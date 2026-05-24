---
title: R사 X5H Day84 - GPU/fence 정상 후 남는 boot animation 전환·launcher attach·first app transaction 타이밍 체크리스트
author: JaeHa
date: 2026-05-25 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day84, display, surfaceflinger, bootanimation, launcher, first-frame, transaction, bringup]
---

Day83에서 `GPU client composition`, `overlay capability`, `present/acquire fence` 까지 정상이면, 이제는 **frame 생성은 되는데 사용자가 봐야 할 첫 장면으로 전환되지 않는 타이밍 절단면** 을 봐야 한다. X5H bring-up 후반에는 panel도 살아 있고 SurfaceFlinger도 frame을 내보내는데 `boot animation exit`, `launcher attach`, `first app transaction apply` 순서가 흔들려 첫 사용자 frame이 늦거나 영영 보이지 않는 경우가 남는다.

## 핵심 요약

- Day83까지 통과했는데도 첫 화면이 안 뜨면 1차 절단 순서는 **bootanimation 종료 조건 → launcher/home attach → first transaction apply → visible layer replacement** 다.
- `retire fence` 가 정상이어도 boot animation layer가 안 내려가거나 home task 첫 `BufferStateLayer` 가 commit되지 않으면 사용자는 여전히 검은 화면처럼 본다.
- 핵심은 `BOOT_EXIT`, `LAUNCHER_READY`, `TX_APPLY`, `VISIBLE_SWAP` 네 줄을 같은 frame 번호 축으로 묶는 것이다.
- display/hwc 문제가 아니라 `system_server ↔ wm ↔ sf` 전환 타이밍 문제면 panel debug를 더 해도 막판 증상은 안 잡힌다.

## 코드 포인트

1. **boot animation 종료 조건이 실제 충족됐는지 먼저 본다**

   ```text
   BOOT_EXIT bootanim_pid=842 service_stopped=0 exit_prop=service.bootanim.exit:0 sf_boot_finished=1
   BOOT_EXIT_WAIT reason=linger_until_home_draw timeout_ms=5000 elapsed_ms=4870
   ```

   `sys.boot_completed=1` 만으로는 충분하지 않다. vendor 쪽에서 `bootanim` 종료를 home draw와 묶어 두면 launcher 첫 frame이 늦을 때 boot animation layer가 계속 topmost로 남을 수 있다.

2. **launcher/home task가 실제 attach 되었는지 window 기준으로 확인한다**

   ```text
   LAUNCHER_READY display=2 task=HomeActivity resumed=1 surface_created=1 first_buffer_queued=0
   WM_FOCUS display=2 focused_window=com.r.launcher/.HomeActivity top_task=home waiting_draw=1
   ```

   activity resume와 실제 surface attach는 다르다. `resumed=1` 이어도 `first_buffer_queued=0` 이면 첫 홈 frame이 없어서 boot animation 이후 검은 장면이 잠시 또는 계속 남을 수 있다.

3. **첫 transaction이 SurfaceFlinger에 apply 되는지 자른다**

   ```text
   TX_APPLY display=2 frame=4512 layer=HomeRoot state=buffer_set crop_set alpha=1 z=210 applied=0 reason=defer_until_sync
   TX_APPLY display=2 frame=4513 layer=HomeRoot applied=1 latch_unsignaled=0 desired_present=733421991
   ```

   WindowManager/Launcher가 transaction을 냈어도 `defer_until_sync`, `latch_unsignaled`, `sync group pending` 으로 한 프레임 이상 밀릴 수 있다. 이 구간이 길면 사용자는 black screen 또는 boot animation freeze처럼 느낀다.

4. **visible layer 교체가 실제로 일어났는지 확인한다**

   ```text
   VISIBLE_SWAP display=2 prev_top=BootAnimation next_top=HomeRoot prev_visible=1 next_visible=0
   VISIBLE_SWAP display=2 frame=4513 prev_top=BootAnimation next_top=HomeRoot prev_visible=0 next_visible=1
   ```

   핵심은 boot animation layer가 사라진 뒤 home root가 같은 frame 또는 바로 다음 frame에 visible로 올라오는지다. 둘 사이가 비면 panel은 정상이어도 사용자 체감은 black gap이다.

5. **첫 app transaction 이후 입력 가능 상태까지 함께 보면 오판이 줄어든다**

   ```text
   FIRST_FRAME_DONE display=2 frame=4513 app=HomeActivity present_ms=16.6
   INPUT_READY display=2 focused=HomeActivity input_dispatch=enabled vsync_after_first_frame=1
   ```

   첫 frame이 떠도 input이 아직 막혀 있으면 사용자는 멈춘 화면으로 오인한다. bring-up 판정은 `first visible frame` 과 `input ready` 를 같이 완료로 잡는 편이 안전하다.

## 리스크

- boot animation 종료 property와 launcher first draw 의존성이 섞여 있으면 cold boot에서만 재현되는 간헐 black gap이 남기 쉽다.
- multi-display 제품에서는 home task가 다른 display area에 먼저 붙고 target panel transaction은 뒤늦게 와서 display route 문제처럼 오인될 수 있다.
- `defer transaction` 원인이 sync fence인지 WM transition인지 분리하지 않으면 SurfaceFlinger/HWC/kernel 팀 사이에서 책임 경계가 흐려진다.
- 첫 frame만 늦는 증상은 로그 없이 체감으로만 보고되기 쉬워 계측 포인트를 미리 심어 두지 않으면 양산 직전까지 잠복한다.

## 다음 액션

다음 글에서는 Day84를 이어서, **첫 home frame까지는 정상인데 이후 특정 앱 진입 시만 cluster/center display가 다시 blank로 떨어질 때 task reparent·display area 이동·surface destroy/recreate race 체크리스트** 를 정리하겠다.

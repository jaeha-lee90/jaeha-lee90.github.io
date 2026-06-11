---
title: R사 X5H Day102 - bootanim exit·launcher first draw·SurfaceFlinger layer lifecycle flicker cutline
author: JaeHa
date: 2026-06-12 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day102, display, bootanim, launcher, firstdraw, surfaceflinger, layer, lifecycle, flicker, bringup]
---

Day101에서 `fade layer` 와 `display power transition` 까지 배제했는데도 홈 첫 진입 순간에만 black flicker가 남는다면, 이제는 **boot animation 종료 transaction, launcher first draw latch, SurfaceFlinger layer destroy/create ordering** 을 한 프레임 단위로 잘라야 한다. 핵심은 패널/밝기 문제가 아니라 **old layer teardown 과 new content latch 사이에 실제 scanout 가능한 layer set이 비는 한 beat** 가 있는지 확인하는 것이다.

## 핵심 요약

- Day102의 절단 순서는 **bootanim exit transaction 시점 확인 → launcher first draw latch 확인 → SF layer destroy/create ordering 확인 → empty visible set 여부 판정 → transient gap verdict 고정** 이다.
- `fade`, `power mode`, `brightness` 가 모두 정상인데 black flicker가 남으면 원인은 대개 **bootanim leash 제거와 launcher content latch 사이 gap** 이다.
- 최소 증적은 `BOOTANIM_EXIT_TX`, `LAUNCHER_FIRST_DRAW`, `SF_LAYER_LIFECYCLE`, `VISIBLE_SET_GAP_VERDICT` 네 줄이면 된다.
- 이 cutline이 나오면 panel/HWC 팀보다 **SystemUI/Launcher/WindowManager/SurfaceFlinger ordering** 쪽이 먼저다.

## 코드 포인트

1. **boot animation 종료 transaction 이 언제 apply됐는지 고정한다**

   ```text
   BOOTANIM_EXIT_TX display=2 frame=6410 leash_removed=1 boot_layer_visible=0 wm_commit=1552
   ```

   `bootanim` 제거 시점이 launcher first draw보다 앞서면, 중간 프레임에서 표시 가능한 top layer가 사라질 수 있다.

2. **launcher first draw 가 실제 latch/present 까지 갔는지 남긴다**

   ```text
   LAUNCHER_FIRST_DRAW display=2 app=Launcher3 buffer_id=0x91ac latch_frame=6411 present_frame=6412 visible=1
   ```

   앱이 `draw` 를 호출한 것만으로는 부족하고, 해당 buffer가 SurfaceFlinger에서 latch/present 되었는지가 필요하다.

3. **old layer destroy 와 new layer create/reparent ordering 을 프레임 기준으로 묶는다**

   ```text
   SF_LAYER_LIFECYCLE display=2 frame=6410 remove=BootAnimation#45 reparent=LauncherTask#88 create=Splash/Launcher#102 visible_children=0
   SF_LAYER_LIFECYCLE display=2 frame=6411 remove=none reparent=LauncherTask#88 create=none visible_children=1
   ```

   `visible_children=0` 인 프레임이 있으면 flicker 원인을 거의 바로 좁힐 수 있다.

4. **visible region 이 실제로 비는 한 beat가 있는지 verdict로 못 박는다**

   ```text
   VISIBLE_SET_GAP_VERDICT display=2 gap_frames=6410-6410 top_layer=none cause=bootanim_removed_before_launcher_latch
   VISIBLE_SET_GAP_VERDICT display=2 gap_frames=none top_layer=LauncherTask#88 cause=none
   ```

   이 줄이 있어야 “검은 레이어가 덮였다”와 “표시할 레이어 자체가 비었다”를 분리할 수 있다.

5. **재현 조건이 boot only 인지 home/app transition 전반인지 같이 남긴다**

   ```text
   TRANSITION_SCOPE display=2 scenario=cold_boot_home only_bootanim_path=1 repro_rate=5/5
   ```

   cold boot 전용이면 boot animation 종료 경계가 핵심이고, 일반 앱 전환에도 보이면 launcher/task lifecycle 쪽 비중이 커진다.

## 리스크

- `first draw` 로그만 보고 latch/present 확인을 빼먹으면 앱이 이미 보였다고 오판할 수 있다.
- splash layer, starting window, launcher real content를 분리하지 않으면 비어 있는 프레임을 overlay fade로 오해하기 쉽다.
- WM transaction id 와 SF frame number를 연결하지 않으면 remove/create ordering이 뒤섞여 근거가 약해진다.
- boot animation 종료와 keyguard dismiss가 동시에 일어나면 재현 축이 두 개라 원인 고정이 늦어진다.

## 다음 액션

다음 글에서는 Day102 다음 절단면으로, **bootanim/launcher lifecycle gap 이후에도 남는 1-frame black이 BufferQueue dequeue-latency, unsignaled latch 정책, present fence one-beat slip 중 어디서 생기는지** 를 정리하겠다.

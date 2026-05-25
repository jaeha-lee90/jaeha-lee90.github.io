---
title: R사 X5H Day85 - task reparent·display area 이동·surface destroy/recreate race 체크리스트
author: JaeHa
date: 2026-05-26 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day85, display, windowmanager, surfaceflinger, launcher, task, race, bringup]
---

Day84에서 `boot animation exit`, `launcher attach`, `first app transaction` 까지 정상이라면, 다음 절단면은 **첫 화면 이후 특정 task가 다른 display area로 이동하면서 surface가 잠깐 비는 race** 다. X5H 멀티디스플레이 bring-up 후반에는 cluster/center 분리 정책, occupant zone routing, task organizer 재배치가 겹치면서 `task reparent → old surface destroy → new surface create` 순서가 어긋나 panel blank처럼 보이는 경우가 남는다.

## 핵심 요약

- Day84를 통과했는데 앱 진입/전환 시만 다시 blank가 뜨면 1차 절단 순서는 **task reparent 요청 → display area 재결정 → old leash/surface destroy → new surface buffer latch** 다.
- `Activity resumed` 와 `task moved` 만으로는 충분하지 않다. 실제 사용자 가시성은 `old surface visible off` 와 `new surface first buffer on` 사이 gap 길이로 판정해야 한다.
- 최소 증적은 `TASK_REPARENT`, `DISPLAY_AREA_MOVE`, `SURFACE_RECREATE`, `FIRST_BUFFER_LATCH` 네 줄이면 된다.
- 이 구간은 panel/HWC 문제가 아니라 `ATMS ↔ WM Shell ↔ SurfaceFlinger` 수명주기 race 인 경우가 많다.

## 코드 포인트

1. **task reparent가 어떤 정책으로 시작됐는지 먼저 고정한다**

   ```text
   TASK_REPARENT task=ClusterNav from_display=0 to_display=2 reason=occupant_zone_policy organizer=CarTaskOrganizer
   TASK_STATE task=ClusterNav resumed=1 top_activity=com.r.cluster/.NavActivity windowing=fullscreen
   ```

   핵심은 누가 이동을 트리거했는지다. occupant zone policy, launch option, shell organizer 중 출발점을 못 고정하면 display route 문제와 lifecycle 문제를 섞어 보게 된다.

2. **display area 이동 결정과 실제 window container 반영 시점을 분리해서 본다**

   ```text
   DISPLAY_AREA_MOVE task=ClusterNav target_area=cluster_main target_display=2 requested_t=81223341 committed_t=81223396
   ROOT_TASK_REORDER display=2 root_task=184 child_pos=top old_display=0 new_display=2
   ```

   policy가 `target_display=2` 를 골랐어도 container commit이 늦으면 old display 쪽 surface destroy만 먼저 발생할 수 있다. 이 시간차가 blank gap의 시작점이다.

3. **old leash/surface 파괴가 새 surface 준비보다 앞서는지 본다**

   ```text
   SURFACE_RECREATE task=ClusterNav old_leash=0x81 destroy=1 old_visible=1 new_leash=0x97 created=0
   WINDOW_VISIBILITY task=ClusterNav old_surface_hidden=1 new_surface_drawn=0 gap_ms=146
   ```

   `destroy old` 와 `create new` 사이가 수 프레임 이상 비면 panel은 정상이어도 사용자는 검은 전환으로 본다. 특히 display 이동 시 `SurfaceControl` 재생성이 들어가면 buffer queue가 새로 붙기 전까지 완전 공백이 날 수 있다.

4. **새 surface의 첫 buffer latch가 transaction apply와 같은 축으로 찍히는지 확인한다**

   ```text
   FIRST_BUFFER_LATCH task=ClusterNav display=2 layer=NavRoot frame=6128 queued=1 latched=1 latch_delay_ms=22
   TX_APPLY display=2 frame=6128 layer=NavRoot reparent_done=1 crop_valid=1 alpha=1 visible=1
   ```

   새 surface가 생성돼도 첫 buffer가 늦으면 `window exists but black` 상태가 된다. `queued` 와 `latched` 를 분리해서 봐야 app, WM, SF 중 어디서 밀리는지 자를 수 있다.

5. **surface destroy/recreate가 configuration 변경과 엮였는지 함께 본다**

   ```text
   CONFIG_CHANGE task=ClusterNav density=changed bounds=1920x720->1280x480 relaunch=1
   RELAUNCH_TRACE task=ClusterNav onPause=1 onStop=1 surface_destroy=1 onResume=1 first_draw=0
   ```

   display area 이동과 동시에 density/bounds/configuration 변경이 들어오면 activity relaunch가 끼어들어 gap이 더 길어진다. 이 경우 route 자체보다 relaunch budget 관리가 먼저다.

## 리스크

- occupant zone/display area 정책과 activity relaunch가 겹치면 cold boot 후 첫 진입에서만 재현되는 간헐 blank가 남기 쉽다.
- old surface destroy를 너무 이르게 하면 HWC/panel 문제처럼 보이는 검은 전환이 생겨 디버깅 축이 틀어진다.
- task organizer, Car launcher, vendor display policy가 각각 이동 결정을 내리면 책임 경계가 흐려져 수정 후 회귀가 반복된다.
- 새 display의 first buffer latch 계측이 없으면 app 문제인지 WM transaction 지연인지 끝까지 감으로 싸우게 된다.

## 다음 액션

다음 글에서는 Day85를 이어서, **task/display 이동은 맞는데 특정 앱만 blank가 남을 때 configuration relaunch·resource qualifier 변경·buffer size 재협상 타이밍을 자르는 체크리스트** 를 정리하겠다.

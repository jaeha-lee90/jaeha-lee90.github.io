---
title: R사 X5H Day86 - configuration relaunch·resource qualifier 변경·buffer size 재협상 타이밍 체크리스트
author: JaeHa
date: 2026-05-27 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day86, display, windowmanager, configuration, relaunch, bufferqueue, bringup]
---

Day85에서 `task reparent`, `display area 이동`, `surface destroy/recreate` 흐름까지 확인했다면, 다음 절단면은 **이동은 맞는데 특정 앱만 configuration relaunch와 buffer 재협상 중 검게 비는 경우** 다. X5H 멀티디스플레이 bring-up 후반에는 display bounds, density, uiMode, qualifier 변경이 한 번에 들어오면서 `activity relaunch → 새 surface 생성 → buffer size 재협상 → first draw` 순서가 늘어지고, 이 gap이 panel blank처럼 보일 수 있다.

## 핵심 요약

- Day85 이후에도 특정 앱만 blank가 남으면 1차 절단 순서는 **configuration diff 발생 → activity relaunch 여부 → buffer size 재협상 → first draw 재개** 다.
- `surface recreated` 만으로는 충분하지 않다. 새 surface의 `requested buffer size` 와 producer/consumer가 합의한 `actual dequeue size` 가 맞지 않으면 첫 frame이 계속 지연된다.
- 최소 증적은 `CONFIG_DIFF`, `RELAUNCH_DECISION`, `BUFFER_RENEGOTIATE`, `FIRST_DRAW_RESUME` 네 줄이다.
- 이 구간은 panel/HWC보다 `ActivityThread ↔ ViewRootImpl ↔ BufferQueue ↔ SurfaceFlinger` 사이 계약 불일치일 가능성이 높다.

## 코드 포인트

1. **어떤 configuration diff가 relaunch를 유발했는지 먼저 고정한다**

   ```text
   CONFIG_DIFF activity=com.r.cluster/.NavActivity display=2 changes=bounds|density|smallestWidth old=1920x720@240 new=1280x480@160
   RESOURCE_QUALIFIER app=com.r.cluster qualifiers=sw720dp-land -> sw600dp-land car_ui=cluster
   ```

   단순 display 이동이 아니라 `bounds`, `densityDpi`, `smallestWidthDp` 가 함께 바뀌면 resource qualifier 재선택이 일어나고 inflate/layout cost가 커진다. 여기서부터 단순 route 문제와 앱 relaunch 문제를 분리할 수 있다.

2. **framework가 relaunch를 택했는지 in-place update를 택했는지 자른다**

   ```text
   RELAUNCH_DECISION activity=com.r.cluster/.NavActivity handle_config_in_place=0 relaunch=1 reason=size_compat_resource_miss
   ACTIVITY_THREAD relaunch_seq=441 onPause=1 onStop=1 onDestroy=1 newToken=0x91
   ```

   앱이 `configChanges` 로 흡수하지 못하면 activity 재생성이 들어간다. 이때 blank gap은 display pipeline이 아니라 app lifecycle budget이 잡아먹는 경우가 많다.

3. **surface 생성 뒤 buffer size 재협상이 실제로 끝났는지 확인한다**

   ```text
   BUFFER_RENEGOTIATE layer=NavRoot requested=1280x480 consumer=1280x480 producer_dequeue=1920x720 format=RGBA_8888
   BUFFER_RENEGOTIATE layer=NavRoot requested=1280x480 consumer=1280x480 producer_dequeue=1280x480 settled=1
   ```

   새 surface가 만들어져도 producer가 이전 display 크기로 dequeue 하면 첫 buffer가 reject 또는 crop mismatch 상태로 밀릴 수 있다. `requested != dequeue` 가 반복되면 blank 원인은 거의 size 재협상 지연이다.

4. **first draw 재개와 transaction apply를 같은 frame 축에서 본다**

   ```text
   FIRST_DRAW_RESUME activity=com.r.cluster/.NavActivity viewroot_drawn=1 frame=6184 draw_delay_ms=97
   TX_APPLY display=2 frame=6184 layer=NavRoot buffer_size=1280x480 crop_valid=1 visible=1
   ```

   `onResume` 가 끝났다고 화면이 나온 게 아니다. `ViewRootImpl` 첫 draw와 `SurfaceFlinger` apply frame을 같은 번호로 묶어야 app/UI 지연과 compositor 지연을 분리할 수 있다.

5. **새 크기에서만 생기는 fence starvation 또는 layout thrash도 함께 본다**

   ```text
   LAYOUT_THRASH activity=com.r.cluster/.NavActivity measure_pass=4 relayout_count=3 stable_insets_pending=1
   ACQUIRE_FENCE_WAIT layer=NavRoot frame=6184 unsignaled_ms=33 producer=RenderThread reason=resize_backpressure
   ```

   size change 직후 `stableInsets`, `cutout`, `system bar` 재계산이 여러 번 돌면 첫 draw 자체가 늦어진다. 여기에 resize backpressure가 겹치면 `surface exists but no visible content` 시간이 길어진다.

## 리스크

- display 이동과 qualifier 변경이 함께 들어오면 특정 화면밀도/해상도 조합에서만 blank가 재현돼 회귀 검출이 어렵다.
- producer가 이전 buffer 크기를 한두 frame 더 쓰는 구현이면 간헐 black frame이 남아도 로그 없이는 HWC 문제로 오인되기 쉽다.
- 앱의 `configChanges` 선언과 실제 리소스 의존성이 어긋나면 relaunch/in-place 처리 경계가 불안정해진다.
- cluster/center 간 서로 다른 inset 정책이 있으면 layout thrash가 cold boot 첫 진입에서만 길어질 수 있다.

## 다음 액션

다음 글에서는 Day86을 이어서, **configuration과 buffer size는 정상인데 특정 앱 전환 직후만 blank가 남을 때 sync group/BLASTBufferQueue/unsignaled latch 허용 정책을 자르는 체크리스트** 를 정리하겠다.

---
title: R사 X5H Day94 - acquire/release/present fence first-frame one-beat slip 절단면
author: JaeHa
date: 2026-06-04 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day94, display, fence, acquire, release, present, surfaceflinger, hwc, bringup]
---

Day93에서 `crop/transform/displayFrame/visible area` 까지 정상인데도 **첫 화면만 검게 보이거나 한 박자 늦게 나타난다면**, 다음으로 잘라야 할 구간은 `producer acquire fence → SurfaceFlinger latch → HWC present fence → release fence recycle` 동기화 체인이다. 즉 geometry는 맞지만 **첫 frame이 scanout 시점에 맞춰 열리지 못해 실제 사용자는 black 한 장을 먼저 보는지**를 확인해야 한다.

## 핵심 요약

- Day94의 1차 절단 순서는 **acquire fence signal → SF latch decision → HWC present completion → release fence recycle** 이다.
- 첫 non-black buffer가 존재해도 `acquire unsignaled`, `desired present miss`, `present fence late` 중 하나면 사용자는 첫 장면 대신 black 한 프레임을 본다.
- 최소 증적은 `ACQUIRE_FENCE_STATE`, `LATCH_DECISION`, `PRESENT_FENCE_RESULT`, `RELEASE_RECYCLE_STATE` 네 줄이면 충분하다.
- 이 구간에서 막히면 app draw나 geometry보다 `SurfaceFlinger/HWC/kernel sync timeline` 정합 문제일 확률이 높다.

## 코드 포인트

1. **첫 non-black buffer의 acquire fence가 제때 열리는지 고정한다**

   ```text
   ACQUIRE_FENCE_STATE layer=MainActivity#0 buffer_id=0x9421 frame=6313 signaled=0 wait_ms=18 timeline=app_gpu
   ACQUIRE_FENCE_STATE layer=MainActivity#0 buffer_id=0x9421 frame=6314 signaled=1 wait_ms=0 timeline=app_gpu
   ```

   buffer 자체는 queue됐어도 acquire fence가 늦게 열리면 SurfaceFlinger는 해당 frame을 latch하지 못한다. 첫 장면 지연인지 black buffer 재사용인지 갈라내려면 `buffer_id`, `signaled`, `timeline` 을 같이 봐야 한다.

2. **SurfaceFlinger가 해당 buffer를 현재 vsync에 실었는지 본다**

   ```text
   LATCH_DECISION display=2 layer=MainActivity#0 frame=6313 latched=0 reason=acquire_fence_pending desired_present=1122334455
   LATCH_DECISION display=2 layer=MainActivity#0 frame=6314 latched=1 reason=ready_for_present desired_present=1122351121
   ```

   `first non-black queue` 와 `first visible present` 사이에 한 프레임 gap이 있으면 사용자는 첫 장면이 늦거나 black flash로 느낀다. 핵심은 `왜 latch가 미뤄졌는지`를 reason으로 남기는 것이다.

3. **HWC present fence가 실제 scanout 완료를 제때 닫는지 확인한다**

   ```text
   PRESENT_FENCE_RESULT display=2 frame=6314 present_fence=4892 signaled=0 wait_ms=16 retire=none
   PRESENT_FENCE_RESULT display=2 frame=6314 present_fence=4892 signaled=1 wait_ms=0 retire=16.6ms
   ```

   latch가 됐더라도 present fence가 늦으면 panel은 이전 black frame을 한 번 더 유지할 수 있다. 이때는 app/SF는 정상처럼 보이는데 사용자는 여전히 첫 화면을 못 본다.

4. **이전 black buffer의 release fence가 늦어 recycle/backpressure를 만들지 분리한다**

   ```text
   RELEASE_RECYCLE_STATE layer=MainActivity#0 prev_buffer=0x9418 release_signaled=0 blocked_slots=1 queue_depth=3
   RELEASE_RECYCLE_STATE layer=MainActivity#0 prev_buffer=0x9418 release_signaled=1 blocked_slots=0 queue_depth=1
   ```

   이전 frame의 release가 늦으면 BufferQueue slot이 막혀 첫 정상 frame이 올라와도 다음 교체가 한 박자 밀린다. 특히 부팅 직후 low-depth queue에서는 이 영향이 크게 보인다.

5. **최종 판정은 first visible present 기준으로 타임라인을 한 줄로 묶는다**

   ```text
   FIRST_VISIBLE_PRESENT display=2 layer=MainActivity#0 first_nonblack_queue_ms=0 acquire_wait_ms=18 latch_gap_frames=1 present_delay_ms=16 black_flash=1
   ```

   `queueBuffer 성공` 또는 `latch 성공` 만 따로 보면 어디서 한 프레임이 비었는지 흐려진다. `queue → acquire → latch → present` 를 한 줄로 묶어야 synchronization cutline이 고정된다.

## 리스크

- acquire fence 지연을 app draw 지연으로 오해하면 렌더링 팀만 붙들고 있게 된다.
- present fence와 release fence를 분리하지 않으면 panel hold와 BufferQueue backpressure가 한 원인처럼 섞인다.
- 첫 프레임 전용 one-beat slip은 평균 FPS 로그에 거의 안 잡혀 재현 불가 이슈로 오인되기 쉽다.
- 멀티디스플레이 환경에서는 다른 display의 present 완료를 잘못 참조하면 실제 target panel black flash 원인을 놓친다.

## 다음 액션

다음 글에서는 Day94 다음 절단면으로, **fence 체인은 정상인데 첫 장면만 black으로 체감될 때 color mode·dataspace·blend/alpha 초기값이 첫 frame에서만 어긋나는 초기 합성 상태 절단면** 을 정리하겠다.

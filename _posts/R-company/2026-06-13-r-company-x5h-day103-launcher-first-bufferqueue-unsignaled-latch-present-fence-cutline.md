---
title: R사 X5H Day103 - launcher first buffer dequeue·unsignaled latch·present fence residual black-frame cutline
author: JaeHa
date: 2026-06-13 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day103, display, launcher, bufferqueue, unsignaledlatch, presentfence, blackframe, bringup]
---

Day102에서 `visible set gap` 이 없다고 확인했는데도 홈 첫 진입에만 1-frame black이 남는다면, 이제는 **launcher first buffer dequeue, unsignaled latch 허용 여부, present fence one-beat slip** 을 같이 묶어 잘라야 한다. 즉 레이어는 살아 있지만 **첫 실콘텐츠 buffer가 늦게 준비되거나, 준비됐어도 latch/present가 한 beat 밀려 이전 black target이 한 프레임 더 스캔아웃되는지** 를 확인하는 단계다.

## 핵심 요약

- Day103의 절단 순서는 **launcher first buffer dequeue 시점 확인 → first buffer queue/latch readiness 확인 → unsignaled latch policy hit 여부 확인 → present fence one-beat slip 판정 → residual black-frame verdict 고정** 이다.
- `visible_children>0` 인데도 black 1-frame이 남으면, 원인은 빈 레이어셋보다 **첫 버퍼 준비/래치/프레젠트 타이밍** 쪽일 가능성이 높다.
- 최소 증적은 `LAUNCHER_BQ_FIRST_BUFFER`, `UNSIGNALED_LATCH_POLICY`, `PRESENT_FENCE_SLIP`, `RESIDUAL_BLACK_VERDICT` 네 줄이면 충분하다.
- 이 cutline이 나오면 Surface 생성 자체보다 **app render thread / BufferQueue / SurfaceFlinger scheduler / HWC present handshake** 쪽으로 소유권이 넘어간다.

## 코드 포인트

1. **launcher 첫 실콘텐츠 buffer dequeue/queue 시점을 먼저 고정한다**

   ```text
   LAUNCHER_BQ_FIRST_BUFFER display=2 layer=LauncherTask#88 dequeue_ok=1 dequeue_ms=0.42 queue_frame=6411 buffer_id=0x91ac
   ```

   `first draw` 이후 실제 `dequeueBuffer → queueBuffer` 가 언제 끝났는지 있어야 app이 그렸는데 버퍼가 아직 안 올라온 상태를 분리할 수 있다.

2. **queued buffer가 같은 beat에 latch 후보가 됐는지 확인한다**

   ```text
   FIRST_BUFFER_LATCH display=2 layer=LauncherTask#88 queue_frame=6411 latch_frame=6411 ready_fence=signaled acquire_wait_ms=0.08
   FIRST_BUFFER_LATCH display=2 layer=LauncherTask#88 queue_frame=6411 latch_frame=6412 ready_fence=late acquire_wait_ms=16.7
   ```

   `queue_frame` 과 `latch_frame` 이 한 beat 어긋나면 launcher surface는 존재해도 실제 보이는 시점은 다음 프레임으로 밀린다.

3. **unsignaled latch 허용 정책이 first frame에서 발동했는지 남긴다**

   ```text
   UNSIGNALED_LATCH_POLICY display=2 layer=LauncherTask#88 allow_unsignaled=0 acquire_fence=pending latch_deferred=1
   UNSIGNALED_LATCH_POLICY display=2 layer=LauncherTask#88 allow_unsignaled=1 acquire_fence=pending latch_deferred=0
   ```

   정책상 보수적으로 defer되면 black target 또는 이전 placeholder가 한 프레임 더 남을 수 있다.

4. **present fence 기준으로 실제 scanout 반영이 한 beat 밀렸는지 본다**

   ```text
   PRESENT_FENCE_SLIP display=2 sf_frame=6411 present_fence=late retire_frame=6412 previous_target=black slip_frames=1
   PRESENT_FENCE_SLIP display=2 sf_frame=6411 present_fence=on_time retire_frame=6411 previous_target=none slip_frames=0
   ```

   latch는 됐는데 `present fence` 가 늦으면 사용자는 여전히 black frame을 보게 된다.

5. **최종 residual black verdict를 한 줄로 고정한다**

   ```text
   RESIDUAL_BLACK_VERDICT display=2 black_frames=6411-6411 cause=first_buffer_late_present cutline=bufferqueue_present
   RESIDUAL_BLACK_VERDICT display=2 black_frames=none cause=none cutline=none
   ```

   이 verdict가 있어야 Day102의 lifecycle gap 문제와 Day103의 buffer/present 문제를 서로 다른 escalation 축으로 유지할 수 있다.

## 리스크

- `draw` 콜백과 `queueBuffer` 를 같은 의미로 취급하면 app 문제인지 SF 스케줄 문제인지 바로 흐려진다.
- unsignaled latch 정책을 보지 않으면 acquire fence가 늦은 건지, 정책상 일부러 한 beat 미룬 건지 구분이 안 된다.
- present fence 기준을 빼먹으면 “latch됐으니 이미 보였다”는 잘못된 결론으로 가기 쉽다.
- black frame 재현이 cold boot에서만 강하면 launcher 초기 asset upload와 scheduler warm-up이 함께 섞여 증적이 흔들릴 수 있다.

## 다음 액션

다음 글에서는 Day103 다음 절단면으로, **launcher 첫 프레임은 제때 올라오는데도 검게 보일 때 GPU clear color, client target reuse, overlay fallback 한정 black composition 경계** 를 정리하겠다.

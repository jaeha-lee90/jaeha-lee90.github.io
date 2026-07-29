---
title: R사 X5H Day131 - secure decoder dequeue·acquire fence·protected first-frame starvation cutline
author: JaeHa
date: 2026-07-30 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day131, display, protected, decoder, fence, first-frame, bringup]
---

Day130에서 `secure compositor fallback`, `capture/writeback 차단`, `clone route` 까지 분리했는데도 여전히 **secure playback 시작 순간만 black으로 보인다면**, 이제 절단면은 `decoder output dequeue 지연`, `acquire fence 미신호`, `protected first-frame starvation` 이다. 즉 secure path는 정책상 살아 있지만, **첫 보호 프레임이 display pipeline에 제때 도착하지 못해 black start처럼 보이는지** 를 잘라야 한다.

## 핵심 요약

- Day131의 핵심 순서는 **decoder output 생성 시점 고정 → BufferQueue enqueue/dequeue 간극 확인 → acquire/present fence 정체 분리 → first protected frame starvation verdict 고정** 이다.
- secure playback에서는 `MediaCodec secure output warm-up`, `BufferQueue 최소 버퍼 미충족`, `HWC acquire fence timeout`, `protected layer 첫 latch skip` 중 하나만 있어도 사용자는 black start로 느낀다.
- `SECURE_DECODE_TRACE`, `BQ_PROTECTED_TRACE`, `ACQUIRE_FENCE_TRACE`, `FIRST_FRAME_STARVATION_TRACE`, `FINAL_PROTECTED_START_VERDICT` 를 같은 `session_id`, `frame`, `buffer_id`, `pts` 로 묶어야 한다.
- 이 단계의 목적은 **policy black인지 timing starvation인지** 를 가르는 것이다. 첫 프레임 starvation이면 Day129~130의 보안정책 이슈와 owner가 다르다.

## 코드 포인트

1. **decoder가 첫 secure output buffer를 언제 실제로 내는지 먼저 고정한다**

   ```text
   SECURE_DECODE_TRACE session=42 frame=0 pts=33333 codec=hevc_secure dequeue_out=late decode_ms=18 output_ready=0
   SECURE_DECODE_TRACE session=42 frame=1 pts=66666 codec=hevc_secure dequeue_out=ok decode_ms=7 output_ready=1 buffer_id=0x91
   ```

   앱은 재생 시작을 눌렀지만 secure decoder가 첫 출력 버퍼를 아직 못 냈다면, black의 owner는 display가 아니라 decode warm-up일 수 있다.

2. **BufferQueue가 protected buffer를 충분히 쌓았는지 본다**

   ```text
   BQ_PROTECTED_TRACE session=42 layer=Video#12 queue_depth=0 min_undequeued=2 last_enqueue_ns=0 starvation=1
   BQ_PROTECTED_TRACE session=42 layer=Video#12 queue_depth=2 min_undequeued=2 last_enqueue_ns=815223991 buffer_id=0x91 starvation=0
   ```

   protected path는 일반 경로보다 버퍼 수급이 빡빡해 첫 enqueue 전까지 SurfaceFlinger가 계속 이전 black state를 유지할 수 있다.

3. **acquire fence가 미신호 상태로 latch를 막는지 분리한다**

   ```text
   ACQUIRE_FENCE_TRACE frame=1201 layer=Video#12 buffer_id=0x91 acquire_fd=211 signaled=0 wait_ms=16.7 result=skip_latch
   ACQUIRE_FENCE_TRACE frame=1202 layer=Video#12 buffer_id=0x91 acquire_fd=211 signaled=1 wait_ms=1.2 result=latch_ok
   ```

   secure decoder output이 enqueue 되었어도 acquire fence가 안 열리면 HWC/SF는 첫 프레임을 잡지 못한다. 이 경우 사용자는 decode 실패처럼 느끼지만 실제 owner는 sync path다.

4. **첫 protected frame이 latch/present 단계에서 한 번 더 밀리는지 기록한다**

   ```text
   FIRST_FRAME_STARVATION_TRACE frame=1201 display=local:0 protected_layer=1 has_buffer=1 latch=0 present=0 reason=acquire_unsignaled
   FIRST_FRAME_STARVATION_TRACE frame=1202 display=local:0 protected_layer=1 has_buffer=1 latch=1 present=1 reason=cleared
   ```

   핵심은 `buffer exists` 와 `actually latched/presented` 를 분리하는 것이다. 첫 프레임이 한 비트 늦게 붙는 문제는 black screen과 달리 재생 시작부에서만 보인다.

5. **최종 owner를 startup timing 기준으로 귀속한다**

   ```text
   FINAL_PROTECTED_START_VERDICT session=42 symptom=start_black owner=decoder_output cause=secure_output_not_ready
   FINAL_PROTECTED_START_VERDICT session=43 symptom=start_black owner=sync_path cause=acquire_fence_unsignaled
   FINAL_PROTECTED_START_VERDICT session=44 symptom=start_black owner=bufferqueue_depth cause=protected_first_frame_starvation
   ```

   이렇게 남겨야 secure policy block과 first-frame timing starvation을 서로 다른 디버그 트랙으로 보낼 수 있다.

## 리스크

- secure codec은 일반 codec보다 초기화 비용이 커 cold start에서 첫 출력 지연이 쉽게 과장된다.
- protected buffer는 CPU dump/screenshot이 막혀 있어 "버퍼가 왔는지" 를 fence와 queue depth로 우회 관측해야 한다.
- vendor HWC가 acquire timeout과 policy black을 같은 검은 client target으로 보여주면 owner 분리가 늦어진다.
- 재생 앱이 시작 애니메이션·placeholder를 얹는 경우 사용자 체감 black 시간이 실제 decoder stall보다 길게 보일 수 있다.

## 다음 액션

다음 글에서는 Day131 다음 절단면으로, **secure first-frame 이후 간헐 black flash로 번지는 release fence backpressure·decoder surface recycle·protected buffer lifetime mismatch 경로** 를 정리하겠다.

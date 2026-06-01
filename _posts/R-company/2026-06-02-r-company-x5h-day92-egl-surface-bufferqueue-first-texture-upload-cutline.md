---
title: R사 X5H Day92 - EGL surface·BufferQueue dequeue·first texture upload 지연 절단면
author: JaeHa
date: 2026-06-02 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day92, display, egl, bufferqueue, textureupload, surfaceflinger, bringup]
---

Day91에서 `starting window 제거 → real content attach` 전환까지 정상인데도 **특정 앱만 첫 장면이 계속 검게 남는다면**, 이제 절단해야 할 구간은 `EGL surface 생성 → BufferQueue dequeue/queue → 첫 texture upload` 이다. 즉 창은 붙었지만 app이 **첫 non-black buffer를 제때 만들어 올리지 못하는지**를 잘라야 한다.

## 핵심 요약

- Day92의 1차 절단 순서는 **EGL surface ready → dequeueBuffer 성공 → first swap/queueBuffer → SurfaceFlinger first non-black latch** 이다.
- `onResume` 나 `first draw` 로그만으로는 부족하다. 앱 프로세스가 살아 있어도 첫 producer buffer가 늦으면 창 내부는 계속 black으로 남는다.
- 최소 증적은 `EGL_SURFACE_READY`, `BQ_DEQUEUE_FLOW`, `FIRST_TEXTURE_UPLOAD`, `FIRST_NONBLACK_QUEUE` 네 줄이면 된다.
- 이 구간에서 막히면 panel/HWC 문제가 아니라 `app render pipeline` 또는 `producer-consumer handshake` 문제일 확률이 높다.

## 코드 포인트

1. **EGL window surface가 실제로 생성 완료됐는지 먼저 고정한다**

   ```text
   EGL_SURFACE_READY app=com.r.cluster surface=0x7c21 native_window=0xb400 config=RGBA8888 size=1920x720
   EGL_SURFACE_READY app=com.r.cluster make_current=1 render_thread=RenderThread tid=2147
   ```

   `Activity resumed` 후에도 `eglCreateWindowSurface` 또는 `eglMakeCurrent` 가 늦으면 app은 draw 경로에 진입해도 실제 producer buffer를 만들 수 없다. `surface handle`, `native_window`, `make_current` 성공 여부를 같이 봐야 한다.

2. **BufferQueue dequeue가 fence/slot 문제 없이 열리는지 본다**

   ```text
   BQ_DEQUEUE_FLOW layer=MainActivity#0 slot=3 dequeue_ms=27 max_buffer_count=3 free_slots=0 blocker=acquire_fence_wait
   BQ_DEQUEUE_FLOW layer=MainActivity#0 slot=1 dequeue_ms=2 gralloc_ok=1 stride=1920 format=RGBA8888
   ```

   첫 프레임이 늦는 대표 원인은 `dequeueBuffer` 장기 대기다. free slot 부족, acquire fence 미해제, gralloc allocation retry가 있으면 앱 창은 살아도 black 상태가 길어진다.

3. **첫 texture upload 또는 UI content 채움이 실제로 일어났는지 분리한다**

   ```text
   FIRST_TEXTURE_UPLOAD app=com.r.cluster frame=1 draw_calls=11 texture_uploads=0 atlas_miss=1
   FIRST_TEXTURE_UPLOAD app=com.r.cluster frame=2 draw_calls=14 texture_uploads=3 first_nonblack_tex=1
   ```

   draw call은 있는데 texture upload가 없으면 실제 결과는 검정 clear color일 수 있다. 특히 첫 이미지 리소스 decode 지연이나 GL upload stall은 `frame submitted` 로그만 보면 놓치기 쉽다.

4. **queueBuffer는 됐는지, 그리고 첫 non-black buffer인지 확인한다**

   ```text
   FIRST_NONBLACK_QUEUE layer=MainActivity#0 buffer_id=0x9321 queue_ms=1 first_queue=1 nonblack_tiles=0
   FIRST_NONBLACK_QUEUE layer=MainActivity#0 buffer_id=0x9324 queue_ms=1 first_nonblack=1 dataspace=srgb
   ```

   첫 queue 자체와 첫 non-black queue는 별개다. 첫 buffer가 검정이면 attach 문제로 착각하기 쉽지만 실제 원인은 앱 초기 렌더 내용 부족일 수 있다.

5. **consumer 쪽 첫 latch와 producer 타임라인 차이를 맞춘다**

   ```text
   SF_FIRST_LATCH display=2 layer=MainActivity#0 buffer_id=0x9324 latch_frame=6310 producer_queue_to_latch_ms=18
   SF_FIRST_LATCH display=2 layer=MainActivity#0 prev_buffer_black=1 current_nonblack=1
   ```

   producer가 정상 queue 했는데 latch가 늦다면 app 문제가 아니라 SurfaceFlinger scheduling 또는 fence 전파 지연으로 다시 잘라야 한다. 반대로 latch가 빠른데도 non-black 전환이 늦으면 앱 초기 draw 내용이 빈약한 쪽이다.

## 리스크

- `first draw` 완료만 보고 정상 판정하면 EGL surface 미완료나 dequeue stall을 놓친다.
- 첫 queue와 첫 non-black queue를 구분하지 않으면 앱 렌더 이슈를 WM 전환 이슈로 오판한다.
- texture upload 지연을 fence 대기로 잘못 해석하면 gralloc/GL 병목과 app asset decode 병목이 섞인다.
- multi-display 환경에서는 wrong native window에 붙은 EGL surface가 조용히 성공해도 사용자가 보는 display는 계속 black일 수 있다.

## 다음 액션

다음 글에서는 Day92 다음 절단면으로, **첫 non-black buffer는 올라오지만 특정 조건에서만 첫 프레임이 확대/잘림/오프셋된 채 검게 체감되는 viewport·transform·crop mis-programming 경계** 를 정리하겠다.

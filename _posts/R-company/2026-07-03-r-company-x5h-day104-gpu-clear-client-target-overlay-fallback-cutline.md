---
title: R사 X5H Day104 - GPU clear color·client target reuse·overlay fallback residual black-frame cutline
author: JaeHa
date: 2026-07-03 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day104, display, gpu, clienttarget, overlay, blackframe, bringup]
---

Day103에서 `first buffer dequeue`, `unsignaled latch`, `present fence slip` 이 모두 정상인데도 첫 홈 진입에만 black 1-frame이 남는다면, 이제는 **GPU clear color, client target 재사용, overlay fallback 순간의 black composition** 을 잘라야 한다. 즉 버퍼는 제때 왔지만 **합성 결과물 자체가 검게 만들어졌는지**, 아니면 **이전 black client target 이 한 beat 재사용됐는지** 를 확인하는 단계다.

## 핵심 요약

- Day104의 절단 순서는 **GPU clear/pass 색 확인 → client target 재사용 여부 확인 → overlay fallback 트리거 확인 → black composition verdict 고정** 이다.
- `queue/latch/present` 가 정상인데 black 1-frame이 남으면 원인은 타이밍보다 **composition 결과물** 쪽일 가능성이 높다.
- 최소 증적은 `GPU_CLEAR_COLOR`, `CLIENT_TARGET_REUSE`, `OVERLAY_FALLBACK_REASON`, `BLACK_COMPOSITION_VERDICT` 네 줄이면 충분하다.
- 이 cutline이 나오면 launcher 앱보다 **SurfaceFlinger RenderEngine / HWC validate-present / composition policy** 쪽이 우선 소유자다.

## 코드 포인트

1. **GPU가 첫 합성 프레임에서 어떤 clear color로 client target을 시작했는지 남긴다**

   ```text
   GPU_CLEAR_COLOR display=2 frame=6412 rgba=0,0,0,1 source=RenderEngine clear_only=1
   GPU_CLEAR_COLOR display=2 frame=6412 rgba=0,0,0,0 source=skip_clear clear_only=0
   ```

   첫 프레임이 black으로 clear되고 실제 layer draw가 비어 있으면, 버퍼는 정상이어도 결과는 black이다.

2. **이전 black client target이 재사용됐는지 확인한다**

   ```text
   CLIENT_TARGET_REUSE display=2 frame=6412 slot=7 reused=1 previous_frame=6411 previous_content=black age=1
   CLIENT_TARGET_REUSE display=2 frame=6412 slot=3 reused=0 previous_content=launcher
   ```

   validate/present 흐름에서 새 client target 생성이 실패하거나 skip되면 직전 black target이 그대로 나갈 수 있다.

3. **overlay fallback 또는 device/client composition 전환 이유를 같이 기록한다**

   ```text
   OVERLAY_FALLBACK_REASON display=2 frame=6412 layer=LauncherTask#88 validate=fail reason=unsupported_dataspace compose=client
   OVERLAY_FALLBACK_REASON display=2 frame=6412 layer=LauncherTask#88 validate=ok reason=none compose=device
   ```

   첫 프레임에서만 overlay capability 판정이 바뀌면 fallback 경계에서 black target이 잠깐 노출될 수 있다.

4. **실제 draw call이 있었는지와 결과 target이 black인지 한 줄로 묶는다**

   ```text
   BLACK_COMPOSITION_VERDICT display=2 frame=6412 draw_calls=0 client_target=black cause=clear_without_layer_draw
   BLACK_COMPOSITION_VERDICT display=2 frame=6412 draw_calls=4 client_target=launcher cause=none
   ```

   이 verdict가 있어야 “보여줄 버퍼가 없었다”와 “합성했지만 검게 만들었다”를 분리할 수 있다.

5. **HWC present 결과와 client/device composition 모드를 같은 frame id로 연결한다**

   ```text
   COMPOSE_PATH_FRAME display=2 frame=6412 mode=client present_ret=0 retire_fence=on_time
   ```

   present가 정상이어도 mode 전환 순간의 target 내용이 black이면 사용자는 동일하게 black 1-frame을 본다.

## 리스크

- GPU clear color를 안 남기면 빈 draw와 정상 draw 후 black overwrite를 구분하기 어렵다.
- client target slot 재사용 여부를 놓치면 “새 프레임이 검었다”와 “이전 black target 재방출”이 섞인다.
- overlay fallback reason 없이 compose mode만 보면 왜 첫 프레임에서만 정책이 달라졌는지 설명이 안 된다.
- RenderEngine 로그와 HWC frame id 매핑이 어긋나면 black composition verdict의 신뢰도가 급격히 떨어진다.

## 다음 액션

다음 글에서는 Day104 다음 절단면으로, **첫 프레임 합성 결과는 정상인데 panel에만 black으로 보일 때 writeback CRC, panel ingress trace, post-blend capture를 이용해 display pipeline 후단 cutline을 고정하는 방법** 을 정리하겠다.

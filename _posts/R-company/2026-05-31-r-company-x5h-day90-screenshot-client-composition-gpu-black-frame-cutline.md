---
title: R사 X5H Day90 - screenshot·client composition·GPU output black frame 절단면
author: JaeHa
date: 2026-05-31 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day90, display, surfaceflinger, screenshot, gpu, composition, blackframe, bringup]
---

Day89에서 `wallpaper/dim/color layer/trusted overlay` 까지 정리됐는데도 사용자가 검은 화면을 본다면, 다음 절단면은 **가리는 layer가 아니라 app이 실제로 낸 첫 frame 자체가 검은지** 확인하는 것이다. 이 단계에서는 panel/HWC 출력 문제가 아니라 `client composition 입력`, `GPU render 결과`, `screenshot path` 셋을 교차 비교해 black frame의 생성 위치를 잘라야 한다.

## 핵심 요약

- Day89 이후의 1차 절단 순서는 **screenshot 캡처값 → SurfaceFlinger client composition 입력 → GPU render target 결과 → app 첫 draw 완료 시점** 이다.
- screenshot도 검으면 HWC 이후가 아니라 `SF 이전 또는 SF 내부 입력` 문제일 가능성이 높다.
- screenshot은 정상인데 실화면만 검으면 panel/backend 쪽으로 다시 내려가고, screenshot도 검고 GPU target도 검으면 app/SF composition 경계에서 자른다.
- 최소 증적은 `SCREENSHOT_PROBE`, `CLIENT_COMP_INPUT`, `GPU_FRAME_RESULT`, `APP_FIRST_DRAW` 네 줄이면 된다.

## 코드 포인트

1. **실화면 체감과 screenshot 결과를 같은 frame id로 묶는다**

   ```text
   SCREENSHOT_PROBE display=2 frame=6224 capture=black nonzero_pixels=0 source=sf-capture-secure-excluded
   SCREENSHOT_PROBE display=2 frame=6224 panel_user_report=black hwc_present=1
   ```

   screenshot도 검으면 `panel only` 문제가 아니라는 뜻이다. 먼저 사용자가 본 black과 캡처된 black이 같은 frame 구간인지 맞춰야 한다.

2. **client composition 입력 layer가 실제 texture/buffer를 가지고 있는지 본다**

   ```text
   CLIENT_COMP_INPUT display=2 layer=NavRoot composition=CLIENT buffer_id=0x8a12 acquire_fence=signaled dataspace=srgb
   CLIENT_COMP_INPUT display=2 layer=StatusBar composition=CLIENT visible_region=full sampled_alpha=1.0
   ```

   `CLIENT` 로 넘어간 layer가 버퍼 없이 검정 clear color만 그리는 상태면 GPU는 정상 동작해도 결과는 black이다. `buffer_id`, fence, dataspace, visible region을 함께 찍어야 한다.

3. **GPU render target 결과가 black clear인지, 실제 draw 후 black인지 분리한다**

   ```text
   GPU_FRAME_RESULT display=2 frame=6224 target=client_fb clear_color=0x000000 draw_calls=14 nonblack_tiles=0
   GPU_FRAME_RESULT display=2 frame=6224 top_client_layer=NavRoot shader_ok=1 texture_bound=0
   ```

   draw call 수가 있어도 texture bind 실패나 sampled image null이면 결과는 그대로 검정이다. `draw_calls>0` 만으로 정상 판정을 내리면 안 된다.

4. **앱의 첫 draw 완료와 실제 buffer 내용 생성을 분리한다**

   ```text
   APP_FIRST_DRAW display=2 app=com.r.cluster first_draw=1 first_buffer_posted=0 relayout_pending=1
   APP_FIRST_DRAW display=2 app=com.r.cluster splash_removed=1 content_nonblack=0 blocker=surface_not_filled
   ```

   WindowManager 기준 `first draw` 완료가 떠도 앱이 실제 UI 픽셀을 채우지 못하면 black screenshot이 남는다. draw lifecycle과 buffer content lifecycle을 따로 봐야 한다.

5. **screenshot path 자체의 제약 여부를 마지막으로 확인한다**

   ```text
   SCREENSHOT_POLICY display=2 secure_layer=0 protected_content=0 capture_allowed=1
   SCREENSHOT_POLICY display=2 capture_black_reason=none path_valid=1
   ```

   secure/protected content 때문에 캡처만 검게 보이는 예외를 빼지 않으면 잘못된 절단을 하게 된다. 정책상 black capture인지 실제 black frame인지 분리해야 한다.

## 리스크

- screenshot만 믿고 내려가면 secure/protected content 예외를 실제 blank로 오판할 수 있다.
- GPU draw call 수만 보고 정상 처리하면 texture bind 실패, null buffer, wrong dataspace 같은 진짜 원인을 놓친다.
- app first draw 이벤트와 first non-black content를 같은 것으로 취급하면 launcher attach 이후의 black window를 재현성 낮은 문제로 남기게 된다.
- multi-display 환경에서는 한 display의 screenshot path만 정상이고 다른 display는 capture pipeline이 달라 비교 기준이 어긋날 수 있다.

## 다음 액션

다음 글에서는 Day90 이후 분기인 **GPU/client composition 결과는 정상인데도 app 창 내부가 검게 유지될 때 splashscreen 전환·starting window 제거·real content attach 타이밍 절단면** 을 정리하겠다.

---
title: R사 X5H Day110 - screenshot·color transform·client target lineage final dark-frame cutline
author: JaeHa
date: 2026-07-09 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day110, display, screenshot, colortransform, clienttarget, darkframe, bringup]
---

Day109에서 `WALLPAPER_ATTACH_TRACE`, `TASK_SNAPSHOT_TRACE`, `TRUSTED_OVERLAY_TRACE` 까지 모두 정상인데도 사용자가 첫 홈 진입 순간 화면을 **검거나 비정상적으로 어둡게** 느낀다면, 이제 남는 절단면은 `screenshot 경로`, `display color transform`, `client target 계보` 다. 즉 cover layer는 없는데 **캡처된 픽셀 자체가 어두운지**, 아니면 **캡처는 정상인데 panel 쪽 색 변환/감쇠가 늦게 풀리는지** 를 잘라야 한다.

## 핵심 요약

- Day110의 절단 순서는 **screenshot vs panel 체감 비교 → color transform/dimming matrix 확인 → client target lineage 확인 → final dark-frame verdict 고정** 이다.
- screenshot은 정상인데 실화면만 어두우면 원인은 app이나 launcher보다 **display color pipeline/panel power handoff 후단** 일 가능성이 높다.
- screenshot도 어둡고 client target lineage가 black/dim이면 SurfaceFlinger 합성 결과를 먼저 의심해야 한다.
- 최소 증적은 `SCREENSHOT_PANEL_COMPARE`, `COLOR_TRANSFORM_TRACE`, `CLIENT_TARGET_LINEAGE`, `FINAL_DARK_FRAME_VERDICT` 네 줄이면 충분하다.

## 코드 포인트

1. **같은 frame에서 screenshot 결과와 사용자가 본 panel 상태를 먼저 분리한다**

   ```text
   SCREENSHOT_PANEL_COMPARE display=2 frame=6414 screenshot=luma_ok panel=dark delta_luma=41 capture_path=sf
   SCREENSHOT_PANEL_COMPARE display=2 frame=6414 screenshot=dark panel=dark delta_luma=2 capture_path=sf
   ```

   `screenshot=luma_ok` 인데 `panel=dark` 면 app first draw나 wallpaper 문제가 아니라 panel ingress 이후 색/광량 경계를 봐야 한다.

2. **display color transform이나 fade/dimming matrix가 아직 남아 있는지 기록한다**

   ```text
   COLOR_TRANSFORM_TRACE display=2 frame=6414 matrix=0.18,0,0,0;0,0.18,0,0;0,0,0.18,0;0,0,0,1 source=wm_transition active=1
   COLOR_TRANSFORM_TRACE display=2 frame=6414 matrix=identity source=none active=0
   ```

   `matrix!=identity` 이고 `active=1` 이면 black frame이 아니라 **잔류 dim/fade transform** 일 수 있다. Day108의 overlay teardown이 끝났어도 color matrix 해제가 한 beat 늦을 수 있다.

3. **dataspace/renderIntent/brightness multiplier가 final output을 눌렀는지 함께 본다**

   ```text
   DISPLAY_COLOR_PIPELINE display=2 frame=6414 dataspace=SRGB render_intent=COLORIMETRIC dimming=0.22 color_mode=NATIVE
   DISPLAY_COLOR_PIPELINE display=2 frame=6414 dataspace=SRGB render_intent=COLORIMETRIC dimming=1.00 color_mode=NATIVE
   ```

   `dimming<1.0` 이면 검은 버퍼가 아니어도 사용자는 순간 암전처럼 느낄 수 있다. bring-up에서는 backlight가 아니라 **논리적 display dimming 잔류** 가 자주 섞인다.

4. **client target이 실제로 어떤 내용을 들고 왔는지 계보를 남겨 black reuse를 배제한다**

   ```text
   CLIENT_TARGET_LINEAGE display=2 frame=6414 slot=8 reused=0 content=launcher_nonblack source=renderengine
   CLIENT_TARGET_LINEAGE display=2 frame=6414 slot=8 reused=1 previous_frame=6413 content=dimmed_black source=reused_target
   ```

   screenshot이 어두우면서 `content=dimmed_black` 이면 color pipeline보다 앞단의 합성 결과가 이미 어두운 것이다. 반대로 lineage가 정상 non-black이면 후단 경계가 우선이다.

5. **최종 verdict를 캡처/합성/패널 중 어디서 어두워졌는지 한 줄로 고정한다**

   ```text
   FINAL_DARK_FRAME_VERDICT display=2 frame=6414 screenshot=ok panel=dark cause=color_transform_residual owner=wm_display_pipeline
   FINAL_DARK_FRAME_VERDICT display=2 frame=6414 screenshot=dark panel=dark cause=client_target_dimmed owner=sf_renderengine
   ```

   이 verdict가 있어야 Day104의 black composition 재사용과 Day109의 cover layer 잔류, Day110의 color pipeline 잔류를 서로 다른 owner로 깔끔하게 분리할 수 있다.

## 리스크

- screenshot 정상 여부를 안 보면 실화면 암전을 launcher/app 문제로 잘못 넘길 수 있다.
- color transform matrix만 보고 overlay가 끝났다고 가정하면 dimming multiplier 잔류를 놓치기 쉽다.
- client target lineage 없이 색 변환만 보면 실제로는 이전 dimmed target 재사용인데 panel 문제로 오판할 수 있다.
- multi-display 환경에서는 cluster/center stack의 color mode가 다를 수 있어 display id 단위 기록이 반드시 필요하다.

## 다음 액션

다음 글에서는 Day110 다음 절단면으로, **screenshot과 color pipeline까지 정상인데도 사용자가 첫 순간 밝기 점프나 색온도 튐을 느낄 때 backlight ownership, local dimming release, CABC/ALS 개입 순서를 어떻게 자를지** 를 정리하겠다.

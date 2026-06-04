---
title: R사 X5H Day95 - color mode·dataspace·alpha 초기값 first-frame black 합성 절단면
author: JaeHa
date: 2026-06-05 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day95, display, dataspace, colormode, alpha, hwc, surfaceflinger, bringup]
---

Day94에서 `acquire/latch/present/release fence` 체인까지 정상인데도 **첫 화면만 검게 보이고 다음 프레임부터 살아난다면**, 이제 잘라야 할 구간은 `SurfaceFlinger layer state → HWC composition attribute → panel 첫 합성 적용값` 이다. 즉 buffer와 timing은 정상인데 **첫 frame에서만 dataspace/alpha/blend/color mode 초기값이 잘못 들어가 실제 출력이 black으로 합성되는지**를 확인해야 한다.

## 핵심 요약

- Day95의 1차 절단 순서는 **SF layer state 확정 → HWC dataspace/blend/plane alpha programming → display color mode 적용 → first visible pixel 검증** 이다.
- 첫 non-black buffer가 올라와도 `dataspace default`, `plane alpha=0`, `blend mode mismatch`, `wrong color mode` 중 하나면 사용자는 black flash로 체감한다.
- 최소 증적은 `SF_COLOR_STATE`, `HWC_COMPOSE_ATTR`, `DISPLAY_COLOR_MODE`, `FIRST_PIXEL_RESULT` 네 줄이면 충분하다.
- 이 구간에서 막히면 producer나 fence가 아니라 `초기 합성 속성 전파` 문제일 가능성이 높다.

## 코드 포인트

1. **SurfaceFlinger가 첫 visible layer에 어떤 색/알파 상태를 붙였는지 먼저 고정한다**

   ```text
   SF_COLOR_STATE layer=MainActivity#0 buffer_id=0x9421 dataspace=SRGB plane_alpha=1.0 blend=PREMULT color_transform=identity
   SF_COLOR_STATE layer=MainActivity#0 buffer_id=0x9421 dataspace=UNKNOWN plane_alpha=0.0 blend=COVERAGE color_transform=identity
   ```

   첫 non-black buffer라도 `dataspace=UNKNOWN` 이거나 `plane_alpha=0.0` 이면 실제 출력은 검게 보일 수 있다. geometry 정상 여부와 별개로 `dataspace`, `alpha`, `blend` 를 함께 봐야 한다.

2. **HWC가 받은 합성 속성이 첫 프레임에서만 잘못 프로그래밍되지 않는지 본다**

   ```text
   HWC_COMPOSE_ATTR display=2 layer=57 dataspace=SRGB pixel_format=RGBA8888 blend=PREMULT plane_alpha=255 dimming=0
   HWC_COMPOSE_ATTR display=2 layer=57 dataspace=BT2020_PQ pixel_format=RGBA8888 blend=NONE plane_alpha=0 dimming=1
   ```

   Android 쪽 layer state는 정상인데 HWC adapter에서 enum 변환이나 default fallback이 틀리면 첫 present 한 장만 black으로 나갈 수 있다. 특히 `plane_alpha=0`, `blend=NONE`, `dimming=1` 조합은 의심 우선순위가 높다.

3. **display color mode 전환이 첫 앱 프레임과 경쟁하지 않는지 확인한다**

   ```text
   DISPLAY_COLOR_MODE display=2 request=NATIVE applied=NATIVE frame=6314
   DISPLAY_COLOR_MODE display=2 request=SRGB applied=PENDING frame=6314 fallback=BOOT_DEFAULT
   ```

   부팅 직후 panel이나 display pipeline이 아직 boot default mode에 머물러 있으면 첫 앱 프레임 한 장만 잘못된 matrix/range로 합성될 수 있다. 이때는 다음 프레임부터 정상이라 재현성이 낮게 보인다.

4. **검정처럼 보이는 원인이 실제 zero-alpha인지 color transform crush인지 분리한다**

   ```text
   FIRST_PIXEL_RESULT display=2 frame=6314 sample_rgb=0,0,0 plane_alpha=0 source_nonblack=1
   FIRST_PIXEL_RESULT display=2 frame=6314 sample_rgb=3,2,3 plane_alpha=255 ctm=limited_to_full_mismatch
   ```

   `source_nonblack=1` 인데 sample이 0에 붙어 있으면 진짜 black buffer가 아니라 합성 속성 문제다. `alpha zero` 와 `CTM/range mismatch` 는 대응 팀이 다르므로 첫 줄에서 갈라야 한다.

5. **첫 frame 전용인지 다음 frame 회복 여부까지 묶어 판정한다**

   ```text
   FIRST_FRAME_COMPOSE_VERDICT display=2 frame=6314 attr_fault=plane_alpha_zero recovered_next_frame=1 black_flash=1
   FIRST_FRAME_COMPOSE_VERDICT display=2 frame=6315 attr_fault=none recovered_next_frame=0 visible=1
   ```

   다음 프레임에서 `plane_alpha=255`, `dataspace=SRGB` 로 즉시 회복되면 지속적 panel 이슈가 아니라 `initial composition state race` 로 좁힐 수 있다.

## 리스크

- fence 체인이 정상이라는 이유로 HWC 합성 속성을 건너뛰면 원인 절단이 한 단계 늦어진다.
- dataspace와 color mode를 분리해서 보지 않으면 `SF state 문제`와 `display pipeline mode apply 지연`이 섞인다.
- alpha/blend default 값은 첫 frame 한 장만 틀어질 수 있어 평균 로그나 steady-state dump로는 잘 안 잡힌다.
- HDR/SDR 혼합 경로가 남아 있으면 특정 display에서만 black flash가 재현되어 multi-display 라우팅 이슈로 오인될 수 있다.

## 다음 액션

다음 글에서는 Day95 다음 절단면으로, **합성 속성도 정상인데 첫 프레임만 사라질 때 idle frame drop·doze/blank transition·display enable sequencing이 첫 app present를 버리는 전원/상태 전이 cutline** 을 정리하겠다.

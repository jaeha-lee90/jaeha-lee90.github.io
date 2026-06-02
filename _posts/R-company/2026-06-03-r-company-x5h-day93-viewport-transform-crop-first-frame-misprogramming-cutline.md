---
title: R사 X5H Day93 - viewport·transform·crop first-frame mis-programming 절단면
author: JaeHa
date: 2026-06-03 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day93, display, viewport, transform, crop, surfaceflinger, hwc, bringup]
---

Day92에서 `first non-black queue/latch` 까지 확인됐는데도 사용자는 **첫 화면이 여전히 검게 보이거나 일부만 번쩍 보였다 사라진다면**, 다음으로 잘라야 할 구간은 `buffer content 정상 → layer crop/transform 계산 → HWC viewport/programming 적용` 이다. 즉 첫 프레임 자체는 올라왔지만 **표시 좌표나 crop이 잘못 잡혀 실제 visible area가 0 또는 off-screen** 으로 밀리는지 확인해야 한다.

## 핵심 요약

- Day93의 1차 절단 순서는 **first non-black latch → layer crop/transform 확정 → HWC source/dest rect programming → panel visible area 검증** 이다.
- `queueBuffer 성공` 만으로는 표시 성공을 보장하지 않는다. `crop=0`, `displayFrame out-of-bounds`, `transform mismatch` 면 사용자는 black으로 체감한다.
- 최소 증적은 `LAYER_GEOM`, `SF_GEOM_APPLY`, `HWC_RECT_PROGRAM`, `VISIBLE_AREA_RESULT` 네 줄이면 된다.
- 이 구간에서 막히면 app rendering보다는 `SurfaceFlinger/HWC geometry programming` 문제일 확률이 높다.

## 코드 포인트

1. **SurfaceFlinger가 첫 non-black buffer에 어떤 geometry를 붙였는지 고정한다**

   ```text
   LAYER_GEOM layer=MainActivity#0 crop=[0,0,1920,720] frame=[0,0,1920,720] transform=0 alpha=1.0
   LAYER_GEOM layer=MainActivity#0 crop=[0,0,1920,720] frame=[1920,0,3840,720] transform=ROT_90 display=2
   ```

   첫 latch 직후 `crop`, `displayFrame`, `transform`, `target display` 를 같이 봐야 한다. buffer는 정상이어도 frame이 다른 display 좌표로 밀리면 현재 panel에서는 black처럼 보인다.

2. **transaction merge 이후 geometry가 조용히 바뀌지 않았는지 확인한다**

   ```text
   SF_GEOM_APPLY layer=MainActivity#0 txn=88421 requested_crop=[0,0,1920,720] applied_crop=[0,0,0,720]
   SF_GEOM_APPLY layer=MainActivity#0 requested_transform=0 inherited_transform=ROT_180 leash=TaskDisplayArea#2
   ```

   launcher attach 직후 leash hierarchy 또는 parent transform 이 섞이면 app이 올린 rect와 실제 applied rect가 달라질 수 있다. 특히 `width=0` 또는 `height=0` 으로 줄어든 crop은 첫 프레임 black 체감의 흔한 절단점이다.

3. **HWC가 source/dest rect를 panel 기준으로 어떻게 프로그래밍했는지 본다**

   ```text
   HWC_RECT_PROGRAM display=2 layer=57 src=0,0,1920,720 dst=0,0,1920,720 blend=PREMULT format=RGBA8888
   HWC_RECT_PROGRAM display=2 layer=57 src=0,0,1920,720 dst=1919,0,1919,720 clipped_w=0 clipped_h=720
   ```

   `src` 는 정상인데 `dst` clipping 후 width/height 가 0 이 되면 panel 쪽에는 아무것도 안 나온다. 이 경우 buffer dump는 정상인데 실화면만 black이라 app 쪽으로 오판하기 쉽다.

4. **rotation/stride/viewport 조합이 첫 프레임에서만 틀어지는지 분리한다**

   ```text
   HWC_RECT_PROGRAM display=2 layer=57 transform=ROT_90 src_w=1920 src_h=720 dst_w=720 dst_h=1920
   VISIBLE_AREA_RESULT display=2 layer=57 visible_px=0 reject_reason=viewport_out_of_range
   ```

   boot 직후 orientation 결정이 늦으면 첫 프레임 한두 장만 portrait/landscape 기준이 엇갈릴 수 있다. 특히 cluster-like wide panel 에서는 `ROT_90 + stale viewport` 조합이 black으로 바로 체감된다.

5. **panel 최종 visible area와 다음 정상 프레임을 비교해 첫-frame-only 결함인지 고정한다**

   ```text
   VISIBLE_AREA_RESULT display=2 frame=6311 layer=57 visible_px=0 panel=1920x720
   VISIBLE_AREA_RESULT display=2 frame=6312 layer=57 visible_px=1382400 panel=1920x720 recovered=1
   ```

   첫 프레임만 `visible_px=0` 이고 바로 다음 프레임이 정상이라면 지속적인 black-screen 이 아니라 `initial geometry programming race` 로 범위를 줄일 수 있다.

## 리스크

- first non-black latch 이후를 보지 않으면 geometry zero-area 문제를 app black frame 으로 오판한다.
- crop/source/dest rect 중 하나만 보면 실제 clipping 원인을 놓친다.
- display rotation/orientation settle 타이밍을 무시하면 첫 프레임 전용 race 를 재현 불가 이슈로 흘리기 쉽다.
- multi-display/mirror 정책이 남아 있으면 올바른 content가 다른 panel에만 보여 현재 panel은 black처럼 보일 수 있다.

## 다음 액션

다음 글에서는 Day93 다음 절단면으로, **geometry는 정상인데 첫 프레임만 acquire/release fence 또는 present fence 정렬이 꼬여 실제 scanout 시점이 한 박자 밀리는 synchronization cutline** 을 정리하겠다.

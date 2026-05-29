---
title: R사 X5H Day89 - wallpaper·dim layer·color layer·trusted overlay occlusion 체크리스트
author: JaeHa
date: 2026-05-30 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day89, display, surfaceflinger, wallpaper, dimlayer, overlay, occlusion, bringup]
---

Day88에서 `transaction merge order`, `leash hierarchy`, `visibility bit propagation` 까지 정상이면, 이제 남는 blank는 **앱 surface는 실제로 올라왔지만 topmost 보조 layer가 화면을 덮어 사용자가 검은 화면으로 체감하는 경우** 로 좁혀진다. 특히 wallpaper fade, dim layer, boot color layer, trusted overlay가 전환 종료 후에도 한 프레임 이상 남으면 app first frame은 존재해도 눈에는 계속 blank처럼 보인다.

## 핵심 요약

- Day88 이후에도 `present ok` 인데 화면이 검게 보이면 1차 절단 순서는 **top layer stack → dim/wallpaper/color layer alpha → trusted overlay occlusion → cleanup transaction** 이다.
- app layer `show=1` 만으로는 충분하지 않다. 더 위 z-order 에 `ColorLayer` 나 `DimLayer` 가 남아 있으면 실제 visible output은 가려진다.
- 최소 증적은 `TOP_LAYER_STACK`, `OCCLUSION_STATE`, `OVERLAY_POLICY`, `LAYER_CLEANUP` 네 줄이다.
- 이 구간은 panel/HWC before-SF 문제가 아니라, `WindowManager ↔ SurfaceFlinger` 의 상위 layer lifecycle 정리 실패일 가능성이 높다.

## 코드 포인트

1. **문제 프레임의 top layer stack을 먼저 고정한다**

   ```text
   TOP_LAYER_STACK display=2 frame=6215 z=245 layer=BootColorLayer type=color alpha=1.0 visible=1
   TOP_LAYER_STACK display=2 frame=6215 z=244 layer=NavRoot type=buffer alpha=1.0 visible=1
   ```

   app layer가 visible 이어도 더 높은 z-order 에 불투명 color layer가 있으면 사용자는 blank만 본다. 첫 절단은 `app not shown` 이 아니라 `shown but covered` 인지 확인하는 것이다.

2. **wallpaper/dim/color layer의 alpha와 crop이 cleanup됐는지 본다**

   ```text
   OCCLUSION_STATE display=2 layer=TaskDimLayer alpha=0.92 crop=full visible=1 owner=Transition#196
   OCCLUSION_STATE display=2 layer=BootColorLayer alpha=1.0 color=0x000000 cleanup_pending=1
   ```

   transition 종료 직전 dim alpha가 0으로 내려가지 않거나 boot color layer가 remove되지 않으면 app buffer는 정상이어도 검은 막이 남는다. alpha, crop, remove 예약 여부를 한 줄에 묶어야 한다.

3. **trusted overlay가 정책상 의도된 것인지 분리한다**

   ```text
   OVERLAY_POLICY display=2 layer=ClusterSafetyOverlay trusted=1 touchable=0 occluding=1 reason=boot_guard
   OVERLAY_POLICY display=2 layer=ClusterSafetyOverlay release_condition=home_drawn latched=0
   ```

   cluster/center bring-up 에서는 안전 overlay가 의도적으로 화면을 덮을 수 있다. 문제는 overlay 존재 자체가 아니라 release condition 이 충족됐는데도 occluding 상태가 유지되는지다.

4. **SurfaceFlinger 최종 composition 결과에서 opaque cover를 확인한다**

   ```text
   SF_COMPOSE_RESULT display=2 frame=6215 output_opaque_cover=1 top_opaque=BootColorLayer client_comp=0 device_comp=1
   SF_COMPOSE_RESULT display=2 frame=6215 app_layer=NavRoot visible_region=full occluded_by=BootColorLayer
   ```

   HWC present 성공만 보면 정상처럼 보일 수 있다. 실제로는 `presented black layer` 일 수 있으므로 최종 composition 결과에서 누가 full-screen opaque cover 인지 봐야 한다.

5. **cleanup transaction과 release trigger의 연결을 검증한다**

   ```text
   LAYER_CLEANUP display=2 owner=Transition#196 target=BootColorLayer action=remove trigger=home_first_present applied=0
   LAYER_CLEANUP display=2 owner=WallpaperFade target=TaskDimLayer action=alpha->0 done=0 blocker=finish_callback_missing
   ```

   대부분의 재현성 낮은 blank는 layer 생성이 아니라 cleanup trigger 유실에서 나온다. `first present`, `transition finish`, `wallpaper fade end` 중 어느 trigger가 빠졌는지 바로 보이게 해야 한다.

## 리스크

- boot color layer를 성급히 제거하면 아직 준비되지 않은 뒷단 frame garbage가 잠깐 노출될 수 있다.
- trusted overlay release 조건을 약하게 잡으면 안전/경고 UI가 너무 빨리 사라져 제품 요구사항을 깨뜨릴 수 있다.
- wallpaper와 dim layer를 한 로그로 뭉뚱그리면 실제 culprit 가 alpha 잔류인지 z-order 역전인지 구분이 안 된다.
- multi-display 에서는 center 쪽 cleanup이 cluster overlay state에 종속될 수 있어 한쪽 display만 간헐 black으로 남을 수 있다.

## 다음 액션

다음 글에서는 Day89를 이어서, **occlusion layer 정리까지 정상인데도 첫 장면이 어둡거나 멎어 보일 때 screenshot/client composition/GPU output 자체가 검은 프레임인지 판별하는 절단면** 을 정리하겠다.

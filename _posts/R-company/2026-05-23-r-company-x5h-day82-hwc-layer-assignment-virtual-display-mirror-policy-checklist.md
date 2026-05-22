---
title: R사 X5H Day82 - backend route 정상 후 남는 HWC layer assignment·virtual display·mirror policy 체크리스트
author: JaeHa
date: 2026-05-23 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day82, display, hwc, surfaceflinger, virtual-display, mirror, layer-assignment, bringup]
---

Day81에서 `backend mux`, `clone/splitter`, `DSC slice`, `output enable sequence` 까지 정상인데도 target panel만 비어 있으면, 이제는 panel ingress가 아니라 **Android display composition policy** 를 먼저 잘라야 한다. X5H bring-up 후반에는 route는 맞지만 `HWC display assignment`, `virtual display/mirror target`, `SurfaceFlinger layer stack ownership` 이 어긋나 실제 표시해야 할 layer가 다른 output으로 빠지는 경우가 자주 남는다.

## 핵심 요약

- Day81을 통과했는데 특정 panel만 비면 1차 절단 순서는 **SurfaceFlinger logical display ↔ HWC physical display mapping → layer assignment → virtual display/mirror policy → secure/protected layer 제약** 이다.
- backend route가 정상이더라도 `layer stack` 이 다른 display id에 묶이면 panel은 valid scanout 없이 검게 남는다.
- `mirror/virtual display` 가 기본 display를 잡아먹으면 app frame은 살아도 target cluster에는 wallpaper/blank layer만 남을 수 있다.
- 최소 증적은 `SF_DISPLAY_MAP`, `HWC_ASSIGN`, `MIRROR_POLICY`, `PRESENT_FENCE` 네 줄이면 충분하다.

## 코드 포인트

1. **logical display 와 physical display 매핑을 먼저 고정한다**

   ```text
   SF_DISPLAY_MAP logical=cluster layer_stack=42 hwc_display=2 type=physical expected_port=dsi1
   HWC_DISPLAY_CAPS hwc_display=2 port=dsi1 connected=1 active_config=3
   ```

   panel route가 맞아도 SurfaceFlinger가 `layer_stack=42` 를 다른 `hwc_display` 에 붙여 놓으면 target panel에는 유효 layer가 올라오지 않는다. bring-up 로그에서 `displayId`, `port`, `layer_stack` 을 한 줄로 같이 봐야 한다.

2. **HWC layer assignment 가 target display에 실제로 붙는지 확인한다**

   ```text
   HWC_ASSIGN display=2 layer=Navigation role=client composition=DEVICE z=12
   HWC_ASSIGN display=0 layer=ClusterApp role=app composition=DEVICE expected_display=2 mismatch=1
   ```

   앱이 떴는데도 panel이 비면 종종 content layer가 default display(0) 로 배정되고 target display에는 dim/wallpaper만 남아 있다. 이 경우 panel debug를 더 해 봐야 소득이 없다.

3. **virtual display / mirror policy 가 main output을 가로채는지 본다**

   ```text
   MIRROR_POLICY source_display=2 sink_display=0 enabled=1 reason=boot_mirror_default
   VIRTUAL_DISPLAY name=CastDisplay owner=systemui layer_stack=42 steals_content=1
   ```

   초기 부팅 policy나 vendor extension이 `mirror-to-main` 을 기본값으로 두면 cluster용 layer stack이 center stack으로 복제되거나 역으로 흡수될 수 있다. 특히 화면은 하나 보이는데 target panel만 비는 패턴이면 mirror policy가 빠른 절단점이다.

4. **secure/protected layer 제약은 '일부 화면만 검음' 패턴을 만든다**

   ```text
   HWC_ASSIGN display=2 layer=RearCamera protected=1 composition=CLIENT blocked_reason=no_protected_path
   PRESENT_FENCE display=2 frame=1187 layers_submitted=4 layers_visible=1 protected_drop=3
   ```

   protected path 미구성 상태에서 secure layer가 target display로 가면 present는 성공해도 핵심 layer만 빠져 검은 화면처럼 보일 수 있다. rear camera, DRM UI, secure overlay 경로에서 특히 자주 보인다.

5. **present fence 와 visible layer count 를 같이 남겨야 마지막 오판을 줄인다**

   ```text
   PRESENT_FENCE display=2 frame=1187 retire_ms=16.7 visible_layers=0
   SF_COMPOSITION_RESULT display=2 framebuffer_written=1 nonempty_stack=1 final_visible=0
   ```

   retire fence만 보면 정상처럼 보이지만 `visible_layers=0` 이면 실제 사용자가 보는 출력은 비어 있다. `submitted` 와 `visible` 을 분리해 남겨야 `composition success but empty scene` 을 빨리 자를 수 있다.

## 리스크

- display route 정상만 확인하고 Android composition policy를 늦게 보면 panel/DSI 쪽에서 며칠을 허비할 수 있다.
- multi-display 제품은 SKU/boot mode별 기본 mirror 정책이 달라 cold boot, service restart, valet mode에서 증상이 다르게 보일 수 있다.
- protected content drop은 전체 black screen이 아니라 핵심 앱만 사라지는 형태라 단순 app crash로 오인하기 쉽다.
- HWC/SF display id 재열거가 hotplug 시 바뀌면 재부팅 때만 재현되는 불안정 결함으로 남는다.

## 다음 액션

다음 글에서는 Day82를 이어서, **HWC assignment까지 정상인데도 특정 panel에서만 blank가 남을 때 GPU client composition fallback, overlay capability mismatch, fence starvation을 자르는 체크리스트** 를 정리하겠다.

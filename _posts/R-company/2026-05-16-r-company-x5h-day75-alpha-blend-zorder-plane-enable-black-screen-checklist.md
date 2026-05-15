---
title: R사 X5H Day75 - IOMMU/cache sync 정상 후 남는 alpha/blend/z-order/plane enable black screen 체크리스트
author: JaeHa
date: 2026-05-16 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day75, display, alpha, blend, z-order, plane-enable, black-screen, bringup]
---

Day74에서 `plane src/iova`, `passthrough`, `cache sync` 까지 모두 정상인데 panel은 여전히 검다면, 다음으로 빨리 잘라야 할 축은 **plane이 실제로 보이도록 합성 조건이 맞았는지** 다. 이 단계에서는 대개 `alpha=0`, `z-order misroute`, `plane disable`, `global blank/cover layer` 네 가지가 원인이다.

## 핵심 요약

- `fetch 가능` 과 `visible` 은 다르다. buffer를 읽을 수 있어도 **blend/enable 상태가 틀리면 결과는 black screen** 이다.
- bring-up 초반 절단 순서는 **plane enable → alpha/global alpha → z-order/top cover layer → blanking/CSC side effect** 가 가장 빠르다.
- `VBLANK alive + IOMMU OK + content OK` 인데 검다면, timing보다 **compositor state ledger** 를 먼저 남겨야 한다.
- 로그는 `plane enabled?`, `alpha 얼마?`, `z 몇 번?`, `top layer 누구?` 네 줄이면 1차 절단이 된다.

## 코드 포인트

1. **plane fetch 성공과 plane visible은 별개다**

   아래처럼 source 계열이 정상이더라도 아직 충분하지 않다.

   ```text
   PLANE_SRC plane=0 iova=0x7c800000 pitch=7680 format=ARGB8888
   IOMMU_MAP master=RVGC domain=disp0 result=OK
   CACHE_SYNC buf=cluster_main dir=to-device result=OK
   ```

   여기에 반드시 붙어야 하는 건 최종 합성 상태다.

   ```text
   PLANE_STATE display_slot=0 plane=0 enable=1 alpha=255 z=12 blend=premult
   TOP_LAYER display_slot=0 layer_id=cluster_home z=12 opaque=1
   ```

   이 두 줄이 없으면 "읽긴 읽었는데 왜 안 보이냐" 를 끝까지 못 자른다.

2. **alpha=0 또는 global alpha default가 black screen을 만든다**

   bring-up 초기에 register default나 binder property 초기화 누락으로 아래 패턴이 자주 나온다.

   ```text
   PLANE_STATE plane=0 enable=1 pixel_alpha=1 global_alpha=0
   BLEND_RESULT plane=0 visible=0 reason=global_alpha_zero
   ```

   source checksum이 멀쩡해도 `global_alpha=0` 이면 사용자는 그냥 검은 화면으로 본다. 특히 Android layer state를 RVGC plane state로 변환할 때 `255` 기본값이 빠지면 이 케이스가 바로 생긴다.

3. **z-order misroute는 wrong content보다 더 조용하게 숨는다**

   ```text
   SF_LAYER name=ClusterHome z=200 display=0
   RVGC_PLANE_BIND plane=0 layer_id=cluster_home z=3
   RVGC_PLANE_BIND plane=1 layer_id=black_mask z=4 opaque=1
   ```

   여기서는 cluster layer가 살아 있어도 `black_mask` 나 clear plane이 더 위에 올라와서 panel이 검게 보인다. 즉 `plane bind OK` 만 보지 말고 **최상위 opaque plane이 누구인지** 까지 같이 찍어야 한다.

4. **plane enable race는 commit success 뒤에 숨어 버린다**

   ```c
   plane->enable = layer_has_content;
   submit_plane(plane);
   post_commit_cleanup(plane);
   ```

   `post_commit_cleanup()` 나 mode-set 직후 restore 단계에서 `enable=0` 으로 덮어쓰면, commit 자체는 성공해도 scanout에는 안 올라간다. 이 경우는 `submit ok` 와 별개로 **post-commit register snapshot** 이 필요하다.

5. **최소 합성 ledger는 네 줄이면 충분하다**

   ```text
   PLANE_STATE plane=0 enable=1 alpha=255 z=12 blend=premult
   TOP_LAYER display_slot=0 layer_id=cluster_home z=12 opaque=1
   COVER_LAYER display_slot=0 present=0
   POST_COMMIT_REG plane=0 enable=1 live_alpha=0xff live_z=12
   ```

   이 네 줄이 같은 `seq/frame_id` 로 묶이면, Day74 이후 남는 black screen을 10분 안에 거의 다 자를 수 있다.

## 리스크

- alpha 기본값 누락은 source/content 쪽 로그가 모두 정상으로 보여 원인 분리가 늦어진다.
- top opaque layer 로그가 없으면 black mask/clear plane이 실제 root cause여도 panel/timing 문제로 오판하기 쉽다.
- commit 전 state만 찍고 post-commit live register를 안 보면 enable race를 놓친다.
- blend mode(`premult`/`coverage`) 해석 차이를 방치하면 특정 UI만 검게 사라지는 간헐 증상으로 번질 수 있다.

## 다음 액션

다음 글에서는 Day75를 이어서, **alpha/z-order/plane enable까지 정상인데도 black screen일 때 CSC/RGB-YUV/range swap과 panel-side blanking 축을 빠르게 자르는 체크리스트** 로 더 좁혀 보겠다.

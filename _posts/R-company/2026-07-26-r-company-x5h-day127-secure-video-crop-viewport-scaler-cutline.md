---
title: R사 X5H Day127 - secure video crop·viewport·scaler 제한 black-frame cutline
author: JaeHa
date: 2026-07-26 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day127, display, protected, crop, viewport, scaler, bringup]
---

Day126에서 `modifier`, `stride`, `header alignment`, `plane modifier allowlist` 까지 맞는데도 **secure video만 검게 남고 non-secure 샘플은 같은 경로에서 보인다** 면 남은 절단면은 `decoder output crop`, `plane src/dst rect`, `viewport/scaler secure 제한` 이다. 즉 메모리 fetch는 성공하지만, **secure plane이 허용하는 crop/scale envelope 밖으로 programming 되어 최종 scanout 직전에 버려지는지** 를 잘라야 한다.

## 핵심 요약

- Day127의 핵심 순서는 **decoder crop 고정 → HWC src/dst rect 대조 → secure scaler capability 확인 → commit reject/silent clamp 증적 결합** 이다.
- secure path에서는 `src_crop`, `dst_rect`, `scale_ratio`, `rotation`, `pixel_extension` 중 하나라도 plane 보안 제약을 넘으면 validate는 DEVICE로 통과해도 실제 output은 black frame 또는 fully clipped layer로 남을 수 있다.
- `DECODER_CROP_TRACE`, `LAYER_RECT_TRACE`, `SECURE_SCALER_TRACE`, `PLANE_CLIP_TRACE`, `FINAL_SECURE_RECT_VERDICT` 를 같은 `buffer_id`, `plane`, `frame` 으로 묶어야 한다.
- 최소 증적은 `decoded_w/h`, `crop(l,t,r,b)`, `dst(x,y,w,h)`, `scale_ratio`, `secure_scale_ok` 다섯 축이면 된다.

## 코드 포인트

1. **decoder가 넘긴 실제 visible crop을 먼저 고정한다**

   ```text
   DECODER_CROP_TRACE buffer_id=0xa241 protected=1 coded=1920x1088 visible=1920x1080 crop=0,0,1920,1080
   DECODER_CROP_TRACE buffer_id=0xa242 protected=1 coded=3840x2160 visible=3840x2160 crop=0,8,3840,2152
   ```

   secure video는 codec alignment 때문에 coded size와 visible crop이 다를 수 있다. 이 차이를 잃으면 downstream clipping 원인을 놓친다.

2. **HWC layer src/dst rect를 plane programming 직전 값으로 남긴다**

   ```text
   LAYER_RECT_TRACE frame=9021 layer=video plane=vp2 src=0,8,3840,2152 dst=80,64,1760,936 transform=0 protected=1
   ```

   SurfaceFlinger/HWC가 crop을 한 번 더 적용하거나 letterbox 조정을 넣으면 decoder가 의도한 visible 영역과 실제 plane src가 어긋날 수 있다.

3. **secure plane의 scaler/crop 허용 범위를 capability로 분리한다**

   ```text
   SECURE_SCALER_TRACE plane=vp2 secure_ok=1 min_scale=0.5 max_scale=2.0 h_align=2 v_align=2 rot_with_scale=0 pixel_extension=limited
   ```

   일반 plane capability와 secure plane capability가 다르면 `validate=DEVICE` 만으로는 안전하지 않다. 특히 rotation+scaling 동시 사용 금지가 흔한 절단면이다.

4. **silent clamp와 fully clipped 결과를 black-frame과 직접 연결한다**

   ```text
   PLANE_CLIP_TRACE frame=9021 plane=vp2 src_req=0,8,3840,2152 src_prog=0,8,3838,2152 dst_req=80,64,1760,936 dst_prog=80,64,0,0 reason=secure_scale_limit
   ```

   일부 backend는 commit fail 대신 dst를 0x0으로 clamp 하거나 src를 align-down 하면서 결과만 검게 만든다. 이 경우 `present fence` 는 정상이라 더 헷갈린다.

5. **최종 원인을 rectangle/scaler owner 기준으로 귀속한다**

   ```text
   FINAL_SECURE_RECT_VERDICT frame=9021 display=2 cause=rotation_with_scaling_not_allowed owner=hwc_plane_caps symptom=black_secure_video
   FINAL_SECURE_RECT_VERDICT frame=9030 display=2 cause=decoder_visible_crop_mismatch owner=codec_to_hwc_contract symptom=right_edge_clipped_then_black
   ```

   이렇게 정리해야 Day126의 fetch-format 문제와 Day127의 geometry/scaler 문제를 섞지 않고 후속 수정 지점을 바로 지정할 수 있다.

## 리스크

- codec이 visible crop metadata를 vendor-private path로만 넘기면 SurfaceFlinger/HWC 로그만으로는 coded/visible 차이를 복원하기 어렵다.
- secure plane은 writeback/capture가 제한되어 fully clipped 상태를 화면 캡처만으로 확인하기 어렵다.
- scaler 비율은 맞아도 홀수 crop, chroma alignment, 90도 회전 결합에서만 secure path가 깨질 수 있어 재현 조건이 좁다.
- backend가 reject 대신 silent clamp를 선택하면 commit success, present success, no fault 조합으로 보여 triage 시간이 길어진다.

## 다음 액션

다음 글에서는 Day127 다음 절단면으로, **secure scaler/crop까지 맞는데도 남는 경우 color format(YUV420/NV12/P010)·chroma siting·CSC 경계가 secure plane에서 black/green frame으로 번지는 경로** 를 정리하겠다.

---
title: R사 X5H Day125 - protected buffer 속성·plane capability·GPU fallback cutline
author: JaeHa
date: 2026-07-24 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day125, display, protected, secure, gralloc, hwc, bringup]
---

Day124에서 `CACHE_POLICY_TRACE`, `CPU_MAP_TRACE`, `DMA_BUF_SYNC_TRACE`, `EXPLICIT_SYNC_TRACE` 까지 정상인데도 **특정 video layer만 검게 나오거나, secure UI에서만 HWC overlay가 사라지고 GPU fallback이 반복** 된다면 다음 절단면은 `protected buffer 속성`, `display plane capability`, `GPU fallback policy` 다. 즉 coherency 문제는 끝났고, 이제는 **이 버퍼를 해당 plane이 합법적으로 스캔아웃할 수 있는지** 를 잘라야 한다.

## 핵심 요약

- Day125의 절단 순서는 **buffer 보호 속성 고정 → plane secure capability 대조 → HWC validate/accept 결과 비교 → fallback owner 확정** 이다.
- `is_protected=1`, `is_secure_display_only=1`, `afbc/protected 조합` 중 하나라도 plane capability와 어긋나면 증상은 black layer처럼 보여도 실제 원인은 **보호 속성-하드웨어 계약 위반** 일 가능성이 높다.
- `PROTECTED_ALLOC_TRACE`, `PLANE_CAP_TRACE`, `HWC_VALIDATE_TRACE`, `GPU_FALLBACK_TRACE`, `FINAL_PROTECTED_VERDICT` 를 같은 `buffer_id`, `layer_id`, `frame` 으로 묶어야 한다.
- 최소 증적은 `usage_protected`, `plane_secure_ok`, `validated_type`, `fallback_reason`, `display_security_mode` 다섯 축이면 충분하다.

## 코드 포인트

1. **gralloc allocation 시점에 보호 속성을 먼저 고정한다**

   ```text
   PROTECTED_ALLOC_TRACE buffer_id=0xa181 layer=video usage_protected=1 secure_display_only=1 compression=afbc producer=codec
   PROTECTED_ALLOC_TRACE buffer_id=0xa182 layer=video usage_protected=0 secure_display_only=0 compression=afbc producer=codec
   ```

   같은 decoder path라도 `usage_protected` 차이 하나로 HWC 경로가 완전히 달라질 수 있다.

2. **plane이 secure/protected scanout을 지원하는지 capability ledger로 바로 자른다**

   ```text
   PLANE_CAP_TRACE plane=vp0 display=cluster0 secure_ok=0 afbc_ok=1 yuv_ok=1 scaler_ok=1
   PLANE_CAP_TRACE plane=vp2 display=cluster0 secure_ok=1 afbc_ok=1 yuv_ok=1 scaler_ok=0
   ```

   `secure_ok=0` plane에 protected layer를 얹으려 하면 import/coherency가 정상이어도 scanout 계약은 이미 깨진 상태다.

3. **HWC validate 단계에서 type 변경 사유를 남긴다**

   ```text
   HWC_VALIDATE_TRACE frame=8122 layer=video requested=DEVICE validated=CLIENT reject_reason=protected_plane_cap_mismatch candidate_plane=vp0
   ```

   validate에서 `DEVICE -> CLIENT` 로 바뀌었다면 black frame처럼 보여도 우선은 **overlay 불수용** 문제로 분류해야 한다.

4. **GPU fallback이 허용된 경로인지, 아니면 정책상 금지돼 black layer가 맞는지 분리한다**

   ```text
   GPU_FALLBACK_TRACE frame=8122 layer=video allowed=0 reason=protected_content client_target_secure=0 result=black_layer
   GPU_FALLBACK_TRACE frame=8123 layer=osd allowed=1 reason=nonsecure_layer result=client_composition
   ```

   protected content는 일반 client target으로 우회 불가한 경우가 많아서 fallback 자체가 해법이 아니라 **정책 위반 표시** 일 수 있다.

5. **최종 verdict를 buffer owner가 아니라 security contract owner 기준으로 고정한다**

   ```text
   FINAL_PROTECTED_VERDICT frame=8122 display=2 cause=plane_secure_cap_missing owner=display_capability symptom=black_video
   FINAL_PROTECTED_VERDICT frame=8124 display=2 cause=client_target_not_secure owner=composition_policy symptom=silent_gpu_fallback
   ```

   이렇게 고정해야 Day124의 coherency 이슈와 Day125의 secure composition 이슈를 같은 black frame으로 뭉개지 않는다.

## 리스크

- vendor HWC는 protected reject reason을 자세히 로그하지 않아 `CLIENT` fallback만 보이고 실제 capability mismatch가 숨을 수 있다.
- secure path에서 writeback/screenshot이 막혀 있으면 검은 화면 증적만 남고 실제 scanout 상태를 직접 관찰하기 어렵다.
- Xen 분리 환경에서는 DomD plane capability와 DomA HWC policy가 어긋나 한쪽만 수정하면 증상이 유지될 수 있다.
- `secure_display_only` 와 DRM/protected playback 정책이 섞이면 특정 display에서는 정상, 다른 display에서는 무조건 black으로 보여 재현 판단을 흐릴 수 있다.

## 다음 액션

다음 글에서는 Day125 다음 절단면으로, **protected plane capability까지 맞는데도 남는 경우 AFBC/stride/plane modifier 제한이 secure video black frame으로 번지는 경로** 를 정리하겠다.

---
title: R사 X5H Day126 - protected AFBC·stride·plane modifier 제한 secure video black-frame cutline
author: JaeHa
date: 2026-07-25 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day126, display, protected, afbc, modifier, stride, bringup]
---

Day125에서 `usage_protected`, `plane_secure_ok`, `GPU fallback policy` 까지 맞는데도 **secure video만 검게 남고 validate는 DEVICE로 통과** 한다면 이제 절단면은 `AFBC/header layout`, `stride/alignment`, `plane modifier allowlist` 다. 즉 secure composition 계약은 살아 있지만, **해당 protected buffer의 메모리 기술 형식을 plane fetch 엔진이 끝까지 해석하지 못하는지** 를 잘라야 한다.

## 핵심 요약

- Day126의 절단 순서는 **buffer modifier 고정 → plane modifier allowlist 대조 → stride/header alignment 검증 → secure fetch fault/underflow 증적 결합** 이다.
- `protected=1` 이고 `validated_type=DEVICE` 여도 `afbc superblock`, `header split`, `stride alignment`, `YUV tile modifier` 중 하나가 plane 제약을 넘으면 증상은 단순 black video처럼 보인다.
- `BUF_MOD_TRACE`, `PLANE_MOD_TRACE`, `STRIDE_ALIGN_TRACE`, `FETCH_FAULT_TRACE`, `FINAL_MODIFIER_VERDICT` 를 같은 `buffer_id`, `plane`, `frame` 으로 묶어야 한다.
- 최소 증적은 `modifier`, `stride_bytes`, `header_addr/alignment`, `plane_modifier_ok`, `fetch_result` 다섯 축이면 충분하다.

## 코드 포인트

1. **import 직후 protected buffer의 실제 modifier를 loss 없이 고정한다**

   ```text
   BUF_MOD_TRACE buffer_id=0xa181 layer=video protected=1 fourcc=NV12 modifier=AFBC_16x16_SPLITBLK stride=4096 header_stride=512
   BUF_MOD_TRACE buffer_id=0xa182 layer=video protected=1 fourcc=NV12 modifier=LINEAR stride=3840 header_stride=0
   ```

   userspace가 `afbc` 라고만 뭉뚱그리면 안 되고, split-block 여부와 header layout까지 내려와야 plane 제약과 1:1 비교할 수 있다.

2. **plane별 secure scanout allowlist를 modifier 단위로 잘라 둔다**

   ```text
   PLANE_MOD_TRACE plane=vp2 secure_ok=1 allowed_modifiers=AFBC_16x16,LINEAR disallow=AFBC_SPLITBLK,YUV_TILED
   ```

   Day125에서 `secure_ok=1` 이어도 modifier allowlist가 좁으면 validate는 통과하고 실제 fetch 단계에서만 실패할 수 있다.

3. **stride/header alignment를 import 성공이 아니라 fetch 가능성 기준으로 본다**

   ```text
   STRIDE_ALIGN_TRACE buffer_id=0xa181 plane=vp2 stride=4096 stride_align_req=256 header_addr=0x7c000 header_align_req=1024 result=header_misaligned
   ```

   secure path는 CPU map으로 내용을 확인하기 어려워서, alignment miss 하나가 silent black frame으로 남기 쉽다.

4. **underflow/fetch fault를 secure black layer와 직접 결합한다**

   ```text
   FETCH_FAULT_TRACE frame=8211 plane=vp2 protected=1 fault=afbc_header_decode_fail underflow=1 iommu_fault=0
   ```

   IOMMU fault가 없다고 메모리 경로가 정상인 건 아니다. AFBC header decode 실패나 tile parse 실패는 MMU 바깥에서 검은 화면으로 끝날 수 있다.

5. **최종 verdict를 composition owner가 아니라 memory-format owner로 고정한다**

   ```text
   FINAL_MODIFIER_VERDICT frame=8211 display=2 cause=afbc_splitblk_not_supported owner=plane_fetch_format symptom=black_secure_video
   FINAL_MODIFIER_VERDICT frame=8214 display=2 cause=header_alignment_violation owner=gralloc_allocator symptom=intermittent_black_after_mode_switch
   ```

   이렇게 고정해야 Day125의 protected policy mismatch와 Day126의 fetch format mismatch를 분리해 다시 재현 가능한 액션으로 연결할 수 있다.

## 리스크

- vendor gralloc/HWC가 modifier 세부값을 숨기면 `AFBC 사용` 정도만 보이고 실제 split-block, wide-block, tiled variant 차이가 로그에서 사라질 수 있다.
- secure buffer는 CPU dump가 막혀 있어 header/crop/stride 이상을 userspace에서 직접 검증하기 어렵다.
- mode switch나 decoder resolution change 뒤에 stride만 바뀌고 plane programming이 이전 값을 재사용하면 간헐 재현으로 보여 원인 고정이 늦어진다.
- DomD import 성공과 DomA composition 성공이 모두 찍혀도 CR52/display backend fetch 제약이 별도로 남아 있으면 black frame이 계속될 수 있다.

## 다음 액션

다음 글에서는 Day126 다음 절단면으로, **protected modifier/stride까지 맞는데도 남는 경우 decoder output crop·plane crop/viewport·secure scaler 제한이 black frame으로 번지는 경로** 를 정리하겠다.

---
title: R사 X5H Day123 - gralloc mapper·dma-buf import·modifier negotiation cutline
author: JaeHa
date: 2026-07-22 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day123, display, gralloc, dmabuf, modifier, mapper, bringup]
---

Day122에서 `COMP_META_TRACE`, `HEADER_FETCH_TRACE`, `LINEBUF_TRACE` 까지 정상인데도 **특정 producer buffer만 black tile / color block / import fail fallback** 로 이어진다면 다음 절단면은 `gralloc mapper`, `dma-buf import`, `modifier negotiation` 이다. 즉 display 후단은 멀쩡하지만 **버퍼가 생성·전달·import 되는 계약 자체가 plane이 기대한 layout와 맞는지** 먼저 잘라야 한다.

## 핵심 요약

- Day123의 절단 순서는 **producer allocation usage/modifier 고정 → mapper metadata/plane layout 확인 → HWC/KMS import 결과 비교 → 최종 buffer-contract verdict 확정** 이다.
- AFBC/UBWC surface 깨짐이 frame timing과 무관하게 특정 앱·특정 layer class에서만 반복되면 원인은 대개 scanout 단계보다 **allocator/import 계약 불일치** 다.
- `ALLOC_TRACE`, `MAPPER_LAYOUT_TRACE`, `IMPORT_NEGOTIATION_TRACE`, `FINAL_BUFFER_CONTRACT_VERDICT` 를 같은 `buffer_id` 와 `frame` 으로 묶어야 owner를 자를 수 있다.
- 최소 증적은 `usage`, `modifier`, `num_planes`, `stride/alignment`, `import_result` 다섯 축이면 충분하다.

## 코드 포인트

1. **producer가 어떤 usage/modifier로 버퍼를 만들었는지 먼저 고정한다**

   ```text
   ALLOC_TRACE buffer_id=0x91a2 format=RGBA8888 usage=GPU_TEXTURE|COMPOSER_OVERLAY modifier=AFBC_16X16 header_split=1 alloc_w=1920 alloc_h=720
   ALLOC_TRACE buffer_id=0x91a3 format=RGBA8888 usage=GPU_TEXTURE modifier=LINEAR header_split=0 alloc_w=1920 alloc_h=720
   ```

   같은 scene인데 `COMPOSER_OVERLAY` 가 빠지거나 `modifier` 가 바뀌면 이후 display path가 정상이어도 **import 가능한 layout 자체** 가 달라진다.

2. **mapper가 export한 plane layout과 display가 기대한 layout이 일치하는지 본다**

   ```text
   MAPPER_LAYOUT_TRACE buffer_id=0x91a2 num_planes=2 plane0_stride=7680 plane1_stride=512 offset1=0x240000 alignment=4096 expected_planes=2
   MAPPER_LAYOUT_TRACE buffer_id=0x91a3 num_planes=1 plane0_stride=7680 plane1_stride=0 offset1=0x0 expected_planes=2
   ```

   `expected_planes` 와 실제 `num_planes` 가 어긋나면 corruption은 fetch 문제가 아니라 **layout contract mismatch** 다.

3. **HWC/KMS import 단계에서 modifier fallback이 발생하는지 분리한다**

   ```text
   IMPORT_NEGOTIATION_TRACE buffer_id=0x91a2 hwc_modifier=AFBC_16X16 drm_modifier=AFBC_16X16 import_result=direct_scanout fallback=0
   IMPORT_NEGOTIATION_TRACE buffer_id=0x91a3 hwc_modifier=AFBC_16X16 drm_modifier=LINEAR import_result=gpu_fallback fallback=1 reason=modifier_mismatch
   ```

   `fallback=1` 이면 panel black tile처럼 보여도 실제 원인은 panel이 아니라 **import negotiation 실패 후 잘못된 합성 경로 진입** 일 수 있다.

4. **cross-domain 전달에서 dma-buf fd는 같아도 metadata sideband가 소실되지 않는지 자른다**

   ```text
   DMABUF_SIDEINFO_TRACE buffer_id=0x91a2 fd=184 domd_modifier=AFBC_16X16 doma_modifier=UNKNOWN metadata_blob_size=0
   ```

   Xen/virt 경계에서 `metadata_blob_size=0` 이면 payload는 살아 있어도 compression header 위치를 잃어 **도메인 간 import 계약** 이 깨진다.

5. **최종 verdict를 buffer-contract owner 기준으로 고정한다**

   ```text
   FINAL_BUFFER_CONTRACT_VERDICT display=2 frame=7418 cause=modifier_mismatch owner=gralloc_hwc symptom=black_tile
   FINAL_BUFFER_CONTRACT_VERDICT display=2 frame=7418 cause=sideband_metadata_drop owner=virt_buffer_bridge symptom=color_block
   ```

   이 verdict가 있어야 Day122의 metadata fetch 이슈와 Day123의 **buffer allocation/import 계약 이슈** 를 다른 수정 경로로 분리할 수 있다.

## 리스크

- 일부 플랫폼은 mapper가 실제 modifier를 직접 노출하지 않아 allocator debug property나 drm debugfs를 함께 봐야 한다.
- GPU fallback이 개입하면 증상이 사라질 수도 있어 direct-scanout 경로와 비교 로그를 반드시 같이 남겨야 한다.
- 같은 dma-buf fd라도 sideband metadata, plane offset, secure flag가 별도 경로로 전달되면 단순 fd 비교만으로는 오판한다.
- alignment를 맞추려고 allocator policy를 바꾸면 메모리 사용량과 bandwidth 특성이 함께 바뀌어 다른 병목을 숨길 수 있다.

## 다음 액션

다음 글에서는 Day123 다음 절단면으로, **buffer import 계약까지 정상인데도 남는 경우 gralloc cacheability·CPU map/unmap·explicit sync 누락이 어떻게 stale frame/부분 corruption으로 번지는지** 정리하겠다.

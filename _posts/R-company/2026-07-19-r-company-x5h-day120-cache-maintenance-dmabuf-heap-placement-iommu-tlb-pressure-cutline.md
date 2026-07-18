---
title: R사 X5H Day120 - cache maintenance·dma-buf heap placement·IOMMU TLB pressure cutline
author: JaeHa
date: 2026-07-19 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day120, display, cache, dma-buf, iommu, tlb, bringup]
---

Day119에서 `DISP_UNDERFLOW_TRACE`, `QOS_VOTE_TRACE`, `ICC_DVFS_TRACE` 까지 정상인데도 특정 transition, camera overlay, 첫 fullscreen animation에서만 **간헐적 black flash / stale line / one-frame corruption** 이 남는다면 다음 절단면은 `cache maintenance`, `dma-buf heap placement`, `IOMMU TLB pressure` 다. 즉 대역폭 총량은 충분하지만 **실제 scanout 시점에 올바른 데이터가 제때 주소 변환을 거쳐 도착하는지** 확인해야 한다.

## 핵심 요약

- Day120의 절단 순서는 **buffer cache sync 확인 → heap/physical placement 고정 → IOMMU fault/TLB miss burst 분리 → 최종 addressability verdict 확정** 이다.
- 특히 GPU/codec/ISP가 생산한 dma-buf를 display가 바로 소비하는 경로에서는 bandwidth보다 **cache clean 누락, non-contiguous heap 선택, map churn** 이 더 자주 원인이 된다.
- `DMA_BUF_SYNC_TRACE`, `HEAP_PLACE_TRACE`, `IOMMU_TLB_TRACE`, `FINAL_ADDR_VERDICT` 를 같은 frame 번호로 묶어야 수정 owner를 자를 수 있다.
- 최소 증적은 `producer_done`, `cache_clean_done`, `sg_nents`, `tlb_miss_burst`, `fault_iova` 다섯 축이면 충분하다.

## 코드 포인트

1. **producer 종료 후 display acquire 직전 cache clean/invalidate가 실제로 끝났는지 먼저 본다**

   ```text
   DMA_BUF_SYNC_TRACE buf=0x91a300 frame=7124 producer=gpu clean_done=1 invalidate_done=1 sync_latency_us=38
   DMA_BUF_SYNC_TRACE buf=0x91a300 frame=7125 producer=isp clean_done=0 invalidate_done=1 sync_latency_us=4
   ```

   `clean_done=0` 인 frame에서만 corruption이 보이면 panel이나 QoS보다 **cache maintenance ownership 누락** 이 더 강하다.

2. **scanout 대상 버퍼가 어떤 heap에서 왔는지와 physical 분절도를 고정한다**

   ```text
   HEAP_PLACE_TRACE buf=0x91a300 frame=7125 heap=system uncached=0 sg_nents=96 contiguous=0 size_kb=8192
   HEAP_PLACE_TRACE buf=0x91a340 frame=7126 heap=cma uncached=1 sg_nents=1 contiguous=1 size_kb=8192
   ```

   같은 해상도인데 `system` heap에서만 `sg_nents` 가 과도하게 크면 display fetch보다 **heap placement 정책** 을 먼저 수정하는 편이 빠르다.

3. **IOMMU fault가 없더라도 TLB miss burst가 frame budget을 깨는지 분리한다**

   ```text
   IOMMU_TLB_TRACE domain=disp frame=7125 tlb_miss_burst=184 walk_latency_us=267 fault_iova=0x0 map_churn=12
   IOMMU_TLB_TRACE domain=disp frame=7126 tlb_miss_burst=9 walk_latency_us=21 fault_iova=0x0 map_churn=1
   ```

   `fault_iova=0x0` 이어도 `tlb_miss_burst` 와 `walk_latency_us` 가 한 frame에만 튀면 hard fault가 아니라 **translation pressure 기반 under-service** 다.

4. **map/unmap churn이 producer lifecycle과 같이 흔들리는지 본다**

   ```text
   MAP_CHURN_TRACE frame=7125 buf_realloc=1 import_count=3 unmap_before_present=1 retained_map=0
   ```

   launch/resize/rotation 순간에 `buf_realloc=1` 과 `unmap_before_present=1` 이 반복되면 steady-state 정상이어도 **first few frames** 만 깨질 수 있다.

5. **최종 verdict를 addressability owner 기준으로 고정한다**

   ```text
   FINAL_ADDR_VERDICT display=2 frame=7125 cause=cache_clean_missing owner=producer_sync symptom=stale_line
   FINAL_ADDR_VERDICT display=2 frame=7125 cause=tlb_miss_burst owner=iommu_mapping symptom=black_flash
   ```

   이 verdict가 있어야 Day119의 bandwidth 문제와 Day120의 **buffer visibility / translation 문제** 를 다른 수정 경로로 분리할 수 있다.

## 리스크

- 일부 드라이버는 cache sync trace를 submit 시점에만 남겨 실제 HW clean completion 시점과 어긋날 수 있다.
- `sg_nents` 가 많아도 prefetch나 large-page mapping이 잘 되면 문제가 안 생길 수 있어, heap만 보고 성급히 결론 내리면 오판한다.
- IOMMU fault가 없다는 이유로 주소 변환 경로를 제외하면 안 된다. TLB walk latency만으로도 single-frame artifact는 충분히 난다.
- CMA로 강제 고정하면 증상은 사라져도 메모리 파편화/장시간 안정성/다른 도메인 할당 실패가 새 리스크가 될 수 있다.

## 다음 액션

다음 글에서는 Day120 다음 절단면으로, **cache/heap/IOMMU까지 정상인데도 남는 경우 fence ordering·prefetch window·plane fetch watermark 경로를 어떻게 자를지** 정리하겠다.

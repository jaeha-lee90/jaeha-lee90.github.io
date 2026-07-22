---
title: R사 X5H Day124 - gralloc cacheability·CPU map/unmap·explicit sync cutline
author: JaeHa
date: 2026-07-23 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day124, display, gralloc, cache, sync, dmabuf, bringup]
---

Day123에서 `ALLOC_TRACE`, `MAPPER_LAYOUT_TRACE`, `IMPORT_NEGOTIATION_TRACE` 까지 정상인데도 **첫 frame stale image / partial corruption / CPU overlay 흔적 잔류** 가 남는다면 다음 절단면은 `gralloc cacheability`, `CPU map/unmap`, `explicit sync` 다. 즉 버퍼 format과 import 계약은 맞지만 **누가 언제 cache를 flush/invalidate 했는지, 그리고 그 시점을 fence로 정말 봉인했는지** 잘라야 한다.

## 핵심 요약

- Day124의 절단 순서는 **buffer cache policy 고정 → CPU map/unmap 구간 확인 → dma-buf sync/fence 선후관계 비교 → 최종 coherency verdict 확정** 이다.
- 특정 layer만 한 frame 늦게 보이거나 CPU annotation/UI overlay 흔적이 섞이면 layout mismatch보다 **cache coherency 또는 explicit sync 누락** 가능성이 더 높다.
- `CACHE_POLICY_TRACE`, `CPU_MAP_TRACE`, `DMA_BUF_SYNC_TRACE`, `EXPLICIT_SYNC_TRACE`, `FINAL_COHERENCY_VERDICT` 를 같은 `buffer_id` 와 `frame` 으로 묶어야 owner를 자를 수 있다.
- 최소 증적은 `cacheable`, `map_owner`, `end_cpu_access`, `acquire_fence`, `invalidate_before_scanout` 다섯 축이면 충분하다.

## 코드 포인트

1. **버퍼가 어떤 cache policy로 할당됐는지 먼저 고정한다**

   ```text
   CACHE_POLICY_TRACE buffer_id=0xa144 heap=system_uncached cacheable=0 cpu_access=rare producer=cpu_overlay
   CACHE_POLICY_TRACE buffer_id=0xa145 heap=system cacheable=1 cpu_access=frequent producer=cpu_overlay
   ```

   같은 producer인데 `cacheable` 정책이 다르면 같은 draw path라도 one-frame stale 여부가 달라진다.

2. **CPU map/unmap 구간이 frame 경계 밖으로 새지 않는지 본다**

   ```text
   CPU_MAP_TRACE buffer_id=0xa145 frame=7501 owner=cpu_overlay begin_cpu_access=1 end_cpu_access=0 map_us=1843 dirty_rect=0,0,320,64
   CPU_MAP_TRACE buffer_id=0xa145 frame=7502 owner=cpu_overlay begin_cpu_access=1 end_cpu_access=1 map_us=411 dirty_rect=0,0,320,64
   ```

   `end_cpu_access=0` 인 채 scanout으로 넘어가면 import는 성공해도 display는 **이전 cache line** 을 읽을 수 있다.

3. **dma-buf sync 호출이 실제 cache flush/invalidate를 동반했는지 분리한다**

   ```text
   DMA_BUF_SYNC_TRACE buffer_id=0xa145 frame=7501 start_cpu_access=1 end_cpu_access=1 flush_done=0 invalidate_before_scanout=0
   DMA_BUF_SYNC_TRACE buffer_id=0xa145 frame=7502 start_cpu_access=1 end_cpu_access=1 flush_done=1 invalidate_before_scanout=1
   ```

   `flush_done=0` 또는 `invalidate_before_scanout=0` 이면 symptom이 partial corruption이어도 본질은 **coherency contract 미완료** 다.

4. **flush 완료보다 acquire fence signal이 먼저 오르는 race를 자른다**

   ```text
   EXPLICIT_SYNC_TRACE buffer_id=0xa145 frame=7501 flush_complete_us=1487 acquire_fence_signal_us=1332 hwc_latch_us=1510 stale_cache_window_us=155
   ```

   `acquire_fence_signal_us < flush_complete_us` 이면 sync는 있어 보여도 실제로는 **flush 이전 버퍼를 latch** 한 것이다.

5. **최종 verdict를 coherency owner 기준으로 고정한다**

   ```text
   FINAL_COHERENCY_VERDICT display=2 frame=7501 cause=missing_end_cpu_access owner=cpu_producer symptom=stale_overlay
   FINAL_COHERENCY_VERDICT display=2 frame=7501 cause=flush_fence_race owner=sync_bridge symptom=partial_corruption
   ```

   이 verdict가 있어야 Day123의 allocation/import 이슈와 Day124의 **cache/sync 이슈** 를 다른 수정 경로로 분리할 수 있다.

## 리스크

- 일부 gralloc 구현은 `dma_buf_begin/end_cpu_access` 를 wrapper로 숨겨 실제 flush 여부를 trace만으로 바로 못 볼 수 있다.
- uncached heap로 임시 우회하면 증상은 사라져도 bandwidth·CPU cost가 급증해 장기 해법을 흐릴 수 있다.
- Xen/virt 경계에서는 DomA와 DomD가 서로 다른 cache domain을 써 한쪽 flush 로그만 보고 정상 판정하면 위험하다.
- dirty-rect 최적화가 켜져 있으면 부분 갱신 영역 밖의 stale pixel이 재현성을 낮춰 root cause를 가릴 수 있다.

## 다음 액션

다음 글에서는 Day124 다음 절단면으로, **cache coherency까지 정상인데도 남는 경우 secure/protected buffer 속성과 display plane capability mismatch가 왜 black frame이나 silent GPU fallback으로 이어지는지** 정리하겠다.

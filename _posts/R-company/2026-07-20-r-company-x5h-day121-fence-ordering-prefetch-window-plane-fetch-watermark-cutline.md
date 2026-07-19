---
title: R사 X5H Day121 - fence ordering·prefetch window·plane fetch watermark cutline
author: JaeHa
date: 2026-07-20 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day121, display, fence, prefetch, watermark, underflow, bringup]
---

Day120에서 `DMA_BUF_SYNC_TRACE`, `HEAP_PLACE_TRACE`, `IOMMU_TLB_TRACE` 까지 정상인데도 첫 전환 frame, overlay 재배치, 고해상도 UI reveal 순간에만 **single-frame black flash / partial fetch / line pop** 이 남는다면 다음 절단면은 `fence ordering`, `prefetch window`, `plane fetch watermark` 다. 즉 버퍼 자체는 올바르지만 **display가 필요한 시점보다 늦게 읽기 시작하거나 안전 여유 없이 scanout에 진입하는지** 확인해야 한다.

## 핵심 요약

- Day121의 절단 순서는 **acquire/present fence 선후관계 고정 → prefetch start margin 확인 → watermark/urgent threshold 비교 → 최종 fetch-timing verdict 확정** 이다.
- cache/IOMMU fault가 없는데도 증상이 첫 frame이나 layer topology 변경 직후에만 나오면, 대개 데이터 무결성보다 **fetch 시작 타이밍** 이 원인이다.
- `FENCE_ORDER_TRACE`, `PREFETCH_MARGIN_TRACE`, `FETCH_WATERMARK_TRACE`, `FINAL_FETCH_VERDICT` 를 같은 frame 번호로 묶어야 owner를 자를 수 있다.
- 최소 증적은 `acquire_to_kick_us`, `prefetch_lines`, `urgent_assert_line`, `scanline_at_data_ready` 네 축이면 충분하다.

## 코드 포인트

1. **acquire fence signal 이후에 실제 plane kick이 일어나는지 먼저 고정한다**

   ```text
   FENCE_ORDER_TRACE frame=7211 layer=primary acquire_signaled=1 acquire_to_kick_us=84 present_to_retire_us=16320 stale_fence_reused=0
   FENCE_ORDER_TRACE frame=7212 layer=primary acquire_signaled=1 acquire_to_kick_us=2410 present_to_retire_us=16655 stale_fence_reused=1
   ```

   `stale_fence_reused=1` 이거나 `acquire_to_kick_us` 가 한 frame budget을 크게 잠식하면 buffer 내용이 멀쩡해도 **늦은 fetch 시작** 으로 black flash가 난다.

2. **prefetch가 active 영역 진입 전에 충분히 선행하는지 본다**

   ```text
   PREFETCH_MARGIN_TRACE frame=7212 plane=0 prefetch_lines=18 required_lines=32 prefetch_start_us=97 before_active=0
   PREFETCH_MARGIN_TRACE frame=7213 plane=0 prefetch_lines=40 required_lines=32 prefetch_start_us=221 before_active=1
   ```

   `prefetch_lines < required_lines` 이 반복되면 bandwidth 총량과 무관하게 **첫 burst가 active scanline을 못 따라간다**.

3. **watermark/urgent threshold가 current mode와 layer mix에 맞는지 비교한다**

   ```text
   FETCH_WATERMARK_TRACE frame=7212 plane=0 fifo_lines=22 low_wm=24 urgent_assert_line=9 latency_budget_us=28
   FETCH_WATERMARK_TRACE frame=7213 plane=0 fifo_lines=44 low_wm=24 urgent_assert_line=21 latency_budget_us=51
   ```

   `fifo_lines < low_wm` 인데 `urgent_assert_line` 도 늦게 올라오면 문제는 memory corrupt가 아니라 **watermark tuning 부족** 이다.

4. **mode switch/overlay fallback 직후 watermark 재계산 누락을 자른다**

   ```text
   WM_RECALC_TRACE frame=7212 event=overlay_fallback mode=2560x1440@60 old_wm=16 new_wm=24 applied=0
   ```

   topology가 바뀌었는데 `applied=0` 이면 steady-state 정상이어도 전환 직후 한두 frame만 깨질 수 있다.

5. **최종 verdict를 fetch-timing owner 기준으로 고정한다**

   ```text
   FINAL_FETCH_VERDICT display=2 frame=7212 cause=stale_fence_reused owner=sync_path symptom=black_flash
   FINAL_FETCH_VERDICT display=2 frame=7212 cause=watermark_too_low owner=display_tuning symptom=line_pop
   ```

   이 verdict가 있어야 Day120의 addressability 이슈와 Day121의 **fetch scheduling/tuning 이슈** 를 다른 수정 경로로 분리할 수 있다.

## 리스크

- 일부 드라이버는 fence timestamp를 userspace 기준으로 남겨 실제 HW kick 시점과 수백 us 차이가 날 수 있다.
- prefetch line 계산은 pixel format, compression, rotation에 따라 달라져 단순 해상도 기준 비교만으로는 오판할 수 있다.
- watermark를 과하게 높이면 artifact는 줄어도 메모리 여유와 전력 효율이 급격히 나빠질 수 있다.
- overlay fallback, DSC on/off, VRR 전환이 섞이면 한 가지 trace만으로는 root cause를 단정하기 어렵다.

## 다음 액션

다음 글에서는 Day121 다음 절단면으로, **fetch timing까지 정상인데도 남는 경우 compression metadata·AFBC/UBWC header fetch·plane rotation/scaler line-buffer 경로를 어떻게 자를지** 정리하겠다.

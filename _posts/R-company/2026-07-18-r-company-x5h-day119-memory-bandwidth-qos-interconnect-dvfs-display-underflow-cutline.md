---
title: R사 X5H Day119 - memory bandwidth QoS·interconnect DVFS·display underflow telemetry cutline
author: JaeHa
date: 2026-07-18 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day119, display, memory-bandwidth, qos, dvfs, underflow, bringup]
---

Day118에서 `PSR_EXIT_TRACE`, `DSC_LINEBUF_TRACE`, `ESD_RECOVERY_TRACE` 까지 정상인데도 장시간 사용 뒤 특정 UI 전환이나 camera overlay 합성 순간에만 **짧은 frame drop, half-line tear, underflow 기반 black flash** 가 남는다면, 이제 절단면은 `memory bandwidth QoS`, `interconnect DVFS`, `display underflow telemetry` 다. 즉 panel 내부 복귀는 끝났지만 **display pipe가 필요한 시점에 메모리/NoC 대역폭을 제때 못 받는지** 확인해야 한다.

## 핵심 요약

- Day119의 절단 순서는 **underflow frame 고정 → QoS vote/latency 확인 → interconnect DVFS 상승 지연 분리 → 최종 bandwidth owner verdict 확정** 이다.
- 증상이 soak 후 첫 UI뿐 아니라 camera preview, map zoom, multi-layer animation처럼 **메모리 burst가 큰 장면** 에서만 반복되면 panel보다 memory path를 먼저 보는 편이 맞다.
- `underflow_cnt`, `rdma_stall_us`, `icc_bw_vote_mb`, `noc_freq_khz`, `dram_busy_pct` 를 같은 frame 축으로 남겨야 owner를 자를 수 있다.
- 최소 증적은 `DISP_UNDERFLOW_TRACE`, `QOS_VOTE_TRACE`, `ICC_DVFS_TRACE`, `FINAL_BW_VERDICT` 네 줄이면 충분하다.

## 코드 포인트

1. **실제 보이는 깜빡임 frame에 underflow가 동반되는지 먼저 고정한다**

   ```text
   DISP_UNDERFLOW_TRACE display=2 frame=6844 plane=primary underflow_cnt=1 rdma_stall_us=312 wb_crc_valid=0
   DISP_UNDERFLOW_TRACE display=2 frame=6845 plane=primary underflow_cnt=0 rdma_stall_us=18 wb_crc_valid=1
   ```

   `wb_crc_valid=0` 와 `underflow_cnt=1` 이 같은 frame에 찍히면 HWC 정책보다 **실제 fetch starvation** 쪽 owner가 더 강하다.

2. **display client의 QoS vote가 peak layer 구성에 맞게 올라가는지 본다**

   ```text
   QOS_VOTE_TRACE client=disp0 frame=6844 avg_bw_mb=420 peak_bw_mb=760 applied=1 latency_ns=180
   QOS_VOTE_TRACE client=disp0 frame=6845 avg_bw_mb=220 peak_bw_mb=280 applied=1 latency_ns=92
   ```

   camera+GPU+UI가 겹친 frame인데도 `peak_bw_mb` 가 low profile에 머물면 display가 스스로 필요한 대역폭을 못 요청한 것이다.

3. **interconnect DVFS가 vote는 받았지만 한 frame 늦게 따라오는지 자른다**

   ```text
   ICC_DVFS_TRACE path=disp-mem frame=6844 vote_mb=760 noc_freq_khz=533000 target_khz=800000 ramp_delay_us=410
   ICC_DVFS_TRACE path=disp-mem frame=6845 vote_mb=760 noc_freq_khz=800000 target_khz=800000 ramp_delay_us=0
   ```

   `ramp_delay_us` 가 한 frame budget보다 크면 steady-state는 정상이어도 burst 시작점에서 **first underflow** 가 난다.

4. **DRAM busy와 경쟁 masters를 같은 시점에 묶어 본다**

   ```text
   DRAM_ARB_TRACE frame=6844 dram_busy_pct=92 gpu_bw_mb=510 isp_bw_mb=640 disp_bw_mb=420 starvation_client=disp0
   ```

   `dram_busy_pct` 가 높아도 `starvation_client=disp0` 로 반복되면 단순 총량 부족보다 **arbiter 우선순위/QoS class 설정 미스** 가능성이 크다.

5. **최종 verdict를 bandwidth owner 기준으로 고정한다**

   ```text
   FINAL_BW_VERDICT display=2 frame=6844 cause=qos_peak_vote_low owner=disp_qos symptom=black_flash
   FINAL_BW_VERDICT display=2 frame=6844 cause=icc_ramp_late owner=interconnect_dvfs symptom=single_frame_underflow
   ```

   이 verdict가 있어야 Day118의 panel internal resume 이슈와 Day119의 **memory service latency 이슈** 를 다른 수정 경로로 분리할 수 있다.

## 리스크

- underflow counter는 display IP revision마다 샘플링 위치가 달라, fetch stall과 scanout miss를 같은 의미로 해석하면 오판할 수 있다.
- interconnect governor는 다른 도메인(camera, GPU, codec) 부하와 강하게 얽혀 있어 display 단독 재현 로그만으로는 root cause가 흐려질 수 있다.
- QoS를 과하게 올리면 artifact는 사라져도 전력/발열이 급증해 양산 세팅으로 유지하기 어렵다.
- WB CRC가 없는 SKU에서는 화면이 검게 보였는지, 단순 jank였는지 증적이 약해질 수 있다.

## 다음 액션

다음 글에서는 Day119 다음 절단면으로, **memory bandwidth 경로까지 정상인데도 특정 장면에서만 끊김이 남을 때 cache maintenance·dma-buf heap placement·IOMMU TLB pressure 경로를 어떻게 자를지** 정리하겠다.

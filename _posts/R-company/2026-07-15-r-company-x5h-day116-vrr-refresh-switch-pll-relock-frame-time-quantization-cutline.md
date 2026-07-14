---
title: R사 X5H Day116 - VRR/refresh switch·PLL relock·frame-time quantization transient jitter cutline
author: JaeHa
date: 2026-07-15 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day116, display, vrr, refresh-switch, pll, frame-time, jitter, bringup]
---

Day115에서 `TEMP_BIN_LATCH_TRACE`, `THERMAL_COMP_TRACE`, `VCOM_AGING_TRACE` 까지 정상인데도 특정 refresh 전환 직후에만 **미세 떨림, one-beat judder, 짧은 shimmer** 가 남는다면, 이제 절단면은 `VRR/refresh switch`, `PLL relock`, `frame-time quantization` 이다. 즉 panel PQ 보정은 안정적이지만 **주사율 재합의 순간 frame cadence가 한두 beat 어긋나는지** 확인해야 한다.

## 핵심 요약

- Day116의 절단 순서는 **refresh switch trigger 확인 → PLL relock window 계측 → frame-time quantization slip 분리 → transient jitter verdict 확정** 이다.
- 증상이 cold/warm과 무관하고 60↔90Hz, 60↔120Hz 같은 refresh 전환 경계에서만 나오면 panel thermal 경로보다 **timing re-lock 경로** 를 먼저 보는 편이 맞다.
- `refresh_from`, `refresh_to`, `pll_lock_us`, `vblank_period_us`, `frame_phase_error_us` 를 같은 frame 축으로 남겨야 미세 떨림의 owner를 자를 수 있다.
- 최소 증적은 `REFRESH_SWITCH_TRACE`, `PLL_RELOCK_TRACE`, `FRAME_TIME_QUANT_TRACE`, `FINAL_TRANSIENT_JITTER_VERDICT` 네 줄이면 충분하다.

## 코드 포인트

1. **refresh switch가 실제 first visible transition 직전/직후에 걸리는지 먼저 본다**

   ```text
   REFRESH_SWITCH_TRACE display=2 frame=6574 from_hz=60 to_hz=90 reason=idle_exit policy=seamless pending=1
   REFRESH_SWITCH_TRACE display=2 frame=6575 from_hz=60 to_hz=90 reason=idle_exit policy=seamless applied=1
   ```

   first UI reveal과 `applied=1` 이 겹치면 black screen 대신 **한 박자 밀린 scroll/judder** 로 체감될 수 있다.

2. **PLL relock 동안 실제 scan cadence가 비는 구간이 있는지 자른다**

   ```text
   PLL_RELOCK_TRACE display=2 frame=6575 pll=dsi0 lock_start_us=18422031 lock_end_us=18422388 relock_us=357 stable=1
   PLL_RELOCK_TRACE display=2 frame=6576 pll=dsi0 target_hz=90 measured_vblank_us=11342 stable=0
   ```

   `relock_us` 가 한 vblank budget을 잠식하면 frame drop은 없어도 **phase wobble** 이 생길 수 있다.

3. **frame-time quantization으로 한 frame이 길거나 짧아지는지 확인한다**

   ```text
   FRAME_TIME_QUANT_TRACE display=2 frame=6576 target_period_us=11111 actual_period_us=11840 phase_error_us=729 repeat=0
   FRAME_TIME_QUANT_TRACE display=2 frame=6577 target_period_us=11111 actual_period_us=10392 phase_error_us=-719 repeat=0
   ```

   평균 FPS는 맞아도 `+/-700us` 급 오차가 두 beat 연속 나오면 사용자는 이를 **짧은 떨림이나 shimmer** 로 본다.

4. **TE/VBLANK 기준으로 source와 sink의 phase mismatch를 묶어 본다**

   ```text
   PHASE_ALIGN_TRACE display=2 frame=6576 sf_present_us=18422310 hwc_flip_us=18422388 te_edge_us=18422441 phase_to_te_us=131
   ```

   SurfaceFlinger present는 정상인데 `phase_to_te_us` 만 전환 직후 튀면 앱/합성보다 **panel timing relock** 쪽 owner가 맞다.

5. **최종 verdict를 refresh transition owner 기준으로 고정한다**

   ```text
   FINAL_TRANSIENT_JITTER_VERDICT display=2 frame=6576 cause=pll_relock_over_budget owner=panel_timing symptom=one_beat_judder
   FINAL_TRANSIENT_JITTER_VERDICT display=2 frame=6576 cause=frame_period_quantization owner=display_scheduler symptom=transient_micro_shimmer
   ```

   이 verdict가 있어야 Day115의 thermal stabilization 이슈와 Day116의 **refresh transition cadence 이슈** 를 다른 수정 경로로 분리할 수 있다.

## 리스크

- seamless switch처럼 보여도 내부적으로는 partial relock이 발생할 수 있어, FPS 로그만 보면 정상으로 오판하기 쉽다.
- VRR policy와 idle exit policy가 동시에 바뀌면 scheduler 문제와 panel timing 문제를 섞어 해석할 수 있다.
- capture 장비가 고정 60fps면 90/120Hz 전환 떨림을 aliasing으로 잘못 읽을 수 있다.
- 특정 panel SKU는 PLL relock 시간 편차가 커서 EVT/DVT/양산 간 재현 조건이 달라질 수 있다.

## 다음 액션

다음 글에서는 Day116 다음 절단면으로, **refresh switch cadence까지 정상인데도 특정 밝기/패턴에서만 미세 수평 노이즈가 남을 때 spread-spectrum clocking·EMI margin·lane deskew 경로를 어떻게 자를지** 정리하겠다.

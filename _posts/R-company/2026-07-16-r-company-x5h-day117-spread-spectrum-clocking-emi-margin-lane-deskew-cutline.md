---
title: R사 X5H Day117 - spread-spectrum clocking·EMI margin·lane deskew transient horizontal noise cutline
author: JaeHa
date: 2026-07-16 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day117, display, ssc, emi, lane-deskew, noise, bringup]
---

Day116에서 `REFRESH_SWITCH_TRACE`, `PLL_RELOCK_TRACE`, `FRAME_TIME_QUANT_TRACE` 까지 정상인데도 특정 밝기 패턴이나 고주파 edge에서만 **짧은 수평 노이즈, fine shimmer, line-level sparkle** 이 남는다면, 이제 절단면은 `spread-spectrum clocking`, `EMI margin`, `lane deskew` 다. 즉 frame cadence 자체는 맞지만 **DSI/PHY 링크가 margin 경계에서 간헐적으로 흔들리는지** 확인해야 한다.

## 핵심 요약

- Day117의 절단 순서는 **SSC enable/ratio 확인 → EMI/noise window 상관관계 확인 → lane deskew/error counter 분리 → transient horizontal noise verdict 확정** 이다.
- 증상이 refresh switch와 무관하고 특정 패턴(세로선, 저채도 그라데이션, 고휘도 edge)에서만 보이면 scheduler보다 **PHY signal integrity 경로** 를 먼저 보는 편이 맞다.
- `ssc_ppm`, `lane_err_crc`, `deskew_step`, `hs_settle`, `noise_window_us` 를 같은 frame 축으로 남겨야 owner를 자를 수 있다.
- 최소 증적은 `SSC_TRACE`, `EMI_MARGIN_TRACE`, `LANE_DESKEW_TRACE`, `FINAL_HORIZONTAL_NOISE_VERDICT` 네 줄이면 충분하다.

## 코드 포인트

1. **SSC 설정이 boot default와 panel 권장값 사이에서 흔들리지 않는지 먼저 본다**

   ```text
   SSC_TRACE display=2 frame=6642 pll=dsi0 enable=1 spread_ppm=450 profile=panel_nominal applied=1
   SSC_TRACE display=2 frame=6643 pll=dsi0 enable=1 spread_ppm=300 profile=fallback_safe override=0
   ```

   `spread_ppm` 이 panel 권장값보다 좁거나 fallback profile로 재진입하면 평균 링크는 살아 있어도 특정 패턴에서 **line shimmer** 가 보일 수 있다.

2. **노이즈가 실제 EMI margin 저점과 겹치는지 확인한다**

   ```text
   EMI_MARGIN_TRACE display=2 frame=6645 lane_clk_mhz=742 hs_settle=93 margin_mv=47 noise_window_us=380 crc_burst=1
   EMI_MARGIN_TRACE display=2 frame=6646 lane_clk_mhz=742 hs_settle=93 margin_mv=64 noise_window_us=0 crc_burst=0
   ```

   `margin_mv` 가 특정 threshold 아래로 떨어지는 frame에서만 `crc_burst=1` 이 나오면 합성 문제가 아니라 **링크 마진 부족** 으로 보는 게 맞다.

3. **lane deskew가 전환 직후 재학습되거나 lane별 편차가 큰지 자른다**

   ```text
   LANE_DESKEW_TRACE display=2 frame=6645 lane0=2 lane1=2 lane2=5 lane3=6 retrain=1 skew_delta_ui=4
   LANE_DESKEW_TRACE display=2 frame=6646 lane0=2 lane1=2 lane2=2 lane3=3 retrain=0 skew_delta_ui=1
   ```

   특정 lane만 `skew_delta_ui` 가 크게 튀면 패널 전체 black-out 대신 **얇은 수평 잡음이나 edge sparkle** 로 나타날 수 있다.

4. **ECC/CRC counter를 line artifact 시점과 같은 축에 묶는다**

   ```text
   DSI_LINK_ERR_TRACE display=2 frame=6645 ecc_corr=3 crc_err=1 timeout=0 lane=3 region=upper_mid
   ```

   `ecc_corr` 가 누적만 증가하는지, 실제 artifact가 나온 frame에 `crc_err` 가 동반되는지 구분해야 한다. 전자가 아니라 후자면 panel content보다 **PHY ingress integrity** 쪽 owner가 더 강하다.

5. **최종 verdict를 PHY owner 기준으로 고정한다**

   ```text
   FINAL_HORIZONTAL_NOISE_VERDICT display=2 frame=6645 cause=emi_margin_low owner=dsi_phy symptom=line_sparkle
   FINAL_HORIZONTAL_NOISE_VERDICT display=2 frame=6645 cause=lane3_deskew_retrain owner=phy_training symptom=horizontal_noise_burst
   ```

   이 verdict가 있어야 Day116의 cadence 문제와 Day117의 **signal integrity 문제** 를 다른 수정 경로로 분리할 수 있다.

## 리스크

- 오실로스코프/패널 캡처 장비 접지 상태가 나쁘면 실제 제품 EMI 이슈보다 더 크게 보일 수 있다.
- SSC 값 변경은 EMI에는 유리해도 panel 호환성이나 인증 마진을 다시 검증해야 한다.
- lane deskew retrain은 온도, 케이블 편차, 패널 SKU 차이 영향을 크게 받아 EVT/DVT/양산에서 재현성이 달라질 수 있다.
- ECC corrected error만 보고 정상으로 넘기면 field에서 간헐적 sparkle complaints가 남을 수 있다.

## 다음 액션

다음 글에서는 Day117 다음 절단면으로, **PHY margin까지 정상인데도 장시간 soak 후만 artifact가 남을 때 panel self-refresh exit retention·DSC line-buffer underflow·ESD recovery 경로를 어떻게 자를지** 정리하겠다.

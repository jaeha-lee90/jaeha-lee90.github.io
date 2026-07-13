---
title: R사 X5H Day115 - thermal compensation·VCOM/aging LUT·temperature bin latch cold-boot flicker cutline
author: JaeHa
date: 2026-07-14 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day115, display, thermal, vcom, aging-lut, temperature-bin, flicker, bringup]
---

Day114에서 `OVERDRIVE_TRACE`, `RESPONSE_COMP_TRACE`, `PANEL_MEMCLR_TRACE` 까지 정상인데도 cold boot 직후 첫 수초 동안만 **미세한 flicker나 밝기 breathing이 남는다면**, 이제 절단면은 `thermal compensation`, `VCOM/aging LUT`, `temperature bin latch` 다. 즉 첫 visible frame 자체는 맞지만 **panel이 온도/노화 보정 테이블을 안정적으로 물기 전 과도구간** 이 사용자에게 깜빡임으로 보이는지 확인해야 한다.

## 핵심 요약

- Day115의 절단 순서는 **temperature bin latch 시점 확인 → thermal compensation enable 분리 → VCOM/aging LUT 안정화 확인 → cold-boot flicker verdict 확정** 이다.
- flicker가 cold boot 초반 수초에만 나오고 warm reboot에서는 사라지면, 합성 파이프보다 **panel PQ 보정 테이블 warm-up 경로** 를 먼저 의심하는 편이 맞다.
- `temp_bin`, `thermal_comp_state`, `vcom_profile`, `aging_lut_id`, `settle_ms` 를 같은 시간축으로 남겨야 원인을 자를 수 있다.
- 최소 증적은 `TEMP_BIN_LATCH_TRACE`, `THERMAL_COMP_TRACE`, `VCOM_AGING_TRACE`, `FINAL_COLD_BOOT_FLICKER_VERDICT` 네 줄이면 충분하다.

## 코드 포인트

1. **temperature bin이 첫 UI 이후 늦게 latch되는지 먼저 본다**

   ```text
   TEMP_BIN_LATCH_TRACE display=2 frame=6490 temp_c=18.4 bin=boot_default latched=0
   TEMP_BIN_LATCH_TRACE display=2 frame=6518 temp_c=18.7 bin=cold latched=1
   ```

   `boot_default` 상태로 몇 frame 진행하다가 뒤늦게 `bin=cold` 로 바뀌면 사용자는 첫 수초 동안 **주기적 luminance 흔들림** 으로 느낄 수 있다.

2. **thermal compensation이 panel enable 직후 step-change로 들어오는지 분리한다**

   ```text
   THERMAL_COMP_TRACE display=2 frame=6492 enable=0 gain=1.00 slope=0 settle_ms=0
   THERMAL_COMP_TRACE display=2 frame=6520 enable=1 gain=1.08 slope=3 settle_ms=180
   ```

   `enable=0 -> 1` 전환이 visible 상태에서 일어나면 실제 픽셀 데이터는 같아도 **panel drive gain 변화** 때문에 전체 화면이 미세하게 숨 쉬듯 흔들린다.

3. **VCOM/aging LUT가 boot fallback에서 패널 실측값으로 재적용되는지 본다**

   ```text
   VCOM_AGING_TRACE display=2 frame=6491 vcom=boot_default aging_lut=none crc=0x00 apply=0
   VCOM_AGING_TRACE display=2 frame=6521 vcom=trim_07 aging_lut=panel_nvm_03 crc=0x8ab4 apply=1
   ```

   `boot_default` 로 올라왔다가 뒤늦게 panel NVM 기반 LUT를 적용하면 저휘도 회색 영역에서 **fine flicker 또는 edge shimmer** 가 나타날 수 있다.

4. **시간 조건을 묶어 cold-only 현상인지 확정한다**

   ```text
   THERMAL_WINDOW_TRACE display=2 since_unblank_ms=2400 flicker=1 temp_bin=cold warm_reboot=0
   THERMAL_WINDOW_TRACE display=2 since_unblank_ms=2400 flicker=0 temp_bin=nominal warm_reboot=1
   ```

   cold boot에서만 재현되고 warm reboot에서는 사라지면 framework race보다 **panel thermal stabilization** 쪽으로 무게를 실을 수 있다.

5. **최종 verdict를 보정 owner 기준으로 고정한다**

   ```text
   FINAL_COLD_BOOT_FLICKER_VERDICT display=2 frame=6521 cause=temp_bin_late_latch owner=panel_pq symptom=global_breathing_flicker
   FINAL_COLD_BOOT_FLICKER_VERDICT display=2 frame=6521 cause=vcom_aging_lut_late_apply owner=panel_nvram symptom=low_gray_shimmer
   ```

   이 verdict가 있어야 Day114의 response history 잔류 문제와 Day115의 **온도/노화 보정 안정화 문제** 를 다른 수정 경로로 분리할 수 있다.

## 리스크

- temperature sensor polling 주기가 길면 실제 panel 온도보다 늦은 값이 들어와 root cause를 흐릴 수 있다.
- thermal compensation과 backlight/ACL이 동시에 움직이면 flicker와 밝기 점프가 섞여 보이므로 Day111/Day112 이슈와 혼동되기 쉽다.
- panel NVM read fallback이 SKU별로 다르면 EVT/DVT/양산 패널 간 재현 조건이 달라질 수 있다.
- VCOM trim 적용 실패를 로그로만 보면 정상처럼 보여도, 실제로는 이전 profile이 유지돼 저휘도 flicker가 남을 수 있다.

## 다음 액션

다음 글에서는 Day115 다음 절단면으로, **thermal/VCOM 안정화까지 정상인데도 특정 주사율 전환 직후만 미세 떨림이 남을 때 VRR/refresh switch·PLL relock·frame-time quantization 경로를 어떻게 자를지** 정리하겠다.

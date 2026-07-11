---
title: R사 X5H Day113 - dither·FRC·DSC restore timing layer-boundary flash/contour break cutline
author: JaeHa
date: 2026-07-12 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day113, display, dither, frc, dsc, contour, bringup]
---

Day112에서 `PANEL_GAMMA_TRACE`, `ACL_TRANSITION_TRACE`, `TONEMAP_TRANSITION_TRACE` 까지 정상인데도 첫 UI 직후 **특정 레이어 경계만 번쩍이거나 gradient contour가 한 박자 깨지는 현상** 이 남는다면, 이제 절단면은 `panel dither`, `FRC(frame rate control)`, `DSC restore timing` 이다. 즉 색정책 자체는 안정적이지만 **panel 후단 비트심도 복원 체인** 이 first visible frame과 엇갈리는지 확인해야 한다.

## 핵심 요약

- Day113의 절단 순서는 **dither enable 시점 확인 → FRC phase lock 분리 → DSC PPS/decoder restore 확인 → boundary flash verdict 고정** 이다.
- full-screen screenshot이 정상인데 일부 경계나 gradient만 깨지면, composition보다 **panel bit-depth reconstruction path** 를 먼저 의심해야 한다.
- `dither_mode`, `frc_phase`, `dsc_pps_seq`, `slice_lock` 를 같은 frame 축으로 남겨야 contour break를 one-beat 단위로 자를 수 있다.
- 최소 증적은 `PANEL_DITHER_TRACE`, `FRC_PHASE_TRACE`, `DSC_RESTORE_TRACE`, `FINAL_BOUNDARY_FLASH_VERDICT` 네 줄이면 충분하다.

## 코드 포인트

1. **first visible frame 전후로 panel dither bank가 늦게 enable되는지 본다**

   ```text
   PANEL_DITHER_TRACE display=2 frame=6420 mode=bypass depth_in=10 depth_out=8 bank=boot seed=0x00
   PANEL_DITHER_TRACE display=2 frame=6421 mode=spatial_temporal depth_in=10 depth_out=8 bank=ui seed=0x31
   ```

   `mode=bypass -> spatial_temporal` 전환이 UI 노출 직후 들어오면 black frame이 아니라 **경계 flash/gradient contour 변화** 로 보일 수 있다.

2. **FRC phase가 첫 UI frame과 lock을 맞추는지 고정한다**

   ```text
   FRC_PHASE_TRACE display=2 frame=6420 frc_enable=1 phase=warmup cadence=2x lock=0
   FRC_PHASE_TRACE display=2 frame=6421 frc_enable=1 phase=steady cadence=2x lock=1
   ```

   `lock=0` 상태에서 한 beat 보이면 동일 색면도 미세하게 출렁이거나 contour가 갈라져 보일 수 있다.

3. **DSC decoder restore가 slice 단위로 늦게 안정화되는지 분리한다**

   ```text
   DSC_RESTORE_TRACE display=2 frame=6420 pps_seq=boot slice_lock=0 rc_model=stale
   DSC_RESTORE_TRACE display=2 frame=6421 pps_seq=ui slice_lock=1 rc_model=latched
   ```

   `slice_lock=0` 이면 전체 black screen 대신 **특정 세로 경계/블록만 번쩍이는 현상** 으로 나타날 수 있다.

4. **screenshot 정상 여부와 panel bit-depth restore 상태를 한 줄로 묶는다**

   ```text
   SCREENSHOT_PANEL_DEPTH_COMPARE display=2 frame=6421 screenshot=stable panel=edge_flash dither=ui frc_lock=0 dsc_lock=1
   ```

   screenshot이 정상인데 panel만 깨지면 앱/UI 수정으로 새지 말고 panel 후단 복원 체인으로 바로 좁혀야 한다.

5. **최종 verdict를 어떤 복원 블록이 늦었는지 기준으로 고정한다**

   ```text
   FINAL_BOUNDARY_FLASH_VERDICT display=2 frame=6421 cause=dither_bank_late owner=panel_pq_restore symptom=edge_flash_on_launcher_icons
   FINAL_BOUNDARY_FLASH_VERDICT display=2 frame=6421 cause=dsc_slice_unlock owner=dsi_dsc_restore symptom=vertical_contour_break
   ```

   이 verdict가 있어야 Day112의 gamma/ACL/tonemap 전환 문제와 Day113의 **bit-depth restore timing 문제** 를 서로 다른 수정 owner로 분리할 수 있다.

## 리스크

- screenshot이 정상이라는 이유만으로 app layer 문제를 배제하지 못하면, 실제 panel 후단 복원 지연을 놓치기 쉽다.
- FRC는 정지 화면보다 low-motion UI에서 더 눈에 띄므로 아이콘 reveal/gradient 배경 기준 재현 시퀀스를 고정해야 한다.
- DSC restore는 slice 수, lane rate, panel SKU에 따라 증상이 달라 panel id와 PPS 버전을 함께 남겨야 한다.
- dither/FRC/DSC 전환이 동시에 일어나면 증상이 비슷해 보여도 수정 지점은 panel PQ, bridge, DSI host로 갈라질 수 있다.

## 다음 액션

다음 글에서는 Day113 다음 절단면으로, **dither/FRC/DSC까지 정상인데도 첫 UI 순간 특정 영역만 잔상처럼 남을 때 overdrive·response compensation·panel memory clear 경로를 어떻게 자를지** 정리하겠다.

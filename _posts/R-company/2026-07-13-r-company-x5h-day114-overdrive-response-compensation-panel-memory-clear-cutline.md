---
title: R사 X5H Day114 - overdrive·response compensation·panel memory clear residual image cutline
author: JaeHa
date: 2026-07-13 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day114, display, overdrive, response-compensation, panel-memory, ghost, bringup]
---

Day113에서 `PANEL_DITHER_TRACE`, `FRC_PHASE_TRACE`, `DSC_RESTORE_TRACE` 까지 정상인데도 첫 UI 순간 **특정 영역만 잔상처럼 남거나 아이콘 외곽이 한 프레임 더 오래 붙어 보이면**, 절단면은 이제 `overdrive`, `response compensation`, `panel memory clear` 다. 즉 전송/합성은 이미 정상인데 **panel 내부 이전 frame 보정값 또는 잔류 메모리 상태** 가 첫 visible frame과 충돌하는지 봐야 한다.

## 핵심 요약

- Day114의 절단 순서는 **overdrive LUT 적용 시점 확인 → response compensation warm-start 분리 → panel memory clear 완료 여부 고정 → residual image verdict 확정** 이다.
- screenshot과 writeback CRC가 정상인데 사람 눈에만 잔상이 보이면, display pipe 앞단보다 **panel 내부 응답보정/메모리 잔류 경로** 를 먼저 의심해야 한다.
- `od_lut_id`, `rc_enable`, `mem_clear_done`, `residual_zone` 를 같은 frame 축으로 남겨야 one-frame ghost를 잘라낼 수 있다.
- 최소 증적은 `OVERDRIVE_TRACE`, `RESPONSE_COMP_TRACE`, `PANEL_MEMCLR_TRACE`, `FINAL_RESIDUAL_IMAGE_VERDICT` 네 줄이면 충분하다.

## 코드 포인트

1. **첫 UI frame 직전 overdrive LUT가 boot용 값에서 UI용 값으로 전환되는지 본다**

   ```text
   OVERDRIVE_TRACE display=2 frame=6488 od_enable=1 lut=boot boost=high temp_bin=cold
   OVERDRIVE_TRACE display=2 frame=6489 od_enable=1 lut=ui boost=nominal temp_bin=cold
   ```

   `lut=boot -> ui` 전환이 first reveal과 겹치면 black frame 대신 **밝은 halo나 edge ghost** 로 보일 수 있다.

2. **response compensation state가 warm-start 없이 stale history를 재사용하는지 자른다**

   ```text
   RESPONSE_COMP_TRACE display=2 frame=6488 rc_enable=1 history=boot preload=1 settle=0
   RESPONSE_COMP_TRACE display=2 frame=6489 rc_enable=1 history=ui preload=0 settle=1
   ```

   `history=boot` 가 첫 UI frame까지 남아 있으면 이전 패턴 기준 보정이 남아 특정 아이콘/텍스트 외곽에 잔상이 맺힌다.

3. **panel memory clear가 실제 scanout 전에 끝났는지 확인한다**

   ```text
   PANEL_MEMCLR_TRACE display=2 frame=6488 op=trigger window=full reason=unblank_exit done=0
   PANEL_MEMCLR_TRACE display=2 frame=6489 op=complete window=full reason=unblank_exit done=1
   ```

   `done=0` 인 상태에서 UI가 노출되면 전체 black screen은 아니어도 **특정 zone만 이전 frame 흔적이 남는 현상** 으로 이어진다.

4. **panel 관측값과 pipeline 관측값을 한 줄로 묶어 app 쪽 오해를 차단한다**

   ```text
   PIPELINE_PANEL_COMPARE display=2 frame=6489 screenshot=clean crc=stable panel=ghost_zone residual_zone=right_top od_lut=ui memclr=0
   ```

   screenshot/CRC는 정상인데 panel만 ghost가 보이면 SurfaceFlinger/HWC보다 panel 보정 체인 쪽 owner가 맞다.

5. **최종 verdict를 잔상 원인별 owner로 고정한다**

   ```text
   FINAL_RESIDUAL_IMAGE_VERDICT display=2 frame=6489 cause=stale_overdrive_lut owner=panel_pq symptom=bright_edge_ghost
   FINAL_RESIDUAL_IMAGE_VERDICT display=2 frame=6489 cause=mem_clear_incomplete owner=panel_tcon symptom=localized_previous_frame_residue
   ```

   이렇게 남겨야 Day113의 bit-depth restore 이슈와 Day114의 **panel response history 잔류 이슈** 를 별도 수정 과제로 분리할 수 있다.

## 리스크

- 잔상은 저온/고휘도에서 더 두드러지므로 온도 bin과 brightness 상태를 같이 기록하지 않으면 재현성이 흔들린다.
- overdrive와 local dimming transition이 겹치면 증상이 비슷해 보여 panel PQ 문제와 backlight 문제를 섞어 해석하기 쉽다.
- memory clear 범위가 full-screen이 아니라 partial zone이면 특정 코너에서만 재현돼 앱 레이아웃 문제로 오판할 수 있다.
- response compensation history reset이 panel SKU별로 다르면 동일 코드라도 양산 패널과 EVT 패널의 증상이 다르게 나타난다.

## 다음 액션

다음 글에서는 Day114 다음 절단면으로, **panel response/memory clear까지 정상인데도 cold boot 직후 첫 수초 동안만 flicker가 남을 때 thermal compensation·VCOM/aging LUT·temperature bin latch 경로를 어떻게 자를지** 정리하겠다.

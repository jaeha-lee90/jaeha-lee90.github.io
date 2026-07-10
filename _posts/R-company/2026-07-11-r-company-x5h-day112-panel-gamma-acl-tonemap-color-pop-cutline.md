---
title: R사 X5H Day112 - panel gamma·ACL·tonemap transition final color-pop cutline
author: JaeHa
date: 2026-07-11 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day112, display, gamma, acl, tonemap, color, bringup]
---

Day111에서 `BRIGHTNESS_OWNER_TRACE`, `LOCAL_DIMMING_RELEASE_TRACE`, `ALS_CABC_OVERRIDE_TRACE` 까지 정상인데도 첫 홈 진입 순간 화면이 **유난히 쨍해지거나 색감이 한 박자 뒤에 바뀌는 현상** 이 남는다면, 이제 절단면은 `panel gamma bank`, `ACL`, `tonemap/HDR bypass 해제` 다. 즉 backlight 값은 안정적이지만 **panel 후단 화질 테이블이 first visible frame 직후 교체되는지** 확인해야 한다.

## 핵심 요약

- Day112의 절단 순서는 **gamma/degamma bank 전환 확인 → ACL 개입 시점 분리 → tonemap/HDR bypass 해제 확인 → final color-pop verdict 고정** 이다.
- 밝기 수치와 screenshot이 정상인데 실화면 색감만 튀면, 합성보다 **panel picture-quality pipeline 전환** 이 우선 의심 대상이다.
- `gamma`, `acl_gain`, `tonemap_mode`, `hdr_bypass` 를 같은 frame 축으로 묶어야 one-beat color pop을 잡을 수 있다.
- 최소 증적은 `PANEL_GAMMA_TRACE`, `ACL_TRANSITION_TRACE`, `TONEMAP_TRANSITION_TRACE`, `FINAL_COLOR_POP_VERDICT` 네 줄이면 충분하다.

## 코드 포인트

1. **first visible frame 전후로 panel gamma LUT bank가 바뀌는지 고정한다**

   ```text
   PANEL_GAMMA_TRACE display=2 frame=6416 bank=boot_gamma crc=0x18a4 ready=1 source=panel_init
   PANEL_GAMMA_TRACE display=2 frame=6417 bank=ui_gamma crc=0x7c21 ready=1 source=pq_policy
   ```

   `frame+1` 에서 `bank` 가 바뀌면 black frame이 아니라 **gamma table swap** 으로 체감 색감이 튈 수 있다.

2. **ACL/ACE 같은 전류 제한 계층이 첫 UI 노출 직후 gain을 바꾸는지 본다**

   ```text
   ACL_TRANSITION_TRACE display=2 frame=6416 acl=off gain=1.00 reason=boot_guard
   ACL_TRANSITION_TRACE display=2 frame=6417 acl=ui gain=0.86 reason=thermal_default
   ```

   `gain` 이 갑자기 낮아지면 사용자는 밝기 드롭으로 느끼지만 실제 owner는 backlight가 아니라 **panel ACL 정책** 이다.

3. **HDR/tonemap bypass 해제가 늦게 들어오는지 분리한다**

   ```text
   TONEMAP_TRANSITION_TRACE display=2 frame=6416 hdr_bypass=1 tonemap=sdr_passthrough gamut=native
   TONEMAP_TRANSITION_TRACE display=2 frame=6417 hdr_bypass=0 tonemap=ui_vivid gamut=srgb_panel
   ```

   `hdr_bypass=1 -> 0` 전환이 first frame 뒤에 오면 같은 버퍼도 채도와 명암이 갑자기 달라 보일 수 있다.

4. **screenshot 정상 여부와 panel PQ 전환을 한 줄로 연결한다**

   ```text
   SCREENSHOT_PQ_COMPARE display=2 frame=6417 screenshot=stable panel=color_pop gamma_bank=ui_gamma tonemap=ui_vivid
   ```

   screenshot이 안정적인데 panel만 튀면 app/UI 문제가 아니라 PQ 후단 절단면으로 바로 좁힐 수 있다.

5. **최종 verdict를 어떤 PQ owner가 바꿨는지 고정한다**

   ```text
   FINAL_COLOR_POP_VERDICT display=2 frame=6417 cause=gamma_bank_swap owner=panel_pq_policy symptom=warm_vivid_pop
   FINAL_COLOR_POP_VERDICT display=2 frame=6417 cause=acl_gain_step owner=panel_thermal_acl symptom=luma_drop_without_backlight_change
   ```

   이 verdict가 있어야 Day111의 광량 owner 문제와 Day112의 **panel PQ policy 전환 문제** 를 서로 다른 수정 경로로 분리할 수 있다.

## 리스크

- brightness/backlight 로그만 보고 안정적이라고 결론내리면 gamma/ACL 전환에 의한 체감 튐을 놓칠 수 있다.
- panel vendor PQ 블록은 register write가 늦게 반영되는 경우가 있어 frame 번호와 실제 latch 시점을 함께 봐야 한다.
- HDR bypass 해제와 night mode/color transform 복원이 겹치면 OS 색정책 문제로 잘못 몰아가기 쉽다.
- multi-panel SKU에서는 같은 Android 빌드라도 panel LUT bank 구조가 달라 panel id 기준 증적이 필요하다.

## 다음 액션

다음 글에서는 Day112 다음 절단면으로, **panel gamma/ACL/tonemap까지 정상인데도 첫 UI 직후 특정 레이어 경계만 번쩍이거나 contour가 깨질 때 dither/FRC/DSC 복원 타이밍을 어떻게 자를지** 정리하겠다.

---
title: R사 X5H Day111 - backlight ownership·local dimming release·CABC/ALS final luminance jump cutline
author: JaeHa
date: 2026-07-10 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day111, display, backlight, localdimming, cabc, als, luminance, bringup]
---

Day110에서 `SCREENSHOT_PANEL_COMPARE`, `COLOR_TRANSFORM_TRACE`, `CLIENT_TARGET_LINEAGE` 까지 정상인데도 사용자가 첫 홈 진입 순간 **밝기가 급격히 튀거나 색온도가 순간 바뀌는 현상** 을 체감한다면, 이제 남는 절단면은 `backlight ownership`, `local dimming release`, `CABC/ALS 개입 순서` 다. 즉 픽셀 내용은 정상인데 **최종 광량 제어 owner가 교대되는 순간** 이 시각 품질을 깨는지 확인해야 한다.

## 핵심 요약

- Day111의 절단 순서는 **brightness owner handoff 확인 → local dimming release 시점 고정 → CABC/ALS override 개입 분리 → final luminance jump verdict 고정** 이다.
- screenshot과 client target이 정상인데 실화면만 밝기 점프를 보이면, 합성보다 **panel 후단 광량 제어 체인** 이 우선 조사 대상이다.
- 동일 frame에서 `brightness 값`, `owner`, `local dimming state`, `ALS override` 를 한 줄로 남겨야 one-beat 밝기 점프를 재현 가능하게 잡을 수 있다.
- 최소 증적은 `BRIGHTNESS_OWNER_TRACE`, `LOCAL_DIMMING_RELEASE_TRACE`, `ALS_CABC_OVERRIDE_TRACE`, `FINAL_LUMINANCE_JUMP_VERDICT` 네 줄이면 충분하다.

## 코드 포인트

1. **최종 brightness write owner가 누구인지 먼저 고정한다**

   ```text
   BRIGHTNESS_OWNER_TRACE display=2 frame=6415 owner=surfaceflinger requested_nits=420 applied_nits=420 reason=boot_complete_handoff
   BRIGHTNESS_OWNER_TRACE display=2 frame=6415 owner=lights_hal requested_nits=420 applied_nits=180 reason=policy_override
   ```

   `requested_nits` 와 `applied_nits` 가 다르면 backlight register write 자체보다 **owner arbitration** 을 먼저 봐야 한다.

2. **local dimming/local contrast enhancement 해제 시점을 frame 축에 맞춘다**

   ```text
   LOCAL_DIMMING_RELEASE_TRACE display=2 frame=6415 state=hold zone_gain=0.42 release_pending=1 source=boot_guard
   LOCAL_DIMMING_RELEASE_TRACE display=2 frame=6416 state=released zone_gain=1.00 release_pending=0 source=panel_policy
   ```

   `state=hold` 에서 다음 frame에 `zone_gain` 이 급변하면 사용자는 black frame이 아니라 **밝기 점프/색온도 변형** 으로 느낀다.

3. **CABC/ALS가 handoff 직후 값을 다시 덮는지 분리한다**

   ```text
   ALS_CABC_OVERRIDE_TRACE display=2 frame=6415 als_lux=185 cabc=movie override=0 final_nits=420
   ALS_CABC_OVERRIDE_TRACE display=2 frame=6415 als_lux=185 cabc=ui override=1 final_nits=260
   ```

   `override=1` 이면 launcher first frame은 정상이어도 바로 다음 beat에서 광량 정책이 바뀌며 튐이 발생할 수 있다.

4. **색온도/white-point 이동이 brightness jump로 오인되지 않게 함께 기록한다**

   ```text
   PANEL_COLOR_STATE_TRACE display=2 frame=6415 cct=6500 white_point=default night_mode=0 dc_dimming=0
   PANEL_COLOR_STATE_TRACE display=2 frame=6415 cct=5200 white_point=warm night_mode=1 dc_dimming=1
   ```

   brightness 수치가 같아도 `cct` 나 `white_point` 가 바뀌면 사용자는 밝기 점프로 느낄 수 있다. 특히 CABC와 night-mode restore가 겹치면 오판하기 쉽다.

5. **최종 verdict를 광량 제어 owner 기준으로 한 줄 고정한다**

   ```text
   FINAL_LUMINANCE_JUMP_VERDICT display=2 frame=6415 cause=backlight_owner_race owner=lights_hal_vs_sf symptom=brightness_drop_after_first_frame
   FINAL_LUMINANCE_JUMP_VERDICT display=2 frame=6416 cause=local_dimming_release_step owner=panel_policy symptom=warm_bright_flash
   ```

   이 verdict가 있어야 Day110의 dark-frame 잔류와 Day111의 **광량/색온도 handoff 문제** 를 다른 owner로 분리할 수 있다.

## 리스크

- screenshot이 정상이라고 바로 panel 전기 특성 문제로 몰아가면, 실제로는 HAL/정책 owner race를 놓칠 수 있다.
- local dimming state를 frame 기준으로 묶지 않으면 first visible UI 직후 한 번만 튀는 현상을 재현 로그에서 놓치기 쉽다.
- ALS/CABC override는 실내 조도에 따라 간헐적으로만 발생하므로 lab 재현 시 lux 입력 고정이 필요하다.
- color temperature 복원과 brightness handoff가 동시에 일어나면 사용자 체감은 같아도 수정 owner가 달라 대응이 엇갈릴 수 있다.

## 다음 액션

다음 글에서는 Day111 다음 절단면으로, **brightness owner와 local dimming까지 정상인데도 첫 UI 순간에 panel gamma/ACL/tonemap 전환으로 색감이 튀는 경우를 어떻게 잘라낼지** 정리하겠다.

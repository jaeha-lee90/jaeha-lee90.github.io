---
title: R사 X5H Day100 - brightness ownership 경쟁·backlight write ordering 깜빡임/재흑화 cutline
author: JaeHa
date: 2026-06-10 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day100, display, backlight, brightness, thermal, vehicle, safety, firstframe, bringup]
---

Day99에서 `first visible UI` 까지는 확인됐는데 직후 화면이 깜빡이거나 다시 검게 떨어진다면, 이제 잘라야 할 구간은 **brightness ownership arbitration** 이다. 즉 `AOSP settings`, `vehicle service`, `safety dimmer`, `thermal policy`, `panel driver restore path` 가 같은 backlight register를 서로 다른 순서로 덮어써서 **첫 visible frame 직후 luminance를 다시 0 근처로 내리는지**를 확인해야 한다.

## 핵심 요약

- Day100의 우선 절단 순서는 **first visible UI 확인 → brightness writer owner 타임라인 고정 → write ordering/last-writer 확인 → clamp source(safety/thermal/vehicle) 판정 → visible flicker verdict 고정** 이다.
- `visible_frame` 이 찍혔는데 곧바로 black 또는 dim으로 떨어지면 display pipeline보다 **brightness owner race** 문제일 확률이 높다.
- 최소 증적은 `BRIGHTNESS_OWNER`, `BACKLIGHT_WRITE`, `BRIGHTNESS_CLAMP`, `VISIBLE_FLICKER_VERDICT` 네 줄이면 충분하다.
- 이 단계에서 막히면 HWC/DSI가 아니라 framework brightness policy와 vendor backlight service 경계를 먼저 봐야 한다.

## 코드 포인트

1. **Day99의 first visible frame을 기준 anchor로 고정한다**

   ```text
   FIRST_VISIBLE_UI display=2 accepted_frame=6315 visible_frame=6318 luminance_nonzero=1 blocker=none
   ```

   이 anchor가 있어야 이후 black/dim 전환이 panel ingress 문제가 아니라 post-visible brightness race임을 분리할 수 있다.

2. **backlight 값을 누가 언제 쓰는지 owner 타임라인을 남긴다**

   ```text
   BRIGHTNESS_OWNER display=2 ts=5123388 owner=panel_restore level=64 reason=resume_defaults
   BRIGHTNESS_OWNER display=2 ts=5123394 owner=vehicle_service level=18 reason=gear_reverse_policy
   BRIGHTNESS_OWNER display=2 ts=5123399 owner=settings_provider level=120 reason=user_restore
   ```

   boot 직후에는 동일 register를 여러 레이어가 순차적으로 쓴다. owner 로그가 없으면 마지막 writer가 누구인지 못 잡는다.

3. **실제 register write 순서와 최종 승자를 분리해서 본다**

   ```text
   BACKLIGHT_WRITE display=2 seq=441 owner=panel_restore req=64 applied=64 pwm_en=1
   BACKLIGHT_WRITE display=2 seq=442 owner=safety_dimmer req=0 applied=0 pwm_en=1 reason=seatbelt_popup_guard
   BACKLIGHT_WRITE display=2 seq=443 owner=settings_provider req=120 applied=0 rejected_by=thermal_cap
   ```

   사용자가 본 현상은 “밝기 변경”이지만 실제 root cause는 `last write wins`, `write rejected`, `write clamped` 셋 중 하나다. 특히 `applied != req` 를 반드시 남겨야 한다.

4. **safety/thermal/vehicle clamp가 black 재진입을 만든 경우를 따로 고정한다**

   ```text
   BRIGHTNESS_CLAMP display=2 owner=thermal_policy cap=8 floor=0 active=1 reason=panel_hot_boot
   BRIGHTNESS_CLAMP display=2 owner=safety_dimmer cap=0 floor=0 active=1 reason=camera_transition_mask
   ```

   밝기 owner race와 clamp는 다르다. owner가 정상이어도 clamp가 활성화되면 결과는 여전히 black이다.

5. **최종적으로 visible 이후 flicker/재흑화 verdict를 한 줄로 묶는다**

   ```text
   VISIBLE_FLICKER_VERDICT display=2 visible_frame=6318 black_return_frame=6320 last_writer=safety_dimmer clamp_owner=safety_dimmer cutline=post_visible_brightness_race
   VISIBLE_FLICKER_VERDICT display=2 visible_frame=6318 black_return_frame=none last_writer=settings_provider clamp_owner=none cutline=none
   ```

   이 verdict가 있어야 panel 팀, Android framework 팀, vehicle app 팀 사이 escalation 경계가 선명해진다.

## 리스크

- 밝기 owner와 clamp를 분리하지 않으면 “누가 0을 썼는지”와 “누가 0으로 제한했는지”가 섞여 잘못된 팀으로 이슈가 간다.
- boot animation 종료 직후 settings restore와 vendor policy replay가 겹치면 재현이 1~2 frame flicker로만 보여 증거가 쉽게 사라진다.
- PWM은 살아 있는데 brightness LUT만 0 근처로 떨어지는 경우가 있어 단순 `pwm_en=1` 확인만으로는 안심하면 안 된다.
- vehicle 상태 연동 정책은 ignition/gear/camera/safety overlay 조건에 따라 달라져 cold boot와 warm boot가 다른 증상을 만들 수 있다.

## 다음 액션

다음 글에서는 Day100 다음 절단면으로, **brightness owner race는 정리됐는데도 첫 홈/앱 전환 구간에서만 짧은 flicker가 남을 때 HWC color fade, fade layer, display power mode transition ordering** 을 정리하겠다.

---
title: R사 X5H Day101 - HWC color fade·fade layer·display power mode transition flicker cutline
author: JaeHa
date: 2026-06-11 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day101, display, hwc, colorfade, fadelayer, powermode, flicker, firstframe, bringup]
---

Day100에서 `brightness owner race` 가 아니라는 것까지 확인했는데도 홈/앱 첫 전환에서만 짧은 깜빡임이 남는다면, 이제 잘라야 할 구간은 **HWC fade composition 과 display power mode transition ordering** 이다. 즉 실제 콘텐츠는 계속 살아 있는데 `color fade layer`, `boot fade animation`, `DisplayPowerController` 전환 순서가 한두 프레임 어긋나면서 **검은 레이어가 실콘텐츠 위를 잠깐 덮는지**를 확인해야 한다.

## 핵심 요약

- Day101의 우선 절단 순서는 **brightness race 배제 → fade layer 제출 여부 확인 → HWC client/device composition 경로 확인 → power mode transition ordering 확인 → transient flicker verdict 고정** 이다.
- `backlight`, `panel ingress`, `visible frame` 이 모두 정상인데도 1~2 frame black flicker가 보이면 pipeline 고장보다 **fade/transition ordering** 문제일 가능성이 높다.
- 최소 증적은 `FADE_LAYER`, `HWC_COMPOSE_PATH`, `DISPLAY_POWER_TRANSITION`, `TRANSIENT_FLICKER_VERDICT` 네 줄이면 충분하다.
- 이 단계에서 막히면 panel 팀보다 framework display/power 팀과 HWC vendor 경계를 먼저 확인해야 한다.

## 코드 포인트

1. **Day100 이후 brightness 경로가 stable인지 먼저 못 박는다**

   ```text
   BRIGHTNESS_STABLE display=2 frame=6320 level=118 pwm_en=1 clamp_owner=none
   ```

   이 anchor가 있어야 이후 black frame을 밝기 0 복귀가 아니라 합성/전환 레이어 문제로 분리할 수 있다.

2. **fade layer 또는 color fade가 실제로 submit 되었는지 기록한다**

   ```text
   FADE_LAYER display=2 frame=6321 source=boot_finish alpha=1.0 color=black visible=1
   FADE_LAYER display=2 frame=6322 source=boot_finish alpha=0.0 color=black visible=0
   ```

   SystemUI/WM fade, boot animation exit fade, trusted overlay fade가 겹치면 실콘텐츠는 준비돼도 검은 레이어가 한 beat 남는다.

3. **해당 프레임이 HWC device composition인지 client composition인지 같이 남긴다**

   ```text
   HWC_COMPOSE_PATH display=2 frame=6321 mode=client gpu_target=valid overlay_layers=0 fade_layer=1
   HWC_COMPOSE_PATH display=2 frame=6322 mode=device gpu_target=none overlay_layers=3 fade_layer=0
   ```

   fade frame이 client composition으로만 처리되면 GPU target clear color가 black이라 transient black으로 보일 수 있다.

4. **display power mode transition ordering이 present 직전/직후에 끼는지 본다**

   ```text
   DISPLAY_POWER_TRANSITION display=2 frame=6321 old=DOZE new=ON blanked=1 commit_seq=918
   DISPLAY_POWER_TRANSITION display=2 frame=6322 old=ON new=ON blanked=0 commit_seq=919
   ```

   `DOZE/ON`, `blank/unblank`, `idle/offload exit` 가 첫 app transition 프레임과 겹치면 실제 콘텐츠가 있어도 패널 쪽에는 black hold처럼 보일 수 있다.

5. **최종 flicker cutline을 한 줄 verdict로 묶는다**

   ```text
   TRANSIENT_FLICKER_VERDICT display=2 visible_frame=6320 black_flicker_frames=6321-6321 cause=fade_layer_late_drop cutline=hwc_fade_transition
   TRANSIENT_FLICKER_VERDICT display=2 visible_frame=6320 black_flicker_frames=none cause=none cutline=none
   ```

   이 verdict가 있어야 앱 attach 문제와 HWC/power transition 문제를 섞지 않고 escalation 할 수 있다.

## 리스크

- fade layer와 실제 black 콘텐츠를 구분하지 않으면 앱 first draw 지연으로 잘못 몰아갈 수 있다.
- power mode transition 로그가 프레임 번호와 묶이지 않으면 `ON 전환이 원인인지`, `전환 후 fade drop 지연이 원인인지`가 섞인다.
- client composition fallback이 한 프레임만 발생해도 사용자는 큰 깜빡임으로 느끼지만 trace가 짧아 증거를 놓치기 쉽다.
- boot animation 종료, keyguard dismiss, 첫 launcher attach가 겹치면 재현 조건이 좁아져 cold boot에서만 보이는 것처럼 오판할 수 있다.

## 다음 액션

다음 글에서는 Day101 다음 절단면으로, **fade/power transition도 아닌데 첫 홈 진입 순간만 black frame이 남을 때 boot animation exit transaction, launcher first draw, SurfaceFlinger layer lifecycle ordering** 을 정리하겠다.

---
title: R사 X5H Day107 - backlight unblank·local dimming release·display power-mode handoff final visibility cutline
author: JaeHa
date: 2026-07-06 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day107, display, backlight, localdimming, power, unblank, blackframe, bringup]
---

Day106에서 `TE_PHASE_TRACE`, `FIRST_FRAME_RELEASE`, `PSR_EXIT_TRACE` 가 모두 정상인데도 첫 visible UI가 검게 보인다면, 이제 남는 절단면은 **패널에 프레임은 들어왔지만 실제 광출력이 아직 열리지 않은 경우** 다. 즉 `backlight unblank`, `local dimming release`, `display power-mode handoff` 셋 중 하나가 늦으면 scanout은 정상이어도 사용자는 여전히 black 1-frame을 보게 된다.

## 핵심 요약

- Day107의 절단 순서는 **backlight unblank 확인 → local dimming release 확인 → power-mode handoff 확인 → final visibility verdict 고정** 이다.
- Day106까지 정상인데 black frame이 남으면 원인은 timing보다 더 뒤쪽인 **광출력 enable/panel power handoff** 일 가능성이 높다.
- 최소 증적은 `BACKLIGHT_UNBLANK_TRACE`, `LOCAL_DIMMING_RELEASE`, `DISPLAY_POWER_HANDOFF`, `FINAL_VISIBILITY_VERDICT` 네 줄이면 충분하다.
- 이 cutline이 나오면 SurfaceFlinger/HWC보다 **panel power/backlight owner** 가 우선 대응해야 한다.

## 코드 포인트

1. **backlight가 첫 유효 frame 이전에 실제로 unblank 되었는지 먼저 고정한다**

   ```text
   BACKLIGHT_UNBLANK_TRACE display=2 frame=6412 state=unblank pwm=182 duty=41 enable_gpio=1 at_present_seq=8801
   BACKLIGHT_UNBLANK_TRACE display=2 frame=6412 state=late_unblank pwm=182 duty=41 enable_gpio=1 at_present_seq=8802
   ```

   `late_unblank` 이면 panel ingress와 TE가 정상이어도 사용자는 첫 beat를 검게 본다.

2. **local dimming/FALD 계층이 black hold를 유지하는지 확인한다**

   ```text
   LOCAL_DIMMING_RELEASE display=2 frame=6412 profile=ui boost=applied black_hold=0 zone_ready=1
   LOCAL_DIMMING_RELEASE display=2 frame=6412 profile=ui boost=pending black_hold=1 zone_ready=0
   ```

   `black_hold=1` 이면 backlight enable 후에도 zone update가 늦어 실제 가시광 출력은 막혀 있을 수 있다.

3. **display power-mode handoff가 ON/DOZE/IDLE 전환 사이에서 끼어들지 않는지 본다**

   ```text
   DISPLAY_POWER_HANDOFF display=2 frame=6412 requested=ON applied=ON blocker=none commit_seq=8801
   DISPLAY_POWER_HANDOFF display=2 frame=6412 requested=ON applied=DOZE blocker=late_policy commit_seq=8801
   ```

   `applied=DOZE` 나 `blocker=late_policy` 면 scanout은 진행돼도 실제 패널 노출 모드는 아직 열리지 않았다고 봐야 한다.

4. **세 신호를 같은 frame/present sequence로 묶어 final visible 여부를 남긴다**

   ```text
   FINAL_VISIBILITY_TRACE display=2 frame=6412 present_seq=8801 backlight=unblank dimming=released power=ON visible=1
   FINAL_VISIBILITY_TRACE display=2 frame=6412 present_seq=8801 backlight=late_unblank dimming=hold power=DOZE visible=0
   ```

   같은 frame 기준 정렬이 안 되면 첫 visible miss와 다음 frame 정상 복구를 혼동하게 된다.

5. **최종 verdict를 광출력 owner 기준으로 한 줄 고정한다**

   ```text
   FINAL_VISIBILITY_VERDICT display=2 frame=6412 ingress=ok timing=ok visible=late cause=backlight_unblank_late owner=backlight_ctl
   FINAL_VISIBILITY_VERDICT display=2 frame=6412 ingress=ok timing=ok visible=late cause=power_handoff_blocked owner=panel_power_policy
   ```

   이 verdict가 있어야 Day106의 timing 이슈와 Day107의 실제 발광 이슈를 깔끔하게 분리할 수 있다.

## 리스크

- backlight GPIO/PWM 로그만 보고 실제 panel 발광 시점을 추정하면 local dimming hold를 놓칠 수 있다.
- 전력 정책 로그가 늦게 찍히면 `power handoff` 와 `late unblank` 가 뒤섞여 잘못 분류될 수 있다.
- debug trace가 frame/present sequence를 공유하지 않으면 첫 frame miss가 다음 frame 정상 로그에 가려진다.
- local dimming 경로는 SKU별 구현차가 커서 공통 verdict 필드 정의가 없으면 팀 간 해석이 어긋난다.

## 다음 액션

다음 글에서는 Day107 다음 절단면으로, **광출력까지 정상인데 사용자가 여전히 순간 black/flicker를 볼 때 bootanimation 제거, launcher window reveal, color-fade overlay teardown 중 어느 레이어가 마지막으로 시야를 가리는지** 를 자르는 체크포인트를 정리하겠다.

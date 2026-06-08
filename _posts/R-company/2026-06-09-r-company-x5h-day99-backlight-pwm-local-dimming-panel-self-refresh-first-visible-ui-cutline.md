---
title: R사 X5H Day99 - backlight/PWM·local dimming·panel self-refresh exit first visible UI cutline
author: JaeHa
date: 2026-06-09 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day99, display, backlight, pwm, localdimming, psr, firstframe, bringup]
---

Day98에서 `DSI flush → TCON latch → panel ingress accept` 까지 모두 정상인데도 사용자는 여전히 검은 화면만 본다면, 이제 남는 절단면은 **광학 출력 enable 시퀀스**다. 즉 first non-black frame은 panel 내부까지 들어갔지만, `backlight/PWM`, `local dimming`, `panel self-refresh(PSR)` exit 중 하나가 늦어서 **첫 visible UI가 가려지는지**를 확인해야 한다.

## 핵심 요약

- Day99의 우선 절단 순서는 **panel accept 확인 → backlight/PWM enable 시점 확인 → local dimming zone release 확인 → PSR/idle self-refresh exit 확인 → first visible luminance 판정** 이다.
- `panel_accept=1` 인데 black이면 display pipeline보다 **optical output gating** 문제일 확률이 높다.
- 최소 증적은 `BACKLIGHT_ENABLE`, `LOCAL_DIMMING_RELEASE`, `PSR_EXIT`, `FIRST_VISIBLE_UI` 네 줄이면 된다.
- 이 단계에서 막히면 HWC나 DSI보다 panel driver/PMIC/backlight policy를 먼저 봐야 한다.

## 코드 포인트

1. **패널이 first frame을 이미 수용했는지 Day98 verdict를 이어받는다**

   ```text
   PANEL_INGRESS_VERDICT display=2 frame=6315 dsi_flush=1 latch_ok=1 panel_accept=1 cutline=none
   ```

   이 전제가 없으면 Day99 분석은 의미가 없다. `panel_accept=0` 이면 다시 Day98 구간으로 돌아가야 한다.

2. **backlight/PWM enable이 first accepted frame보다 늦지 않는지 본다**

   ```text
   BACKLIGHT_ENABLE display=2 frame=6315 pwm_en=0 bl_level=0 reason=boot_hold
   BACKLIGHT_ENABLE display=2 frame=6317 pwm_en=1 bl_level=32 delta_from_accept_ms=27
   ```

   first frame이 들어왔어도 `pwm_en=0` 또는 `bl_level=0` 이면 사용자는 계속 black으로 본다. 흔한 패턴은 panel init 완료 전까지 backlight enable을 보류하는 것이다.

3. **local dimming 또는 zone brightness clamp가 첫 UI를 가리는지 확인한다**

   ```text
   LOCAL_DIMMING_RELEASE display=2 frame=6315 zones_ready=0 global_gain=0 clamp=boot_black
   LOCAL_DIMMING_RELEASE display=2 frame=6318 zones_ready=1 global_gain=96 clamp=none
   ```

   HDR/자동차용 panel은 zone map 준비 전까지 global gain을 0으로 두는 경우가 있다. 이때 frame 자체는 정상인데 광학 출력만 0 nit에 머문다.

4. **panel self-refresh(PSR) 또는 idle self-refresh exit가 실제 scan source 전환을 늦추는지 본다**

   ```text
   PSR_EXIT display=2 frame=6315 psr_state=active exit_req=1 exit_done=0 source=self_refresh
   PSR_EXIT display=2 frame=6316 psr_state=inactive exit_req=1 exit_done=1 source=live_scanout
   ```

   PSR exit가 한 beat 늦으면 accepted frame은 panel memory에 묻히고, 사용자는 다음 live scanout부터 보게 된다.

5. **최종적으로 사용자가 본 첫 visible UI frame을 따로 기록한다**

   ```text
   FIRST_VISIBLE_UI display=2 accepted_frame=6315 visible_frame=6318 luminance_nonzero=1 blocker=backlight_hold
   FIRST_VISIBLE_UI display=2 accepted_frame=6315 visible_frame=6316 luminance_nonzero=1 blocker=psr_exit_delay
   ```

   `accepted_frame` 과 `visible_frame` 차이를 남겨야 BSP, panel vendor, 앱팀이 같은 frame gap을 기준으로 이야기할 수 있다.

## 리스크

- panel accept만 보고 display path 완료로 판정하면 backlight hold나 dimming clamp를 놓치기 쉽다.
- boot logo에서 UI로 넘어가는 순간 backlight policy가 재적용되면 first-app 전용 black window처럼 보일 수 있다.
- PSR/idle refresh는 정상 절전 기능이라 로그가 없으면 결함으로 안 보이기 쉽다.
- 광학 출력 gating 문제를 SurfaceFlinger 지연으로 오인하면 디버깅 팀이 완전히 잘못 배정된다.

## 다음 액션

다음 글에서는 Day99 다음 절단면으로, **first visible UI는 떴지만 곧바로 깜빡이거나 다시 black으로 떨어질 때 brightness ownership 경쟁(AOSP setting, vehicle service, safety dimmer, thermal policy)과 backlight write ordering** 을 정리하겠다.

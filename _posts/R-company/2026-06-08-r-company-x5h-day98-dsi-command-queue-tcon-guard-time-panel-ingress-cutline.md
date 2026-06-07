---
title: R사 X5H Day98 - DSI command queue flush·TCON frame-start latch·bridge/panel guard-time panel ingress cutline
author: JaeHa
date: 2026-06-08 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day98, display, dsi, tcon, bridge, panel, firstframe, bringup]
---

Day97에서 `writeback/CRC/TE/VBLANK` 로 **SoC scanout은 실제로 돌고 있음**이 확인됐는데도 첫 앱 프레임 한 장만 패널에서 보이지 않는다면, 이제 잘라야 할 구간은 `DSI command/video egress → bridge/TCON ingress latch → panel guard-time` 이다. 즉 source는 정상인데 **패널이 첫 유효 frame을 받기 전에 command flush나 frame-start latch 조건을 놓치는지**를 확인해야 한다.

## 핵심 요약

- Day98의 1차 절단 순서는 **DSI packet flush 확인 → frame-start latch 조건 확인 → reset/ULPS/blank 해제 후 guard-time 충족 여부 확인 → first accepted panel frame 판정** 이다.
- Day97에서 `scanout_observed=1` 인데 화면이 계속 black이면 `cmd queue not flushed`, `frame-start missed`, `bridge blank release late`, `panel guard-frame discard` 중 하나일 가능성이 높다.
- 최소 증적은 `DSI_TX_FLUSH`, `PANEL_FRAME_START_LATCH`, `PANEL_GUARD_WINDOW`, `FIRST_PANEL_ACCEPT` 네 줄이면 충분하다.
- 이 구간에서 막히면 HWC나 display engine이 아니라 **panel ingress sequencing** 이 주원인일 확률이 높다.

## 코드 포인트

1. **SoC가 첫 frame용 DSI payload를 실제로 밀어냈는지 flush 단위로 고정한다**

   ```text
   DSI_TX_FLUSH display=2 frame=6314 mode=video pkt_seq=881 last_line_ts=5123334 flush_done=1 fifo_empty=1
   DSI_TX_FLUSH display=2 frame=6314 mode=cmd pkt_seq=881 flush_done=0 fifo_empty=0 reason=kick_missing
   ```

   Day97에서 CRC가 바뀌어도 DSI egress가 `flush_done=0` 이면 패널 입장에서는 첫 frame이 아직 오지 않은 것이다. 특히 command mode는 shadow register write 후 explicit kick 누락이 흔하다.

2. **bridge/TCON이 그 frame을 latch 가능한 frame-start 경계에서 받았는지 본다**

   ```text
   PANEL_FRAME_START_LATCH display=2 frame=6314 fs_ts=5123335 latched=1 line=0 shadow_to_active=1
   PANEL_FRAME_START_LATCH display=2 frame=6314 fs_ts=5123335 latched=0 line=742 reason=late_shadow_apply
   ```

   첫 frame이 active window 도중에 들어오면 다음 frame-start까지 미뤄질 수 있다. `line!=0` 또는 `late_shadow_apply` 가 보이면 한 장만 검게 보이고 다음 장부터 정상인 패턴이 설명된다.

3. **reset/ULPS exit/blank 해제 후 panel guard-time이 남아 있는지 확인한다**

   ```text
   PANEL_GUARD_WINDOW display=2 frame=6314 reset_release_ms=6 ulps_exit_ms=2 blank_off_ms=1 required_ms=10 guard_open=0
   PANEL_GUARD_WINDOW display=2 frame=6315 reset_release_ms=18 ulps_exit_ms=11 blank_off_ms=9 required_ms=10 guard_open=1
   ```

   패널이나 bridge가 `blank off 후 N ms`, `ULPS exit 후 1 frame`, `sleep out 후 120 ms` 같은 보호 구간을 갖고 있으면 첫 frame은 의도적으로 버려진다. bring-up 때 이 guard-time이 문서화되지 않으면 재현이 들쭉날쭉해진다.

4. **패널이 첫 frame을 실제 유효 frame으로 수용했는지 verdict를 남긴다**

   ```text
   FIRST_PANEL_ACCEPT display=2 frame=6314 ingress_seen=1 accepted=0 reason=guard_frame_discard next_accept=6315
   FIRST_PANEL_ACCEPT display=2 frame=6315 ingress_seen=1 accepted=1 reason=none next_accept=6315
   ```

   `ingress_seen=1` 이지만 `accepted=0` 이면 black screen이라기보다 첫 frame discard다. 이 한 줄이 있으면 panel vendor와 BSP 팀이 같은 현상을 서로 다른 문제로 부르지 않게 된다.

5. **최종 cutline을 SoC egress / bridge latch / panel policy 셋 중 하나로 찍는다**

   ```text
   PANEL_INGRESS_VERDICT display=2 frame=6314 scanout_observed=1 dsi_flush=0 latch_ok=0 panel_accept=0 cutline=dsi_flush_kick
   PANEL_INGRESS_VERDICT display=2 frame=6314 scanout_observed=1 dsi_flush=1 latch_ok=1 panel_accept=0 cutline=panel_guard_policy
   ```

   이 verdict가 있어야 다음 액션이 선명해진다. `dsi_flush=0` 면 host driver, `latch_ok=0` 면 bridge/TCON timing, `panel_accept=0` 면 panel init/guard policy를 우선 본다.

## 리스크

- CRC/TE/VBLANK만 보고 visible path 전체를 통과했다고 결론내리면 panel ingress discard를 놓치기 쉽다.
- command mode panel은 shadow apply kick 한 번 빠져도 steady-state에서는 회복돼 first-frame 전용 버그로 남는다.
- guard-time이 문서화되지 않으면 온도·boot timing·display index 순서에 따라 재현률이 흔들린다.
- bridge와 panel이 각각 한 frame씩 discard하면 현상은 "두 장 늦게 뜸"으로 보이므로 frame 번호 매핑 없이는 잘못된 팀으로 escalation 되기 쉽다.

## 다음 액션

다음 글에서는 Day98 다음 절단면으로, **panel ingress는 정상인데 실제 사용자에게 보이는 첫 UI만 늦을 때 backlight/PWM enable·local dimming·panel self-refresh exit가 첫 non-black frame을 가리는 경우**를 정리하겠다.

---
title: R사 X5H Day77 - CSC/range 정상 후 남는 backlight-PWM/TE sync/lane status panel electrical 체크리스트
author: JaeHa
date: 2026-05-18 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day77, display, backlight, pwm, te, dsi, lane-status, panel, electrical, bringup]
---

Day76에서 `CSC`, `RGB/YUV`, `range`, `panel blanking` 까지 정상이면 이제 남는 큰 축은 **panel 전기적 활성 상태와 live-link** 다. 이 단계에서는 compositor/plane 쪽은 거의 정상인데도, `backlight PWM duty=0`, `TE sync 미수신`, `DSI lane stopstate/HS 진입 실패`, `panel reset-sequence 미완료` 때문에 사용자는 계속 black screen만 보게 된다.

## 핵심 요약

- Day76까지 통과했다면 다음 절단 순서는 **backlight on 사실 확인 → PWM/live brightness → TE(sync) 수신 → DSI lane state/ULPS/HS 진입** 이 가장 빠르다.
- `backlight=ON` 로그 한 줄만으로는 부족하다. 실제로는 `enable GPIO high` 인데 `PWM duty=0` 이거나 `brightness register not applied` 인 경우가 많다.
- `VBLANK alive` 와 `plane commit OK` 가 있어도 panel link가 HS transmission에 못 들어가면 화면은 끝까지 검다.
- 최소 증적은 `BL_PWM`, `TE_EVENT`, `DSI_LANE_STATE`, `PANEL_RESET_SEQ` 네 줄이면 된다.

## 코드 포인트

1. **backlight enable과 실제 광량은 분리해서 봐야 한다**

   ```text
   PANEL_PWR panel=cluster vdd=1 iovcc=1 reset=1 backlight_en=1
   BL_PWM panel=cluster pwm_ch=3 period_ns=20000 duty_ns=0 brightness=0
   ```

   `backlight_en=1` 이어도 `duty_ns=0` 이면 사용자가 보는 결과는 black screen이다. bring-up 초반에는 enable GPIO만 확인하고 넘어가다가 PWM 적용 누락을 놓치기 쉽다.

2. **brightness write race는 panel init success처럼 보이면서도 화면을 죽인다**

   ```c
   panel_hw_enable(panel);
   panel_send_init_seq(panel);
   panel_set_backlight(panel, ui_brightness);
   ```

   여기서 `ui_brightness` 가 기본값 0 이거나, init 직후 다른 worker가 0으로 덮어쓰면 로그상 panel init은 성공인데 실광량은 없다.

   ```text
   BACKLIGHT_REQ src=bootanim level=128
   BACKLIGHT_APPLY panel=cluster level=0 reason=late-default-override
   ```

3. **TE(sync) 미수신은 '그림은 보내는데 panel이 latch 안 함' 패턴으로 이어진다**

   ```text
   TE_CFG panel=cluster expected=enabled gpio=42 irq=224
   TE_EVENT panel=cluster count=0 timeout_ms=120
   FRAME_PUSH panel=cluster seq=381 hs_enter=1 eof=1
   ```

   `FRAME_PUSH` 는 되는데 `TE_EVENT count=0` 이면, panel이 frame latch 타이밍을 못 받고 있을 가능성이 높다. 이 경우 content path보다 TE polarity, routing, irq binding을 먼저 봐야 한다.

4. **DSI lane state는 live-link 확인의 핵심이다**

   ```text
   DSI_LANE_STATE d0=STOPSTATE d1=STOPSTATE clk=STOPSTATE ulps=0 hs_req=1 hs_ack=0
   ```

   `hs_req=1` 인데 `hs_ack=0` 이면 panel 쪽이 HS entry를 못 받거나 PHY timing/lane mapping이 어긋난 상태다. 반대로 아래처럼 보이면 live-link는 상당 부분 통과한 것이다.

   ```text
   DSI_LANE_STATE d0=HS d1=HS clk=HS ulps=0 hs_req=1 hs_ack=1
   ```

5. **reset sequence와 panel init command 완료 여부를 별도 ledger로 남겨야 한다**

   ```text
   PANEL_RESET_SEQ rst_low_ms=5 rst_high_ms=15 post_reset_delay_ms=120 result=OK
   PANEL_INIT_CMD seq=vendor_init_v3 sent=142 ack=142 last=0x29
   ```

   `0x29`(display on) 까지 갔는지, reset hold/release timing이 spec에 맞는지 분리해서 남겨야 한다. reset sequence가 미묘하게 틀리면 DSI lane은 살아도 panel 내부 state machine이 display-on까지 못 간다.

## 리스크

- `backlight_en=1` 만 보고 panel이 켜졌다고 판단하면 PWM duty 0, brightness override, polarity 오류를 오래 놓친다.
- TE 기반 panel에서 sync 미수신을 무시하면 frame push는 계속 보이는데 실제 latch가 안 되는 애매한 black screen이 지속된다.
- DSI lane state를 남기지 않으면 PHY timing/lane swap/link training 실패를 상위 software 문제로 오판하기 쉽다.
- reset/display-on 시퀀스가 spec과 조금만 어긋나도 부팅마다 재현성이 흔들려 intermittent issue로 번질 수 있다.

## 다음 액션

다음 글에서는 Day77을 이어서, **backlight/TE/lane status까지 정상인데도 검을 때 panel register dump·test pattern·BIST를 이용해 SoC 출력 문제와 panel 내부 처리 문제를 분리하는 절단 순서** 를 정리하겠다.

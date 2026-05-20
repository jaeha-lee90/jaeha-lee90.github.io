---
title: R사 X5H Day80 - mode/timing 정상 후 남는 continuous clock·ULPS exit·reset 후 첫 frame race 체크리스트
author: JaeHa
date: 2026-05-21 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day80, display, dsi, continuous-clock, ulps, reset, first-frame, race, bringup]
---

Day79에서 `command/video mode`, `porch`, `packetization` 까지 맞췄는데도 외부 frame이 여전히 안 보이거나 부팅 직후 한두 프레임만 놓친 뒤 영구 black screen으로 굳는 경우가 있다. 이때는 payload 형식보다 **링크 상태 전이와 첫 frame 수용 타이밍** 을 먼저 봐야 한다. 특히 `continuous clock 미유지`, `ULPS exit 불완전`, `mode switch 뒤 panel reset/blank 재개입`, `first-frame discard race` 네 축이 bring-up 막바지에서 자주 남는다.

## 핵심 요약

- Day79를 통과했는데 black screen이 남으면, 다음 절단 순서는 **continuous clock 상태 → ULPS exit 완료 여부 → mode switch 뒤 reset/blank 재개입 → first valid frame 보장** 이다.
- panel spec상 `continuous clock required` 인데 LP gap이 길게 들어가면 mode/timing이 모두 맞아도 첫 active frame을 panel이 버릴 수 있다.
- `ULPS exit ack` 없이 바로 frame push를 시작하면 trace상 전송 성공처럼 보여도 panel 수신 상태머신은 아직 sleep에 남아 있을 수 있다.
- 최소 증적은 `DSI_CLK_STATE`, `ULPS_TRACE`, `PANEL_RESET_TIMELINE`, `FIRST_FRAME_ACCEPT` 네 줄이면 충분하다.

## 코드 포인트

1. **continuous clock requirement를 먼저 고정한다**

   ```text
   DSI_CLK_STATE panel=cluster expected=continuous actual=non-continuous lp_gap_us=180
   PANEL_SPEC panel=cluster hs_clock=continuous_required te_window_us=120
   ```

   일부 panel은 video burst 자체보다 `HS clock continuity` 에 더 민감하다. frame payload는 정상이어도 line 사이 LP 진입이 길면 panel이 stream lock을 잃고 첫 frame 전체를 버릴 수 있다.

2. **ULPS exit 완료 전 frame 전송은 '성공처럼 보이는 실패' 를 만든다**

   ```text
   ULPS_TRACE lane=clk event=exit_req ts=42133
   ULPS_TRACE lane=clk event=exit_done ts=42290 latency_us=157
   FRAME_PUSH panel=cluster seq=801 ts=42180 eof=1 before_ulps_done=1
   ```

   `exit_req` 이후 `exit_done` 전에 밀어 넣은 첫 frame은 상위 stack에서 성공처럼 남기 쉽다. 하지만 panel/D-PHY 상태머신은 아직 수신 재개 조건을 만족하지 못했을 수 있다. 이 경우 첫 frame을 재전송하거나 `exit_done` 이후로 flush를 늦춰야 한다.

3. **mode switch 뒤 reset/blank 제어가 재개입하면 설정이 다시 무효화된다**

   ```text
   MODE_SWITCH panel=cluster from=cmd to=video result=ok ts=51820
   PANEL_RESET_TIMELINE gpio=reset state=toggle ts=51892 reason=late_init_worker
   PANEL_BLANK panel=cluster state=on ts=51910 owner=backlight_sync
   ```

   bootloader, kernel panel driver, vendor worker thread가 각각 panel state를 만지면 `video mode 진입 성공` 뒤에도 reset/blank가 다시 들어와 첫 표시 프레임을 날려 버린다. 이 패턴이면 packetization보다 ownership 충돌이 본질이다.

4. **panel이 첫 유효 frame을 버리는지 명시적으로 측정해야 한다**

   ```text
   FIRST_FRAME_ACCEPT panel=cluster frame_seq=802 crc_change=0 te_seen=0 accepted=0
   FIRST_FRAME_ACCEPT panel=cluster frame_seq=803 crc_change=1 te_seen=1 accepted=1
   ```

   어떤 panel은 `sleep-out/display-on` 직후 첫 frame 한 장을 안정화 용도로 버린다. 이 특성을 모른 채 단일 splash frame만 보내면 영구 black screen처럼 보인다. 최소 2~3 frame을 강제로 보내고 `crc_change/te_seen` 으로 acceptance를 확인하는 편이 안전하다.

5. **최종 판정은 state transition과 first-frame acceptance를 한 타임라인으로 묶는다**

   ```text
   PANEL_TIMELINE reset_release=41000 ulps_done=42290 mode_video=51820 first_flush=52100 first_accept=52340
   ```

   이 한 줄이 있으면 `flush는 했지만 아직 ULPS exit 전이었다`, `mode switch 뒤 reset이 다시 들어왔다`, `첫 frame은 버리고 둘째부터 받는다` 를 빠르게 분리할 수 있다. bring-up 후반의 black screen은 데이터 자체보다 이런 상태 전이 race로 남는 경우가 많다.

## 리스크

- command/video mode와 porch가 맞았다는 이유로 link state race를 배제하면 `간헐 1회 성공/재부팅 후 실패` 같은 재현성 낮은 black screen을 오래 못 잡는다.
- ULPS/continuous clock 정책이 bootloader와 kernel 사이에서 다르면 cold boot와 warm boot 결과가 달라져 원인 추적이 더 어려워진다.
- 첫 frame discard 특성을 고려하지 않고 splash나 단발 test frame만 보내면 false negative가 발생한다.
- reset/backlight worker ownership을 정리하지 않으면 수정 후에도 다른 thread가 panel state를 덮어써 회귀하기 쉽다.

## 다음 액션

다음 글에서는 Day80을 이어서, **panel 링크와 첫 frame acceptance까지 정상인데도 특정 display에서만 안 보일 때 multi-display clone/splitter/backend mux/DSC slice 경로를 자르는 체크리스트** 를 정리하겠다.

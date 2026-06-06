---
title: R사 X5H Day97 - writeback·CRC·TE/VBLANK로 first visible pixel 실스캔 여부 절단하기
author: JaeHa
date: 2026-06-07 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day97, display, writeback, crc, te, vblank, firstframe, bringup]
---

Day96에서 `display state=ON`, `first present accepted=1`, `enable ack` 까지 모두 정상인데도 **첫 앱 프레임 한 장만 눈에 보이지 않는다면**, 이제 잘라야 할 구간은 `scanout 증적 부재` 다. 즉 software 관점에서는 제출이 끝났지만 **실제 display engine이 첫 프레임을 읽어 패널 쪽으로 내보냈는지**를 `writeback/CRC/TE/VBLANK` 로 증명해야 한다.

## 핵심 요약

- Day97의 1차 절단 순서는 **first present 시점 고정 → writeback/CRC 증적 확보 → TE/VBLANK 발생 여부 확인 → visible pixel verdict 판정** 이다.
- `present accepted=1` 이어도 `writeback 미갱신`, `frame CRC 정지`, `TE/VBLANK 누락`, `scanout source freeze` 중 하나면 첫 장은 실제로 화면에 나가지 않았을 수 있다.
- 최소 증적은 `FIRST_PRESENT_ACCEPT`, `WRITEBACK_FRAME`, `FRAME_CRC_SAMPLE`, `TE_VBLANK_OBS` 네 줄이면 충분하다.
- 이 구간에서 막히면 HWC 합성이나 power state보다 아래인 **display engine scanout 관측성 부족** 문제가 핵심일 가능성이 높다.

## 코드 포인트

1. **첫 present와 관측 기준 프레임 번호를 같은 키로 묶는다**

   ```text
   FIRST_PRESENT_ACCEPT display=2 frame=6314 accepted=1 retire_fence=18 present_ts=5123315 buffer_id=0x9421
   FIRST_PRESENT_ACCEPT display=2 frame=6314 accepted=1 retire_fence=18 present_ts=5123315 wb_slot=none
   ```

   Day96까지 정상이어도 `frame=6314` 를 hardware telemetry와 연결하지 못하면 이후 로그는 전부 추정이 된다. `frame`, `buffer_id`, `present_ts` 를 writeback/CRC 쪽 키로 재사용해야 한다.

2. **writeback 또는 capture path가 첫 frame을 실제로 읽었는지 본다**

   ```text
   WRITEBACK_FRAME display=2 src_frame=6314 wb_seq=410 buffer_id=0x9421 nonblack=1 complete=1
   WRITEBACK_FRAME display=2 src_frame=6314 wb_seq=none buffer_id=0x9421 nonblack=unknown complete=0 reason=no_fetch
   ```

   writeback 경로가 있으면 가장 빠른 절단선이다. `accepted=1` 인데 `wb_seq` 가 안 늘면 display engine fetch 이전에서 멈춘 것이다. 반대로 writeback은 non-black인데 panel만 black이면 panel/link 쪽으로 바로 넘어갈 수 있다.

3. **frame CRC/hash가 첫 장에서만 정지하는지 확인한다**

   ```text
   FRAME_CRC_SAMPLE display=2 frame=6314 crc=0x8a31ff20 changed=1 prev_crc=0x00000000
   FRAME_CRC_SAMPLE display=2 frame=6314 crc=0x00000000 changed=0 prev_crc=0x00000000 source_nonblack=1
   ```

   writeback이 없거나 비싸면 line CRC, pipe CRC, histogram hash 같은 경량 지표로 대체할 수 있다. 첫 non-black source인데 CRC가 boot default 그대로면 scanout source가 갱신되지 않은 것이다.

4. **TE/VBLANK가 실제 첫 frame 이후 발생했는지 상대 순서로 본다**

   ```text
   TE_VBLANK_OBS display=2 frame=6314 te_ts=5123332 vblank_ts=5123333 post_present_ms=1.8 count_delta=1
   TE_VBLANK_OBS display=2 frame=6314 te_ts=none vblank_ts=5123350 post_present_ms=3.5 count_delta=0
   ```

   `accepted=1` 이더라도 `count_delta=0` 이면 첫 frame이 scanout cadence에 편입되지 않았을 수 있다. 특히 TE는 있는데 VBLANK 카운터가 안 움직이면 backend interrupt routing이나 display index binding을 의심해야 한다.

5. **최종적으로 source-visible/panel-visible을 분리해 verdict를 찍는다**

   ```text
   FIRST_VISIBLE_PIXEL_VERDICT display=2 frame=6314 source_ready=1 scanout_observed=0 panel_visible=0 cutline=display_engine_fetch
   FIRST_VISIBLE_PIXEL_VERDICT display=2 frame=6314 source_ready=1 scanout_observed=1 panel_visible=0 cutline=panel_link_or_tcon
   ```

   이 한 줄이 있어야 다음 담당 팀이 갈린다. `scanout_observed=0` 이면 SoC display engine/HWC/backend, `scanout_observed=1` 인데 `panel_visible=0` 이면 DSI/bridge/panel/TCON 쪽이다.

## 리스크

- writeback/CRC instrumentation 없이 `present 성공`만 믿으면 실제 미스캔 상황을 software success로 오판한다.
- CRC 기준 프레임 키가 SF/HWC 로그와 다르면 first-frame 문제를 steady-state frame과 섞어 잘못 결론내릴 수 있다.
- TE와 VBLANK는 발생해도 잘못된 display instance에서 온 interrupt일 수 있어 `display id/index` 매칭 검증이 빠지면 위험하다.
- writeback이 non-black이어도 panel path가 별도 mux/bridge를 타면 visible pixel 부재를 놓칠 수 있으니 verdict에서 `scanout` 과 `panel` 을 분리해야 한다.

## 다음 액션

다음 글에서는 Day97 다음 절단면으로, **scanout 관측은 정상인데 패널에서 첫 장만 놓칠 때 DSI command queue flush·TCON frame-start latch·bridge/panel guard-time 문제를 자르는 panel ingress cutline** 을 정리하겠다.

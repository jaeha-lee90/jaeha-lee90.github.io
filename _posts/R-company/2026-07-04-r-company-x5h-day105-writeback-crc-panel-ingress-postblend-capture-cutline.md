---
title: R사 X5H Day105 - writeback CRC·panel ingress trace·post-blend capture display-pipe 후단 cutline
author: JaeHa
date: 2026-07-04 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day105, display, writeback, crc, panel, postblend, blackframe, bringup]
---

Day104에서 `GPU clear`, `client target reuse`, `overlay fallback` 이 모두 정상인데도 사용자가 black 1-frame을 본다면, 이제는 **합성 결과는 정상인데 display pipeline 후단에서만 검게 보이는지** 를 잘라야 한다. 즉 SurfaceFlinger/HWC가 만든 프레임은 살아 있지만 **writeback CRC, post-blend capture, panel ingress trace 중 어디까지 정상인지** 를 같은 frame id로 묶어 확인하는 단계다.

## 핵심 요약

- Day105의 절단 순서는 **writeback CRC 확보 → post-blend capture 확인 → panel ingress trace 확인 → display-pipe 후단 verdict 고정** 이다.
- Day104가 정상인데도 black 1-frame이 남으면 원인은 composition보다 **scanout 후단 전송/패널 ingress** 쪽일 가능성이 높다.
- 최소 증적은 `WRITEBACK_CRC`, `POSTBLEND_CAPTURE`, `PANEL_INGRESS_TRACE`, `DISPLAY_PIPE_TAIL_VERDICT` 네 줄이면 충분하다.
- 이 cutline이 나오면 launcher나 RenderEngine보다 **HWC writeback path / DSI-bridge / panel timing ingress** 쪽이 우선 소유자다.

## 코드 포인트

1. **writeback CRC로 HWC 출력 프레임이 실제로 검지 않았는지 먼저 고정한다**

   ```text
   WRITEBACK_CRC display=2 frame=6412 crc=0x8f13a4c2 source=postcomp content=non_black valid=1
   WRITEBACK_CRC display=2 frame=6412 crc=0x00000000 source=postcomp content=black valid=1
   ```

   CRC가 non-black이면 합성 결과물은 살아 있고, black이면 Day104 판단을 다시 의심해야 한다.

2. **post-blend capture로 scanout 직전 plane 합성 결과를 캡처한다**

   ```text
   POSTBLEND_CAPTURE display=2 frame=6412 capture_ok=1 histogram_luma=37 preview=launcher_ui
   POSTBLEND_CAPTURE display=2 frame=6412 capture_ok=1 histogram_luma=0 preview=all_black
   ```

   writeback CRC와 post-blend capture가 둘 다 정상인데 panel만 black이면, 문제는 패널 ingress 이후로 좁혀진다.

3. **panel ingress trace로 DSI/bridge/panel이 첫 유효 frame을 실제로 받았는지 남긴다**

   ```text
   PANEL_INGRESS_TRACE display=2 frame=6412 dsi_pkt=ok hs_clk=on te_seen=1 frame_start=latched underrun=0
   PANEL_INGRESS_TRACE display=2 frame=6412 dsi_pkt=ok hs_clk=on te_seen=0 frame_start=miss underrun=1
   ```

   `frame_start=miss` 나 `underrun=1` 이면 합성 정상과 무관하게 panel ingress에서 첫 프레임이 버려질 수 있다.

4. **frame id를 CRC/capture/panel trace에 동일하게 묶는다**

   ```text
   DISPLAY_PIPE_FRAME display=2 frame=6412 wb_crc=0x8f13a4c2 capture=launcher_ui panel_ingress=miss
   ```

   세 경로의 frame id가 어긋나면 “정상 프레임이 어느 단계에서 사라졌는지”를 잘못 결론내리기 쉽다.

5. **최종 verdict를 post-comp vs panel-tail 두 축으로 한 줄 고정한다**

   ```text
   DISPLAY_PIPE_TAIL_VERDICT display=2 frame=6412 postcomp=ok panel_tail=fail cause=panel_ingress_miss cutline=postblend_to_panel
   DISPLAY_PIPE_TAIL_VERDICT display=2 frame=6412 postcomp=fail panel_tail=unknown cause=black_writeback cutline=reopen_day104
   ```

   이 verdict가 있어야 Day104의 composition 문제와 Day105의 panel-tail 문제를 분리한 채 소유권을 넘길 수 있다.

## 리스크

- writeback CRC 없이 panel trace만 보면 합성 black과 panel ingress fail이 섞인다.
- post-blend capture가 없으면 CRC non-black이어도 실제 UI가 맞는지, 단순 노이즈/old frame인지 확신하기 어렵다.
- panel ingress trace의 frame id 매핑이 틀리면 첫 프레임 drop과 다음 프레임 회복이 뒤바뀌어 보일 수 있다.
- writeback path 자체가 debug-only일 경우 성능 오버헤드로 재현 타이밍이 흔들릴 수 있다.

## 다음 액션

다음 글에서는 Day105 다음 절단면으로, **panel ingress까지 정상인데 첫 visible UI만 늦을 때 TE/vsync phase, first-frame release window, panel self-refresh exit 간 경쟁을 1-frame 단위로 자르는 방법** 을 정리하겠다.

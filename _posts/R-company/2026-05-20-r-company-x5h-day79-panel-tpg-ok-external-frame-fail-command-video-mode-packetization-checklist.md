---
title: R사 X5H Day79 - panel TPG는 보이는데 외부 frame만 안 보일 때 command/video mode·porch·packetization 체크리스트
author: JaeHa
date: 2026-05-20 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day79, display, panel, tpg, command-mode, video-mode, porch, packetization, dsi, bringup]
---

Day78에서 `panel internal test pattern/BIST` 가 정상적으로 보였다면 panel 전원·광량·내부 구동축은 대체로 살아 있다는 뜻이다. 이제 남는 축은 **SoC가 panel 입구까지 보내는 외부 frame이 panel이 기대한 전송 모드와 타이밍으로 들어오는가** 다. 이 단계에서는 `command mode ↔ video mode 전환`, `porch/timing`, `long packet payload/word count`, `sync event/burst 설정` 네 가지를 먼저 자르는 편이 가장 빠르다.

## 핵심 요약

- `TPG visible + external frame invisible` 조합이면 panel 자체보다 **external ingress contract mismatch** 가능성이 훨씬 높다.
- 1차 절단 순서는 **mode 전환 상태 확인 → lane/frame timing 확인 → packetization/word count 확인 → frame counter 변화 비교** 다.
- 특히 panel은 `video mode` 를 기대하는데 SoC가 `command mode refresh` 로 밀거나, `hactive/vactive` 는 맞아도 `hfp/hbp/hsync` 가 어긋나면 black screen이 계속된다.
- 최소 증적은 `DSI_MODE`, `DSI_TIMING`, `PKT_TRACE`, `FRAME_COUNTER` 네 줄이면 충분하다.

## 코드 포인트

1. **panel internal pattern이 보인다고 external mode contract까지 맞는 것은 아니다**

   ```text
   TPG_STATE panel=cluster enable=1 pattern=color_bar visible=1
   DSI_MODE panel=cluster expected=video-burst actual=command refresh=te-triggered mismatch=1
   ```

   Day78에서 internal pattern이 보였다는 사실은 panel 내부가 살아 있다는 의미일 뿐이다. 외부 frame 단계에서는 panel init 시점의 `mode switch` 가 제대로 반영됐는지 별도로 고정해야 한다.

2. **mode mismatch는 frame push 성공처럼 보이면서 실제 표시만 실패한다**

   ```text
   FRAME_PUSH panel=cluster seq=512 tx_bytes=8294400 eof=1
   DSI_MODE panel=cluster mode=command max_pkt=256 tear_sync=1
   PANEL_EXPECT panel=cluster ingress=video continuous_clock=1
   ```

   상위 stack에서는 `FRAME_PUSH eof=1` 이라 성공처럼 보이지만, panel이 video stream latch를 기대하면 command burst 전송은 실제 scanout으로 이어지지 않는다. 이 패턴이면 buffer나 CSC보다 `mode hand-off` 를 먼저 봐야 한다.

3. **porch/timing mismatch는 active 해상도가 맞아도 화면을 죽인다**

   ```text
   DSI_TIMING panel=cluster hact=1920 hfp=24 hbp=32 hsync=8 vact=720 vfp=12 vbp=18 vsync=4
   PANEL_SPEC panel=cluster hfp=40 hbp=40 hsync=4 vfp=8 vbp=20 vsync=2
   ```

   `1920x720` 같은 active 영역이 맞아도 `hfp/hbp/vsync` 가 spec과 어긋나면 panel이 frame boundary를 잘못 해석하거나 display-on 이후 blank 상태를 유지할 수 있다. bring-up에서는 active만 맞추고 porch를 대충 두는 실수가 자주 난다.

4. **packetization/word count mismatch는 lane live 상태에서도 조용히 숨어든다**

   ```text
   PKT_TRACE vc=0 dt=RGB888 long_pkt_wc=5760 expected_wc=5760 ecc=ok crc=ok
   PKT_TRACE vc=0 dt=RGB888 long_pkt_wc=5756 expected_wc=5760 ecc=ok crc=ok trunc=1
   ```

   ECC/CRC 가 깨지지 않아도 `word count` 가 4 byte만 모자라면 panel은 매 line 마지막을 잘못 자른다. 그 결과 전체 frame이 무효 처리되거나 검은 화면처럼 보일 수 있다. 특히 stride/packing 변환 뒤 `wc` 계산 누락이 흔하다.

5. **최종 판정은 external frame 투입 전후 frame counter 변화로 묶는 게 안전하다**

   ```text
   FRAME_COUNTER panel=cluster source=internal before=900 after=960 delta=60
   FRAME_COUNTER panel=cluster source=external before=960 after=960 delta=0
   ```

   internal pattern에서는 counter가 증가하는데 external frame에서만 멈춘다면, panel 내부 구동보다 ingress contract가 문제라는 증거가 된다. 이때는 `command/video mode`, `continuous clock`, `packet type`, `porch` 네 축을 우선 수정하는 편이 효율적이다.

## 리스크

- internal TPG가 보인다는 이유로 panel 이슈를 완전히 배제하면 `mode switch 미반영` 같은 panel ingress 조건을 놓칠 수 있다.
- active resolution만 맞추고 porch/timing을 느슨하게 두면 부팅 직후 간헐 black screen이나 특정 frame-rate에서만 실패하는 문제가 남는다.
- packet CRC/ECC만 보고 정상 판정하면 `word count`, `virtual channel`, `pixel packing` mismatch를 오래 못 잡는다.
- command/video mode 전환 책임이 bootloader, kernel panel driver, display FW 사이에 흩어져 있으면 수정 후에도 다른 단계가 다시 덮어써 재발하기 쉽다.

## 다음 액션

다음 글에서는 Day79를 이어서, **mode/timing/packetization까지 맞는데 외부 frame만 불안정할 때 continuous clock·ULPS exit·reset-after-mode-switch·first-frame discard race를 자르는 체크리스트** 를 정리하겠다.

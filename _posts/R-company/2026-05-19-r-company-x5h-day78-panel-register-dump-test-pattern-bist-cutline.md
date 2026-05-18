---
title: R사 X5H Day78 - panel register dump/test pattern/BIST로 SoC 출력 vs panel 내부 처리 절단하기
author: JaeHa
date: 2026-05-19 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day78, display, panel, register-dump, test-pattern, bist, dsi, black-screen, bringup]
---

Day77에서 `backlight PWM`, `TE sync`, `DSI lane state`, `reset/display-on sequence` 까지 정상이라면, 이제 남는 질문은 하나다. **SoC는 정상 frame을 panel 입구까지 보내고 있는데 panel 내부가 그 frame을 못 그리는가, 아니면 사실 panel 입구까지도 기대한 command/video 상태가 안 갔는가**. 이 구간은 `panel register dump`, `internal test pattern`, `BIST` 세 축으로 자르면 빠르다.

## 핵심 요약

- Day77까지 통과했는데 black screen이면, 다음 절단 순서는 **panel status register → internal test pattern/BIST → external frame path 비교** 가 가장 빠르다.
- panel 자체 color-bar/test pattern이 보이면, 전원/광량/소자 쪽은 대체로 살아 있고 **SoC video path 또는 panel command state mismatch** 가능성이 더 크다.
- 반대로 test pattern/BIST도 안 보이면, SoC 상위 stack보다 **panel 내부 state machine, register latch, analog/power rail** 쪽을 먼저 의심해야 한다.
- 최소 증적은 `PANEL_REG`, `PANEL_ERR`, `TPG_STATE`, `FRAME_COUNTER` 네 줄이면 된다.

## 코드 포인트

1. **display-on 직후 panel status register를 먼저 고정한다**

   ```text
   PANEL_REG panel=cluster cmd=0x0A value=0x9C meaning=display_on+normal_mode
   PANEL_REG panel=cluster cmd=0x0B value=0x00 meaning=no_error
   PANEL_REG panel=cluster cmd=0x0D value=0x06 meaning=booster_on+source_on
   ```

   Day77까지 live-link가 정상이어도 `0x29` 전송과 panel 내부 latch는 다를 수 있다. `display_on`, `sleep_out`, `error_status` 계열 register를 같이 남기면 **command path OK / panel internal state NG** 를 바로 분리할 수 있다.

2. **register는 'write success' 보다 'readback consistency' 가 더 중요하다**

   ```text
   PANEL_CMD_WRITE cmd=0x53 value=0x2C result=OK
   PANEL_REG panel=cluster cmd=0x53 value=0x00 expected=0x2C mismatch=1
   ```

   DSI write ACK만 보고 넘어가면 안 된다. readback 값이 다르면 lane은 살아 있어도 `page select`, `command mode timing`, `vendor unlock sequence` 가 틀린 상태일 수 있다.

3. **internal test pattern은 SoC frame path를 우회하는 가장 강한 절단선이다**

   ```text
   TPG_STATE panel=cluster enable=1 pattern=color_bar src=panel_internal
   SCREEN_OBS panel=cluster visible=1 colors=stable flicker=0
   ```

   panel 내부 color bar가 보이면 `backlight/power/gamma/source driver` 는 대체로 살아 있다. 이때 black screen 원인은 panel 자체보다 **SoC scanout payload 또는 command/video mode 전환** 에 남는다.

4. **BIST는 panel이 frame counter를 스스로 증가시키는지 확인해 준다**

   ```text
   BIST_ENABLE panel=cluster mode=internal-ramp result=OK
   FRAME_COUNTER panel=cluster before=381 after=447 delta=66
   ```

   `FRAME_COUNTER` 가 내부 패턴에서만 증가하고 외부 frame push 때는 멈춘다면, panel 내부 refresh는 가능하지만 외부 ingress가 막히는 것이다. 이 경우 `video mode entry`, `porch/timing`, `payload packetization` 쪽이 더 유력하다.

5. **test pattern이 안 보이면 analog/power rail 쪽으로 즉시 내려가야 한다**

   ```text
   TPG_STATE panel=cluster enable=1 pattern=white result=OK
   SCREEN_OBS panel=cluster visible=0
   PANEL_ERR panel=cluster vgh_uv=1 vgl_uv=0 source_fault=1 gate_fault=0
   ```

   internal pattern까지 invisible이면 compositor나 buffer 문제를 더 파도 시간만 잃는다. 이때는 `VGH/VGL/AVDD/IOVCC`, source/gate driver fault, panel vendor self-check를 바로 붙여야 한다.

## 리스크

- panel command write ACK만 보고 register readback을 안 남기면 vendor page/unlock 누락을 놓쳐서 SoC 쪽을 괜히 오래 의심하게 된다.
- internal test pattern/BIST를 안 쓰면 SoC payload 문제와 panel 내부 처리 문제를 분리하지 못해 디버깅 시간이 크게 늘어난다.
- test pattern visible 여부만 보고 끝내면 안 되고, **외부 frame 복귀 후 frame counter/err register 변화** 까지 같이 봐야 한다.
- panel마다 readback 가능 register와 BIST 진입 조건이 달라서, vendor sequence를 문서화하지 않으면 재현성이 떨어진다.

## 다음 액션

다음 글에서는 Day78을 이어서, **panel internal test pattern은 보이는데 외부 frame만 안 보일 때 command-mode/video-mode 전환, porch/timing, packetization mismatch를 1차로 자르는 체크리스트** 를 정리하겠다.

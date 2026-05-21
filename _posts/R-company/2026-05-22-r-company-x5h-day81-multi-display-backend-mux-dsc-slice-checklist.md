---
title: R사 X5H Day81 - panel link 정상 후 남는 multi-display backend mux·clone·DSC slice 체크리스트
author: JaeHa
date: 2026-05-22 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day81, display, multi-display, backend-mux, clone, splitter, dsc, bringup]
---

Day80까지 확인해 `continuous clock`, `ULPS exit`, `reset-after-mode-switch`, `first-frame acceptance` 가 모두 정상인데도 특정 panel이나 특정 display path에서만 화면이 안 나오면, 이제는 **single-panel ingress 문제보다 display fabric routing 문제** 를 먼저 의심해야 한다. X5H 계열 bring-up 후반에서는 `frontend plane` 은 살아 있는데 `backend mux`, `clone/splitter selection`, `DSC slice allocation` 이 어긋나 특정 출력만 검게 남는 패턴이 자주 보인다.

## 핵심 요약

- Day80을 통과했는데 특정 output만 죽어 있으면 1차 절단 순서는 **backend mux 연결 → clone/splitter fanout → DSC slice/map → per-display enable/order** 다.
- `flush/vblank/first-frame accept` 가 살아 있어도 backend가 다른 output에 묶여 있으면 panel 입장에서는 유효 frame을 한 번도 못 받는다.
- dual-display 또는 wide panel 구성에서는 `DSC slice count`, `slice boundary`, `left/right half ownership` 이 한 칸만 틀어져도 한쪽 black screen 또는 전체 discard가 난다.
- 최소 증적은 `DISPLAY_ROUTE`, `BACKEND_MUX`, `DSC_MAP`, `OUTPUT_ENABLE_SEQ` 네 줄이면 충분하다.

## 코드 포인트

1. **frontend 성공과 실제 output route는 별개로 고정해야 한다**

   ```text
   DISPLAY_ROUTE crtc=1 plane=video0 frontend=rvgc0 backend=vc1 expected_panel=cluster actual_panel=center mismatch=1
   BACKEND_MUX vc1 src=rvgc0 sink=dsi1 active=1
   ```

   plane reserve와 flush가 정상이어도 backend sink가 다른 DSI/eDP 포트로 묶여 있으면 현재 panel은 아무 것도 못 받는다. `frontend alive` 만으로 표시 경로를 정상 판정하면 안 된다.

2. **clone/splitter 경로는 '한쪽만 보임' 패턴을 만든다**

   ```text
   CLONE_STATE group=main_cluster mode=split left=dsi0 right=dsi1 frame_seq=901
   SPLITTER_STATUS group=main_cluster left_push=1 right_push=0 right_block_reason=late_enable
   ```

   cluster panel이나 dual-link 경로에서 좌/우 half 중 하나만 enable되면 panel 내부 recombine이 실패해 전체 black으로 보일 수 있다. 이때는 한쪽 lane이 살아 있어도 단순 link issue로 오해하기 쉽다.

3. **DSC slice map 불일치는 payload가 멀쩡해 보여도 panel discard를 유발한다**

   ```text
   DSC_MAP panel=cluster slices_expected=4 slices_actual=2 slice_width=480 chunk=960
   DSC_RC panel=cluster pps_crc=0x82af expected_crc=0x6c31 mismatch=1
   ```

   DSC가 들어간 경로는 `slice count`, `slice width`, `PPS` 중 하나만 달라도 panel이 frame 전체를 버릴 수 있다. 특히 single-display에서는 괜찮다가 split-path에서만 실패하면 DSC map을 먼저 봐야 한다.

4. **output enable 순서가 틀리면 route는 맞아도 첫 steady-state를 못 만든다**

   ```text
   OUTPUT_ENABLE_SEQ panel=cluster dsi0=on dsi1=off splitter=on backlight=on ts=77120
   OUTPUT_ENABLE_SEQ panel=cluster dsi1=on ts=77380 gap_ms=260
   ```

   split/clone 구성에서는 관련 output이 거의 동시에 올라와야 한다. enable gap이 길면 먼저 올라온 쪽 frame이 버려지거나 panel이 lock 실패 상태에 머문다. Day80의 first-frame race가 fabric 차원으로 확장된 형태다.

5. **최종 판정은 route, slice, enable 순서를 한 줄로 묶는 편이 가장 빠르다**

   ```text
   DISPLAY_FABRIC_TIMELINE route_ok=77010 splitter_on=77040 dsc_pps_ok=77066 dsi0_on=77120 dsi1_on=77380 first_visible=none
   ```

   이 타임라인이 있으면 `route는 맞지만 secondary output enable이 늦다`, `splitter는 켜졌지만 DSC PPS가 다르다`, `한쪽 backend가 다른 sink로 샌다` 를 한 번에 자를 수 있다.

## 리스크

- single-panel 기준으로만 debug하면 multi-display fabric misroute를 늦게 발견해 link/panel 쪽에서 시간을 낭비한다.
- clone/splitter 경로는 간헐적으로 한 번 보였다가 다시 안 보이는 재현성을 만들기 쉬워 cold/warm boot 차이와 섞여 오판하기 쉽다.
- DSC slice/PPS mismatch는 CRC/ECC가 멀쩡해 보여도 숨어들 수 있어 packet layer 정상만으로는 안전하지 않다.
- backend ownership이 bootloader, display FW, kernel driver에 분산돼 있으면 한 단계 수정 후 다른 단계가 다시 mux를 덮어써 회귀할 수 있다.

## 다음 액션

다음 글에서는 Day81을 이어서, **multi-display route까지 정상인데 Android/HWC layer assignment와 virtual display/mirror 정책 때문에 실제 target panel만 비는 경우를 자르는 체크리스트** 를 정리하겠다.

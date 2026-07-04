---
title: R사 X5H Day106 - TE/vsync phase·first-frame release window·panel self-refresh exit residual black-frame cutline
author: JaeHa
date: 2026-07-05 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day106, display, te, vsync, firstframe, selfrefresh, blackframe, bringup]
---

Day105에서 `WRITEBACK_CRC`, `POSTBLEND_CAPTURE`, `PANEL_INGRESS_TRACE` 가 모두 정상인데도 사용자가 첫 visible UI에서 black 1-frame을 본다면, 이제는 **패널 ingress 이후의 표시 타이밍 경쟁** 을 잘라야 한다. 핵심은 첫 유효 프레임이 패널에 들어왔더라도 **TE/vsync 위상, first-frame release window, panel self-refresh exit** 중 하나가 어긋나면 실제 노출은 한 beat 늦어질 수 있다는 점이다.

## 핵심 요약

- Day106의 절단 순서는 **TE/vsync phase 확인 → first-frame release window 확인 → panel self-refresh exit 확인 → residual black-frame verdict 고정** 이다.
- Day105가 정상인데 black 1-frame이 남으면 원인은 composition이나 ingress보다 **표시 타이밍 경계** 쪽일 가능성이 높다.
- 최소 증적은 `TE_PHASE_TRACE`, `FIRST_FRAME_RELEASE`, `PSR_EXIT_TRACE`, `VISIBLE_UI_TIMING_VERDICT` 네 줄이면 충분하다.
- 이 cutline이 나오면 SurfaceFlinger보다 **display timing owner / panel power-sequence owner** 가 먼저 받아야 한다.

## 코드 포인트

1. **TE/vsync 위상이 첫 유효 frame과 같은 beat에 맞았는지 먼저 남긴다**

   ```text
   TE_PHASE_TRACE display=2 frame=6412 te_seq=8801 present_seq=8801 phase=aligned delta_us=312
   TE_PHASE_TRACE display=2 frame=6412 te_seq=8801 present_seq=8802 phase=missed_one_beat delta_us=17102
   ```

   `phase=missed_one_beat` 면 frame ingress 자체는 정상이어도 panel scan start가 다음 beat로 밀리며 black 1-frame처럼 보일 수 있다.

2. **first-frame release window에서 fence 해제와 panel latch 시점이 겹쳤는지 확인한다**

   ```text
   FIRST_FRAME_RELEASE display=2 frame=6412 acquire=done release=armed latch_window=hit present_ready=1
   FIRST_FRAME_RELEASE display=2 frame=6412 acquire=done release=late latch_window=miss present_ready=0
   ```

   `release=late` 또는 `latch_window=miss` 면 HWC가 프레임을 만들었어도 패널이 그 beat에서는 이전 black 상태를 유지할 수 있다.

3. **panel self-refresh exit가 첫 visible UI 이전에 끝났는지 확인한다**

   ```text
   PSR_EXIT_TRACE display=2 frame=6412 psr_state=exit_done exit_us=420 blank_hold=0
   PSR_EXIT_TRACE display=2 frame=6412 psr_state=exit_inflight exit_us=18430 blank_hold=1
   ```

   `blank_hold=1` 이면 panel ingress가 정상이어도 self-refresh exit 보호 구간 때문에 첫 UI가 한 프레임 늦게 드러날 수 있다.

4. **세 신호를 같은 frame/present sequence로 묶어야 오판을 막을 수 있다**

   ```text
   FIRST_VISIBLE_UI_TRACE display=2 frame=6412 present_seq=8801 te_phase=aligned release_window=hit psr=exit_done visible=1
   FIRST_VISIBLE_UI_TRACE display=2 frame=6412 present_seq=8801 te_phase=missed_one_beat release_window=miss psr=exit_inflight visible=0
   ```

   frame id만 같고 present sequence가 다르면 실제 black 1-frame 원인을 잘못 넘겨짚기 쉽다.

5. **최종 verdict를 timing owner 기준으로 한 줄 고정한다**

   ```text
   VISIBLE_UI_TIMING_VERDICT display=2 frame=6412 composition=ok ingress=ok visible=late cause=te_phase_miss owner=panel_timing
   VISIBLE_UI_TIMING_VERDICT display=2 frame=6412 composition=ok ingress=ok visible=late cause=psr_exit_hold owner=panel_power_seq
   ```

   verdict가 있어야 Day105의 ingress 문제와 Day106의 timing 문제를 분리해서 후속 조치를 맡길 수 있다.

## 리스크

- TE/vsync phase를 남기지 않으면 panel ingress 정상인데도 black ingress처럼 오판할 수 있다.
- first-frame release window 계측이 없으면 fence starvation과 panel latch miss를 구분하지 못한다.
- self-refresh exit trace가 없으면 전력 절감 복귀 지연이 display backend 결함으로 잘못 분류될 수 있다.
- frame/present sequence 정렬이 틀리면 실제 1-frame slip보다 더 큰 jitter로 해석될 위험이 있다.

## 다음 액션

다음 글에서는 Day106 다음 절단면으로, **timing path까지 정상인데도 사용자가 여전히 검은 첫 화면을 볼 때 backlight unblank, local dimming release, display power-mode handoff 중 무엇이 마지막으로 화면을 가리는지** 를 자르는 체크포인트를 정리하겠다.

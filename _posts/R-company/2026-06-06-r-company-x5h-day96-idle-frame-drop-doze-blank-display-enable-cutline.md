---
title: R사 X5H Day96 - idle frame drop·doze/blank transition·display enable sequencing first-frame discard cutline
author: JaeHa
date: 2026-06-06 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day96, display, doze, blank, enable, firstframe, bringup]
---

Day95에서 `dataspace/alpha/blend/color mode` 까지 정상인데도 **첫 앱 프레임 한 장만 사라지고 다음 프레임부터 바로 보인다면**, 이제 잘라야 할 구간은 `display power state → blank/unblank transition → first present accept/drop` 이다. 즉 buffer도 정상이고 합성 속성도 맞지만 **display enable sequencing이 아직 완료되지 않아 첫 present가 idle/drop 처리되는지**를 확인해야 한다.

## 핵심 요약

- Day96의 1차 절단 순서는 **display state transition 확정 → first present 수용 여부 확인 → idle/drop reason 분리 → 다음 frame 회복 여부 판정** 이다.
- 첫 non-black frame이 producer/HWC까지 정상 도달해도 `blank pending`, `doze exit incomplete`, `display enable ack late`, `idle frame suppression` 중 하나면 사용자 눈에는 black flash로 보인다.
- 최소 증적은 `DISPLAY_STATE_TRANSITION`, `FIRST_PRESENT_ACCEPT`, `FRAME_DROP_REASON`, `NEXT_FRAME_RECOVERY` 네 줄이면 충분하다.
- 이 구간에서 막히면 합성 내용이 아니라 `전원/상태 전이 완료 전에 첫 frame을 버리는 정책` 문제일 가능성이 높다.

## 코드 포인트

1. **display pipeline이 실제로 visible state까지 올라왔는지 먼저 고정한다**

   ```text
   DISPLAY_STATE_TRANSITION display=2 old=DOZE_SUSPEND new=ON blanked=0 lanes=up enable_seq=done ts=4123312
   DISPLAY_STATE_TRANSITION display=2 old=DOZE_SUSPEND new=ON blanked=1 lanes=up enable_seq=pending ts=4123312
   ```

   `new=ON` 만 보면 안 된다. `blanked=1` 이거나 `enable_seq=pending` 이면 첫 present는 이미 제출됐어도 panel 쪽에서는 아직 수용 불가일 수 있다.

2. **첫 app present가 실제 scanout 후보로 accepted 되었는지 본다**

   ```text
   FIRST_PRESENT_ACCEPT display=2 frame=6314 layer_stack=app accepted=1 retire_fence=18 ready_for_scanout=1
   FIRST_PRESENT_ACCEPT display=2 frame=6314 layer_stack=app accepted=0 retire_fence=-1 ready_for_scanout=0 reason=display_not_unblanked
   ```

   `presentDisplay()` 성공만으로 충분하지 않다. vendor HWC나 backend에서 첫 frame을 `accepted=0` 으로 버릴 수 있으니 scanout 후보 등록 여부를 따로 남겨야 한다.

3. **idle/drop 정책이 첫 frame 한 장만 누락시키는지 분리한다**

   ```text
   FRAME_DROP_REASON display=2 frame=6314 reason=idle_suppression idle_budget_ms=16 last_kick_ms=0
   FRAME_DROP_REASON display=2 frame=6314 reason=doze_exit_guard guard_frames=1 remaining=1
   ```

   panel 보호 정책이나 bandwidth 절감 로직이 `guard frame` 을 하나 두는 구현이면 첫 앱 프레임만 의도치 않게 폐기된다. 이 경우 black frame이라기보다 `frame never scanned out` 에 가깝다.

4. **enable ACK와 first vblank의 상대 순서를 고정한다**

   ```text
   DISPLAY_ENABLE_ACK display=2 frame=6314 ack_ts=4123321 first_vblank_ts=4123338 first_present_ts=4123315
   DISPLAY_ENABLE_ACK display=2 frame=6314 ack_ts=4123346 first_vblank_ts=4123359 first_present_ts=4123315
   ```

   `first_present_ts < ack_ts` 이면 첫 frame은 살아 있어도 실제 hw enable 완료 전에 제출된 것이다. 이 패턴이면 다음 frame부터 정상 보이는 이유가 설명된다.

5. **다음 frame 회복까지 묶어 first-frame discard로 판정한다**

   ```text
   NEXT_FRAME_RECOVERY display=2 dropped_frame=6314 visible_frame=6315 cause=enable_ack_late recovered=1
   NEXT_FRAME_RECOVERY display=2 dropped_frame=6314 visible_frame=6316 cause=doze_guard_frames recovered=1
   ```

   회복 프레임 번호가 바로 이어지면 panel 지속 장애가 아니라 상태 전이 경계에서의 discard 문제다. recovery offset이 1~2 frame이면 우선 `display state machine` 쪽을 본다.

## 리스크

- `present 성공`만 보고 실제 scanout acceptance를 확인하지 않으면 first-frame discard를 정상 합성으로 오판한다.
- doze/blank/unblank state와 enable ACK를 같은 타임라인으로 묶지 않으면 전원 전이 문제를 fence 지연으로 착각하기 쉽다.
- idle suppression 로직은 성능 최적화 코드에 숨어 있는 경우가 많아 steady-state dump만으로는 원인이 드러나지 않는다.
- multi-display 환경에서는 특정 display만 guard frame 정책이 달라 재현 조건이 흔들릴 수 있다.

## 다음 액션

다음 글에서는 Day96 다음 절단면으로, **display state는 ON이고 첫 present도 accepted인데 여전히 첫 장만 안 보일 때 writeback/CRC/TE/VBLANK 관측으로 실제 scanout 여부를 확인하는 최종 visible-pixel cutline** 을 정리하겠다.

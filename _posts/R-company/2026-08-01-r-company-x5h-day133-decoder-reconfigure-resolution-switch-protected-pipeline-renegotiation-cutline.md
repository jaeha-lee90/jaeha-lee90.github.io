---
title: R사 X5H Day133 - decoder reconfigure·resolution switch·protected pipeline renegotiation cutline
author: JaeHa
date: 2026-08-01 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day133, display, protected, decoder, reconfigure, resolution-switch, bringup]
---

Day132에서 `release fence`, `surface recycle`, `protected lifetime` 까지 정상인데도 **재생 중간 해상도 전환이나 adaptive bitrate 직후만 black glitch가 반복되면**, 이제 절단면은 `decoder reconfigure`, `resolution switch`, `protected pipeline renegotiation` 이다. 즉 steady-state 자체는 살아 있지만, **stream parameter 변경 순간 secure path의 버퍼/plane/세션 계약이 다시 맞물리지 않아 검은 프레임이 끼는지** 를 잘라야 한다.

## 핵심 요약

- Day133의 핵심 순서는 **codec reconfigure 시작점 고정 → 새 해상도/stride/modifier 계약 반영 시점 확인 → protected import 재협상 지연 분리 → switch-window black glitch verdict 고정** 이다.
- secure playback에서는 `INFO_OUTPUT_FORMAT_CHANGED`, `surface generation mismatch`, `protected plane capability 재판정 지연`, `old/new session buffer 혼재` 중 하나만 있어도 전환 직후 black burst가 생길 수 있다.
- `DECODER_RECONFIG_TRACE`, `FORMAT_SWITCH_TRACE`, `PROTECTED_RENEGOTIATION_TRACE`, `SWITCH_GLITCH_TRACE`, `FINAL_SWITCH_VERDICT` 를 같은 `session_id`, `switch_seq`, `buffer_id`, `generation` 으로 묶어야 한다.
- 이 단계의 목적은 **steady-state backpressure 문제(Day132)** 와 **parameter switch 시점 계약 재수립 실패** 를 분리하는 것이다.

## 코드 포인트

1. **codec이 실제로 reconfigure에 들어간 시점을 먼저 고정한다**

   ```text
   DECODER_RECONFIG_TRACE session=42 switch_seq=7 event=format_changed old=1920x1080 new=1280x720 secure=1 generation=18
   DECODER_RECONFIG_TRACE session=42 switch_seq=7 event=output_port_reset buffers_outstanding=2 drain_required=1
   ```

   black glitch가 해상도 전환 직후에만 보인다면, 시작점은 display가 아니라 codec의 `format_changed` 이벤트다.

2. **새 출력 포맷 계약이 BufferQueue/HWC까지 같은 세대로 반영됐는지 본다**

   ```text
   FORMAT_SWITCH_TRACE switch_seq=7 layer=Video#12 generation=18 queued=0xA1 1280x720 stride=1536 modifier=afbc
   FORMAT_SWITCH_TRACE switch_seq=7 layer=Video#12 generation=17 latched=0x97 1920x1080 stride=2048 modifier=linear stale=1
   ```

   old generation buffer가 latch 경로에 남아 있으면 secure path는 보호정책상 안전하게 검은 프레임으로 빠질 가능성이 높다.

3. **protected import/plane capability 재협상이 늦는지 분리한다**

   ```text
   PROTECTED_RENEGOTIATION_TRACE switch_seq=7 buffer_id=0xA1 import=ok protected=1 modifier=afbc plane_cap_pending=1
   PROTECTED_RENEGOTIATION_TRACE switch_seq=7 buffer_id=0xA1 import=ok protected=1 modifier=afbc plane_cap_pending=0 result=commit_ok
   ```

   decoder는 새 버퍼를 냈지만 HWC plane 쪽 capability 재판정이 늦으면 한두 프레임 동안 fallback 금지 상태의 black hole이 생긴다.

4. **switch window 동안 old/new buffer 혼재 여부를 기록한다**

   ```text
   SWITCH_GLITCH_TRACE session=42 switch_seq=7 old_gen=17 new_gen=18 visible=0 black_ms=66 reason=mixed_generation_protected_buffers
   SWITCH_GLITCH_TRACE session=42 switch_seq=7 old_gen=17 new_gen=18 visible=1 black_ms=0 reason=cleared
   ```

   secure playback은 세대가 다른 버퍼를 느슨하게 섞어 보여주기보다 차라리 안 보여주는 쪽으로 구현된 경우가 많아서, 혼재 window를 명시적으로 잡아야 한다.

5. **최종 owner를 reconfigure contract 기준으로 귀속한다**

   ```text
   FINAL_SWITCH_VERDICT session=42 symptom=post_switch_black owner=decoder_reconfigure cause=output_port_reset_gap
   FINAL_SWITCH_VERDICT session=43 symptom=post_switch_black owner=protected_plane_contract cause=capability_renegotiation_late
   FINAL_SWITCH_VERDICT session=44 symptom=post_switch_black owner=buffer_generation_control cause=old_new_protected_mix
   ```

   이렇게 남겨야 Day132의 release backpressure와 Day133의 switch-time contract mismatch를 분리할 수 있다.

## 리스크

- adaptive playback이 자주 일어나는 스트림은 문제가 재현되어도 steady-state 구간이 짧아 원인 분리가 더 어렵다.
- secure decoder는 재configure 중 buffer dump가 제한되어 generation bookkeeping이 틀리면 false verdict가 나기 쉽다.
- vendor HWC가 modifier/plane capability 재판정 로그를 안 남기면 codec 문제로 과오인할 가능성이 높다.
- resolution switch와 HDCP/session refresh가 동시에 겹치면 renegotiation owner가 둘로 나뉘어 보일 수 있다.

## 다음 액션

다음 글에서는 Day133 다음 절단면으로, **secure playback switch 이후에도 남는 subtitle/UI overlay reattach·trusted composition 정책·protected/unprotected layer 재합성 경로** 를 정리하겠다.

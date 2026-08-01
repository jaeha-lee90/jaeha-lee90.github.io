---
title: R사 X5H Day134 - subtitle/UI overlay reattach·trusted composition policy·protected/unprotected recompose cutline
author: JaeHa
date: 2026-08-02 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day134, display, protected, subtitle, overlay, trusted-composition, bringup]
---

Day133에서 `decoder reconfigure`, `generation switch`, `protected pipeline renegotiation` 까지 정상인데도 **해상도 전환 직후 subtitle, playback UI, trusted overlay가 다시 붙는 순간만 black frame 또는 video hide가 생기면**, 이제 절단면은 `overlay reattach`, `trusted composition policy`, `protected/unprotected layer recompose` 다. 즉 secure video plane 자체는 살아 있지만, **보호 레이어와 비보호 레이어를 다시 한 화면으로 합치는 정책이 틀어져 전체 visible set이 검게 떨어지는지** 를 잘라야 한다.

## 핵심 요약

- Day134의 핵심 순서는 **video layer는 계속 visible인지 고정 → subtitle/UI/trusted overlay 재attach 시점 기록 → protected/unprotected 혼합 정책 위반 여부 분리 → recomposition owner verdict 고정** 이다.
- secure playback에서는 `subtitle layer reparent`, `trusted overlay promote`, `GPU client target fallback 금지`, `secure display policy mismatch` 중 하나만 있어도 비디오는 decode되고 있어도 최종 화면은 black 또는 video hidden으로 보일 수 있다.
- `VIDEO_VISIBILITY_TRACE`, `OVERLAY_REATTACH_TRACE`, `TRUSTED_COMPOSE_TRACE`, `RECOMPOSE_POLICY_TRACE`, `FINAL_MIXED_COMPOSE_VERDICT` 를 같은 `session_id`, `switch_seq`, `layer_id`, `display_id` 로 묶어야 한다.
- 이 단계의 목적은 **Day133의 secure pipeline 재협상 성공 이후** 남는 문제를, `video path` 가 아니라 **mixed composition policy** 쪽으로 귀속하는 것이다.

## 코드 포인트

1. **먼저 protected video layer가 전환 뒤에도 계속 살아 있는지 고정한다**

   ```text
   VIDEO_VISIBILITY_TRACE session=42 switch_seq=8 layer=Video#12 protected=1 visible=1 z=4 plane=DISP1-V0
   VIDEO_VISIBILITY_TRACE session=42 switch_seq=8 layer=Video#12 protected=1 visible=1 crop=1280x720 alpha=1.0
   ```

   subtitle/UI가 붙는 순간 검어져도 video layer 자체가 계속 visible이면 decoder나 panel보다 composition policy를 먼저 봐야 한다.

2. **subtitle/UI/trusted overlay가 언제 어떤 parent로 재attach되는지 기록한다**

   ```text
   OVERLAY_REATTACH_TRACE switch_seq=8 layer=Subtitle#31 type=buffer parent=Task#A trusted=0 attach=ok
   OVERLAY_REATTACH_TRACE switch_seq=8 layer=PlaybackUI#44 type=color parent=RootTask#0 trusted=1 attach=ok
   ```

   reconfigure 뒤 overlay layer가 새 leash나 새 task parent로 이동하면, secure video와 섞이는 composition 규칙이 한 프레임 동안 달라질 수 있다.

3. **trusted composition 정책 때문에 GPU fallback이 금지되는지 분리한다**

   ```text
   TRUSTED_COMPOSE_TRACE switch_seq=8 display=cluster secure_video=1 overlay=Subtitle#31 trusted=0 gpu_fallback=blocked reason=secure_mix_forbidden
   TRUSTED_COMPOSE_TRACE switch_seq=8 display=cluster secure_video=1 overlay=PlaybackUI#44 trusted=1 gpu_fallback=allowed result=device_compose
   ```

   secure video 위에 비신뢰 overlay가 올라오면, 어떤 플랫폼은 client composition으로 못 내리고 그냥 video hide 또는 black fill로 처리한다.

4. **protected/unprotected 레이어 혼합 시 recomposition decision을 같은 프레임에서 묶는다**

   ```text
   RECOMPOSE_POLICY_TRACE frame=2281 layer_stack=video+subtitle secure_layers=1 unsecure_layers=1 decision=reject visible_set=0
   RECOMPOSE_POLICY_TRACE frame=2282 layer_stack=video+trusted_ui secure_layers=1 unsecure_layers=0 decision=accept visible_set=1
   ```

   핵심은 `video decode ok` 가 아니라 `최종 layer stack verdict` 다. 같은 switch window에서 어떤 레이어 조합은 통과하고 어떤 조합은 막히는지 남겨야 한다.

5. **최종 owner를 mixed composition contract 기준으로 귀속한다**

   ```text
   FINAL_MIXED_COMPOSE_VERDICT session=42 symptom=post_switch_black owner=subtitle_policy cause=untrusted_overlay_on_secure_video
   FINAL_MIXED_COMPOSE_VERDICT session=43 symptom=post_switch_black owner=trusted_ui_hierarchy cause=overlay_reparent_to_nonsecure_root
   FINAL_MIXED_COMPOSE_VERDICT session=44 symptom=post_switch_black owner=hwc_compose_policy cause=gpu_fallback_blocked_for_secure_mix
   ```

   이렇게 남겨야 Day133의 renegotiation 문제와 Day134의 mixed-layer policy 문제를 분리할 수 있다.

## 리스크

- subtitle/UI는 영상보다 늦게 attach되어 재현 창이 짧으므로 trace window를 switch 직후 1~2초까지 넓혀야 한다.
- vendor HWC가 `reject reason` 을 자세히 남기지 않으면 black frame이 decoder stall처럼 잘못 해석될 수 있다.
- trusted overlay 정의가 product policy마다 달라 같은 앱이라도 build variant에 따라 결과가 바뀔 수 있다.
- screenshot/capture가 막힌 secure product에서는 visible 여부를 writeback 대신 layer-state와 HWC verdict 조합으로만 추적해야 한다.

## 다음 액션

다음 글에서는 Day134 다음 절단면으로, **secure video 위 subtitle/UI 혼합은 허용되는데 특정 display clone/cast/PIP 경로에서만 실패하는 mirror·virtual display·secondary output policy cutline** 을 정리하겠다.

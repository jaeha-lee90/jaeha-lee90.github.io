---
title: R사 X5H Day129 - secure session·HDCP·display policy final black-frame cutline
author: JaeHa
date: 2026-07-28 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day129, display, protected, tee, hdcp, bringup]
---

Day128에서 `YUV format`, `chroma siting`, `CSC/range` 까지 맞는데도 **secure video만 끝까지 black frame** 이면 이제 절단면은 `TEE session`, `decrypt state`, `content protection policy`, `HDCP/auth state`, `secure display routing` 이다. 즉 pixel contract는 맞지만, **콘텐츠 보호 상태기가 ready가 아니어서 backend가 의도적으로 scanout을 막는지** 를 잘라야 한다.

## 핵심 요약

- Day129의 핵심 순서는 **decoder crypto session 확인 → secure buffer/usage 비트 확인 → display protection state와 HDCP 연동 확인 → 최종 blank owner 확정** 이다.
- secure playback에서는 `Widevine/TEE session not ready`, `key ladder 지연`, `HDCP 미인증`, `internal-only secure display policy`, `protected path break` 중 하나라도 걸리면 format이 맞아도 최종 출력은 black frame이 된다.
- `SECURE_SESSION_TRACE`, `CRYPTO_STATUS_TRACE`, `DISPLAY_PROTECTION_TRACE`, `HDCP_STATE_TRACE`, `FINAL_SECURE_POLICY_VERDICT` 를 같은 `session_id`, `buffer_id`, `display`, `frame` 으로 묶어야 한다.
- 최소 증적은 `session_state`, `key_status`, `buffer_protected`, `display_protection_state`, `blank_reason` 다섯 축이다.

## 코드 포인트

1. **TEE/DRM crypto session이 실제로 ready인지 먼저 고정한다**

   ```text
   SECURE_SESSION_TRACE session_id=0x44a drm=widevine level=L1 tee=optee state=ready key_slot=7
   CRYPTO_STATUS_TRACE frame=9812 session_id=0x44a decrypt=ok key_valid=1 subsample_err=0
   ```

   decoder 인스턴스가 살아 있어도 key ladder나 secure world session attach가 늦으면 첫 수 프레임은 정상 decode처럼 보여도 display hand-off 시점에서 보호 경로가 끊긴다.

2. **buffer usage/heap/handle이 protected contract를 끝까지 유지하는지 본다**

   ```text
   DISPLAY_PROTECTION_TRACE buffer_id=0xb811 protected=1 usage=0x40200 heap=secure_carveout import=ok protected_path=0
   ```

   gralloc allocate 시점에는 `protected=1` 이었는데 import 후 `protected_path=0` 으로 내려가면 HWC/backend는 의도적으로 blank 처리할 수 있다.

3. **display policy가 외부/가상 display를 차단하는지 분리 기록한다**

   ```text
   DISPLAY_PROTECTION_TRACE display=HDMI-A-1 protected_content=1 sink_allowed=0 policy=internal_only action=force_black
   ```

   secure content는 panel에는 나오지만 HDMI, wireless display, virtual display로는 금지되는 경우가 많다. clone/mirror 정책과 섞이면 단순 black screen처럼 오해하기 쉽다.

4. **HDCP/auth state를 frame blackout과 직접 연결한다**

   ```text
   HDCP_STATE_TRACE display=HDMI-A-1 state=auth_pending version=2.3 link_integrity=0 blanking=enabled
   HDCP_STATE_TRACE display=HDMI-A-1 state=authenticated version=2.3 link_integrity=1 blanking=disabled
   ```

   sink hotplug 직후나 re-auth 중에는 plane programming이 정상이어도 backend가 콘텐츠를 흑화한다. 이때는 display pipe 이슈가 아니라 policy 이슈로 귀속해야 한다.

5. **최종 black-frame owner를 보호 상태기로 귀속한다**

   ```text
   FINAL_SECURE_POLICY_VERDICT frame=9812 display=1 cause=hdcp_not_authenticated owner=display_policy symptom=black_frame
   FINAL_SECURE_POLICY_VERDICT frame=9824 display=0 cause=tee_session_attach_late owner=crypto_session symptom=first_frames_blacked
   ```

   이렇게 남겨야 Day128의 pixel-format 이슈와 Day129의 content-protection 이슈를 분리해 수정 우선순위를 정확히 잡을 수 있다.

## 리스크

- secure world/DRM 관련 로그는 릴리즈 빌드에서 축약되거나 masking 되어 session state를 직접 보기 어렵다.
- HDCP 재인증은 케이블/싱크 상태에 따라 간헐적으로만 터져 재현성 낮은 black frame처럼 보일 수 있다.
- 일부 vendor stack은 `policy block` 과 `decode failure` 를 동일한 generic error로 표기해 owner 분리가 어렵다.
- internal panel, HDMI, virtual display가 동시에 얽히면 어느 경로가 정책으로 막혔는지 trace 상관관계를 먼저 정리하지 않으면 triage 시간이 길어진다.

## 다음 액션

다음 글에서는 Day129 다음 절단면으로, **secure session/HDCP/policy까지 맞는데도 남는 경우 compositor fallback·screen capture block·writeback/clone path 단절이 사용자 체감 black frame으로 이어지는 경로** 를 정리하겠다.

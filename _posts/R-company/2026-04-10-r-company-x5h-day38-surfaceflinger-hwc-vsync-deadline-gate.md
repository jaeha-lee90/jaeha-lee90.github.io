---
title: R사 X5H Day38 - SurfaceFlinger/HWC/VSync deadline 안정화 게이트
author: JaeHa
date: 2026-04-10 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day38, SurfaceFlinger, HWC, VSync, deadline, frame, bringup]
---

Day38은 Day37의 ANR/watchdog 다음 단계로, **프로세스는 응답하지만 프레임 deadline을 반복적으로 놓쳐 실제 UX가 깨지는 상태**를 다룬다. X5H bring-up에서는 `system_server`가 살아 있고 binder timeout도 제한적이어도, SurfaceFlinger-HWC-vendor display stack의 타이밍 불일치가 남아 있으면 검은 화면 복귀 지연, 첫 홈 화면 stutter, 입력 후 표시 지연이 길게 이어진다.

## 핵심 요약

- post-boot 안정화는 ANR 0건만으로 충분하지 않다. **SurfaceFlinger/HWC/VSync deadline miss**도 별도 게이트로 관리해야 한다.
- display 경로는 crash보다 **프레임 생성-합성-present가 주기 안에 끝나는지**가 더 중요하다.
- X5H처럼 Xen, vendor display HAL, GPU/DMABUF 경계가 긴 구조는 binder timeout보다 **frame pacing 붕괴**가 먼저 UX 품질을 무너뜨릴 수 있다.

## 코드 포인트

1. **관측 축을 앱이 아니라 display pipeline 기준으로 고정**
   - 최소 수집 대상:
     - `SurfaceFlinger` frame miss / jank 관련 로그
     - HWC present fence, retire fence 지연
     - VSync period 대비 compose-present 완료 시간
     - display HAL 재시도/timeout 로그
   - 해석 원칙:
     - ANR 없음 + boot complete = 성공이 아니다.
     - 첫 홈 화면 이후라도 연속 deadline miss가 있으면 bring-up fail 후보로 본다.

2. **deadline miss taxonomy를 고정**
   - 우선 분류할 축:
     - `APP_LATE` : app/dequeue 단계에서 이미 늦음
     - `SF_LATE` : SurfaceFlinger composition 구간 지연
     - `HWC_PRESENT_LATE` : HWC present 또는 validate 단계 지연
     - `FENCE_SIGNAL_LATE` : retire/acquire fence signal 지연
     - `DISPLAY_HAL_RETRY` : vendor display HAL 재시도 누적
   - 핵심은 "끊긴다"가 아니라 **어느 계층이 주기를 깨는지**를 일관되게 남기는 것이다.

3. **부팅 직후 안정화 윈도우를 별도 계측**
   - 권장 체크 구간:
     - `sys.boot_completed=1` 이후 첫 5분
     - launcher 진입 직후 30초
     - 화면 on/off 또는 home 전환 직후 30초
   - 이유:
     - 초기 shader/cache warm-up, display ownership 전환, vendor service late-start가 이 구간에 집중된다.
     - steady-state 평균 fps만 보면 bring-up 초반의 critical glitch를 놓친다.

4. **pass/fail 기준을 frame deadline 기준으로 승격**
   - 예시 게이트:
     - 초기 5분 동안 black screen recovery timeout 0건
     - 연속 3프레임 이상 present deadline miss 0회
     - launcher/home 전환 시 severe jank 상한 고정
     - display HAL retry burst 허용치 초과 0건
   - 이렇게 해야 "화면이 나온다"와 "제품처럼 자연스럽게 나온다"를 구분할 수 있다.

## 리스크

- binder/ANR 관측만으로는 display stack의 deadline miss 누적을 놓쳐, 양품처럼 보이는 저품질 부팅 이미지를 통과시킬 수 있다.
- HWC present 지연이 fence 지연으로만 보이면 GPU, display HAL, Xen 경계 중 어디가 병목인지 소유권이 흐려진다.
- 첫 부팅 직후의 짧은 검은 화면 복귀 지연은 수치화하지 않으면 재현성 낮은 현상으로 취급되어 계속 릴리스까지 남는다.

## 다음 액션

- post-boot stability 수집 항목에 SurfaceFlinger jank, present fence 지연, display HAL retry 카운터를 추가.
- launcher 진입, home 전환, 화면 on/off 직후의 deadline miss를 공통 시나리오로 고정해 회귀 테스트에 편입.
- Day30 SOP 후속판에 `ANR/watchdog 0건`뿐 아니라 `display deadline miss budget`을 통합 pass 조건으로 추가.

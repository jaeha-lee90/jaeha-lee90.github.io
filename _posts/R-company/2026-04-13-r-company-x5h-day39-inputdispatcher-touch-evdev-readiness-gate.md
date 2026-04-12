---
title: R사 X5H Day39 - InputDispatcher/Touch/EVDEV readiness 안정화 게이트
author: JaeHa
date: 2026-04-13 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day39, InputDispatcher, InputReader, EVDEV, touch, bringup]
---

Day39는 Day38의 display deadline 다음 단계로, **화면은 살아 있지만 입력이 늦거나 먹지 않아 실제 조작이 불가능한 상태**를 다룬다. X5H bring-up에서는 SurfaceFlinger가 정상이고 홈 화면이 떠도, EVDEV 노드 권한/등록 순서, InputReader device classification, vendor touch service 초기화 지연 중 하나만 틀어져도 첫 터치 유실, wake 후 입력 무응답, 특정 화면 전환 직후 ghost timeout이 발생한다.

## 핵심 요약

- display stable만으로 bring-up 성공을 선언하면 안 된다. **InputReader/InputDispatcher/touch EVDEV 경로**도 별도 readiness gate로 관리해야 한다.
- 입력 문제는 crash보다 **디바이스 등록 시점, 권한, focus window, dispatch latency** 불일치로 더 자주 나타난다.
- X5H처럼 Android-vendor driver-Xen 경계가 있는 구조에서는 첫 입력 성공 시점과 wake 후 재활성화 시점을 수치화해야 재현 불량을 줄일 수 있다.

## 코드 포인트

1. **입력 경로를 4단계로 분리해서 본다**
   - `kernel/input driver` : IRQ 발생, `/dev/input/event*` 생성, EV_SYN cadence
   - `ueventd/permission` : device node mode/owner/group 정합
   - `InputReader` : classifying, calibration, device enabled 상태
   - `InputDispatcher` : focused window 대상 dispatch, timeout/ANR 연계
   - 이 4단계 중 어디서 끊겼는지 구분하지 않으면 touch 불량이 전부 `framework issue`로 뭉개진다.

2. **첫 입력 readiness 시점을 고정 계측**
   - 최소 계측 포인트:
     - boot complete 이후 첫 `InputReader device added`
     - launcher foreground 이후 첫 touch down dispatch 성공 시각
     - suspend/resume 후 첫 입력 재수신 시각
     - focus 변경 중 dropped event 수
   - 핵심은 "터치가 된다"가 아니라 **언제부터 안정적으로 되느냐**다.

3. **실패 taxonomy를 표준화**
   - `NODE_MISSING` : `/dev/input/event*` 미생성 또는 번호 변동
   - `PERMISSION_DENIED` : `ueventd.rc` 또는 sepolicy 누락으로 접근 실패
   - `MISCLASSIFIED_DEVICE` : touch가 keyboard/mouse/unknown으로 분류
   - `FOCUS_MISMATCH` : 화면은 보이지만 입력 대상 window가 잘못 잡힘
   - `RESUME_INPUT_LATE` : suspend/resume 후 device re-enable 지연
   - 이렇게 분리해야 kernel/vendor/framework 소유권 경계가 선다.

4. **pass/fail 기준을 UX 관점으로 승격**
   - 예시 게이트:
     - boot complete 이후 launcher 첫 입력 성공까지 상한 고정
     - 화면 on/off, suspend/resume 후 첫 입력 복귀 지연 상한 고정
     - 5분 안정화 윈도우 동안 dropped touch event 0건
     - InputDispatcher timeout 또는 input 관련 ANR 0건
   - 그래야 "화면이 보임"이 아니라 **조작 가능한 제품 상태**를 판정할 수 있다.

## 리스크

- EVDEV 노드 생성 순서나 권한 차이를 방치하면 build마다 입력 장치 번호가 흔들려 재현성 없는 초기 불량이 남는다.
- InputReader misclassification은 로그가 약하면 panel/driver 불량처럼 보여 디버깅 소유권이 틀어질 수 있다.
- resume 이후 첫 입력 지연을 계측하지 않으면 저전력 전환 후 무응답 문제가 출시 직전까지 숨어 있을 수 있다.

## 다음 액션

- boot/resume 공통 수집 항목에 `InputReader`, `InputDispatcher`, `/dev/input/event*` 생성 로그를 추가.
- touch panel별 `ueventd.rc`/sepolicy/device id 매핑을 표로 고정해 번호 변동과 권한 drift를 차단.
- launcher 진입, 화면 on/off, suspend/resume 직후 첫 입력 성공 시간을 Day38 display deadline 지표와 묶어 post-boot gate에 편입.

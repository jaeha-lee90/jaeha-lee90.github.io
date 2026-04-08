---
title: R사 X5H Day37 - ANR/Watchdog/Binder Starvation 부팅 후 안정화 게이트
author: JaeHa
date: 2026-04-09 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day37, ANR, watchdog, binder, system_server, stability, bringup]
---

Day37은 Day36의 native crashloop 다음 단계로, **프로세스는 살아 있지만 binder starvation이나 main-thread stall 때문에 시스템이 사실상 멈추는 상태**를 다룬다. X5H bring-up에서는 tombstone이 없어도 `system_server`, vendor HAL, UI 프로세스가 binder 대기열에 묶이면 첫 홈 화면 이후 입력 불능·검은 화면·지연 ANR로 이어진다.

## 핵심 요약

- post-boot 안정화는 crashloop 0건만으로 충분하지 않다. **ANR, watchdog, binder threadpool starvation**도 별도 게이트로 잡아야 한다.
- 특히 `system_server`와 display/audio/input 경로는 "살아 있음"보다 **정해진 시간 안에 응답을 반환하는지**가 더 중요하다.
- X5H처럼 Xen·vendor HAL·binder hop이 긴 구조는 한 프로세스 crash보다 **binder 지연 누적이 더 늦게, 더 끈질기게 품질을 무너뜨린다.**

## 코드 포인트

1. **ANR와 crash를 분리해 관측**
   - 최소 수집 대상:
     - `/data/anr/traces.txt` 또는 anr trace 디렉터리 생성 시각
     - `system_server watchdog` 로그
     - `ActivityManager`, `InputDispatcher`, `WindowManager` timeout 로그
   - 해석 원칙:
     - tombstone 없음 + ANR 존재 = "죽지 않았지만 기능적으로 실패"
     - 특히 launcher, system UI, 핵심 설정 앱 ANR은 boot pass가 아니라 fail로 본다.

2. **binder starvation 관측 키를 고정**
   - 우선 추적할 축:
     - binder transaction timeout
     - threadpool max 도달 여부
     - 특정 서비스 응답 p95/p99
     - `blocked for XXms` 계열 system_server 로그
   - 실무적으로는 아래 분류가 유용하다.
     - `FRAMEWORK_WAIT_VENDOR_HAL`
     - `VENDOR_WAIT_BINDER_REPLY`
     - `INPUT_DISPATCH_TIMEOUT`
     - `WATCHDOG_HALF_TIMEOUT`

3. **watchdog를 단순 리셋 장치가 아니라 blocker detector로 사용**
   - 확인 포인트:
     - watchdog trigger 직전 blocked thread 목록
     - monitor 대상 서비스(display/input/package/activity/power)
     - vendor binder call stack이 상단에 반복 노출되는지 여부
   - 핵심은 watchdog 발생 후 재부팅 횟수를 세는 게 아니라, **어떤 대기열과 호출 경로가 전체 boot-ready를 멈췄는지 구조적으로 남기는 것**이다.

4. **post-boot SLA를 응답시간 기반으로 승격**
   - 권장 게이트 예시:
     - boot 후 10분 안정화 구간 동안 critical ANR 0건
     - `system_server watchdog` 0건
     - display/audio/input 관련 핵심 binder call timeout 0건
     - 첫 foreground interaction 지연 상한 고정
   - 이렇게 해야 "부팅 완료 + 잠깐 사용 가능" 상태와 "연속 사용 가능한 상태"를 구분할 수 있다.

## 리스크

- crashloop가 없더라도 binder thread starvation이 누적되면 UI freeze와 delayed ANR로 이어져 soak test 후반에 실패한다.
- vendor HAL 응답 지연이 framework watchdog으로만 보이면 원인 소유권이 흐려져 triage가 길어진다.
- boot 완료 직후 짧은 smoke test만 돌리면 입력 경로 교착이나 system_server half-watchdog를 놓칠 수 있다.

## 다음 액션

- post-boot stability 스크립트에 ANR trace 생성, watchdog 로그, binder timeout 카운터를 함께 수집하도록 확장.
- display/audio/input/power 축의 핵심 binder call 목록을 정하고 timeout 발생 시 blocker taxonomy로 자동 분류.
- Day30 SOP 후속판에 `crashloop 0건 + ANR 0건 + watchdog 0건`을 통합한 안정화 pass 조건을 추가.

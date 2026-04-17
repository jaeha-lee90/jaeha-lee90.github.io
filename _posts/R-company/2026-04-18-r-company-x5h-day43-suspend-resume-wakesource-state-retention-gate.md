---
title: R사 X5H Day43 - Suspend/Resume Wake Source/State Retention 안정화 게이트
author: JaeHa
date: 2026-04-18 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day43, Suspend, Resume, Wake Source, bringup]
---

Day43은 Day42의 PowerHAL/Thermal readiness 다음 단계로, **화면은 정상 부팅되지만 절전 진입/복귀 이후 입력이 늦거나 디바이스 일부가 죽어 있고, 첫 복귀 직후 성능·기능 상태가 흔들리는 문제**를 다룬다. X5H bring-up에서는 `suspend blocker/wakelock 정합`, `wake source routing`, `driver suspend/resume callback`, `framework state restore` 중 하나만 어긋나도 장시간 데모나 차량 전원 사이클에서 안정성이 급격히 무너진다.

## 핵심 요약

- post-boot 안정화의 다음 게이트는 `userspace suspend request -> kernel suspend entry -> wake IRQ/source -> driver/framework state restore`를 한 묶음으로 봐야 한다.
- bring-up 초기에는 deep sleep 도달 자체보다 **예상한 wake source로 깨어나고, 복귀 후 입력/오디오/네트워크/차량 상태가 지정 시간 안에 원상복구되는가**가 더 중요하다.
- X5H처럼 Android/vendor/kernel/domain 경계가 많은 구조에서는 wakelock leak, IRQ wake enable mismatch, resume ordering 문제, property/state restore 누락이 가장 흔한 초기 실패축이다.

## 코드 포인트

1. **suspend/resume 경로를 4단계로 절단해서 본다**
   - `framework/userspace` : suspend 진입 요청, wake lock release, autosuspend 조건 충족 여부
   - `kernel PM core` : suspend state 진입, device callback 순서, sleep state residency
   - `wake source plane` : GPIO/RTC/CAN/modem/touch 등 실제 wake IRQ 활성 여부와 원인 기록
   - `restore path` : input/audio/network/VHAL/PowerHAL 상태 복원과 첫 유효 이벤트 도달 시점
   - 이 4단계를 분리해야 `resume 후 뭔가 이상함`을 ownership 있는 결함으로 바꿀 수 있다.

2. **entry-to-first-usable latency를 readiness 지표로 고정**
   - 최소 계측 항목:
     - suspend request 시각과 실제 suspend entry 완료 시각
     - wake IRQ 발생 시각과 kernel resume 시작 시각
     - display on, input first event, network link restore, 주요 vehicle property first valid event 시각
     - resume 직후 첫 performance hint 재적용 및 thermal/power policy 복원 시각
   - 핵심은 깨어났다는 사실보다 **사용 가능한 시스템 상태로 복귀하는 전체 latency 상한**을 관리하는 것이다.

3. **실패 taxonomy를 표준화**
   - `WAKELOCK_LEAK` : userspace/kernel wakelock이 남아 deep sleep 진입 실패
   - `WRONG_WAKE_SOURCE` : 기대하지 않은 IRQ나 noisy source가 system wake를 유발
   - `CALLBACK_ORDER_BROKEN` : driver suspend/resume callback 순서 불량으로 dependency device가 먼저/늦게 살아남
   - `STATE_RESTORE_LOSS` : input/audio/network/VHAL 등 상위 기능 상태가 resume 뒤 재초기화되지 않음
   - `RESUME_LATENCY_SPIKE` : 특정 드라이버 timeout 또는 재연결 지연으로 first usable state가 SLA 초과
   - 이렇게 분류해야 PM/core driver/framework triage가 짧아진다.

4. **pass/fail 기준을 차량 사용 시나리오 기준으로 승격**
   - 예시 게이트:
     - 지정 suspend state 진입 성공률과 abort 원인 분포 고정
     - wake source whitelist 외 원인으로 깨어나는 비율 0건
     - resume 후 display on / first input / 핵심 vehicle property restore 시간 상한 충족
     - 50회 반복 suspend/resume 동안 driver hang, service death, network reattach failure 0건
   - 그래야 `절전은 됨`이 아니라 **차량 전원 사이클에서도 기능이 유지되는가**를 판정할 수 있다.

## 리스크

- wake source ownership을 명확히 하지 않으면 noisy IRQ가 배터리 drain과 random wake를 동시에 만든다.
- driver resume ordering 문제가 남아 있으면 display/input은 살아도 audio/VHAL/network 일부만 죽는 반쯤 고장난 상태가 반복된다.
- 복귀 직후 state restore 시간을 계측하지 않으면 현장에서는 "가끔 한 번씩 먹통"으로 보이지만 실험실에서는 재현이 길어져 양산 리스크가 커진다.

## 다음 액션

- suspend entry, wake IRQ, resume 완료, display/input/network/VHAL first usable event를 한 타임라인으로 묶어 수집.
- wake source whitelist와 각 source의 owner driver, IRQ, expected policy를 표로 고정해 random wake를 조기 검출.
- 50회 반복 suspend/resume smoke를 post-boot readiness gate에 편입하고, 실패 taxonomy별로 로그 수집 위치를 표준화.

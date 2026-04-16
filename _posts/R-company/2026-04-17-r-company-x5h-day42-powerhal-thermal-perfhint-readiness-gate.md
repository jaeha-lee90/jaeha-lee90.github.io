---
title: R사 X5H Day42 - PowerHAL/Thermal/Performance Hint readiness 안정화 게이트
author: JaeHa
date: 2026-04-17 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day42, PowerHAL, Thermal, Performance Hint, bringup]
---

Day42는 Day41의 VHAL readiness 다음 단계로, **기능은 살아 있지만 화면 전환이 버벅이거나 발열 이후 성능이 급락하고, suspend/resume 이후 체감 성능이 흔들리는 상태**를 다룬다. X5H bring-up에서는 `Power HAL hint 전달`, `thermal zone/power budget`, `scheduler/cpufreq governor 반응`, `resume 이후 state restore` 중 하나만 틀어져도 부팅 직후는 그럴듯해 보여도 실제 주행/데모 조건에서 금방 불안정해진다.

## 핵심 요약

- post-boot 성능 게이트는 `framework hint -> Power HAL -> kernel DVFS/scheduler -> thermal feedback`를 한 묶음으로 봐야 한다.
- bring-up 초기에는 최고 성능보다 **hint가 제때 적용되고 thermal throttling이 예측 가능하게 들어오며 resume 뒤 상태가 복원되는가**가 더 중요하다.
- X5H처럼 Android/vendor/kernel 경계가 두꺼운 구조에서는 hint 무시, thermal sensor naming drift, cpufreq policy mismatch가 가장 흔한 초기 실패축이다.

## 코드 포인트

1. **성능 경로를 4단계로 절단해서 본다**
   - `framework/client` : launch, touch boost, animation, audio/video playback 등 성능 힌트 발생 시점
   - `Power HAL` : mode/boost 수신, hint duration, vendor plugin 적용 여부
   - `kernel control plane` : cpufreq/devfreq/scheduler governor 반응, CPU cluster residency 변화
   - `thermal loop` : skin/SoC zone 임계치, throttling level, cooling device 적용 순서
   - 이 4단계를 나누면 `느리다`는 모호한 증상을 ownership 있는 failure로 바꿀 수 있다.

2. **hint-to-effect latency를 readiness 지표로 고정**
   - 최소 계측 항목:
     - launcher 전환/앱 cold start 시 boost hint 발행 시각
     - Power HAL이 해당 hint를 수신하고 mode를 적용한 시각
     - cpufreq/devfreq가 목표 state에 도달한 시각
     - thermal throttling 진입/해제 시각과 그 직전 부하 조건
     - suspend/resume 이후 첫 hint 재적용 성공 시각
   - 핵심은 힌트 API가 존재하느냐가 아니라 **힌트가 실제 성능 변화로 이어지는 시간 상한**을 관리하는 것이다.

3. **실패 taxonomy를 표준화**
   - `HINT_DROP` : framework hint는 발생하지만 Power HAL 또는 vendor layer에서 무시됨
   - `POLICY_MISMATCH` : CPU cluster/devfreq policy 이름 또는 governor 설정이 기대와 다름
   - `THERMAL_DRIFT` : thermal zone/cooling device naming 또는 threshold가 build마다 흔들림
   - `RESUME_STATE_LOSS` : suspend/resume 뒤 hint mode, governor, affinity 상태가 복원되지 않음
   - `THROTTLE_OSCILLATION` : thermal 제어가 과도하게 출렁이며 UI frame drop과 latency spike 유발
   - 이렇게 나눠야 framework/vendor/kernel/thermal triage가 짧아진다.

4. **pass/fail 기준을 체감 성능 기준으로 승격**
   - 예시 게이트:
     - cold start/launcher transition 구간의 hint-to-effect latency 상한 고정
     - 일정 부하 시나리오에서 thermal throttling 진입 시점과 최저 성능 하한 고정
     - suspend/resume 10회 반복 후 boost mode restore failure 0건
     - 안정화 구간 동안 severe frame drop 또는 runaway throttle oscillation 0건
   - 그래야 `부팅은 됨`이 아니라 **지속 가능한 성능 상태로 올라왔는가**를 판정할 수 있다.

## 리스크

- Power HAL hint 경로를 계측하지 않으면 성능 저하가 앱 문제인지 vendor policy 문제인지 오래 섞여 보인다.
- thermal zone/threshold drift를 방치하면 동일 이미지라도 환경 온도나 데모 부하에 따라 재현성이 무너진다.
- resume 이후 governor/state restore 누락은 주행 재개나 절전 복귀 직후의 첫 체감 불량으로 이어져 양산 리스크가 크다.

## 다음 액션

- boot/post-boot/perf 수집 항목에 Power HAL mode/boost 로그, cpufreq/devfreq transition, thermal throttling 이벤트를 함께 묶어 기록.
- launcher cold start, map render, media playback, suspend/resume 대표 시나리오별 hint-to-effect latency 상한을 수치로 고정.
- thermal zone 이름, trip point, cooling device 매핑과 cluster/devfreq policy를 한 표로 정리해 build 간 drift를 조기 검출.

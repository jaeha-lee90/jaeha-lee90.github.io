---
title: R사 X5H Day48 - RTC/System Time/NTP time-base readiness 안정화 게이트
author: JaeHa
date: 2026-04-23 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day48, RTC, Time, NTP, bringup]
---

Day48은 Day47의 Ethernet readiness 다음 단계로, **네트워크는 붙지만 시스템 시간이 1970으로 부팅되거나, RTC 기준시가 흔들리거나, NTP 동기화 전후로 인증서/로그 순서/스케줄러 동작이 꼬이는 상태**를 다룬다. X5H bring-up에서는 `PMIC/SoC RTC 유지`, `bootloader의 time seed`, `kernel rtc driver`, `persisted time offset`, `userspace time sync`, `suspend/resume 뒤 clock continuity` 중 하나만 흔들려도 현상은 단순히 "가끔 시간이 이상하다"로 보이지만 실제 영향은 TLS, telemetry, OTA, watchdog evidence, crash forensics 전반으로 퍼진다.

## 핵심 요약

- time-base readiness 게이트는 `RTC source -> kernel system clock seed -> userspace correction -> network time sync -> suspend/resume continuity`를 한 체인으로 봐야 한다.
- bring-up 초기에는 절대 정밀도보다 **cold boot 시 초기 시각의 안전성, NTP lock latency, resume 뒤 monotonic/realtime 일관성, 로그 timestamp 신뢰성**이 우선이다.
- X5H처럼 Android/Yocto/가상화 경계가 섞인 구조에서는 `RTC battery/back-up domain`, `rtc driver probe`, `persist.sys.timezone 및 persisted time`, `network available 이후 SNTP/NTP sequencing`, `certificate validation before sync`가 가장 흔한 초기 실패축이다.

## 코드 포인트

1. **time-base stack을 5단계로 절단해서 본다**
   - `rtc source plane` : PMIC/SoC RTC, backup domain, battery/supercap, reset 이후 기준시 유지 여부를 확인하는 구간
   - `kernel seed plane` : boot 시 RTC read, `CLOCK_REALTIME` 초기화, rtc driver probe, invalid-time fallback 처리 구간
   - `userspace correction plane` : persisted clock, timezone property, init/service sequencing으로 사용자 공간 시각을 보정하는 구간
   - `network sync plane` : Ethernet/Wi-Fi ready 이후 SNTP/NTP가 실행되고 실제 offset/step/slew가 반영되는 구간
   - `continuity plane` : suspend/resume, warm reboot, power cycle 뒤 monotonic/realtime/log ordering이 유지되는 구간
   - 이렇게 잘라야 `TLS가 가끔 실패한다`, `로그 순서가 뒤틀린다`, `job scheduler가 빨리/늦게 돈다`를 ownership 있는 결함으로 바꿀 수 있다.

2. **초기 시각 안전성과 sync latency를 readiness 지표로 고정**
   - 최소 계측 항목:
     - boot 직후 kernel이 읽은 RTC 시각
     - `init` 완료 시점의 realtime/monotonic 차이
     - 네트워크 usable 시점과 첫 NTP sync 성공 시각
     - sync 전후 certificate validation, HTTPS, telemetry upload 성공 여부
     - suspend/resume 뒤 realtime jump, monotonic continuity, log ordering 이상 여부
   - 핵심은 단순히 시간이 맞느냐가 아니라 **시간이 틀려도 시스템이 안전하게 버티는지, 그리고 언제 안정 시간축으로 수렴하는지**다.

3. **실패 taxonomy를 표준화**
   - `RTC_SOURCE_INVALID` : backup domain 미유지, battery 불량, reset 영향으로 RTC가 epoch 또는 쓰레기 값으로 올라옴
   - `KERNEL_SEED_FAIL` : rtc driver probe/order 문제로 커널이 잘못된 초기 realtime을 심거나 fallback 처리에 실패함
   - `USERSPACE_CORRECTION_MISORDER` : persisted time/timezone/service ordering 문제로 부팅 중 time step이 반복되거나 로그 순서가 뒤틀림
   - `NETWORK_SYNC_BLOCKED` : network ready 전후 sequencing, DNS, firewall, SNTP/NTP daemon 문제로 sync가 장시간 실패함
   - `POST_SYNC_SIDE_EFFECT` : 큰 time jump 뒤 TLS session, watchdog evidence, scheduler, cache expiry가 비정상 동작함
   - 이렇게 분류해야 bootloader/kernel/framework/network 팀이 시간을 서로 미루지 않고 같은 failure bucket으로 본다.

4. **pass/fail 기준을 lab 운영 기준으로 승격**
   - 예시 게이트:
     - cold boot 직후 realtime이 허용 하한 이하(예: build time 이전/epoch)로 내려가지 않을 것
     - network ready 후 NTP sync latency 상한 고정
     - sync 전 HTTPS/TLS 실패는 허용하되 sync 후 자동 회복 100%
     - suspend/resume 50회 뒤 monotonic 역행 0건, log ordering anomaly 0건
   - 그래야 `시간은 나중에 맞춰지겠지`가 아니라 **인증/로그/스케줄 기반 기능을 믿고 올릴 수 있는 플랫폼인지**를 판정할 수 있다.

## 리스크

- RTC/초기 시각 게이트를 안 잡으면 인증서 검증 실패가 네트워크 문제처럼 보이고, 실제로는 time-base 결함인데 triage가 길어진다.
- 큰 time step을 허용한 채 bring-up을 진행하면 로그 상관관계, crash forensics, OTA evidence 해석이 계속 오염된다.
- suspend/resume 뒤 clock continuity를 검증하지 않으면 장시간 주행/검증에서 타이머, job, metrics timestamp가 누적 오차를 만든다.

## 다음 액션

- RTC read 시각, kernel seed 시각, userspace correction, NTP sync 완료 시각을 한 타임라인으로 수집하는 계측 포인트를 고정한다.
- invalid RTC, no network, delayed network, large time jump, resume 후 drift 케이스를 failure taxonomy에 연결해 dmesg/logcat/time service snapshot 수집 위치를 표준화한다.
- TLS/telemetry/OTA/log pipeline이 sync 전후 어떤 상태 전이를 보이는지 체크리스트로 묶어 time-base readiness를 독립 게이트로 승격한다.

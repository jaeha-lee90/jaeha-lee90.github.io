---
title: R사 X5H Day41 - VHAL/Property/Event readiness 안정화 게이트
author: JaeHa
date: 2026-04-16 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day41, VHAL, Vehicle HAL, Property, bringup]
---

Day41은 Day40의 오디오 readiness 다음 단계로, **Android 프레임워크는 올라왔지만 차량 상태 연동이 늦거나 틀려서 HVAC/gear/ignition/speed 기반 앱 로직이 비정상 동작하는 상태**를 다룬다. X5H bring-up에서는 `vehicle HAL 서비스 등록`, `property config 정합`, `초기 property seed`, `event publish cadence` 중 하나만 어긋나도 UI는 떠 있어도 실제 차량 제품으로는 쓸 수 없는 상태가 된다.

## 핵심 요약

- post-boot 차량 기능 게이트는 `VINTF 선언 -> vehicle HAL 등록 -> property 초기값 -> event publish`를 한 묶음으로 봐야 한다.
- bring-up 초기에는 **값이 존재하느냐보다 언제 유효한 값이 들어오고, 구독자가 언제 안정적으로 받기 시작하느냐**가 더 중요하다.
- X5H처럼 vendor/domain 경계가 있는 구조에서는 property ID 정의 mismatch, initial value 부재, event rate 불안정이 가장 흔한 초기 실패축이다.

## 코드 포인트

1. **VHAL readiness를 4단계로 절단해서 본다**
   - `manifest/VINTF` : `android.hardware.automotive.vehicle` 선언과 version/instance 정합
   - `service bring-up` : `hwservicemanager` 또는 binder 서비스 등록 성공 시점
   - `property model` : supported property list, area config, access/change mode 정합
   - `event delivery` : framework/client 구독 이후 첫 event publish와 steady-state cadence
   - 이 4단계를 나누지 않으면 `차량 값이 안 보인다`는 한 문장으로 뭉개져 triage가 길어진다.

2. **초기 property seed를 boot gate로 고정**
   - 최소 계측 항목:
     - boot complete 이전/직후 VHAL service registration 완료 시각
     - ignition/gear/speed/power state 계열 핵심 property의 첫 valid value 도달 시각
     - CarService 또는 구독 client가 첫 callback을 수신한 시각
     - subscribe 이후 N초 동안 stale value 없이 event가 유지되는지 여부
   - 핵심은 property API 존재 여부가 아니라 **프레임워크가 신뢰 가능한 차량 상태를 언제부터 갖는가**다.

3. **실패 taxonomy를 표준화**
   - `SERVICE_NOT_REGISTERED` : VHAL 서비스가 등록되지 않거나 instance mismatch
   - `PROPERTY_CONFIG_MISMATCH` : property ID/area/access/change mode 정의가 framework 기대와 불일치
   - `INITIAL_VALUE_MISSING` : 서비스는 떴지만 초기값이 비어 client가 unknown 상태에 머묾
   - `EVENT_CADENCE_UNSTABLE` : on-change/continuous publish 주기 흔들림으로 UI/logic이 flap
   - `WRITER_READER_OWNERSHIP_BLUR` : CAN/domain/vendor service 중 누가 authoritative source인지 불명확
   - 이렇게 나눠야 framework/vendor/vehicle-data 쪽 책임 분리가 선다.

4. **pass/fail 기준을 앱 체감 기준으로 승격**
   - 예시 게이트:
     - boot complete 전후 지정 시간 내 VHAL registration 완료
     - 핵심 property 세트(ignition, gear, speed, power, HVAC 일부)의 첫 valid value SLA 충족
     - subscribe 후 stale/unknown state 지속 시간 상한 고정
     - 10분 안정화 구간 동안 service death 0건, invalid area/property access 0건
   - 그래야 `서비스가 보임`이 아니라 **상위 앱이 차량 상태를 믿고 동작 가능한가**를 판정할 수 있다.

## 리스크

- property ID/area 정의 drift를 방치하면 앱 문제처럼 보이지만 실제로는 HAL/framework 계약 불일치가 누적된다.
- 초기값 seed가 늦으면 부팅 직후 UI 상태가 잘못 고정되고 이후 정상 event가 와도 사용자 체감 불량이 남는다.
- publish cadence 기준이 없으면 continuous property가 binder/CPU 부하를 만들거나 반대로 stale state를 장시간 숨긴다.

## 다음 액션

- boot/post-boot 수집 항목에 `vehicle HAL`, `CarService`, 주요 property callback 시각을 추가해 registration-to-first-valid-value 구간을 계측.
- 핵심 property 세트별로 ID/area/access/change mode/authoritative source를 한 표로 고정해 drift를 조기 검출.
- ignition/gear/speed/HVAC 대표 property를 기준으로 stale state 시간, event cadence, service death 횟수를 post-boot readiness gate에 편입.

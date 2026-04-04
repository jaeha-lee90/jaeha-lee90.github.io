---
title: R사 X5H Day33 - early-init SELinux/property/service 브링업 게이트
author: JaeHa
date: 2026-04-05 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day33, SELinux, init, property, service, bringup]
---

Day33는 Day31(Timeout/Watchdog), Day32(fstab/verity) 다음 단계로, **마운트 이후 early-init 구간에서 SELinux와 property/service 계약 불일치로 부팅이 지연·실패하는 축**을 고정한다. 이 구간은 커널과 파일시스템이 정상이어도 `service never-ready`, `restart loop`, `late bootanim`으로 체감 장애를 만든다.

## 핵심 요약

- early-init 실패의 다수는 기능 코드가 아니라 `sepolicy deny`와 `property trigger 누락` 같은 계약 불일치에서 발생한다.
- `init.rc ↔ sepolicy ↔ property_contexts`를 하나의 검증 단위로 묶어야 부팅 초반 실패를 선제 차단할 수 있다.
- bring-up 게이트에 “부팅 성공”만 두지 말고, **핵심 서비스 ready 시간과 deny 허용치 0**을 함께 넣어야 재발을 줄일 수 있다.

## 코드 포인트

1. **init 서비스-정책 정합성 정적 검사 추가**
   - 예: `tools/bringup/check_init_sepolicy_contract.py`
   - 입력:
     - `vendor/etc/init/*.rc`
     - `system/sepolicy/vendor/*.te`
     - `property_contexts`
   - 검사 항목:
     - `service <name>`의 실행 파일 context와 `domain transition` 존재 여부
     - `on property:*` 트리거 키가 `property_contexts`에 정의되어 있는지
     - `disabled/oneshot` 서비스가 의도치 않게 부팅 크리티컬 경로에 배치되지 않았는지

2. **early-init 관측 키 표준화**
   - 부팅 로그에 아래 키를 고정 출력:
     - `BOOT_PHASE=early_init`
     - `SERVICE=<name>`
     - `TRIGGER=<property|class_start>`
     - `STATE=<start|ready|timeout|denied>`
     - `SEPOLICY_DENY=<0|1>`
   - 목표: 1회 로그 수집으로 “트리거 누락 vs 권한 거부 vs 실행 파일 문제”를 즉시 분류.

3. **property 기반 의존성 타임버짓 명시**
   - 예: `boot/contracts/service_ready_budget.json`
   - 필드: `service`, `depends_on_property`, `deadline_ms`, `fallback_action`
   - 권장 정책:
     - deadline 초과 시 무한 재시도 금지
     - `fallback_action=degrade` 또는 `safe-mode mark`

4. **게이트 연결 (Day30 SOP 확장)**
   - cold boot 20회 기준:
     - 부팅 크리티컬 서비스(`vendor.display`, `vendor.audio`, `vendor.input`) ready 실패 0건
     - `avc: denied` 중 허용되지 않은 신규 deny 0건
     - early-init timeout 재현율 < 1%

## 리스크

- bring-up 중 임시 permissive/allow 규칙이 누적되면, 이후 정식 정책 전환 시 대량 회귀가 발생한다.
- property 이름/접두사 변경이 문서화 없이 반영되면 트리거 체인이 조용히 끊겨 장애 탐지가 늦어진다.
- 서비스 ready를 로그 문자열에만 의존하면, 실제 기능 미동작인데도 정상으로 오판할 수 있다.

## 다음 액션

- 타깃 보드 `init.rc`/`property_contexts`/`vendor sepolicy` 정합성 리포트 v1 생성.
- early-init 표준 로그 키(`BOOT_PHASE/SERVICE/TRIGGER/STATE/SEPOLICY_DENY`)를 init 래퍼에 반영.
- 부팅 크리티컬 서비스 3종의 ready deadline을 수집해 `service_ready_budget.json` 초안 작성 및 SOP 게이트에 연결.
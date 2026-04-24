---
title: R사 X5H Day50 - Boot Count/Boot Success/A-B Rollback readiness 안정화 게이트
author: JaeHa
date: 2026-04-25 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day50, bootcount, boot_success, A/B, rollback, bringup]
---

Day50은 Day49의 crash evidence readiness 다음 단계로, **부팅 실패가 반복될 때 언제 현재 슬롯을 포기하고 fallback/rollback으로 넘어갈지**를 다룬다. X5H bring-up에서는 watchdog reset, late-boot crashloop, boot-complete 미수렴이 발생해도 `boot count`, `boot_success`, `slot retry`, `rollback reason` 계약이 느슨하면 보드는 끝없이 같은 불량 슬롯을 재부팅하거나 반대로 정상 복구 기회를 너무 빨리 잃는다.

## 핵심 요약

- rollback readiness 게이트는 `slot select -> boot attempt count -> boot_success 판정 -> retry budget 소진 -> fallback/rollback`을 하나의 상태기계로 관리해야 한다.
- 핵심은 단순 재부팅 횟수가 아니라 **어느 단계까지 갔을 때 성공으로 인정할지**를 엄격히 고정하는 것이다.
- X5H처럼 bootloader, kernel, Android init/framework, watchdog이 모두 얽힌 구조에서는 `boot_success를 너무 이르게 세팅`하거나 `retry counter reset 조건이 불명확`한 것이 가장 흔한 초기 실패축이다.

## 코드 포인트

1. **boot 성공 판정 시점을 late stage로 고정**
   - `boot_success=1`은 kernel 진입이나 zygote 기동 시점이 아니라, 최소한 아래 조건 이후에만 기록하는 편이 안전하다.
     - `sys.boot_completed=1`
     - 핵심 vendor daemon crashloop 없음
     - boot 후 안정화 window(예: 60~120초) 통과
   - 그렇지 않으면 실제로는 불량 이미지인데도 bootloader 입장에서는 정상 슬롯으로 학습해 rollback이 막힌다.

2. **slot retry budget과 watchdog budget을 분리**
   - 예: `slot_retry_max=3`, `same_slot_watchdog_retry=2`
   - watchdog 1~2회는 일시 glitch 흡수용으로 같은 슬롯 재시도 가능하지만, `boot_success` 미달 상태에서 누적 budget을 넘으면 다음 슬롯 또는 safe image로 내려가야 한다.
   - 핵심은 watchdog 정책과 슬롯 승격 정책을 같은 카운터로 섞지 않는 것이다.

3. **rollback reason을 표준 코드로 남김**
   - 최소 reason 예시:
     - `BOOT_TIMEOUT`
     - `BOOT_COMPLETED_MISSING`
     - `POST_BOOT_CRASHLOOP`
     - `WATCHDOG_REPEATED`
     - `MANUAL_FORCE_ROLLBACK`
   - bootloader reason, kernel last reboot reason, userspace 판정을 같은 문자열 집합으로 맞춰야 Day49의 evidence chain과 바로 연결된다.

4. **A/B metadata와 userspace health signal을 연결**
   - 점검 포인트:
     - slot active/successful/unbootable 상태 전이
     - retry count 감소 시점
     - userspace health daemon 또는 boot-complete hook가 언제 성공 플래그를 쓰는지
   - 권장 규칙:
     - 부팅 중간 단계에서는 `active but not successful`
     - 최종 health gate 통과 후에만 `successful`
     - rollback 발생 시 `reason + failing stage + build_id`를 함께 기록

## 리스크

- `boot_success`를 너무 빨리 세팅하면 late-boot crashloop 이미지가 영구적으로 active slot에 남는다.
- retry budget이 너무 작으면 일시적인 주변기기 attach 지연만으로 정상 슬롯이 불필요하게 rollback될 수 있다.
- rollback reason이 bootloader/kernel/userspace에서 서로 다른 어휘를 쓰면, 실제 실패율보다 "unknown reset" 비율이 높아져 triage 속도가 급격히 떨어진다.

## 다음 액션

- Day51에서는 이 rollback readiness와 직접 맞닿는 **persist/pdata 영역 무결성과 부팅 간 상태 보존 계약**을 정리해, retry/rollback 판단 근거가 재부팅 후에도 일관되게 남는지 이어서 보겠다.

---
title: R사 X5H Day54 - Service Restart Budget와 Crashloop Escalation 임계치 표준화
author: JaeHa
date: 2026-04-29 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day54, restart-budget, crashloop, escalation, watchdog, bringup]
---

Day54는 Day53의 deadline owner 규칙을 실제 복구 정책으로 연결하기 위해 **service restart budget**과 **crashloop escalation 임계치 표준화**를 다룬다. X5H bring-up 후반으로 갈수록 흔한 실패는 "죽은 서비스를 살리지 못한다"보다 "살리기는 하는데 너무 자주 살려서 시스템 전체를 더 불안정하게 만든다"에 가깝다. 특히 `init`의 service restart, vendor launcher 재기동, framework watchdog, 도메인 재시작 정책이 서로 다른 카운터를 쓰면 같은 장애가 `local restart -> subsystem reset -> full reboot`로 과승격된다. 따라서 restart는 많이 하는 것보다 **누가 몇 번까지 어떤 시간창에서 재시도할지**를 먼저 고정해야 한다.

## 핵심 요약

- restart 정책은 서비스별 임의값이 아니라 **criticality class별 budget 표준**으로 관리해야 한다.
- `restart_count`만 세지 말고 `window_ms`, `success_hold_ms`, `escalation_target`, `evidence_key`를 함께 기록해야 crashloop와 일시적 지연을 구분할 수 있다.
- 동일 장애에 대해 `service-level`, `subsystem-level`, `system-level` 조치를 동시에 발동하지 말고 **단일 escalation ladder**로 승격 순서를 고정해야 한다.

## 코드 포인트

1. **service restart budget은 시간창 기반으로 정의한다**
   - 단순히 `3회 실패 시 재부팅`으로 끝내면 부팅 직후 burst failure와 8시간 뒤 간헐 실패가 같은 심각도로 취급된다.
   - 최소 필드는 아래처럼 고정하는 편이 낫다.
     - `service`
     - `criticality_class`
     - `max_restarts`
     - `window_ms`
     - `success_hold_ms`
     - `escalation_target`
     - `owner`
     - `evidence_key`
   - 예시:
     - `surfaceflinger`: `max_restarts=1`, `window_ms=120000`, `success_hold_ms=30000`, `escalation_target=subsystem_reset`
     - `vendor.vhal`: `max_restarts=2`, `window_ms=180000`, `success_hold_ms=60000`, `escalation_target=vehicle-stack-restart`
     - `non-critical logger`: `max_restarts=5`, `window_ms=300000`, `success_hold_ms=10000`, `escalation_target=none`
   - 이렇게 해야 Day31, Day36, Day53에서 다룬 watchdog/restart/deadline 규칙이 같은 정책표로 수렴한다.

2. **crashloop 판정은 연속 실패와 안정 구간을 함께 봐야 한다**
   - `restart_count++`만 있으면 서비스가 1초 만에 죽는 경우와 50초 동안 정상 동작 후 죽는 경우를 구분하지 못한다.
   - 운영 기준으로는 아래 상태 전이가 필요하다.
     - `STARTING`
     - `RUNNING_UNSTABLE`
     - `RUNNING_STABLE`
     - `CRASHLOOP_SUSPECT`
     - `ESCALATED`
   - 핵심은 `success_hold_ms`를 넘긴 뒤에만 restart budget 일부를 회복하거나 카운터를 리셋하는 것이다.
   - 예를 들어 VHAL이 3회 죽었더라도 각 실행이 10분 이상 유지됐다면 hardware bring-up blocker가 아니라 환경성 이벤트일 가능성이 높다.

3. **escalation ladder는 서비스 → 기능 스택 → 시스템 순으로 단일화한다**
   - 잘못된 패턴:
     - `init`가 서비스 재시작
     - vendor launcher가 도메인 재시작
     - watchdog이 전체 reboot
     - framework healthd가 rollback 표시
   - 권장 패턴:
     1. 서비스 local restart
     2. 기능 스택 재초기화 (`display/audio/vehicle` 단위)
     3. 도메인/서브시스템 재시작
     4. 최종 system reboot 또는 slot rollback
   - 같은 장애 인스턴스에는 오직 한 단계만 활성화되어야 한다.
   - 이를 위해 모든 조치는 `failure_instance_id`, `boot_id`, `escalation_level`을 공통 키로 공유해야 한다.

4. **restart evidence는 원인 추정이 아니라 승격 근거를 남겨야 한다**
   - 최소 증거 세트:
     - `service`
     - `pid`
     - `exit_reason`
     - `restart_count_in_window`
     - `window_ms`
     - `last_stable_duration_ms`
     - `escalation_level`
     - `trigger_anchor`
   - `service restarted` 로그만 남기면 운영자는 crashloop인지 수동 restart인지 구분할 수 없다.
   - 반대로 `trigger_anchor=hwservicemanager registration timeout after binder thread starvation`처럼 남기면 Day34, Day37의 readiness 게이트와 직접 연결된다.

## 리스크

- 서비스별 restart budget이 제각각이면 동일 계열 HAL이라도 어떤 것은 무한 재시작, 어떤 것은 즉시 재부팅으로 가서 운영 일관성이 무너진다.
- `success_hold_ms` 없이 카운터만 누적하면 일시적인 bring-up 노이즈가 장시간 뒤의 정상 장애와 합산되어 오탐 escalation이 발생한다.
- subsystem reset과 full reboot가 같은 failure instance에 동시에 걸리면 복구 속도는 빨라지지 않고 evidence만 파괴된다.

## 다음 액션

- Day55에서는 이 escalation 정책이 실제 릴리스 품질 게이트로 이어지도록 **service criticality class와 reboot authority matrix 표준화**를 정리하겠다.

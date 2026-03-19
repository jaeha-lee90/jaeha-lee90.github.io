---
title: R사 X5H Day20 - Fault Taxonomy 표준화와 주입 커버리지 고정
author: JaeHa
date: 2026-03-20 01:00:00 +0900
categories: [R사, X5H, Domain-Integration, Bring-up]
tags: [R사, X5H, Day20, Fault-Injection, Taxonomy, Recovery, Bring-up]
---

Day20은 Day19의 p95 기준선을 신뢰 가능한 게이트로 만들기 위해, 채널 장애를 공통 분류체계로 고정하는 작업을 다룬다. 핵심은 장애를 "증상"이 아니라 "주입 가능한 원인" 단위로 정의해 채널/브랜치 간 비교 가능성을 확보하는 것이다.

## 핵심 요약

- 채널 복구 게이트는 `blk/net/serial/audio`별로 동일한 fault taxonomy를 공유해야 회귀 비교가 가능하다.
- Taxonomy는 최소 `fault_class`, `inject_method`, `expected_recover_path`, `timeout_ms`를 포함해야 한다.
- CI에서는 "케이스 개수"가 아니라 "클래스별 최소 커버리지"(happy-path + hard-failure)를 강제해야 한다.

## 코드 포인트

1. **공통 taxonomy 스키마 정의**
   - 예시 필드:
     - `id`: `NET_LINK_DOWN`, `VQ_STALL`, `PEER_RESTART`, `DEVICE_REMOVE`
     - `channel`: `blk|net|serial|audio|multi`
     - `class`: `transport|peer|driver|resource`
     - `inject_method`: 스크립트/커널 훅/서비스 재시작 명령
     - `recover_expect`: `auto_reconnect|service_restart|required_reboot`
     - `timeout_ms`: 클래스별 복구 타임아웃
   - `taxonomy_version`을 산출물(`recover-events.ndjson`)에 포함해 데이터 해석 기준을 고정.

2. **주입 시나리오와 복구 경로 매핑**
   - `transport` 계열(예: link down, queue stall)은 재연결 루틴 검증.
   - `peer` 계열(예: DomD service restart)은 소유 서비스 재기동/의존 서비스 전파 확인.
   - `resource` 계열(예: shared memory pressure)은 degraded 상태 허용 여부를 명시.
   - 각 케이스는 `expected_state_transition`(예: `DEGRADED -> RECOVERED`)을 명확히 기록.

3. **커버리지 게이트 규칙**
   - 채널별 최소 규칙:
     - `transport` 2케이스 이상
     - `peer` 1케이스 이상
     - `hard-failure` 1케이스 이상(자동 복구 실패를 의도적으로 포함)
   - PR 게이트는 다음을 동시 검증:
     - 필수 클래스 커버리지 충족 여부
     - 클래스별 `recover_ms p95`와 실패율 변화율

4. **결과 집계 포맷 통일**
   - raw: `recover-events.ndjson` (케이스 단위 이벤트)
   - summary: `recovery-baseline.json` (클래스/채널별 p50/p95/max/fail_ratio)
   - 실패 이벤트는 `reason_code`를 강제(`TIMEOUT`, `NO_HEARTBEAT`, `PARTIAL_RECOVERY`)하여 후속 triage 자동화.

## 리스크

- taxonomy가 느슨하면 팀/브랜치마다 다른 장애를 같은 이름으로 집계해 기준선 신뢰도가 무너진다.
- hard-failure 케이스를 제외하면 "복구 가능한 장애"만 측정되어 실제 양산 리스크를 과소평가할 수 있다.
- `reason_code` 표준이 없으면 실패 분류가 사람 의존적으로 변해 회귀 원인 추적 시간이 급증한다.

## 다음 액션

- Day21에서 3주차 통합 요약을 작성하고, `taxonomy_version + baseline_version` 동기화 정책을 릴리스 게이트 항목으로 승격한다.
- fault catalog를 코드 저장소의 단일 소스(`fault-taxonomy.yaml`)로 고정하고 리뷰 시 스키마 변경 diff를 필수 확인한다.
- 클래스별 최소 샘플 수(예: 30회)를 파이프라인 조건으로 강제해 p95 신뢰구간을 안정화한다.

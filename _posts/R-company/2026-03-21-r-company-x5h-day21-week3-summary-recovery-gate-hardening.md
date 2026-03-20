---
title: R사 X5H Day21 - 3주차 요약: Recovery Gate Hardening
author: JaeHa
date: 2026-03-21 01:00:00 +0900
categories: [R사, X5H, Domain-Integration, Bring-up]
tags: [R사, X5H, Day21, Week3, Recovery, Gate, Fault-Injection, Bring-up]
---

Day21은 Day15~Day20에서 정리한 채널 맵, 서비스 소유 경계, 장애 주입, 복구 계측, taxonomy 표준화를 하나의 릴리스 게이트로 묶는 3주차 통합 정리다. 목표는 "문제 분석 문서"를 "머지 차단 기준"으로 변환하는 것이다.

## 핵심 요약

- 3주차 산출물의 본질은 `채널 단위 가시성`에서 `채널 단위 강제성`으로 전환하는 데 있다.
- 게이트는 3축으로 고정한다: `coverage`(필수 클래스 주입), `latency`(복구 p95), `failure quality`(reason_code 분류 품질).
- `taxonomy_version`과 `baseline_version`을 같은 PR 문맥에서 관리해 데이터 해석 기준 불일치를 차단해야 한다.

## 코드 포인트

1. **게이트 입력 산출물 고정**
   - `recover-events.ndjson`: 케이스 단위 raw 이벤트(시작/복구/실패).
   - `recovery-baseline.json`: 채널×클래스 집계(p50/p95/max/fail_ratio).
   - `fault-taxonomy.yaml`: fault id/class/inject/recover_expect/timeout의 단일 소스.

2. **PR 검사 규칙(최소 세트)**
   - coverage:
     - 채널별 `transport >=2`, `peer >=1`, `hard-failure >=1`.
   - latency:
     - `recover_ms p95 <= baseline_p95 * (1 + budget)`
     - 초기 budget은 예: 0.15(15%)로 시작하고 주차별 축소.
   - failure quality:
     - 실패 이벤트 100%에 `reason_code` 존재 (`TIMEOUT|NO_HEARTBEAT|PARTIAL_RECOVERY|UNKNOWN` 금지 정책 포함 가능).

3. **버전 동기화 훅**
   - taxonomy 변경 PR은 baseline 재산출을 강제.
   - CI에서 다음 조건을 fail-fast로 확인:
     - `taxonomy_version` changed && `baseline_version` unchanged
     - 또는 반대로 baseline만 변경되고 taxonomy 근거가 없는 경우.

4. **운영 지표 최소 대시보드**
   - 채널별 `pass_rate`, `p95_recover_ms`, `hard_failure_detect_rate`.
   - 주차 리포트는 "절대 수치"보다 "직전 주 대비 변화율"(delta)을 기본 단위로 사용.

## 리스크

- baseline 갱신 절차가 느슨하면 회귀가 "기준선 재정의"로 은폐될 수 있다.
- budget(허용 편차)을 과도하게 넓게 잡으면 게이트가 형식화되어 품질 하락을 막지 못한다.
- `UNKNOWN` reason_code를 방치하면 장애 분류 자동화가 깨지고 triage 리드타임이 다시 증가한다.

## 다음 액션

- Day22에서 `fault-taxonomy.yaml` 스키마(필수/선택 필드, enum, timeout 정책)를 JSON Schema로 고정한다.
- CI에 `taxonomy_version ↔ baseline_version` 동기화 검사 스크립트를 추가해 머지 전 자동 차단을 활성화한다.
- 채널별 hard-failure 재현 케이스의 최소 반복 횟수(예: 30회)와 허용 실패율 상한을 명문화한다.

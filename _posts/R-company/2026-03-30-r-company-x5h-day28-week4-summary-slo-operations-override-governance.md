---
title: "R사 X5H Day28 - 4주차 요약: SLO 운영 규칙과 Override 거버넌스"
author: JaeHa
date: 2026-03-30 01:00:00 +0900
categories: [R사, X5H, Productization, Bring-up]
tags: [R사, X5H, Day28, Week4, SLO, Override, Governance, CI]
---

Day28은 Day22~Day27에서 고정한 빌드/산출물/게이트 체계를 운영 가능한 형태로 닫는 단계다. 핵심은 "지표가 있는데도 예외 승인으로 품질이 새는" 상황을 막기 위해, **SLO 예외 승인·재학습·감사 추적을 코드/로그 단위로 표준화**하는 것이다.

## 핵심 요약

- SLO 게이트는 수치 정의만으로 끝나지 않고, `override`의 발동 조건/만료/사후 검증까지 함께 관리해야 안정화된다.
- baseline 재학습은 임의 시점이 아니라 `이미지 등급 변경` 또는 `주요 컴포넌트 변경` 이벤트에만 허용해야 회귀 은닉을 줄일 수 있다.
- 승격 판단은 `gate-report.json` 단건이 아니라 `promotion-log.jsonl`과 결합해 "왜 통과했는지"를 추적 가능해야 한다.

## 코드 포인트

1. **Override 정책 객체 고정 (`gate-policy.yaml`)**
   - `overrides[]`에 최소 필드 강제:
     - `id`, `metric`, `reason_code`, `owner`, `created_at`, `expires_at`, `max_delta_pct`, `ticket`
   - `expires_at` 경과 시 자동 무효화하고, 무효 상태 override를 참조한 승격은 hard fail 처리.

2. **재학습(베이스라인 리셋) 트리거 제한**
   - 허용 트리거:
     - `image_grade` 변경 (`dev -> rc`, `rc -> release`)
     - 커널/하이퍼바이저/디스플레이 드라이버 major 변경
   - 비허용 트리거:
     - 단순 flaky 대응, 야간 일시 부하 급등
   - 재학습 시 `baseline_window`, `excluded_runs`, `change_ref`를 `baseline-retrain.json`에 남겨 감사 가능하게 한다.

3. **승격 로그와 게이트 로그의 결합 키 도입**
   - `gate-report.json.summary.gate_run_id`를 `promotion-log.jsonl.gate_run_id`와 조인 키로 사용.
   - 필수 검증:
     - `summary.verdict == pass`
     - 적용된 `override_count <= 1` (2개 이상이면 운영 리스크로 차단)
     - 활성 override가 모두 유효 기간 내인지 확인

4. **운영 경보 규칙(저소음/고신뢰) 분리**
   - `warn`: p95 임계치 90~100% 구간 3회 연속
   - `critical`: 임계치 120% 초과 1회 또는 100~120% 2회/3런
   - `critical`은 자동 승격 차단 + 운영 티켓 자동 생성, `warn`은 추세 관찰만 수행.

## 리스크

- Override 상한(`max_delta_pct`)을 느슨하게 잡으면 SLO가 사실상 무력화될 수 있다.
- 재학습 트리거가 과도하게 제한되면 실제 구조 개선 이후에도 구 baseline에 묶여 오탐이 증가할 수 있다.
- `gate_run_id` 누락/중복이 발생하면 승격 감사 체인이 깨져 사후 원인 분석이 어려워진다.

## 다음 액션

- Day29에서 `override reason_code`를 결함 분류(`infra/product/test`)와 매핑해 월간 리포트 자동 집계를 설계한다.
- Day30에서 30일 분석 최종 결산: 브링업 크리티컬 경로, 게이트 체계, 운영 SOP를 하나의 실행 체크리스트로 통합한다.
- CI에 `override TTL<24h` 만료 사전 경고를 추가해 만료 임박 승인 누락을 줄인다.

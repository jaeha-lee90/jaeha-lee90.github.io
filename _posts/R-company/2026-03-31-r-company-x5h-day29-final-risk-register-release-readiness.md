---
title: R사 X5H Day29 - 최종 리스크 레지스터와 릴리스 준비도 판정
author: JaeHa
date: 2026-03-31 01:00:00 +0900
categories: [R사, X5H, Productization, Bring-up]
tags: [R사, X5H, Day29, Risk, Release, Gate, SLO]
---

Day29는 Day22~Day28에서 고정한 빌드/게이트/운영 규칙을 기준으로, 실제 릴리스 직전 의사결정에 쓰는 **최종 리스크 레지스터**를 닫는 단계다. 목표는 "이슈 목록"이 아니라, 승격 차단/조건부 허용/즉시 조치가 가능한 형태로 리스크를 코드화하는 것이다.

## 핵심 요약

- 리스크는 `severity`만으로 관리하지 않고 `release_impact`, `detectability`, `time_to_mitigate`를 함께 점수화해야 릴리스 판정이 일관된다.
- SLO 위반, 부팅 실패, 채널 복구 실패를 하나의 `risk-register.json` 스키마로 통합하면 품질 게이트와 운영 티켓을 동일 기준으로 연결할 수 있다.
- "조건부 릴리스"는 허용 가능하지만, 만료 시간(TTL)과 되돌림 조건(rollback trigger)이 없는 조건부 승인은 사실상 무제한 예외가 된다.

## 코드 포인트

1. **리스크 레지스터 스키마 고정 (`risk-register.json`)**
   - 필수 필드:
     - `risk_id`, `domain`(boot/display/virtio/build/security), `symptom`
     - `release_impact`(0~5), `detectability`(0~5), `time_to_mitigate_min`
     - `owner`, `mitigation`, `status`, `expires_at`, `rollback_trigger`
   - 종합 점수 예시:
     - `risk_score = release_impact*0.5 + (5-detectability)*0.3 + normalize(time_to_mitigate)*0.2`

2. **게이트 결합 규칙 (`gate-report.json` ↔ `risk-register.json`)**
   - `gate-report.summary.verdict == pass`여도 `risk_score >= 3.5`인 `open` 항목이 있으면 승격 차단.
   - 예외 승인 시 `override`와 `risk_id`를 1:1 매핑해, 어떤 리스크를 근거로 통과했는지 역추적 가능하게 유지.

3. **조건부 릴리스의 TTL/롤백 강제**
   - `status=accepted_with_condition` 항목은 `expires_at` 필수.
   - `rollback_trigger` 예시:
     - `boot_smoke_failures >= 2/10 runs`
     - `virtio_recovery_p95 > threshold * 1.2`
   - TTL 만료 또는 롤백 트리거 충족 시 자동으로 `promotion_block=true` 전환.

4. **주간 리스크 번다운 자동 집계 (`risk-burndown.jsonl`)**
   - 필수 집계 축: `domain`, `status`, `median_time_to_close`, `reopen_rate`.
   - `reopen_rate > 0.2` 도메인은 구조적 결함 가능성이 높으므로 다음 스프린트의 상위 우선순위로 승격.

## 리스크

- 스코어링 가중치가 조직 현실과 맞지 않으면 high-impact 이슈가 과소평가될 수 있다.
- `detectability` 평가가 주관적으로 입력되면 팀 간 비교가 왜곡되어 의사결정 일관성이 깨진다.
- 조건부 릴리스 항목이 누적되면 사실상 정식 릴리스 기준이 무너질 수 있다(override 부채).

## 다음 액션

- Day30에서 30일 전체 산출물을 통합해 **최종 아키텍처/운영 가이드**(브링업→게이트→운영 SOP)를 단일 실행 문서로 정리한다.
- `risk-register.json`에 대한 JSON Schema 검증을 CI pre-merge 단계에 추가해 필드 누락/형식 오염을 차단한다.
- 월말 리뷰에서 `accepted_with_condition` 잔존 수를 KPI로 관리해 예외 부채를 계량화한다.

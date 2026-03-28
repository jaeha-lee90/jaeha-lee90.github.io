---
title: R사 X5H Day27 - Boot-smoke/Healthcheck SLO 임계치 고정
author: JaeHa
date: 2026-03-29 01:00:00 +0900
categories: [R사, X5H, Productization, Bring-up]
tags: [R사, X5H, Day27, Boot, Healthcheck, SLO, Gate, CI]
---

Day27은 Day26의 DAG 승격 제어를 실제 품질 게이트로 완성하기 위해, boot-smoke/healthcheck를 "성공/실패"가 아니라 **시간 기반 SLO 임계치**로 고정하는 단계다. 핵심은 회귀를 조기에 수치로 감지해 승격 전에 차단하는 것이다.

## 핵심 요약

- `boot-smoke`와 `healthcheck`는 결과값(binary)만으로는 부족하고, p95 시간 임계치를 함께 강제해야 한다.
- `full.img`와 `android_only.img`는 동일 지표를 쓰되 임계치는 분리해야 오탐/미탐을 줄일 수 있다.
- 임계치 판정은 파이프라인 Job 내부 스크립트가 아니라 공통 `gate-report.json` 스키마로 일원화해야 승격 감사가 가능하다.

## 코드 포인트

1. **게이트 지표 스키마 고정 (`gate-report.json`)**
   - 필수 키:
     - `boot.kernel_to_userspace_ms`
     - `boot.userspace_to_launcher_ms`
     - `health.domd_ready_ms`
     - `health.android_ready_ms`
     - `channel.virtio_serial_ready_ms`
   - 각 키는 `{ value, p95_baseline, threshold, verdict }` 구조로 기록해 후속 자동 비교를 단순화한다.

2. **트랙별 임계치 분리**
   - `full.img`:
     - `domd_ready_ms`를 승격 필수 키로 지정 (Xen/서비스 체인 영향 반영).
   - `android_only.img`:
     - `android_ready_ms` 가중치를 높이고 DomD 관련 키는 참고 지표로 강등.
   - 공통 키(boot, virtio_serial)는 두 트랙 모두 fail gate로 유지.

3. **판정 로직의 히스테리시스 적용**
   - 단일 런 초과로 즉시 차단하지 않고, `N=3` 중 `2회 초과` 시 fail 처리.
   - 단, 임계치 20% 초과는 즉시 fail (중대 회귀 fast-fail).
   - 이 규칙을 `gate-policy.yaml`로 외부화해 코드 변경 없이 정책 조정 가능하게 한다.

4. **승격 연동**
   - `promote_verified`는 `gate-report.json`의 `summary.verdict == pass`를 하드 조건으로 사용.
   - `summary`에 `baseline_window`(예: 최근 20런)와 `sample_count`를 포함해, 저표본 상태 승격을 차단한다.
   - 리포트 누락/스키마 불일치는 infra fail이 아니라 product fail로 분류해 재시도 금지.

## 리스크

- 초기 baseline 표본이 부족하면 정상 빌드가 과도하게 fail될 수 있다(학습 구간 리스크).
- 트랙별 임계치가 문서와 파이프라인에서 불일치하면 운영자가 수동 override를 남발할 가능성이 높다.
- 히스테리시스 규칙이 과도하면 실제 성능 회귀를 늦게 탐지해 필드 반영 시점이 늦어진다.

## 다음 액션

- Day28에서 Week4 요약과 함께 **SLO 운영 규칙(예외 승인, baseline 재학습, override 감사)** 을 표준 운영 절차로 정리한다.
- 최근 2주 `boot-smoke` 로그를 재파싱해 지표별 baseline window(10/20/30런) 민감도를 비교한다.
- `gate-policy.yaml` 변경 이력과 승격 이벤트(`promotion-log.jsonl`)를 연결한 감사 대시보드 초안을 만든다.

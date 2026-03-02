---
title: R사 X5H Day13 - 디버깅 포인트와 로그 전략
author: JaeHa
date: 2026-03-02 09:00:00 -0800
categories: [R사, X5H, Display, Bring-up]
tags: [R사, X5H, Debug, Logging, Bring-up, Display]
---

Day13은 Day12에서 정의한 병목 축(`copy_*`, `fence_wait_ms`, `queue_depth`)을 실제 재현 루프에 태우기 위해, 수집 지점과 로그 스키마를 최소 집합으로 고정한다. 목표는 "원인 추정"이 아니라 **프레임 단위 인과관계 복원**이다.

## 핵심 요약

- 로그는 기능별 문자열 덤프가 아니라, **frame_id 중심 이벤트 스트림**으로 통합해야 병목 구간을 정확히 역추적할 수 있다.
- 초기 브링업에서는 정밀도보다 일관성이 중요하므로, 모든 경로에 같은 키(`ts`, `frame_id`, `domain`, `stage`, `metric`)를 강제한다.
- 1차 운영 기준은 `p50/p95`와 timeout count이며, 평균값 단독 관찰은 금지한다.

## 코드 포인트

1. **프레임 상관키 고정 (`frame_id`, `buffer_id`)**
   - producer enqueue 시점에 `frame_id`를 생성하고 copy/sync/commit 전 구간에 전파한다.
   - dmabuf fd 재사용으로 혼선이 생기므로, `buffer_id`(핸들 해시)와 함께 기록해 오탐을 줄인다.

2. **Stage 기반 타임라인 로깅**
   - 최소 stage: `enqueue -> import -> compose -> fence_wait -> commit -> present`.
   - 각 stage에서 `ts_start`, `ts_end`, `duration_ms`를 남겨 구간별 체류 시간을 분리한다.

3. **복사 fallback 원인 코드 표준화**
   - `copy_fallback_reason`을 enum으로 고정(`perm`, `format`, `stride`, `cache`, `unknown`).
   - free-text 로그를 배제하고 reason별 카운트를 집계해 우선순위 오류를 줄인다.

4. **Queue/Sync 경고 임계치 분리**
   - `queue_depth` 경고와 `fence_wait_ms` 경고를 분리 발화한다.
   - 동일 프레임에서 동시 발생 시 `bottleneck_hint=sync|copy|mixed`를 자동 태깅해 triage 시간을 단축한다.

## 리스크

- 로그 항목을 급격히 늘리면 I/O 부하가 새 병목을 만들어 계측 결과를 오염시킬 수 있다.
- frame_id 전파 누락이 일부 경로에 남으면 잘못된 인과관계(가짜 상관)가 생성된다.
- 팀마다 다른 스크립트로 후처리하면 동일 데이터에서도 다른 결론이 나와 의사결정이 흔들릴 수 있다.

## 다음 액션

- 공통 로그 키 스키마(`ts`, `frame_id`, `buffer_id`, `stage`, `duration_ms`, `queue_depth`, `fence_wait_ms`, `copy_fallback_reason`)를 Day14 전까지 동결한다.
- Day11 ownership 고정 시나리오 10분 런 3회(정상/고부하/복구)에서 p95와 timeout count를 수집해 기준선을 만든다.
- Day14에서는 2주차 요약으로, copy/sync 병목 분류 결과와 "즉시 수정/보류" 항목을 분리해 운영 체크리스트로 승격한다.

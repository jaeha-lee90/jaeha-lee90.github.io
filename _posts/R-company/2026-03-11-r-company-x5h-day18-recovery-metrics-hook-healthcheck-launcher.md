---
title: R사 X5H Day18 - 복구시간 계측 훅 표준화 (Launcher/Healthcheck)
author: JaeHa
date: 2026-03-11 09:00:00 -0800
categories: [R사, X5H, Domain-Integration, Bring-up]
tags: [R사, X5H, Day18, Recovery, SLO, Healthcheck, Bring-up]
---

Day18은 Day17의 실패 주입/SLO 정의를 실제 운영 지표로 고정하기 위해, `recover_ms` 계측 훅을 launcher와 healthcheck 경로에 공통 삽입하는 규칙을 정리한다. 목표는 채널 장애의 복구시간을 코드 레벨에서 동일 방식으로 기록해 pre-merge 게이트에서 자동 판정 가능하도록 만드는 것이다.

## 핵심 요약

- `recover_ms`는 채널별 임의 로그가 아니라 공통 라이프사이클 이벤트로 측정해야 비교 가능하다.
- 측정 시작/종료 시점을 고정하지 않으면 p95 지표가 팀/브랜치마다 달라진다.
- launcher 재기동과 healthcheck 정상 전이를 묶어 **실사용 관점 복구 완료**를 선언해야 한다.

## 코드 포인트

1. **계측 이벤트 스키마 고정**
   - 공통 필드: `trace_id`, `channel`, `fault_type`, `t_detect_ns`, `t_recover_ns`, `recover_ms`, `result`.
   - 계산식: `recover_ms = (t_recover_ns - t_detect_ns) / 1_000_000`.
   - `result` 값: `recovered | degraded | failed`.

2. **시작/종료 시점 정의**
   - 시작(`t_detect_ns`): healthcheck가 최초 `UNHEALTHY` 또는 I/O 에러 임계치 초과를 감지한 시점.
   - 종료(`t_recover_ns`):
     - launcher 경로: 대상 서비스 프로세스 재기동 완료 + readiness 통과
     - non-restart 경로: healthcheck가 연속 `N`회 `HEALTHY` 복귀
   - 기본 `N=3`, interval 200ms(총 600ms 안정 창).

3. **삽입 지점(최소 변경)**
   - launcher: restart orchestration 함수 진입/종료에 `RECOVERY_BEGIN/END` 이벤트 삽입.
   - healthcheck: 상태 전이(`HEALTHY↔UNHEALTHY`) 직전/직후에 `FAULT_DETECT/RECOVER_CONFIRM` 이벤트 삽입.
   - 채널 식별자는 Day15 맵(`blk/net/serial/audio`)의 canonical name만 허용.

4. **게이트 연동 규칙**
   - CI 아티팩트: 채널별 `recover_ms` raw + `p50/p95/max` 저장.
   - Day17 기준과 연결: `expected_level_max <= L2` 채널은 `p95 recover_ms <= 5000` 미준수 시 fail.
   - `failed` 발생 1회라도 있으면 merge block.

## 리스크

- 단일 monotonic clock 미사용 시 launcher/healthcheck 로그 결합에서 음수 또는 과대 `recover_ms`가 발생할 수 있다.
- 재기동 성공 직후 readiness 기준이 느슨하면 실제 트래픽 실패를 놓친 채 조기 복구 판정될 수 있다.
- trace_id 누락 시 다중 장애 동시 발생 구간에서 이벤트 상관관계가 깨져 원인 분석 시간이 증가한다.

## 다음 액션

- Day19에서 `blk/net/serial/audio` 채널별 기준선(baseline) `p95 recover_ms`를 최초 측정해 임계값을 보정한다.
- launcher/healthcheck 공통 계측 모듈을 분리해 중복 구현을 제거하고 스키마 드리프트를 차단한다.
- CI 리포트에 `recovered/degraded/failed` 비율을 추가해 복구시간 외 품질 저하 신호를 함께 추적한다.

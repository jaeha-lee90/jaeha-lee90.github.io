---
title: R사 X5H Day19 - 채널별 복구시간 기준선 수립 (p95 Baseline)
author: JaeHa
date: 2026-03-12 09:00:00 -0800
categories: [R사, X5H, Domain-Integration, Bring-up]
tags: [R사, X5H, Day19, Recovery, Baseline, p95, Bring-up]
---

Day19는 Day18에서 표준화한 `recover_ms` 계측 훅을 실제 운영 게이트에 연결하기 위한 첫 단계로, `blk/net/serial/audio` 채널의 **초기 p95 기준선**을 수립하는 절차를 정리한다. 핵심은 "절대적으로 좋은 수치"를 찾는 것이 아니라, 현재 브랜치/보드 조건에서 재현 가능한 분포를 확보해 이후 회귀(regression)를 즉시 탐지하는 것이다.

## 핵심 요약

- 채널별 복구시간은 절댓값보다 **분포 안정성(p50/p95/max + 실패율)**이 게이트 품질을 결정한다.
- 기준선은 단일 런 결과가 아니라 최소 샘플 수를 채운 반복 주입으로 산출해야 한다.
- 초기 임계값은 보수적으로 시작하고, 1~2주 데이터로 단계적으로 조정해야 오탐/미탐 균형이 맞는다.

## 코드 포인트

1. **측정 프로토콜 고정**
   - 채널별 fault 주입 30회(권장 50회), 주입 간격은 복구 완료 후 2초 hold.
   - 공통 수집 필드: `channel`, `fault_type`, `recover_ms`, `result`, `build_id`, `board_rev`.
   - `result != recovered`는 분포 계산에서 제외하지 말고 별도 실패율로 집계.

2. **기준선 계산식**
   - `p50`, `p95`, `max`, `fail_ratio = failed_or_degraded / total` 산출.
   - 초기 gate threshold 제안:
     - `threshold_p95 = max(observed_p95 * 1.15, observed_p95 + 300ms)`
     - `threshold_fail_ratio = 0`(초기 bring-up 단계는 강제 차단)
   - 채널별 독립 임계값 사용(특히 `net`과 `audio`의 분산 차이를 동일 기준으로 묶지 않음).

3. **CI 반영 구조**
   - 아티팩트: `recovery-baseline.json` + raw events(`recover-events.ndjson`).
   - `baseline_version` 필드를 추가해 임계값 변경 이력을 추적.
   - PR gate는 "기준선 대비 변화율"도 함께 체크:
     - `delta_p95_ratio = (candidate_p95 - baseline_p95) / baseline_p95`
     - `delta_p95_ratio > 0.20`이면 경고, `> 0.30`이면 fail.

4. **재현성 보호 장치**
   - 측정 중 CPU governor/thermal 상태를 함께 기록해 외란을 분리.
   - 런 시작 전 warm-up 3회는 통계에서 제외.
   - 동일 이미지 해시(`build_id`)가 아니면 기준선 비교를 금지.

## 리스크

- 샘플 수 부족 상태에서 p95를 확정하면 초기 임계값이 과도하게 낙관적이거나 비관적으로 고정될 수 있다.
- 보드 열 상태/백그라운드 부하를 통제하지 않으면 채널 성능 회귀가 아닌 환경 노이즈를 gate fail로 오인할 가능성이 높다.
- 기준선 파일 버전 관리가 느슨하면, 어떤 변경이 성능 저하를 유발했는지 회귀 추적이 어려워진다.

## 다음 액션

- Day20에서 채널별 fault taxonomy(링크 다운, virtqueue stall, peer restart 등)를 표준화해 기준선의 케이스 커버리지를 고정한다.
- `recovery-baseline.json` 스키마를 코드 리뷰 체크리스트에 포함해 임계값 변경 시 근거(샘플 수/환경)를 의무 첨부한다.
- 1주 누적 데이터 확보 후 `threshold_p95` 계수를 채널별로 재보정해 오탐률을 낮춘다.

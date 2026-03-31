---
title: R사 X5H Day30 - 브링업·게이트·운영 SOP 단일화
author: JaeHa
date: 2026-04-01 01:00:00 +0900
categories: [R사, X5H, Productization, Bring-up]
tags: [R사, X5H, Day30, SOP, Bring-up, Gate, Operations]
---

Day30는 30일 동안 정리한 부팅 체인, 드라이버/채널 의존성, CI 승격 게이트, 운영 SLO/리스크 규칙을 하나의 실행 문서로 접는 단계다. 핵심은 "잘 아는 팀원" 없이도 같은 판단을 재현할 수 있는 **단일 SOP(표준운영절차)**를 만드는 것이다.

## 핵심 요약

- 브링업(초기 부팅 성공)과 제품화 게이트(승격 판정), 운영(장애 대응)은 분리 문서가 아니라 동일 상태모델로 연결되어야 한다.
- `build_id`를 중심 키로 `boot-smoke`, `healthcheck`, `virtio recovery`, `risk-register`를 join하면 릴리스 판정과 롤백 트리거가 자동화 가능해진다.
- 최종 SOP는 "체크리스트"가 아니라 `입력 → 판정 규칙 → 출력 액션`의 머신-검증 가능한 형태(JSON/YAML)로 유지해야 drift를 줄일 수 있다.

## 코드 포인트

1. **단일 실행 스펙 정의 (`release-sop.yaml`)**
   - 섹션:
     - `inputs`: `gate-report.json`, `risk-register.json`, `slo-metrics.json`, `artifact-manifest.json`
     - `rules`: hard gate / soft gate / conditional gate
     - `actions`: promote, hold, rollback, create_ticket
   - 예시 규칙:
     - `boot_smoke.pass_rate < 0.95` → `hold`
     - `virtio.recovery_p95 > threshold` and `risk.open_critical > 0` → `rollback`

2. **판정 엔진의 결정 로그 고정 (`decision-log.jsonl`)**
   - 각 판정에 `build_id`, `rule_id`, `evidence_ref`, `verdict`, `operator`를 기록.
   - 수동 override 시 `override_reason`, `expires_at`, `risk_id` 필수로 강제.
   - 목표: 사후 분석 시 "왜 통과/차단됐는지"를 1-hop으로 재구성.

3. **브링업 크리티컬 우선 라인 유지 (초기 안정화 축)**
   - SOP의 첫 게이트를 Day1~Day11 축(boot chain/init/driver ownership)으로 고정:
     - 부팅 불가/패닉/필수 디바이스 미기동은 즉시 `promotion_block=true`.
   - Day15~Day21 축(virtio/복구 SLO)은 두 번째 게이트에서 품질 판정.

4. **운영 연동 자동화 (`ops-hooks/`)**
   - `promote` 시: 배포 노트 + 기준선 스냅샷 생성.
   - `hold` 시: 도메인 오너 라우팅 티켓 자동 발행.
   - `rollback` 시: 최근 안정 `build_id` 자동 선택 + blast radius 요약 출력.

## 리스크

- SOP가 지나치게 복잡해지면 현장 우회(수동 판단) 비율이 올라가고 규칙 신뢰도가 떨어진다.
- 입력 데이터 스키마 버전이 맞지 않으면 잘못된 hold/rollback이 발생할 수 있다.
- override 만료(TTL) 관리가 느슨하면 예외가 누적되어 사실상 무규칙 운영으로 회귀한다.

## 다음 액션

- 30일 산출물 기반으로 `release-sop.yaml` v1을 작성하고 CI에서 schema lint를 강제한다.
- `decision-log.jsonl` 대시보드를 붙여 도메인별 hold/rollback 패턴을 주간 리뷰 지표로 운영한다.
- 다음 사이클(심화편)은 보안/OTA/전력(thermal 포함) 축을 추가한 Day31+ 확장 트랙으로 분리한다.

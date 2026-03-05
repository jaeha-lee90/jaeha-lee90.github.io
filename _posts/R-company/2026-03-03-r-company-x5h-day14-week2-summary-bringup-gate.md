---
title: R사 X5H Day14 - 2주차 요약: 브링업 게이트와 우선 수정축
author: JaeHa
date: 2026-03-03 09:00:00 -0800
categories: [R사, X5H, Display, Bring-up]
tags: [R사, X5H, Week2, Bring-up, Triage, Display]
---

Day14는 Day8~Day13에서 확보한 모듈 의존성/KM-UM 정합/Display ownership/Copy-Sync 계측을 한 번에 묶어, 실제 브링업에서 "부팅 성공"보다 먼저 통과해야 할 **운영 게이트**를 고정한다. 핵심은 기능 추가가 아니라 실패 재현성과 복구 시간을 통제하는 것이다.

## 핵심 요약

- 2주차 결론은 성능 튜닝 이전에 **정합성 게이트(버전/소유권/로그 스키마)**를 먼저 잠그는 것이며, 이 3축이 흔들리면 병목 분석 결과도 신뢰할 수 없다.
- 즉시 수정 항목은 `KM/UM 버전 고정`, `DomD/DomA ownership 단일화`, `frame_id 기반 로그 전파`의 세 가지다.
- 보류 가능 항목은 미세 최적화(copy 경량화, queue 파라미터 조정)이며, 게이트 통과 전에는 적용 우선순위를 낮춘다.

## 코드 포인트

1. **브링업 게이트 A: KM/UM 정합 강제**
   - 부팅 직후 `ddk-km`과 `ddk-um-bin` 빌드 태그를 한 쌍으로 검증하고 불일치 시 display init을 진행하지 않는다.
   - 초기 실패를 빠르게 드러내기 위해 `compat_check=hard_fail` 정책을 유지한다.

2. **브링업 게이트 B: Display ownership 고정**
   - Day11 원칙대로 ownership 경계를 런타임 전환 금지로 고정한다.
   - DomD가 scanout을 소유하면 DomA 경로는 명시적 bypass 상태를 로그에 남겨 오해를 차단한다.

3. **브링업 게이트 C: 관측 가능성(Observability) 확보**
   - Day13 스키마(`frame_id`, `stage`, `duration_ms`, `fence_wait_ms`) 미준수 경로를 빌드 경고 대상으로 지정한다.
   - 최소 stage 누락(`import`, `fence_wait`, `present`) 시 성능 비교 리포트를 무효 처리한다.

4. **Triage 라벨링 규칙 통합**
   - 장애 분류를 `compat`, `ownership`, `sync`, `copy`, `mixed` 5개로 제한해 주간 리포트와 현장 디버깅 용어를 통일한다.
   - 한 이슈에 라벨 2개 이상이면 primary/secondary를 분리 기록해 대응 순서를 자동화한다.

## 리스크

- 게이트를 강하게 걸면 초기엔 실패 건수가 늘어 팀이 "불안정해졌다"고 오판할 수 있다.
- ownership 단일화 과정에서 기존 우회 스크립트가 무력화되어 단기 생산성이 떨어질 수 있다.
- 로그 스키마 강제가 완전하지 않으면 mixed 라벨이 과다 발생해 실제 원인 분리가 지연된다.

## 다음 액션

- Day15에서 게이트 A/B/C를 CI preflight로 옮겨 merge 이전 차단이 가능한지 검증한다.
- 최근 1주 로그를 5개 라벨로 재분류해 `즉시 수정(이번 주)` vs `보류(게이트 통과 후)` 백로그를 분리한다.
- ownership/compat 실패 케이스별 복구 플레이북(탐지→격리→재기동)을 1페이지 런북으로 정리한다.

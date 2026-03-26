---
title: R사 X5H Day26 - CI 게이트 DAG와 이미지 승격 제어면 고정
author: JaeHa
date: 2026-03-27 01:00:00 +0900
categories: [R사, X5H, Productization, Bring-up]
tags: [R사, X5H, Day26, CI, CD, DAG, Gate, Promotion, Artifact]
---

Day26은 Day25의 브랜치/태그 정책을 실제 파이프라인으로 강제하는 단계다. 핵심은 "검증 순서"를 문서가 아니라 **CI Job DAG(Directed Acyclic Graph)** 로 고정해, 잘못된 이미지가 승격 경로로 올라오지 못하게 만드는 것이다.

## 핵심 요약

- 이미지 승격은 사람 판단이 아니라 DAG의 선행 조건으로 제어해야 한다.
- `full.img`와 `android_only.img`는 공통 게이트 + 트랙별 게이트를 분리해 관리해야 실패 절단면이 명확해진다.
- bring-up 초기에 필요한 최소 집합은 `build -> flashability -> boot-smoke -> healthcheck -> promotion`이며, 이 순서가 깨지면 회귀 분석 속도가 급락한다.

## 코드 포인트

1. **승격 단계를 Job으로 분리하고 needs 체인으로 고정**
   - 예시 단계:
     - `build_full`, `build_android_only`
     - `flashability_check`
     - `boot_smoke_full`, `boot_smoke_android_only`
     - `service_healthcheck_domd`, `service_healthcheck_android`
     - `promote_verified`, `promote_release`
   - `promote_*` Job은 테스트 요약 JSON(`gate-report.json`)이 없으면 실행 자체를 차단한다.

2. **공통 게이트와 이미지별 게이트 분리**
   - 공통: 산출물 무결성(SHA256), 메타데이터(`artifact-manifest.json`) 필수 필드, 서명 검증.
   - `full.img` 전용: Xen/도메인 기동 확인, virtio 핵심 채널(blK/net/serial/audio) 최소 핸드셰이크.
   - `android_only.img` 전용: Android framework 부팅 완료 시그널, 필수 HAL probe 성공.

3. **실패 즉시 중단(fail-fast) + 재시도 정책 분리**
   - 인프라성 실패(네트워크 타임아웃, runner 장애)만 제한 재시도 허용.
   - 제품성 실패(부팅 실패, healthcheck 실패)는 재시도 금지 후 원인 분석 티켓 자동 생성.
   - 재시도로 가려진 불안정성 제거를 위해 flaky 비율을 주간 리포트로 고정 추적.

4. **승격 이벤트 감사 로그 표준화**
   - `promotion-log.jsonl`에 `commit`, `tag`, `pipeline_run_id`, `gate_digest`, `actor`를 남긴다.
   - `release` 승격은 서명 태그 존재 + `verified` 유지 시간(예: 24h 무회귀) 조건을 추가한다.
   - 배포 시스템은 `promotion-log` 불일치 시 다운로드 링크를 비활성화한다.

## 리스크

- DAG 선행 조건이 느슨하면 테스트 일부 누락 상태로 `verified` 승격이 가능해져, 필드 결함이 증폭된다.
- 공통/트랙 게이트를 섞어두면 `full.img` 전용 결함이 `android_only.img` 품질 신뢰도까지 오염시킨다.
- 재시도 정책이 과도하면 실제 회귀가 인프라 이슈로 오판되어 MTTR이 늘어난다.

## 다음 액션

- Day27에서 boot-smoke/healthcheck의 **정량 SLO(부팅 시간, 서비스 기동 시간, 채널 준비 시간)** 를 수치로 고정한다.
- 현재 파이프라인에 `promotion-log.jsonl` 생성/검증 훅을 추가하고, 누락 시 승격 차단을 기본값으로 전환한다.
- 최근 2주 빌드 이력을 재평가해 flaky 재시도 의존 Job을 분리 태깅한다.

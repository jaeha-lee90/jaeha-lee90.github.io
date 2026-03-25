---
title: R사 X5H Day25 - 브랜치/태그 운영 정책과 이미지 등급 게이팅
author: JaeHa
date: 2026-03-26 01:00:00 +0900
categories: [R사, X5H, Productization, Bring-up]
tags: [R사, X5H, Day25, Branch, Tag, Release, Artifact, Governance]
---

Day25는 Day24 산출물 추적 규칙을 실제 개발 흐름에 붙이는 단계다. 핵심은 브랜치/태그를 "협업 편의"가 아니라 `full.img`/`android_only.img`의 **신뢰 등급을 결정하는 제어면**으로 다루는 것이다.

## 핵심 요약

- 브랜치는 변경의 속도를, 태그는 상태의 불변성을 담당해야 한다.
- 이미지 등급은 브랜치에서 자동 추론하지 않고, **검증 통과 + 서명된 태그**로만 승격한다.
- bring-up 초기에는 규칙이 느슨해 보이더라도, 등급 승격 경로(`dev` → `staging` → `release`)를 초기에 고정해야 회귀 원인 추적이 가능하다.

## 코드 포인트

1. **브랜치 역할 고정**
   - `feature/*`: 개발/실험 전용. 산출물은 내부 검증용(`candidate`)으로만 취급.
   - `integration/x5h`: 기능 통합 브랜치. 부팅/기본 회귀 통과 시 `verified` 후보 생성.
   - `release/x5h/*`: 릴리스 안정화. hotfix 외 직접 기능 추가 금지.

2. **태그를 등급 승격의 단일 관문으로 사용**
   - 예시 태그 체계:
     - `x5h-v2026.03.26-rc1` (검증 후보)
     - `x5h-v2026.03.26` (릴리스)
   - 태그 생성 조건:
     - Day24의 `artifact-manifest.json` 완비
     - 필수 게이트(`boot-smoke-full`, `android-regression-only`) 통과
     - 태그 생성 주체/시간/커밋 SHA 감사 로그 보존

3. **이미지 등급 매핑 테이블 강제**
   - `feature/*` 산출물: `experimental`
   - `integration/x5h` + 필수 테스트 통과: `verified`
   - `release/*` + 서명 태그: `release`
   - 배포 시스템은 `release` 등급 외 외부 공유 차단(내부 테스트 채널만 허용).

4. **브랜치 상태와 산출물 메타데이터 연결**
   - `artifact-manifest.json`에 `source_branch`, `source_tag`, `pipeline_run_id` 필드 추가.
   - 태그 없는 이미지 업로드는 허용하되 `ungraded`로 분리 저장하고 자동 만료 정책 적용.
   - 동일 커밋에서 재빌드된 이미지라도 SHA가 다르면 별도 revision으로 취급한다.

## 리스크

- 브랜치와 이미지 등급이 분리되지 않으면, 테스트용 산출물이 운영 경로로 유입되어 장애 전파 범위가 커진다.
- 태그 생성 기준이 모호하면 "같은 버전"에 서로 다른 이미지가 매핑되어 회귀 분석이 불가능해진다.
- release 브랜치에 기능성 변경이 섞이면, 결함 책임 구간이 확장되어 핫픽스 속도가 급격히 떨어진다.

## 다음 액션

- Day26에서 CI/CD 안정화 포인트를 정리하며, 등급 승격 게이트를 파이프라인 Job DAG로 고정한다.
- 태그 생성 자동화 스크립트에 서명/검증 단계(`git tag -s`, verify)를 포함한다.
- 기존 최근 산출물 중 태그 미연결 항목을 `ungraded`로 재분류하고 만료 정책을 적용한다.

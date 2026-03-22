---
title: R사 X5H Day22 - meta-xt-gen5-platform 빌드 흐름과 실패 절단면
author: JaeHa
date: 2026-03-23 01:00:00 +0900
categories: [R사, X5H, Productization, Bring-up]
tags: [R사, X5H, Day22, Yocto, meta-xt-gen5-platform, Build, Bring-up]
---

Day22는 4주차(Productization) 시작점으로, `meta-xt-gen5-platform`을 중심으로 X5H 이미지 생성 경로를 "재현 가능한 빌드 파이프라인" 관점에서 정리한다. 브링업 크리티컬 관점의 핵심은 기능 추가보다 **빌드 입력(레이어/설정/아티팩트)의 드리프트를 먼저 차단**하는 것이다.

## 핵심 요약

- bring-up 후반의 주요 장애는 런타임 버그보다 `빌드 입력 불일치`에서 자주 발생한다.
- `meta-xt-gen5-platform`은 보드 공통 정책 레이어로 보고, 보드/프로덕트 변형은 하위 dev 레이어에서 분리해 책임을 고정해야 한다.
- 우선 고정해야 할 것은 3가지다: `레이어 우선순위`, `MACHINE/DISTRO 설정`, `산출물 매핑(full/android_only)`.

## 코드 포인트

1. **레이어 구성의 단일 소스화**
   - `bblayers.conf`에서 `meta-xt-gen5-platform`의 포함 순서와 우선순위를 명시적으로 고정.
   - 같은 레시피를 다중 레이어가 제공할 때 `BBFILE_PRIORITY` 충돌을 로그로만 보지 말고 lint 단계에서 fail 처리.
   - CI preflight에서 `bitbake-layers show-layers`, `show-recipes` 스냅샷을 아티팩트로 보존.

2. **빌드 입력 변수의 드리프트 차단**
   - `local.conf` 임시 오버라이드(개발자 로컬 수정)를 금지하고, 변경은 레이어 conf로 승격.
   - 최소 고정 변수: `MACHINE`, `DISTRO`, `TCLIBC`, 이미지 타입(`IMAGE_FSTYPES`), 서명/배포 관련 플래그.
   - 환경변수 주입(`BB_ENV_PASSTHROUGH`)은 allowlist만 허용해 재현성 확보.

3. **산출물 경로와 이름 규약 고정**
   - `tmp/deploy/images/<machine>/` 하위 결과물 중 릴리스 대상으로 `full.img`, `android_only.img` 매핑 규칙을 문서/스크립트 양쪽에 동일 반영.
   - 심볼릭 링크/복사 스크립트는 SHA256 기록을 강제해 "같은 이름 다른 바이너리" 사고를 방지.

4. **실패 절단면(First-Cut) 정의**
   - configure 실패, fetch 실패, do_compile 실패, do_rootfs 실패를 단계별로 분리해 triage 진입점을 고정.
   - `sstate` hit/miss 비율과 fetch mirror fallback 횟수를 함께 수집해 인프라성 실패와 코드성 실패를 분리.

## 리스크

- 레이어 우선순위가 불명확하면 동일 브랜치에서도 개발자마다 다른 이미지가 생성될 수 있다.
- `local.conf` 수동 수정이 누적되면 재현 불가 상태가 되어 버그 재현/회귀 검증 비용이 급증한다.
- 산출물 이름만 같고 실제 해시가 다른 상태로 배포되면, 현장 이슈 역추적이 사실상 불가능해진다.

## 다음 액션

- Day23에서 `meta-xt-gen5-platform` 대비 `meta-xt-x5h-dev` 오버레이 지점을 diff 기준으로 정리해 책임 경계를 수치화한다.
- 빌드 preflight 스크립트(레이어/변수/해시 검증) 초안을 작성하고 CI의 첫 단계로 배치한다.
- 릴리스 후보 생성 시 아티팩트 manifest(파일명, SHA256, 생성 시각, git revision) 자동 출력 규칙을 추가한다.

---
title: R사 X5H Day23 - meta-xt-x5h-dev 오버레이 경계와 우선순위 충돌 제어
author: JaeHa
date: 2026-03-24 01:00:00 +0900
categories: [R사, X5H, Productization, Bring-up]
tags: [R사, X5H, Day23, Yocto, meta-xt-x5h-dev, Overlay, Priority]
---

Day23은 Day22의 빌드 입력 고정 작업을 이어서, `meta-xt-gen5-platform`(공통 정책)과 `meta-xt-x5h-dev`(보드/개발 오버레이) 사이의 책임 경계를 코드 관점으로 절단한다. 핵심은 "어디까지가 공통이고, 어디부터가 실험/보드 특화인지"를 레시피 우선순위와 append 범위로 강제하는 것이다.

## 핵심 요약

- `meta-xt-x5h-dev`는 공통 정책 대체 레이어가 아니라 **제한된 오버레이 레이어**로 운영해야 한다.
- 충돌을 줄이는 최소 규칙은 3가지다: `bbappend 우선`, `동일 PN 신규 recipe 금지`, `override 이유 추적`.
- bring-up 초기에는 속도 때문에 dev 레이어가 비대해지기 쉬우므로, 승격 기준(공통화 기준)을 수치로 관리해야 한다.

## 코드 포인트

1. **오버레이 범위 제한 (`.bbappend` 우선)**
   - `meta-xt-x5h-dev`는 가능한 한 기존 레시피에 대한 `.bbappend`만 사용하고, 동일 `PN`의 신규 `.bb` 제공은 예외로 제한.
   - 예외 허용 조건: 상위 레이어 레시피 부재 또는 임시 핫픽스(만료일 포함).
   - CI에서 `bitbake-layers show-overlayed` 결과를 수집해 신규 overlay 항목을 PR diff에서 강제 노출.

2. **우선순위 충돌의 자동 탐지**
   - `BBFILE_PRIORITY_meta-xt-x5h-dev`가 높더라도, 공통 레이어 대체가 발생하면 경고가 아니라 fail 처리.
   - `bitbake -e <target>`에서 최종 변수 소유 레이어(`FILE`, `LAYERDIR`)를 추출해 기준값과 비교.
   - 핵심 체크 변수: `SRC_URI`, `DEPENDS`, `RDEPENDS`, `PACKAGECONFIG`, `SYSTEMD_SERVICE`.

3. **보드 실험값과 제품 기본값 분리**
   - 성능/디버그 목적의 실험 플래그는 `DISTRO_FEATURES` 기본 경로에 두지 않고, `MACHINEOVERRIDES` 또는 별도 feature 토글로 격리.
   - `IMAGE_INSTALL:append` 남용을 줄이고, 패키지 그룹(`packagegroup-*`)으로 의존을 명시해 회귀 범위를 추적 가능하게 유지.

4. **승격(Dev → Platform) 게이트 정의**
   - 동일 수정이 N회 이상 반복 적용되면(예: 3회) 공통 레이어 승격 후보로 자동 분류.
   - 승격 시 필수 산출물: 변경 사유, 영향 타깃, 롤백 방법, sstate 캐시 영향.

## 리스크

- dev 레이어가 공통 정책을 덮어쓰기 시작하면, 브랜치별 빌드 결과 차이가 누적되어 "동일 태그 비동일 바이너리" 상태를 만든다.
- overlay 원인 추적이 안 되면 장애 분석 시간이 코드 수정 시간보다 길어지는 역전이 발생한다.
- 실험 플래그가 제품 기본값으로 잔류하면, 양산 직전 안정화 단계에서 예측 불가능한 부팅/성능 편차가 발생한다.

## 다음 액션

- Day24에서 `kas`/`repo manifest` 기준으로 레이어 lock(커밋 SHA pinning) 전략을 정리해 "재현 가능한 소스 입력"을 완성한다.
- `show-overlayed` + `bitbake -e` 파서를 묶은 preflight 스크립트를 작성해 PR 단계에 붙인다.
- dev 레이어 항목을 `임시/상시`로 분류하는 태그 규칙(만료일 포함)을 도입한다.

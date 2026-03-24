---
title: R사 X5H Day24 - 산출물(full.img/android_only.img) 관리와 재현성 게이트
author: JaeHa
date: 2026-03-25 01:00:00 +0900
categories: [R사, X5H, Productization, Bring-up]
tags: [R사, X5H, Day24, Yocto, Artifact, full.img, android_only.img, Reproducibility]
---

Day24는 Day22~23에서 정리한 레이어 경계 위에, 최종 전달물인 `full.img`와 `android_only.img`를 어떻게 분리/추적/검증할지에 집중한다. 핵심은 "파일만 만들어지는 빌드"가 아니라 "입력-출력 매핑이 보존되는 산출물 체계"를 고정하는 것이다.

## 핵심 요약

- `full.img`와 `android_only.img`는 파일명만 다른 변형이 아니라 **소유 도메인과 검증 항목이 다른 독립 산출물**로 취급해야 한다.
- 산출물 관리의 최소 단위는 `(manifest SHA, layer SHA set, build config hash, image hash)` 4종 메타데이터다.
- bring-up 단계에서 가장 위험한 패턴은 "같은 이름의 이미지가 다른 입력으로 재생성"되는 경우이며, 이를 막기 위해 immutable 아카이브 규칙이 필요하다.

## 코드 포인트

1. **이미지 타입별 책임 분리**
   - `full.img`: 하이퍼바이저/도메인 구성/Android 포함 통합 부팅 검증 대상.
   - `android_only.img`: Android 영역 변경 검증 및 빠른 회귀 테스트 대상.
   - CI 파이프라인에서 두 이미지를 동일 잡에서 생성하더라도, 테스트 게이트는 분리(`boot-smoke-full`, `android-regression-only`)한다.

2. **산출물 메타데이터 매니페스트 강제**
   - 각 이미지와 함께 `artifact-manifest.json`을 생성해 아래를 기록:
     - `repo_manifest_revision`
     - `layers`(레이어별 commit SHA)
     - `build_target`, `machine`, `distro`
     - `artifact_sha256`, `build_time_utc`
   - 릴리스/공유는 이미지 단독 업로드 금지, `img + manifest` 쌍으로만 허용.

3. **이름 재사용 금지와 보관 정책**
   - `latest/full.img` 같은 mutable 링크는 허용하되, 실제 저장 경로는 `artifacts/<date>/<shortsha>/...`로 고정.
   - 동일 파일명 재업로드 시 overwrite 금지(immutable). 변경 필요 시 새 revision 발급.
   - 문제 재현 시 `manifest`만으로 입력 소스를 역추적할 수 있어야 한다.

4. **검증 체크섬과 배포 전 preflight**
   - 업로드 직전 `sha256sum` 검증, 수신 측에서도 재검증을 파이프라인 단계로 강제.
   - preflight 실패 조건 예시:
     - manifest 누락
     - 이미지-매니페스트 SHA 불일치
     - 레이어 SHA 중 floating ref(`HEAD`, branch name) 포함

## 리스크

- `full.img`/`android_only.img`의 테스트 경계가 섞이면, Android 변경이 하이퍼바이저/도메인 회귀로 오판되거나 반대로 근본 원인을 놓친다.
- 산출물과 소스 입력의 매핑이 약하면, 현장 이슈 재현 시 "어떤 코드로 만든 이미지인지"를 확정하지 못해 MTTR이 급증한다.
- mutable 파일명 overwrite가 허용되면, 동일 태그/동일 파일명에서도 실행 결과가 달라져 품질 지표가 무의미해진다.

## 다음 액션

- Day25에서 브랜치/태그 운영 정책을 산출물 규칙과 연결해, "브랜치 상태 → 이미지 등급(실험/검증/릴리스)" 매핑을 정의한다.
- CI에 `artifact-manifest.json` 스키마 검증 단계를 추가하고, 실패 시 업로드를 차단한다.
- 기존 저장소의 최근 이미지에 대해 역으로 manifest를 보강해 최소 추적성 baseline을 맞춘다.

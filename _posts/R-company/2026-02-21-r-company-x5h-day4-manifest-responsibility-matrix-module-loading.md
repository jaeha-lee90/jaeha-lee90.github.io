---
title: R사 X5H Day4 - Manifest 책임 매트릭스와 벤더 모듈 로딩 경로
author: JaeHa
date: 2026-02-21 09:00:00 -0800
categories: [R사, X5H, Bring-up, Kernel]
tags: [R사, X5H, Bring-up, Manifest, Kernel, Module]
pin: false
published: true
---

Day4는 Day3의 후속으로, 브링업 초기에 가장 자주 꼬이는 지점인 **manifest 책임 경계**와 **벤더 모듈 로딩 타이밍**을 한 번에 정리했다.

## 핵심 요약

- `android-manifest`는 제품/플랫폼 전체 조합(상위 레이어), `android-kernel-manifest`는 커널/툴체인/외부모듈 결합(하위 레이어) 책임이 크다.
- 장애 triage는 “상위 조합 문제인지, 커널/모듈 결합 문제인지”를 1차 분기해야 시간을 줄일 수 있다.
- `ddk-km`, `disfwk_fe`는 커널 부팅 완료 이후 모듈 로딩 단계에서 실패가 드러나는 경우가 많아, boot 성공만으로 정상 판정을 하면 안 된다.

## 코드 포인트

1. 책임 매트릭스(실무용 1차 분기)
   - `android-manifest`
     - 대상: product 조합, 다중 repo pin, 플랫폼 통합 버전
     - 증상: 특정 제품만 빌드 실패/런타임 앱-서비스 연동 깨짐
   - `android-kernel-manifest`
     - 대상: 커널 소스/브랜치, clang·gcc prebuilts, Xen override, 벤더 커널 모듈 태그
     - 증상: 커널 빌드 실패, 부팅 후반 드라이버 probe 실패, Xen guest 디바이스 attach 실패

2. 모듈 로딩 관점 점검 순서
   - 빌드 시점: `EXT_MODULES` 포함 여부 확인(누락 시 산출물 자체가 비어 있을 수 있음)
   - 패키징 시점: `ko` 배치 경로와 이미지 포함 여부 확인
   - 부팅 시점: `dmesg`에서 모듈 insert/probe 실패 원인 확인(의존 심볼, 버전, init order)

3. 실패 패턴(초기 브링업에서 흔한 형태)
   - manifest 태그 mismatch → 빌드는 통과하지만 런타임 probe 실패
   - toolchain drift → 특정 모듈만 ABI/심볼 이슈
   - Xen override 누락/충돌 → virtio/xenbus 계열 초기화 지연 또는 실패

## 리스크

- 팀이 `repo sync` 기준점을 단일 문서로 고정하지 않으면, 같은 브랜치명인데 실제 소스 조합이 달라지는 비결정성 빌드가 발생한다.
- 커널/모듈 검증을 CI에서 분리하지 않으면 “부팅됨=정상”으로 오판해 결함을 후반 통합 단계로 넘길 위험이 크다.
- 벤더 모듈 태그 변경 이력이 추적되지 않으면 회귀 분석 시간이 급격히 늘어난다.

## 다음 액션

- Day5에서 `EXT_MODULES`, `BUILD_CONFIG`, toolchain revision까지 포함한 **재현 가능한 커널 빌드 최소 명령셋**을 확정한다.
- CI preflight에 “manifest 해시 + 모듈 태그 + toolchain 버전” 3종 체크를 추가해 드리프트를 조기 차단한다.
- `ddk-km/disfwk_fe`에 대해 부팅 직후 자동 수집 로그 템플릿(`uname -a`, `dmesg`, 모듈 목록)을 표준화한다.

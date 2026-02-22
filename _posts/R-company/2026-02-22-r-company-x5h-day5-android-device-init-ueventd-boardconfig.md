---
title: R사 X5H Day5 - android_device init/ueventd/BoardConfig 브링업 크리티컬 포인트
author: JaeHa
date: 2026-02-22 09:00:00 -0800
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Bring-up, android_device, init, ueventd, BoardConfig]
pin: false
published: true
---

Day5는 초기 부팅 성공률에 직접 영향을 주는 `android_device` 레이어를 점검한다. 핵심은 **init 서비스 기동 순서**, **ueventd 권한/노드 생성**, **BoardConfig 산출물 경계**를 하나의 체인으로 보는 것이다.

## 핵심 요약

- `BoardConfig`는 "무엇을 빌드/포함할지"를 결정하고, `init*.rc`는 "언제/어떻게 띄울지"를 결정한다. 둘 중 하나만 맞아도 부팅 완성도가 떨어진다.
- `ueventd.rc`의 퍼미션/ownership 누락은 드라이버 자체 문제처럼 보이는 가짜 장애를 만든다.
- 브링업 초반에는 `servicemanager 등록 실패`, `HAL 소켓/디바이스 접근 거부`, `vendor service class 미기동`이 반복 패턴이다.

## 코드 포인트

1. `BoardConfig*.mk` 확인 포인트
   - 커널/DTB/벤더 모듈 산출물 경로가 현재 product make 타깃과 일치하는지 확인
   - `BOARD_VENDOR_RAMDISK_*`, `BOARD_BOOT_HEADER_VERSION`, `TARGET_ARCH*` 등 부트 이미지 규격 관련 변수 drift 점검
   - SELinux 정책/분할 이미지(system/vendor/product) 경계가 실제 파티션 전략과 맞는지 확인

2. `init.${ro.hardware}.rc`, `init.vendor.rc` 확인 포인트
   - `class core/main/hal` 기동 순서와 `on boot`, `on late-init`, `on property:*` 트리거가 의도대로 연결되는지 확인
   - 핵심 vendor daemon이 `disabled` 상태로 남아 property trigger를 못 받는지 점검
   - 서비스별 `user/group/capabilities`가 실제 디바이스 노드 권한과 호환되는지 교차 확인

3. `ueventd*.rc` 확인 포인트
   - `/dev/*` 노드 mode/uid/gid가 HAL 기대값과 일치하는지 확인
   - 펌웨어 로딩 경로(`/vendor/firmware` 등)와 소유권 설정 검증
   - coldboot 타이밍 이슈로 노드가 늦게 생성되는 경우, init 서비스 시작 조건을 지연/재시도로 분리

4. 실전 triage 최소 순서
   - `logcat -b all | grep -E "init|hwservicemanager|servicemanager|avc:"`
   - `dmesg | grep -E "uevent|firmware|denied|probe"`
   - `ls -l /dev/<문제 노드>` + `getprop | grep -E "ro.hardware|ro.boot|init.svc"`

## 리스크

- device 트리와 product 조합이 늘어날수록 `init rc` 분기가 중복되어, 특정 SKU에서만 서비스 미기동 회귀가 발생할 수 있다.
- `ueventd` 권한을 임시로 넓혀 장애를 우회하면, 후반 보안 점검에서 대량 수정이 필요해져 일정 리스크가 커진다.
- BoardConfig 변수 변경 이력이 문서화되지 않으면 부트 이미지 규격 mismatch를 빌드 단계에서 조기 감지하기 어렵다.

## 다음 액션

- Day6에서 "기본 부팅 실패 포인트"를 `init/SELinux/firmware/모듈` 4축으로 통합 체크리스트화한다.
- `android_device` 기준으로 서비스 의존 그래프(서비스명, class, trigger, required device node)를 표준 템플릿으로 정리한다.
- CI preflight에 `init rc lint + ueventd permission diff + BoardConfig 핵심 변수 diff`를 추가해 회귀를 조기 탐지한다.

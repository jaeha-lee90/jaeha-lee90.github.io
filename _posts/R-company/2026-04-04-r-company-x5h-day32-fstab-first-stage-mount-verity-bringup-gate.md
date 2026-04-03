---
title: R사 X5H Day32 - fstab/first-stage mount/verity 브링업 게이트 고정
author: JaeHa
date: 2026-04-04 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day32, fstab, first-stage-mount, dm-verity, AVB]
---

Day32는 초기 브링업에서 재현 빈도가 높은 **first-stage mount 실패 축**을 잠그는 데 집중한다. kernel/init가 정상이라도 `fstab 파티션 매핑`, `AVB/verity 상태`, `동적 파티션(super)` 해석이 어긋나면 부팅은 곧바로 복구 모드 또는 bootloop로 떨어진다.

## 핵심 요약

- first-stage mount는 Android userspace 진입 전 마지막 관문이라, 여기서의 실패는 상위 서비스 디버깅보다 우선 처리해야 한다.
- `fstab 항목(blk device, fs type, flags)`과 실제 GPT/DTB 노출 경로를 1:1 검증하는 정적 체크가 필요하다.
- verity/AVB 실패는 "보안 문제"이자 동시에 "브링업 차단 문제"이므로, 원인 코드를 표준화해 triage 시간을 줄여야 한다.

## 코드 포인트

1. **fstab-실디바이스 정합성 검사 스크립트 추가**
   - 예: `tools/bringup/check_fstab_devices.py`
   - 입력: `vendor/etc/fstab.*`, `by-name` 심볼릭 링크 덤프
   - 검증:
     - fstab의 logical partition명이 실제 super metadata에 존재하는지
     - `first_stage_mount` 대상이 init 1st stage에서 접근 가능한지

2. **first-stage mount 실패 로그 표준화**
   - `init_first_stage.cpp` 경로에서 실패 시 아래 키를 반드시 출력:
     - `FSM_STAGE=mount`
     - `PARTITION=<name>`
     - `ERRNO=<code>`
     - `AVB_SLOT=<slot>`
   - 목표: 현장 로그 1회 수집으로 "장치 미노출 / fs 불일치 / verity 실패"를 즉시 분류.

3. **AVB/verity 판정과 복구 정책 분리**
   - 정책 분리:
     - `VERITY_CORRUPTION`: 즉시 fail-fast + 슬롯 롤백 평가
     - `DEVICE_NOT_FOUND`: 제한 재시도 후 DTB/uevent 경로 점검
   - 동일 재부팅 루프 방지를 위해 boot reason에 `mount_fail_class`를 기록.

4. **브링업 게이트 조건 추가**
   - Day30 SOP 게이트에 아래 조건을 추가:
     - cold boot 20회에서 first-stage mount 실패 0건
     - AVB 실패 원인코드 미분류 건수 0건

## 리스크

- fstab를 기기 변형별로 분기 관리할 때 누락이 발생하면 특정 SKU만 간헐 부팅 실패가 발생한다.
- 재시도 정책이 과도하면 근본 원인(파티션 매핑 오류)이 숨겨져 장애 탐지가 지연된다.
- verity 우회 디버그 설정이 현장 이미지로 유출되면 보안/품질 이슈가 동시에 커진다.

## 다음 액션

- 현재 타깃 보드 기준 `fstab.*`와 `super metadata`를 대조한 정합성 리포트 v1 작성.
- first-stage mount 실패 로그 키 표준(`FSM_STAGE/PARTITION/ERRNO/AVB_SLOT`)을 init 패치로 반영.
- bootloop 샘플 로그를 `mount_fail_class` 3종(DEVICE_NOT_FOUND, FS_MISMATCH, VERITY_CORRUPTION)으로 재분류해 triage 템플릿 업데이트.
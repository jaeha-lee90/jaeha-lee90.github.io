---
title: R사 X5H Day16 - DomD 서비스 책임 경계 (소유 채널/재시작 범위/실패 전파)
author: JaeHa
date: 2026-03-08 09:00:00 -0800
categories: [R사, X5H, Domain-Integration, Bring-up]
tags: [R사, X5H, Day16, DomD, Virtio, Bring-up]
---

Day16은 Day15의 virtio 채널 맵을 실제 운영 가능한 형태로 고정하기 위해, DomD 서비스 단위의 책임 경계를 명시한다. 목표는 장애 발생 시 "어디를 재시작할지"를 즉시 결정하고, 단일 채널 오류가 전체 시스템 재기동으로 확산되는 것을 막는 것이다.

## 핵심 요약

- bring-up 단계에서 DomD 서비스는 **기능 묶음이 아닌 채널 소유 단위**로 분리해 관리해야 한다.
- 각 서비스마다 `소유 채널`, `허용 재시작 범위`, `실패 전파 대상`을 문서/런북/헬스체크에 동일하게 반영해야 triage 시간이 줄어든다.
- 게이트 기준은 "프로세스 생존"이 아니라 **채널 단위 입출력 성공 + 전파 차단 검증**까지 포함해야 한다.

## 코드 포인트

1. **서비스 책임 템플릿 고정**
   - 권장 필드:
     - `service_name`
     - `owned_channels[]` (예: `net.doma.0`)
     - `restart_scope` (`channel-only` | `service-only` | `host-impact`)
     - `propagation_targets[]` (영향 받는 도메인/기능)
     - `fallback_action` (degrade/readonly/fail-fast)
   - 템플릿을 Git 내 단일 파일로 유지해 코드 변경 리뷰 시 함께 검증한다.

2. **재시작 정책 코드화**
   - 동일 바이너리가 여러 채널을 제공하더라도, 런처에서 채널 인스턴스별 재초기화 경로를 우선 사용한다.
   - `restart_scope=host-impact` 항목은 기본 금지하고, 예외 승인 없이는 자동 재시작 대상에서 제외한다.

3. **실패 전파 라벨링 표준화**
   - 로그 공통 필드: `channel`, `owner_service`, `failure_class`, `propagation_level`.
   - `propagation_level` 예시:
     - `L1`: 단일 채널 국소 장애
     - `L2`: 동일 서비스 내 다채널 영향
     - `L3`: DomA 기능 저하
     - `L4`: 시스템 레벨 재부팅 필요

4. **preflight 게이트 강화**
   - merge 전 필수 검증:
     - 채널별 hello I/O 1회 성공
     - 서비스 재시작 후 소유 외 채널 무영향 확인
     - 실패 주입 시 전파 레벨이 설계값 이하인지 확인

## 리스크

- 서비스 경계 문서와 실제 런처 설정이 분리되면 운영자가 잘못된 재시작 결정을 내릴 수 있다.
- 전파 레벨 기준이 팀마다 다르면 장애 티켓 분류가 일관되지 않아 통계가 무의미해진다.
- host-impact 예외가 누적되면 결국 "부분 복구" 전략이 붕괴해 full reboot 의존으로 회귀할 수 있다.

## 다음 액션

- Day17에서 `blk/net/serial/audio` 채널별 실패 주입 시나리오를 정의하고 전파 레벨 측정 기준을 수치화한다.
- DomD 런처/서비스 설정 파일과 책임 템플릿 간 자동 정합 체크 스크립트를 추가한다.
- 최근 2주 장애 이력을 `propagation_level` 기준으로 재분류해 상위 반복 패턴을 추출한다.

---
title: R사 X5H Day8 - ddk-km 구조와 핵심 모듈 의존 관계
author: JaeHa
date: 2026-02-25 09:00:00 -0800
categories: [R사, X5H, Display, GPU]
tags: [R사, X5H, ddk-km, GPU, Bring-up]
---

Day8은 2주차(Display/GPU) 시작점으로, `ddk-km/rogue_km`의 빌드 진입점과 서비스 계층 의존 관계를 먼저 고정한다. 목표는 성능 튜닝이 아니라 **GPU 커널 모듈의 안정 로딩 조건**을 만드는 것이다.

## 핵심 요약

- X5H 기준 ddk-km의 실질 진입점은 `rogue_km/Makefile`과 `build/linux/x5h_android/Makefile` 조합이며, 여기서 타깃/툴체인/커널 헤더 정합이 먼저 결정된다.
- 런타임 관점 핵심 경로는 `pvrsrv` 공통 서비스 레이어(`services/server/common`)와 RGX 디바이스 레이어(`services/server/devices`)의 결합이다.
- 브링업 초반에는 기능 확장보다 **(1) 모듈 로드 성공, (2) 펌웨어 초기화 성공, (3) sync/dmabuf 인터페이스 정상화** 3가지를 게이트로 잡는 것이 안전하다.

## 코드 포인트

1. **빌드 엔트리/플랫폼 분기 확인**
   - `rogue_km/Makefile`
   - `rogue_km/build/linux/x5h_android/Makefile`
   - `rogue_km/build/linux/config/core.mk`, `core_volcanic.mk`
   - 점검 포인트: 커널 버전 매크로, 타깃 아키텍처, 외부 모듈 빌드 플래그(`KDIR`, `CROSS_COMPILE`) 일치 여부

2. **서비스 코어 계층 (공통 런타임)**
   - `services/server/common/pvrsrv.c`
   - `services/server/common/pvrsrv_bridge_init.c`
   - `services/server/common/sync_server.c`, `dma_km.c`
   - 의미: 유저/커널 인터페이스 브리지, 메모리/동기화 관리의 기본 축을 형성

3. **RGX 디바이스 계층 (GPU 기능 경로)**
   - `services/server/devices/rgx_bridge_init.c`
   - `services/server/devices/rgxpower.c`, `rgxmmuinit.c`, `rgxhwperf_common.c`
   - 의미: 전원/메모리 매핑/펌웨어 상호작용의 초기화 순서가 깨지면 로딩은 되어도 실제 submit 경로가 비정상화됨

4. **브리지 자동 생성 코드의 역할 분리**
   - `generated/rogue/*_bridge/*.c`
   - `generated/volcanic/*_bridge/*.c`
   - 의미: 기능 추가 시 수동 수정 지점과 생성 코드 지점을 분리하지 않으면 merge/리베이스 시 회귀 확률이 급증

## 리스크

- 커널 헤더/설정 mismatch 상태에서 모듈 빌드가 통과해도 런타임 unresolved symbol로 실패할 수 있다.
- sync/dmabuf 경로가 불안정하면 Display ownership 분석(Day11) 이전에 프레임 파이프라인이 간헐적으로 멈춘다.
- generated bridge 코드를 수동 패치하면 차기 버전 업 시 재생성으로 변경이 소실될 수 있다.

## 다음 액션

- `x5h_android` 타깃으로 모듈 빌드 1회 고정하고, 빌드 아티팩트/커널 버전/컴파일러 버전을 로그로 보존한다.
- 부팅 후 `dmesg`에서 RGX/pvrsrv 초기화 시퀀스와 error 패턴을 템플릿화해 Day9(`ddk-um-bin` 정합) 입력으로 사용한다.
- Day9 주제는 계획 순서대로 **`ddk-um-bin` 의존성과 KM/UM 버전 정합 실패 패턴**으로 진행한다.

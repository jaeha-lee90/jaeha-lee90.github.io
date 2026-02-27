---
title: R사 X5H Day10 - disfwk_fe 인터페이스/초기 표시 경로 분석
author: JaeHa
date: 2026-02-27 09:00:00 -0800
categories: [R사, X5H, Display, Bring-up]
tags: [R사, X5H, disfwk_fe, Display, Bring-up]
---

Day10은 Day9(KM/UM 정합 통과 조합) 기준선 위에서, 부팅 직후 화면이 실제로 살아나는 최소 경로를 `disfwk_fe` 중심으로 정리한다. 목표는 “GPU가 붙었는데 화면이 안 나온다” 문제를 display front-end 경계에서 빠르게 절단하는 것이다.

## 핵심 요약

- `disfwk_fe`는 패널/출력 체인과 상위 그래픽 스택 사이의 front-end 접점으로 동작하며, 초기 표시 성공 여부는 **probe → 리소스 바인딩 → 첫 frame commit** 3단계로 수렴된다.
- 브링업 초반에는 기능 확장보다 **초기 commit 성공/실패를 로그로 명확히 분기**하는 것이 우선이다.
- Day9의 KM/UM 정합이 깨진 상태에서는 Day10 이슈를 정확히 분류할 수 없으므로, 입력 조건을 고정한 뒤 분석해야 한다.

## 코드 포인트

1. **모듈 로딩/프로브 시퀀스 고정**
   - 점검 대상: 부팅 로그의 `disfwk_fe` probe/bind 관련 메시지, 모듈 로딩 순서
   - 확인 포인트: 의존 리소스(clock/reset/interrupt/device-tree 노드) 획득 실패 여부

2. **초기 표시 경로의 최소 IOCTL/콜 체인 확인**
   - 점검 대상: 사용자/서비스 계층에서 display 초기화 요청 후 kernel front-end까지 도달하는 호출 경로
   - 확인 포인트: 첫 mode-set 또는 first-frame commit 직전에 반환되는 에러 코드(`-EINVAL`, `-ETIMEDOUT` 등) 위치

3. **도메인 경계(특히 DomD↔DomA) 책임 분리**
   - 점검 대상: display ownership이 어느 도메인에서 결정되는지, front-end 제어 권한이 단일 소유로 고정되는지
   - 확인 포인트: ownership 전환/중복 제어로 인한 commit race, blank 화면 상태

4. **로그 게이트 정의 (Bring-up 필수)**
   - Gate A: `disfwk_fe` probe 완료
   - Gate B: 첫 display pipeline enable 완료
   - Gate C: 첫 frame commit ack
   - 세 게이트 중 멈춘 지점으로 장애 분류를 표준화

## 리스크

- probe는 성공하지만 first-frame commit에서 멈추는 경우, 원인이 panel timing/ownership/race 중 어디인지 빠르게 섞여 보일 수 있다.
- DomD/DomA 양측에서 display 제어를 시도하면 간헐 blank/깜빡임으로 나타나 재현성이 낮아진다.
- 로그 포맷이 팀마다 다르면 같은 증상도 다른 이름으로 기록되어 트리아지 비용이 커진다.

## 다음 액션

- `disfwk_fe` 기준 부팅 직후 표준 수집 로그를 1세트로 고정(`dmesg`, 첫 frame 시점 logcat, 모듈 목록).
- ownership 단일화 원칙(누가 mode-set/commit 최종 책임자인지)을 문서 한 장으로 명시한다.
- Day11에서는 계획 순서대로 **display ownership (DomD vs DomA)**를 상세 분석하되, Day10의 Gate A/B/C 통과 케이스와 실패 케이스를 비교 입력으로 사용한다.

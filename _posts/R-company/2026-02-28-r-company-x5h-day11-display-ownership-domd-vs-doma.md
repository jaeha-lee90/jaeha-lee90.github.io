---
title: R사 X5H Day11 - Display Ownership (DomD vs DomA) 경계 고정
author: JaeHa
date: 2026-02-28 09:00:00 -0800
categories: [R사, X5H, Display, Bring-up]
tags: [R사, X5H, Display, DomD, DomA, Bring-up]
---

Day11은 Day10의 `disfwk_fe` Gate(A/B/C) 기준선 위에서, 화면 제어권을 DomD와 DomA 중 어디에 둘지 명확히 고정하는 데 집중한다. 목표는 “부팅은 되는데 화면이 간헐적으로 검정/깜빡임” 증상을 ownership 충돌 관점에서 빠르게 절단하는 것이다.

## 핵심 요약

- 초기 브링업 단계에서는 **display ownership 단일화**가 최우선이며, DomD 또는 DomA 중 한쪽만 최종 mode-set/commit 권한을 가져야 한다.
- DomD를 소유자로 두면 플랫폼 제어 일관성은 높지만, DomA UI 초기화 타이밍과의 인터페이스 계약이 필수다.
- DomA를 소유자로 두면 앱/UI 민첩성은 좋지만, 하이퍼바이저/가상화 경계에서 권한·복구 정책이 복잡해진다.

## 코드 포인트

1. **단일 커밋 경로 보장**
   - 점검: mode-set/atomic commit 호출 주체가 런타임에 1개인지 확인
   - 구현 포인트: 소유자 외 도메인 호출은 early reject 또는 proxy 경유로 강제

2. **도메인 간 제어 인터페이스 명시**
   - 점검: DomA↔DomD 사이 display 제어 요청 채널(virtio/rpc 등)에서 요청/응답 타임아웃과 재시도 정책
   - 구현 포인트: “권한 없음”, “리소스 점유”, “복구 중” 오류 코드를 표준화해 상위 서비스가 분기 가능하게 유지

3. **부팅 단계별 ownership 상태 머신 정의**
   - 점검: Boot → Splash → UI-ready 전환 시 ownership 전환 이벤트 유무
   - 구현 포인트: 전환이 필요하면 단발성 handover만 허용하고, 재진입 방지 플래그로 race 차단

4. **Day10 Gate와 연동한 관측 포인트**
   - Gate A/B 통과 후 Gate C(첫 frame commit ack) 실패 시 ownership 충돌 여부를 최우선 확인
   - 동일 부팅에서 commit issuer(도메인/프로세스) 1개만 로그에 남도록 추적 키를 부여

## 리스크

- DomD/DomA가 동시에 복구 루틴을 수행하면 first-frame 이후 blank로 회귀하는 간헐 장애가 발생할 수 있다.
- ownership 정책 문서 없이 구현만 앞서면 팀별 가정 차이로 디버깅 시간이 급증한다.
- handover 시점이 불명확하면 suspend/resume 또는 서비스 재시작 때 권한 유실이 재발한다.

## 다음 액션

- 브링업 단계 기준 기본 정책을 “단일 소유자 + 비소유자 proxy 요청”으로 고정한다.
- 부팅 로그에 `owner_domain`, `commit_issuer`, `handover_seq` 3개 추적 필드를 공통 삽입한다.
- Day12에서는 계획 순서대로 **성능 병목 후보(copy/sync)**를 ownership 고정 케이스 기준으로 분석한다.

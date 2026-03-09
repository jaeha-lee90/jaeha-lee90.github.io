---
title: R사 X5H Day17 - 채널 실패 주입 시나리오와 전파 레벨 SLO
author: JaeHa
date: 2026-03-09 09:00:00 -0800
categories: [R사, X5H, Domain-Integration, Bring-up]
tags: [R사, X5H, Day17, Virtio, Fault-Injection, Bring-up]
---

Day17은 Day16에서 정의한 서비스 책임 경계를 실제로 검증하기 위해, `blk/net/serial/audio` 채널별 실패 주입 패턴과 전파 레벨 판정 기준(SLO)을 고정한다. 목표는 장애 재현-분류-복구를 같은 언어로 수행해 bring-up 병목을 줄이는 것이다.

## 핵심 요약

- 채널별 실패를 동일 템플릿으로 주입해야 원인 비교가 가능하다.
- 전파 레벨은 정성 판단이 아닌 **시간/영향 범위 기반 수치 기준**으로 판정한다.
- merge 게이트에 "실패 주입 1회 + 레벨 상한 준수"를 포함해야 부분 복구 전략이 유지된다.

## 코드 포인트

1. **실패 주입 템플릿 표준화**
   - 공통 필드: `channel`, `inject_type`, `duration_ms`, `expected_level_max`, `recovery_deadline_ms`.
   - `inject_type` 예시:
     - `timeout` (응답 지연)
     - `disconnect` (링크 단절)
     - `corrupt-header` (프레임 헤더 손상)
     - `queue-stall` (ring 정체)

2. **채널별 최소 시나리오 세트**
   - `blk`: I/O timeout 3초, 재시도 후 read-only degrade 확인
   - `net`: tx queue stall 2초, DomA ping 복구 시간 측정
   - `serial`: 단절 1초, 세션 재동기화 성공 여부 검증
   - `audio`: 버퍼 underrun 500ms, 무음 삽입 후 스트림 지속 확인

3. **전파 레벨 판정 SLO**
   - `L1` (국소): 단일 채널 영향, 복구 `<= 2s`
   - `L2` (서비스): 동일 서비스 다채널 영향, 복구 `<= 5s`
   - `L3` (도메인): DomA 기능 저하, 복구 `<= 15s`
   - `L4` (시스템): 재부팅 필요 또는 복구 실패
   - 게이트 기준: Day17 범위 채널은 `expected_level_max <= L2`.

4. **로그/메트릭 수집 필드 고정**
   - 필수 로그 키: `channel`, `inject_type`, `propagation_level`, `recover_ms`, `fallback_action`.
   - CI 아티팩트에 채널별 `p95 recover_ms`를 저장해 회귀를 추적한다.

## 리스크

- 주입 도구가 실제 장애와 다르면 낮은 레벨로 과소평가될 수 있다.
- 복구 시간 측정 시점(start/end) 정의가 팀별로 다르면 SLO 비교가 무의미해진다.
- audio/serial처럼 체감 품질 중심 채널은 기능 정상처럼 보여도 사용자 영향이 누락될 수 있다.

## 다음 액션

- Day18에서 채널별 `recover_ms` 측정 훅을 런처/헬스체크에 공통 삽입한다.
- 실패 주입 스크립트를 CI pre-merge 잡으로 연결해 `L3/L4` 발생 시 merge를 차단한다.
- 최근 장애 리포트를 Day17 SLO 포맷으로 재작성해 기준선(baseline) 수치를 확보한다.

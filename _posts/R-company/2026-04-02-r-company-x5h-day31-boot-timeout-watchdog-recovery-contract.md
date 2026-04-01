---
title: R사 X5H Day31 - 부팅 Timeout/Watchdog 복구 계약 고정
author: JaeHa
date: 2026-04-02 01:00:00 +0900
categories: [R사, X5H, Bring-up, Reliability]
tags: [R사, X5H, Day31, Boot, Timeout, Watchdog, Recovery]
---

Day31부터는 30일 계획의 확장 트랙으로 들어가되, 우선순위는 여전히 초기 브링업 크리티컬 축을 먼저 잠그는 것이다. 이번 주제는 **부팅 단계 Timeout과 Watchdog 복구 경계**를 코드 계약으로 고정해, "무한 대기"와 "의미 없는 재부팅 루프"를 동시에 줄이는 데 초점을 둔다.

## 핵심 요약

- 부팅 실패의 다수는 기능 결함보다도 `대기 조건 미충족 + timeout 부재`에서 발생한다.
- Watchdog은 단순 리셋 장치가 아니라, 단계별 증거(`boot_stage`, `last_error`, `retry_count`)를 남기는 복구 오케스트레이터로 설계해야 한다.
- `stage timeout budget`과 `max reboot attempts`를 분리 정의해야, 느린 부팅과 비정상 루프를 구분해 판정할 수 있다.

## 코드 포인트

1. **단계별 Timeout 계약 테이블화**
   - 예: `boot/contracts/boot_timeout_table.json`
   - 필드: `stage`, `deadline_ms`, `on_timeout`, `owner`
   - 권장 최소 단계:
     - `kernel_init`
     - `init_first_stage`
     - `vendor_service_ready`
     - `display_stack_ready`

2. **init/launcher에서 공통 타임아웃 핸들러 통일**
   - 분산된 `sleep+retry` 패턴을 공통 API(`wait_with_deadline()`)로 대체.
   - timeout 시 표준 이벤트를 남김:
     - `BOOT_TIMEOUT(stage=..., elapsed_ms=..., dependency=...)`
   - 목표: 로그 파싱으로 실패 상위 원인 Top-N 자동 집계.

3. **Watchdog 복구 정책 2단계 분리**
   - `N1`회 이하: 동일 슬롯/동일 설정 재시도(일시 glitch 흡수)
   - `N2`회 초과: safe profile(최소 드라이버/서비스)로 강등 부팅 후 원인 덤프
   - 정책 예시:
     - `N1=2`, `N2=4`, `N2 초과 시 promotion_block=true`

4. **부팅 게이트와 연계**
   - CI/현장 smoke 결과에 `timeout_signature`를 포함해 Day30 SOP 입력으로 연결.
   - 같은 `stage timeout`이 3회 이상 반복되면 자동으로 risk-register에 `critical` 생성.

## 리스크

- timeout 값을 과도하게 타이트하게 잡으면 정상 지연 케이스를 장애로 오탐할 수 있다.
- 반대로 deadline이 느슨하면 실제 hang 탐지가 늦어져 부팅 시간과 디버그 비용이 증가한다.
- safe profile 강등 경로가 검증되지 않으면, 복구 정책 자체가 새로운 부팅 실패 지점을 만든다.

## 다음 액션

- 실제 부팅 로그 기준으로 stage별 p95/p99를 산출해 `boot_timeout_table.json` v1 초안 작성.
- launcher/init 공통 `wait_with_deadline()` 래퍼를 도입하고 기존 busy-wait 지점을 순차 치환.
- Watchdog 재시도/강등 정책을 테스트케이스(정상 지연, 영구 hang, 간헐 hang)로 분리 검증 후 SOP 게이트에 연결.

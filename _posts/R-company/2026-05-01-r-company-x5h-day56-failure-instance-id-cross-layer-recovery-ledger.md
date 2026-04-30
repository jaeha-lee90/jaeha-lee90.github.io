---
title: R사 X5H Day56 - Failure Instance ID와 Cross-layer Recovery Ledger 표준화
author: JaeHa
date: 2026-05-01 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day56, failure-instance-id, recovery, ledger, bringup]
---

Day56은 Day55의 criticality class/authority matrix를 실제 운영 증거 체계로 닫기 위해 **failure instance ID**와 **cross-layer recovery ledger 표준화**를 다룬다. X5H bring-up에서 같은 장애가 `init`, vendor launcher, domain manager, framework watchdog 로그에 각각 다른 이름으로 찍히면, restart budget과 reboot authority를 아무리 잘 나눠도 같은 사건을 여러 건으로 오판하게 된다. 따라서 복구 정책이 작동하려면 먼저 **"지금 보고 있는 실패가 동일한 incident인가"**를 cross-layer에서 같은 키로 식별해야 한다.

## 핵심 요약

- 모든 복구 이벤트는 단순 서비스명 대신 `failure_instance_id`를 기준으로 묶어야 한다.
- ledger는 observer별 개별 로그가 아니라 **incident 단위의 누적 상태 기계**로 관리해야 한다.
- restart/reboot 판단은 "새 장애인가"보다 먼저 **"같은 instance가 이미 어디까지 승격되었는가"**를 확인하고 나서 내려야 한다.

## 코드 포인트

1. **failure instance ID를 component + symptom + time window 조합으로 고정한다**
   - `vendor.audio-hal died` 같은 문자열만으로는 같은 사건 재발과 다른 사건 신규 발생을 구분하기 어렵다.
   - bring-up 운영용 최소 키는 아래 정도가 적당하다.
     - `component_id`
     - `failure_class`
     - `first_seen_boot_id`
     - `first_seen_elapsed_ms_bucket`
     - `symptom_fingerprint`
   - 조합 예시는 아래처럼 잡을 수 있다.
     - `failure_instance_id = sha1(component_id + failure_class + boot_id + 5s_bucket + symptom_fingerprint)`
   - 여기서 `symptom_fingerprint`는 tombstone signal, timeout owner, missing HAL name, binder target, watchdog reason처럼 **원인 근처 정보**로 구성해야 한다. 단순 PID는 restart마다 바뀌므로 주키로 쓰기 어렵다.

2. **cross-layer recovery ledger를 단일 schema로 누적 갱신한다**
   - ledger는 텍스트 로그 한 줄이 아니라, 같은 incident에 대해 각 레이어가 상태를 덧붙이는 구조여야 한다.
   - 최소 필드는 아래처럼 가져가는 편이 좋다.
     - `failure_instance_id`
     - `boot_id`
     - `component_id`
     - `criticality_class`
     - `observer_chain[]`
     - `symptom_summary`
     - `current_escalation_level`
     - `attempted_actions[]`
     - `final_decider`
     - `final_verdict`
     - `closed_at_elapsed_ms`
   - 예를 들어 `init`는 crashloop 관측을 기록하고, vendor health daemon은 stack restart 시도를 append하고, watchdog은 최종 reboot 여부만 append해야 한다. 같은 incident를 각자 새 row로 만들면 Day54/Day55 정책이 무의미해진다.

3. **승격 판단 전에 기존 instance open 여부를 먼저 조회해야 한다**
   - 흔한 안티패턴은 `service crash -> local restart`, `watchdog timeout -> subsystem reset`, `framework boot miss -> reboot`가 서로 독립 판정되는 구조다.
   - 올바른 순서는 아래에 가깝다.
     1. observer가 새 symptom을 받는다.
     2. open ledger에서 같은 `failure_instance_id` 또는 merge 가능한 sibling incident를 찾는다.
     3. 이미 `current_escalation_level >= subsystem_restart`면 하위 조치는 생략한다.
     4. authority matrix에 지정된 decider만 다음 승격을 확정한다.
   - 이 조회 단계가 없으면 동일 incident에 대해 `restart -> reset -> reboot`가 병렬로 중첩되고, 실제론 "한 사건"인데 세 건의 실패처럼 집계된다.

4. **ledger close 조건과 carry-over 규칙을 분리해야 한다**
   - incident는 단순히 서비스가 살아났다고 바로 닫으면 안 된다. 최소한 아래 중 하나는 만족해야 한다.
     - `stability_hold_ms` 경과
     - 다음 boot stage watermark 도달
     - dependent healthcheck green
   - 반대로 reboot 이후까지 이어지는 사건은 새 boot에서도 참조 가능해야 한다.
   - 따라서 ledger에는 아래 두 상태가 필요하다.
     - `closed_with_recovery`
     - `carried_over_to_next_boot`
   - Day49~Day52에서 다룬 reboot reason, boot success, watermark timeline과 연결하면 "이 reboot가 어떤 open incident를 닫으려 한 조치였는지"를 역추적할 수 있다.

## 리스크

- failure instance ID 규칙이 약하면 같은 root cause가 서로 다른 incident로 쪼개져 restart budget이 우회될 수 있다.
- ledger가 레이어별로 따로 존재하면 observer마다 다른 최종 원인을 주장하게 되어 authority matrix의 단일 decider 원칙이 붕괴된다.
- close 조건 없이 즉시 종료 처리하면 짧은 flap이 매번 신규 incident로 재생성되어 crash storm와 reboot storm 분석이 왜곡된다.

## 다음 액션

- Day57에서는 이 ledger를 실제 운영 지표로 연결하기 위해 **recovery attempt idempotency key와 duplicate action suppression 규칙**을 정리하겠다.

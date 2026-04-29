---
title: R사 X5H Day55 - Service Criticality Class와 Reboot Authority Matrix 표준화
author: JaeHa
date: 2026-04-30 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day55, criticality, reboot, authority, matrix, bringup]
---

Day55는 Day54의 restart budget/escalation 규칙을 실제 시스템 권한 모델로 닫기 위해 **service criticality class**와 **reboot authority matrix 표준화**를 다룬다. X5H bring-up에서 흔한 실패는 서비스가 죽는 것 자체보다, 여러 레이어가 자신이 "가장 중요한 감시자"라고 판단해 각자 reset을 때리는 데 있다. `init`, vendor launcher, domain manager, framework watchdog가 모두 reboot 권한을 가진 상태에서는 같은 장애가 중복 승격되고 증거가 사라진다. 따라서 복구 정책은 `몇 번 재시작할지`뿐 아니라 **어떤 class의 장애에 대해 누가 reboot를 선언할 수 있는지**까지 같이 고정해야 한다.

## 핵심 요약

- 모든 서비스는 `criticality class`를 가져야 하며, class마다 허용 가능한 `restart scope`와 `reboot authority`를 고정해야 한다.
- reboot는 "먼저 감지한 주체"가 아니라 **authority matrix에서 지정된 decider 1개**만 실행해야 한다.
- escalation은 `service -> functional stack -> domain/subsystem -> full system` 순서로 올리되, 각 단계의 승인 주체와 증거 키를 함께 남겨야 한다.

## 코드 포인트

1. **service criticality class를 기능 중요도와 복구 반경 기준으로 나눈다**
   - 단순히 `critical=true/false` 이분법으로는 부족하다. bring-up 기준으로는 최소 아래 정도의 등급이 필요하다.
     - `C0_OBSERVABILITY`: logger, metrics exporter, debug helper
     - `C1_FEATURE`: 개별 feature service, 비핵심 UX 보조 기능
     - `C2_BOOT_PATH`: boot complete/first frame/first property ready를 직접 막는 서비스
     - `C3_SAFETY_PATH`: 전원, 열, 차량 상태, watchdog 판정에 연결되는 서비스
   - 각 class는 아래 필드를 함께 가진다.
     - `max_restart_scope`
     - `max_escalation_level`
     - `reboot_authority_owner`
     - `rollback_eligible`
     - `required_evidence_keys`
   - 예를 들어 `vendor.vhal`은 차량 상태 전파와 framework readiness에 걸치므로 보통 `C2_BOOT_PATH` 또는 SKU에 따라 `C3_SAFETY_PATH`로 관리하는 편이 안전하다.

2. **reboot authority matrix는 감지자와 조치자를 분리해 한 장 표로 고정한다**
   - 최소 행/열은 아래처럼 생각할 수 있다.
     - 행: `failure class` (`service crashloop`, `boot deadline miss`, `domain heartbeat loss`, `thermal safety violation`, `storage corruption suspect`)
     - 열: `observer`, `local action`, `escalation gate`, `final decider`, `max allowed action`
   - 예시 규칙:
     - `service crashloop / C1_FEATURE`: observer=`init`, final decider=`vendor healthd`, max action=`functional stack restart`
     - `boot deadline miss / C2_BOOT_PATH`: observer=`boot metrics hook`, final decider=`boot health state machine`, max action=`subsystem restart`
     - `thermal safety violation / C3_SAFETY_PATH`: observer=`thermal HAL`, final decider=`system safety manager`, max action=`full reboot`
   - 핵심은 관측자가 직접 reboot하지 못하게 막는 것이다. 감지자는 evidence만 남기고, 권한자는 현재 `boot_id`, `failure_instance_id`, `current_escalation_level`을 보고 1회만 결정해야 한다.

3. **restart scope와 reboot scope를 코드/설정에서 분리 표현한다**
   - 흔한 실수는 `restart_policy=critical` 한 줄에 모든 의미를 넣는 것이다.
   - 실제로는 아래처럼 분리하는 편이 운영성이 높다.
     - `local_restart_allowed=true`
     - `functional_stack_restart_allowed=true`
     - `domain_restart_allowed=false`
     - `full_reboot_allowed=false`
     - `decider=boot_healthd`
   - 이렇게 하면 Day53의 timeout owner, Day54의 restart budget, Day49/Day50의 reboot evidence/rollback 로직이 같은 schema로 묶인다.
   - 특히 Android init rc, vendor launcher config, watchdog config가 서로 다른 표현을 쓰더라도 최종 정책 레이어는 공통 필드로 수렴시켜야 drift를 줄일 수 있다.

4. **authority 행사 시 남겨야 할 증거 키를 최소 세트로 강제한다**
   - reboot 또는 subsystem reset 전에는 최소 아래 필드가 남아야 한다.
     - `failure_instance_id`
     - `service_or_component`
     - `criticality_class`
     - `observer`
     - `final_decider`
     - `requested_action`
     - `executed_action`
     - `evidence_anchor`
   - `reboot due to watchdog` 같은 뭉뚱그린 로그는 운영 가치가 낮다.
   - 반대로 `executed_action=subsystem_restart`, `observer=init`, `final_decider=boot_healthd`, `evidence_anchor=hwservicemanager registration timeout + vhal crashloop window overflow`처럼 남기면 원인-판정-조치 체인이 바로 복원된다.

## 리스크

- criticality class가 정의되지 않으면 비핵심 서비스 장애가 안전 경로 장애처럼 과승격되어 reboot storm가 발생할 수 있다.
- observer와 decider가 분리되지 않으면 `init restart`, `domain reset`, `watchdog reboot`가 같은 failure instance에 중첩되어 crash evidence가 유실된다.
- SKU별로 authority matrix가 문서 없이 갈라지면 한 플랫폼에서는 허용되는 조치가 다른 플랫폼에서는 금지되어 현장 재현성과 릴리스 판단 일관성이 무너진다.

## 다음 액션

- Day56에서는 이 class/authority 규칙을 검증 가능한 운영물로 굳히기 위해 **failure instance ID와 cross-layer recovery ledger 표준화**를 정리하겠다.

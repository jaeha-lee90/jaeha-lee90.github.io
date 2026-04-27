---
title: R사 X5H Day53 - Service Critical Path Deadline Budget와 Timeout Ownership 분리 규칙
author: JaeHa
date: 2026-04-28 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day53, deadline, timeout, ownership, critical-path, bringup]
---

Day53은 Day52의 단일 타임라인 규칙을 실제 장애 판정에 연결하기 위해 **service critical path별 deadline budget**과 **timeout ownership 분리 규칙**을 다룬다. X5H bring-up에서 자주 보이는 문제는 timeout이 많다는 사실이 아니라, 같은 구간에 서로 다른 watchdog과 retry가 겹쳐서 `누가 실패를 선언해야 하는지`가 모호해지는 점이다. `init service restart`, `framework boot timeout`, `vendor healthd`, `system watchdog`가 같은 컴포넌트를 각자 감시하면 timeout 자체보다 false recovery와 중복 reset이 먼저 시스템을 망가뜨린다.

## 핵심 요약

- bring-up 안정화의 핵심은 timeout 숫자를 줄이는 것이 아니라 **critical path별 deadline owner를 하나로 고정**하는 것이다.
- 각 stage는 `owner`, `budget_ms`, `grace_ms`, `retry_policy`, `evidence_key`를 함께 정의해야 한다.
- timeout은 하위 레이어가 먼저 감지하더라도, 최종 fail/rollback 판정은 상위 상태기계 하나만 내리도록 분리해야 한다.

## 코드 포인트

1. **critical path를 stage 단위가 아니라 dependency chain 단위로 자른다**
   - 예를 들어 `BOOT_COMPLETED` 하나로 뭉개지 말고 아래처럼 끊는다.
     - `init -> servicemanager -> hwservicemanager`
     - `vold -> first visible storage state`
     - `zygote -> system_server -> bootanim exit`
     - `surfaceflinger -> HWC ready -> first frame`
     - `vendor hal -> car service/vhal property ready`
   - 그래야 Day34~Day41에서 다룬 VINTF, SystemServer, SurfaceFlinger, VHAL 게이트를 같은 budget 표로 연결할 수 있다.

2. **deadline budget은 누적 예산과 구간 예산을 같이 가진다**
   - 각 레코드는 최소 아래 필드를 가져야 한다.
     - `stage`
     - `component`
     - `budget_ms`
     - `cumulative_budget_ms`
     - `owner`
     - `retry_limit`
     - `evidence_key`
   - 예시:
     - `component=surfaceflinger`, `budget_ms=8000`, `cumulative_budget_ms=42000`
     - `component=vhal_service`, `budget_ms=5000`, `cumulative_budget_ms=47000`
   - 단일 구간 예산만 있으면 앞단 지연이 뒷단 timeout으로 잘못 귀속되고, 누적 예산만 있으면 실제 병목 컴포넌트를 못 찾는다.

3. **timeout ownership을 감지자(observer)와 판정자(decider)로 분리한다**
   - 감지자 역할:
     - kernel watchdog
     - init service crash counter
     - vendor health probe
     - framework boot metrics
   - 판정자 역할:
     - boot health state machine 1개
   - 감지자는 `deadline miss evidence`만 남기고, 실제 조치는 판정자가 현재 `boot_id`, `stage`, `retry_budget`를 보고 한 번만 결정해야 한다.
   - 즉, `detected_by=mtee_probe`와 `decision_by=boot_healthd`를 분리 기록해야 중복 reboot storm를 막을 수 있다.

4. **timeout 이벤트는 항상 증거 키와 함께 남긴다**
   - 최소 증거 세트:
     - `boot_id`
     - `stage`
     - `component`
     - `deadline_ms`
     - `observed_ms`
     - `owner`
     - `detected_by`
     - `last_log_anchor`
   - `component timeout` 한 줄만 남기면 운영에서는 다시 원시 로그를 뒤져야 한다.
   - 반대로 `last_log_anchor=SurfaceFlinger: HWC present fence stalled` 같은 고정 anchor를 남기면 Day52의 타임라인과 바로 붙는다.

## 리스크

- watchdog, init restart, vendor healthd가 각각 reboot를 직접 수행하면 동일 실패 1건이 연쇄 reset 3건으로 증폭될 수 있다.
- component budget이 SKU/빌드 타입별로 다르면서 문서화되지 않으면 eng/userdebug에서는 정상인데 user 빌드에서만 timeout이 터지는 비재현성 문제가 생긴다.
- evidence key 없이 timeout owner만 정하면 rollback은 되더라도 실제 병목이 I/O인지 binder starvation인지 HAL deadlock인지 구분이 안 된다.

## 다음 액션

- Day54에서는 이 budget/ownership 규칙을 운영 가능한 형태로 굳히기 위해 **service restart budget과 crashloop escalation 임계치 표준화**를 정리하겠다.

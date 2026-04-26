---
title: R사 X5H Day52 - Boot Stage Watermark/Health Verdict/Event 로그 단일 타임라인 정렬 규칙
author: JaeHa
date: 2026-04-27 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day52, watermark, timeline, health, eventlog, bringup]
---

Day52는 Day51의 상태 보존 계약을 실제 디버깅과 rollback 판정에 연결하는 **단일 타임라인 정렬 규칙**을 다룬다. X5H bring-up에서 가장 흔한 실패는 로그는 많지만 서로 시간이 안 맞는 경우다. bootloader는 monotonic tick, kernel은 `ktime`, Android는 `logcat wall clock`, vendor daemon은 제각각의 timestamp를 남기면 `어디서 죽었는지`보다 `어느 로그를 믿어야 하는지`부터 헷갈린다. 그래서 boot stage watermark, health verdict, event 로그를 하나의 정렬 가능한 축으로 고정해야 Day50~Day51의 retry/rollback 상태기계가 실제 운영에서 의미를 가진다.

## 핵심 요약

- bring-up 판정은 로그 수집량보다 **타임라인 정렬 규칙**이 먼저다.
- 최소 공통 축은 `boot_id + boot attempt + stage sequence + relative monotonic time` 네 개다.
- wall clock은 참고용으로만 두고, rollback/health 판정은 반드시 monotonic 기반 상대시간과 stage watermark 기준으로 계산해야 한다.

## 코드 포인트

1. **모든 이벤트에 공통 상관키를 넣는다**
   - 최소 필드:
     - `boot_id`
     - `boot_attempt`
     - `slot`
     - `stage`
     - `t_rel_ms`
     - `source`
     - `result`
   - 예를 들어 bootloader metadata, kernel printk marker, init property, vendor healthd report가 같은 `boot_id`를 공유하면 재부팅 후에도 같은 부팅 시도의 사건들을 한 줄로 묶을 수 있다.

2. **stage watermark는 순서번호를 가진 enum으로 고정한다**
   - 문자열만 남기면 구현체마다 오타/변형이 생긴다.
   - 예시:
     - `10=BL2_DONE`
     - `20=KERNEL_ENTER`
     - `30=FIRST_STAGE_MOUNT_OK`
     - `40=INIT_SERVICES_OK`
     - `50=VINTF_OK`
     - `60=SYSTEMSERVER_OK`
     - `70=BOOT_COMPLETED`
     - `80=POST_BOOT_STABLE`
   - rollback 로직은 "마지막 성공 stage < 60" 같은 식으로 비교 가능해야 한다.

3. **relative monotonic time을 기준축으로 사용한다**
   - bootloader는 부팅 시작 시점을 `t0`로 두고 tick을 기록한다.
   - kernel은 `boottime` 또는 monotonic 기준으로 marker를 남긴다.
   - Android userspace는 `elapsedRealtime()` 계열 값을 health/event 기록에 포함한다.
   - 이렇게 하면 RTC 불안정, NTP 보정, timezone 변경이 있어도 timeout과 stage budget 계산이 깨지지 않는다.

4. **health verdict는 단일 이벤트가 아니라 stage-aware 집계 결과로 남긴다**
   - 나쁜 예: `healthy=false`
   - 좋은 예:
     - `stage=60`
     - `component=surfaceflinger`
     - `check=service_ready`
     - `deadline_ms=45000`
     - `observed_ms=61200`
     - `verdict=timeout`
   - verdict를 stage와 deadline에 묶어야 Day38~Day49에서 정의한 readiness 게이트와 직접 연결된다.

## 리스크

- wall clock 기준으로만 사건을 정렬하면 RTC reset/NTP sync 직후 로그 순서가 뒤집혀 false root-cause가 나온다.
- `boot_id` 없이 stage marker만 남기면 crashloop 상황에서 이전 부팅의 late log가 다음 부팅의 early log와 섞여 triage가 오염된다.
- health verdict가 component 수준 세부정보 없이 pass/fail 한 줄로만 남으면 rollback은 가능해도 수정 우선순위 결정을 못 한다.

## 다음 액션

- Day53에서는 이 타임라인 규칙을 실제 운영 데이터로 굳히기 위해 **service critical path별 deadline budget 표준화와 timeout ownership 분리 규칙**을 정리하겠다.

---
title: R사 X5H Day51 - persist/pdata 상태 보존 계약과 재부팅 간 판정 일관성
author: JaeHa
date: 2026-04-26 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day51, persist, pdata, state-retention, rollback, bringup]
---

Day51은 Day50의 bootcount/rollback 게이트를 실제로 성립시키는 전제조건인 **재부팅 간 상태 보존 계약**을 다룬다. X5H bring-up에서 rollback 판단은 한 번의 부팅 로그만으로 끝나지 않는다. `boot attempt`, `last failing stage`, `watchdog reason`, `userspace health verdict`가 다음 부팅에서도 읽혀야 같은 실패를 연속 실패로 인식할 수 있다. 이때 `persist`나 vendor `pdata` 성격의 저장영역이 불안정하면 retry budget과 rollback reason이 매번 초기화되어 Day50의 상태기계가 사실상 무력화된다.

## 핵심 요약

- rollback/readiness 설계는 상태기계만 정의해서 끝나지 않고, **어떤 상태를 어디에 얼마나 원자적으로 남길지**까지 계약해야 완성된다.
- 핵심 저장 대상은 `slot retry count`, `boot stage watermark`, `last reboot reason`, `health verdict`, `build fingerprint` 다섯 축이다.
- bring-up 초기에 가장 위험한 패턴은 `persist mount 이전에 필요한 상태를 persist에만 쓰는 것`, 그리고 `write torn/corruption 대비 없이 단일 파일만 덮어쓰는 것`이다.

## 코드 포인트

1. **상태를 boot stage별로 분리 저장**
   - early stage(bootloader/kernel 직후)에서 필요한 값:
     - active slot
     - retry count
     - last reboot reason
     - previous boot stage watermark
   - late stage(Android userspace 안정화 후)에서만 가능한 값:
     - `sys.boot_completed` 도달 여부
     - post-boot stabilization pass/fail
     - vendor daemon health summary
   - 즉, bootloader가 읽어야 하는 상태와 Android가 나중에 보강하는 상태를 같은 파일/같은 타이밍에 섞으면 안 된다.

2. **persist 의존 전용 설계를 피하고 fallback metadata를 둔다**
   - `persist`는 fs check, mount 지연, encryption, SELinux label mismatch 영향으로 초기에 바로 신뢰하지 못할 수 있다.
   - 따라서 최소 rollback 필수 정보는 아래 둘 중 하나로 별도 보존하는 편이 안전하다.
     - bootloader control block / misc partition
     - 작은 이중화 metadata blob (A/B copy + generation counter)
   - `persist`는 상세 진단 데이터 보강용, `misc/pdata`는 부팅 의사결정용으로 역할을 나누는 구성이 안정적이다.

3. **원자적 갱신 규칙을 명문화**
   - 권장 필드:
     - `version`
     - `generation`
     - `slot`
     - `retry_remaining`
     - `last_stage`
     - `last_reason`
     - `health_verdict`
     - `crc`
   - 갱신 규칙 예시:
     1. 새 레코드 작성
     2. CRC 검증
     3. generation 증가
     4. active copy 포인터 전환
   - 전원 차단이나 watchdog reset이 write 중간에 발생해도 이전 유효 레코드로 복구 가능해야 한다.

4. **stage watermark를 명확히 정의**
   - 예: `BL2_DONE`, `KERNEL_ENTER`, `FIRST_STAGE_MOUNT_OK`, `INIT_SERVICES_OK`, `BOOT_COMPLETED`, `POST_BOOT_STABLE`
   - reboot 후 `last_stage`를 읽으면 실패 절단면을 즉시 좁힐 수 있다.
   - Day32~Day37에서 다룬 mount/init/VINTF/SystemServer/crashloop 게이트를 이 watermark 값에 그대로 매핑하면 triage와 rollback reason taxonomy가 일관된다.

## 리스크

- retry counter가 `persist` mount 성공 이후에만 갱신되면, early boot reset은 영원히 같은 첫 실패로 보이게 된다.
- metadata write가 원자적이지 않으면 watchdog reset 직후 `retry_remaining=0` 또는 `unknown` 같은 깨진 상태가 남아 잘못된 rollback을 유발할 수 있다.
- build fingerprint 없이 health verdict만 남기면 OTA/이미지 교체 후 이전 실패 이력이 새 빌드에 오염되어 false rollback이 생긴다.

## 다음 액션

- Day52에서는 이 상태 보존 계약을 실제 관측 체계와 연결하기 위해 **boot stage watermark/health verdict/event 로그의 단일 타임라인 정렬 규칙**을 정리하겠다.

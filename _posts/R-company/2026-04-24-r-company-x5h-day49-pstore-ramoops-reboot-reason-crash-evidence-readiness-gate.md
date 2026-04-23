---
title: R사 X5H Day49 - pstore/ramoops/Reboot Reason crash evidence readiness 안정화 게이트
author: JaeHa
date: 2026-04-24 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day49, pstore, ramoops, reboot-reason, crash, bringup]
---

Day49는 Day48의 time-base readiness 다음 단계로, **커널 패닉이나 watchdog reset이 났는데 다음 부팅에서 원인 증거가 남지 않거나, reboot reason이 뒤섞여 triage가 길어지는 상태**를 다룬다. X5H bring-up에서는 `bootloader reset cause`, `kernel panic path`, `pstore/ramoops reserved memory`, `last_kmsg/console persistence`, `userspace tombstone/dropbox`, `watchdog reboot metadata` 중 하나만 비어도 현상은 단순히 "가끔 죽는다"로 보이지만 실제로는 재현/분류/수정 우선순위 결정이 거의 불가능해진다.

## 핵심 요약

- crash evidence readiness 게이트는 `reset cause capture -> kernel panic evidence persist -> next-boot collection -> userspace correlation -> triage bucketization`을 한 체인으로 봐야 한다.
- bring-up 초기에는 crash 빈도 자체보다 **증거 보존율, reboot reason 일관성, panic-to-next-boot 수집 성공률, watchdog vs brownout vs manual reset 구분성**이 더 중요하다.
- X5H처럼 bootloader, kernel, Android userspace, 가상화 경계가 섞인 구조에서는 `reserved-memory 충돌`, `ramoops 크기/정렬`, `reboot reason register ownership`, `pstore mount timing`, `watchdog reset 후 metadata 유실`이 가장 흔한 초기 실패축이다.

## 코드 포인트

1. **crash evidence stack을 5단계로 절단해서 본다**
   - `reset-cause plane` : PMIC/SoC reset reason register, bootloader reason propagation, warm/cold reset 분류가 이뤄지는 구간
   - `kernel evidence plane` : panic/oops/watchdog 시 console, ftrace, pmsg, ramoops record를 reserved memory에 남기는 구간
   - `boot collection plane` : 다음 부팅에서 pstore mount, last_kmsg 수집, crash marker 정리가 수행되는 구간
   - `userspace correlation plane` : tombstone, dropbox, logcat, service restart history를 reboot reason과 묶는 구간
   - `triage plane` : 수집된 증거를 `kernel panic`, `watchdog bite`, `brownout`, `manual reboot`, `unknown reset`으로 분류하는 구간
   - 이렇게 잘라야 `재부팅됐다`를 ownership 있는 결함으로 바꿀 수 있다.

2. **증거 보존율과 reboot reason 정합성을 readiness 지표로 고정**
   - 최소 계측 항목:
     - bootloader가 읽은 reset cause와 kernel이 인지한 reboot reason
     - panic 발생 시 pstore/ramoops record 생성 여부와 record 크기
     - 다음 부팅에서 pstore mount 성공 시각, record 파싱 성공 여부
     - userspace tombstone/dropbox와 reboot reason correlation 성공률
     - watchdog/panic/manual reboot 주입별 expected bucket 매칭률
   - 핵심은 crash가 났다는 사실보다 **다음 부팅에서 그 crash를 같은 이름으로 다시 설명할 수 있는가**다.

3. **실패 taxonomy를 표준화**
   - `RESET_REASON_LOST` : bootloader 또는 register ownership 문제로 reset cause가 다음 stage에 전달되지 않음
   - `RAMOOPS_LAYOUT_INVALID` : reserved-memory 주소/크기/정렬/overlap 문제로 panic evidence가 저장되지 않음
   - `PSTORE_COLLECTION_FAIL` : kernel에는 남았지만 다음 부팅에서 mount/parsing/permission 문제로 수집에 실패함
   - `WATCHDOG_CLASSIFICATION_DRIFT` : watchdog bite, hard reset, brownout이 같은 generic reboot로 뭉개짐
   - `USERSPACE_CORRELATION_GAP` : tombstone/dropbox/service restart 정보와 kernel evidence가 시간축/ID 기준으로 연결되지 않음
   - 이렇게 분류해야 bootloader/kernel/framework/lab triage가 짧아진다.

4. **pass/fail 기준을 lab 운영 기준으로 승격**
   - 예시 게이트:
     - induced kernel panic 20회 중 pstore evidence 보존율 100%
     - watchdog reset 20회 중 reboot reason bucket 정확도 100%
     - 다음 부팅 후 evidence collection 완료 시간 상한 고정
     - brownout/manual reboot와 panic/watchdog 간 오분류 0건
   - 그래야 `재부팅 후 살아난다`가 아니라 **죽은 원인을 다음 부팅에서 신뢰성 있게 설명 가능한 플랫폼인지**를 판정할 수 있다.

## 리스크

- pstore/ramoops 게이트를 초기에 안 잡으면 불안정 이슈가 생길수록 증거는 더 부족해져, 수정 속도가 오히려 느려진다.
- reboot reason register ownership이 모호하면 watchdog, brownout, manual reboot가 한 bucket으로 섞여 HW/SW 책임 분리가 무너진다.
- crash evidence와 time-base가 함께 불안정하면 로그 순서와 reset cause가 동시에 오염돼 장시간 soak 이슈의 원인 판별이 거의 불가능해진다.

## 다음 액션

- reset cause register, bootloader handoff, kernel pstore/ramoops, userspace collection 경로를 한 타임라인으로 계측 포인트화한다.
- induced panic, watchdog bite, manual reboot, power drop 케이스별 expected reboot reason bucket과 수집 아티팩트 목록을 체크리스트로 고정한다.
- reserved-memory 배치, pstore mount timing, tombstone/dropbox correlation 규칙을 표준화해 crash evidence readiness를 독립 게이트로 승격한다.

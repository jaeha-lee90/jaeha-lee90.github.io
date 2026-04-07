---
title: R사 X5H Day36 - Native Crashloop/Tombstone/Restart Budget 안정화 게이트
author: JaeHa
date: 2026-04-08 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day36, tombstone, crashloop, init, servicemanager, restart, bringup]
---

Day36은 Day35의 `sys.boot_completed=1` 이후를 다룬다. X5H bring-up에서는 화면이 떴다고 끝이 아니다. **vendor native service가 짧은 주기로 죽고 다시 뜨는 crashloop 상태**여도 framework는 일시적으로 살아 보일 수 있고, 이 상태는 곧바로 audio/display/input 지터나 장시간 soak 실패로 이어진다.

## 핵심 요약

- boot complete 뒤에도 **native daemon crashloop, tombstone 발생률, service restart budget 초과 여부**를 별도 게이트로 잡아야 한다.
- `init` 재기동이 계속 성공한다고 해서 정상은 아니다. 핵심은 **재시작 횟수/주기/전파 범위**를 수치화해 unstable boot를 실패로 분류하는 것이다.
- X5H처럼 Xen·vendor HAL·binder 의존성이 큰 구조는 framework ready보다 그 다음 단계인 **post-boot stability window**를 잠가야 실제 제품 품질이 나온다.

## 코드 포인트

1. **관측 대상을 init service 단위로 고정**
   - 최소 추적 대상:
     - display/audio/input 관련 vendor daemon
     - `servicemanager`, `hwservicemanager`, `vndservicemanager`
     - surface/compositor 경로 보조 daemon
   - 수집 키 예시:
     - `init.svc.<name>` 상태 변화
     - `init.svc_debug_pid.<name>` 변동
     - tombstone 생성 시각/프로세스명
     - 최근 N분 restart count

2. **restart budget을 bring-up gate로 정의**
   - 권장 규칙 예시:
     - boot 후 10분 안정화 구간 동안 핵심 vendor daemon restart 0회
     - 비핵심 daemon은 1회까지 허용하되 동일 원인 2회째부터 fail
     - 60초 내 3회 재기동이면 crashloop로 즉시 fail
   - 핵심은 "재기동되면 복구된 것"이 아니라 **재기동이 필요한 상태 자체를 결함**으로 보는 것이다.

3. **tombstone을 기능별 blocker taxonomy에 연결**
   - tombstone 메타데이터에서 최소 추출 항목:
     - process name / pid / tid
     - signal (`SIGSEGV`, `SIGABRT`, `SIGBUS` 등)
     - fault addr / abort message
     - backtrace top frame / vendor lib 이름
   - 분류 축 예시:
     - `BINDER_TIMEOUT`
     - `HAL_CONTRACT_MISMATCH`
     - `MMAP_ION_DMA_FAILURE`
     - `SEPOLICY_DENIAL_FOLLOWUP`
     - `RACE_ON_BOOT_ORDER`

4. **framework success와 stability success를 분리**
   - Day35 완료 조건은 `boot_completed`
   - Day36 완료 조건은 추가로:
     - 안정화 윈도우 내 tombstone 0건
     - 핵심 daemon restart budget 초과 0건
     - `dmesg` fatal/oops/panic follow-up 0건
   - 이 두 단계를 분리해야 "부팅은 됨"과 "실사용 가능"를 혼동하지 않는다.

## 리스크

- display/audio/input daemon이 재기동되면 홈 화면은 보여도 UX는 간헐 실패 상태가 된다.
- tombstone을 모으기만 하고 restart budget과 연결하지 않으면 soak test에서 같은 이슈를 반복 재현하게 된다.
- binder/HAL 계약 불일치가 짧은 crashloop로 나타나는 경우, boot KPI만 보면 양품으로 오판할 수 있다.

## 다음 액션

- boot 완료 후 10분 동안 `init.svc.*` 변화와 tombstone 생성을 묶어 수집하는 post-boot stability 스크립트 초안 작성.
- 핵심 daemon 목록(display/audio/input/vendor HAL bridge)을 확정하고 restart budget을 서비스별로 수치화.
- tombstone 원인을 blocker taxonomy와 연결해 Day30 SOP의 최종 pass/fail 규칙에 추가.

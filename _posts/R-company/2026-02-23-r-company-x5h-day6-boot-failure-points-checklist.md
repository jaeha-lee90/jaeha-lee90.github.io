---
title: R사 X5H Day6 - 기본 부팅 실패 포인트 체크리스트
author: JaeHa
date: 2026-02-23 09:00:00 -0800
categories: [R사, X5H, Bring-up]
tags: [R사, X5H, Boot, Bring-up, Debug]
---

Day1~Day5에서 확인한 boot chain, manifest, kernel module, android_device 초기화 경로를 바탕으로, 초기 브링업에서 실제로 부팅을 멈추게 만드는 실패 지점을 우선순위로 정리한다.

## 핵심 요약

- X5H 초기 부팅 실패는 대체로 **(1) 부팅 이미지/DTB 불일치**, **(2) early init 서비스 실패**, **(3) 벤더 모듈 로딩 실패**, **(4) 파티션/SELinux 정책 불일치** 네 축으로 수렴한다.
- 디버깅 효율을 높이려면 로그를 순차적으로 보지 말고, `bootloader → kernel → first-stage init → late init` 경계에서 **"다음 단계로 넘어갔는지"**를 기준으로 체크해야 한다.
- R사 X5H 브링업 초반에는 기능 확대보다 **정상 부팅 재현성(10회 연속 성공)**을 먼저 달성하는 것이 전체 일정 리스크를 줄인다.

## 코드 포인트

1. **커널 진입 이전(이미지/DTB 정합)**
   - 증상: watchdog reset 반복, 커널 배너 미출력
   - 확인 포인트
     - 부트 아티팩트 조합(boot/vendor_boot/dtb) 버전 매칭
     - bootargs의 console/earlycon 설정 유효성
   - 체크 기준
     - UART에 커널 배너가 보이면 다음 단계로 이동

2. **커널 초기화(드라이버/루트마운트)**
   - 증상: `VFS: Unable to mount root fs`, panic/reboot
   - 확인 포인트
     - cmdline의 root/slot 정보
     - 필수 스토리지 드라이버 built-in 여부
     - dtb의 storage 노드 상태(enabled/주소)
   - 체크 기준
     - `init` 프로세스 시작 로그 확인

3. **first-stage init / ueventd 구간**
   - 증상: 디바이스 노드 미생성, 서비스 시작 전 정지
   - 확인 포인트
     - `init*.rc` import 경로 누락 여부
     - `ueventd*.rc` 권한/owner/mode 설정 오류
     - 필요한 노드(`/dev/*`) 생성 시점
   - 체크 기준
     - 핵심 HAL/daemon 시작 로그 확인

4. **벤더 모듈 로딩 구간**
   - 증상: 디스플레이/네트워크/오디오 등 특정 서브시스템 무응답
   - 확인 포인트
     - 모듈 의존 순서(`modprobe` alias, softdep)
     - 커널 ABI와 모듈 빌드 버전 일치
     - `dmesg`의 unresolved symbol / taint
   - 체크 기준
     - 필수 모듈 로딩 후 해당 서비스가 `running` 상태로 전환

5. **정책/권한(SELinux + property + service context)**
   - 증상: 서비스 반복 재시작, permission denied, property set 실패
   - 확인 포인트
     - `avc: denied` 빈도와 대상 도메인
     - init service의 seclabel/context 불일치
     - vendor/system property namespace 충돌
   - 체크 기준
     - 부팅 완료 후 critical service restart 0회

## 리스크

- 부팅 실패를 단일 원인으로 가정하고 한 레이어만 보는 습관은 해결 시간을 크게 늘린다.
- 임시 우회(SELinux permissive, 모듈 강제 skip) 상태로 진행하면 후속 통합 단계에서 결함이 누적된다.
- manifest pinning 없이 여러 저장소를 동시에 갱신하면 “재현 불가능한 부팅” 상태가 된다.

## 다음 액션

- 부팅 단계별 게이트 체크리스트를 CI smoke 항목으로 축소 적용:
  1) 커널 배너 출력
  2) init 진입
  3) 핵심 서비스 시작
  4) 10회 재부팅 연속 성공
- 실패 시 로그 수집 템플릿을 고정(UART, `dmesg`, `logcat -b all`, `getprop`).
- Day7에서 Day1~Day6 결과를 하나의 **브링업 트리아지 표준 절차**로 정리한다.

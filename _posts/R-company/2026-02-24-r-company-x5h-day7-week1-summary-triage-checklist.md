---
title: R사 X5H Day7 - 1주차 요약과 브링업 트리아지 체크리스트
author: JaeHa
date: 2026-02-24 09:00:00 -0800
categories: [R사, X5H, Bring-up]
tags: [R사, X5H, Bring-up, Triage, Checklist]
---

Day1~Day6에서 정리한 boot chain, manifest 계층, kernel/module 로딩, android_device 초기화, 부팅 실패 패턴을 하나의 브링업 트리아지 절차로 통합한다. 목표는 "원인 탐색"보다 "재현 가능한 정상 부팅 확보"다.

## 핵심 요약

- 1주차 결론은 X5H 초기 브링업 이슈가 **이미지 정합 → 커널 진입 → init/ueventd → 벤더 모듈 → 정책/권한** 5단계 게이트에서 대부분 설명된다는 점이다.
- 디버깅 우선순위는 기능 완성보다 **게이트 통과율 안정화(10회 재부팅 연속 성공)**로 둔다.
- 트리아지는 "로그를 많이 보는 방식"이 아니라, 각 단계의 **진입/탈출 신호를 체크하는 상태기계 방식**이 시간 대비 효율이 가장 높다.

## 코드 포인트

1. **Gate A: Boot artifact 정합성**
   - 입력: `boot.img`, `vendor_boot.img`, `dtb`, bootargs
   - 성공 신호: UART 커널 배너 출력
   - 실패 시 즉시 확인
     - 아티팩트 빌드 버전/manifest revision 매칭
     - `earlycon`, `console` 파라미터 유효성

2. **Gate B: Kernel → init 전이**
   - 성공 신호: `init` 프로세스 시작 로그
   - 실패 시 즉시 확인
     - rootfs/slot 파라미터
     - storage 필수 드라이버 built-in 여부
     - dtb storage node enable 상태

3. **Gate C: first-stage init / ueventd 기본 구성**
   - 성공 신호: 핵심 디바이스 노드 생성 + 필수 daemon start
   - 실패 시 즉시 확인
     - `init*.rc` import 누락
     - `ueventd*.rc` 권한/owner/mode
     - 서비스 class 순서와 property trigger 조건

4. **Gate D: Vendor module 체인**
   - 성공 신호: 필수 모듈 로드 완료 + 관련 서비스 `running`
   - 실패 시 즉시 확인
     - 모듈 의존 순서(softdep/alias)
     - 커널 ABI/모듈 빌드 버전
     - `dmesg` unresolved symbol/taint

5. **Gate E: SELinux/property/service context**
   - 성공 신호: critical service restart 0회, 반복 deny 없음
   - 실패 시 즉시 확인
     - `avc: denied` 도메인/타깃 패턴
     - seclabel/service context 매칭
     - vendor/system property namespace 충돌

## 리스크

- 게이트 구분 없이 병렬 수정하면 원인-결과 연결이 깨져 회귀가 증가한다.
- permissive/임시 우회 상태를 장기 유지하면 2주차(Display/GPU) 진입 시 결함이 폭발한다.
- manifest pinning 없이 팀 단위 실험을 진행하면 동일 증상 재현이 불가능해진다.

## 다음 액션

- Day8부터 Display/GPU 영역 진입 전, 아래 **사전 통과 조건**을 고정한다.
  1) 10회 재부팅 연속 성공
  2) 핵심 서비스 restart 0회
  3) 로그 수집 세트(UART/`dmesg`/`logcat -b all`/`getprop`) 자동 저장
- 브랜치 기준 브링업 베이스라인 태그 1개를 고정해 실험군/대조군 비교 가능 상태를 만든다.
- Day8 주제는 계획 순서대로 **ddk-km 구조와 핵심 모듈 의존 관계** 분석으로 진행한다.

---
title: R사 X5H Day15 - Virtio 채널 맵 정리 (blk/net/serial/audio)
author: JaeHa
date: 2026-03-06 09:00:00 -0800
categories: [R사, X5H, Domain-Integration, Bring-up]
tags: [R사, X5H, Day15, Virtio, DomD, DomA, Bring-up]
---

Day15는 3주차 시작점으로, DomD↔DomA 간 데이터 경로를 기능별(`blk/net/serial/audio`)로 고정해 부팅 후 "장치가 보이는데 동작하지 않는" 통합 초반 장애를 줄이는 데 집중한다. 핵심은 성능이 아니라 **채널 정의의 일관성**이다.

## 핵심 요약

- 초기 통합에서는 virtio 성능 튜닝보다 **채널 존재/바인딩/소유 도메인**을 먼저 고정해야 재현 가능한 디버깅이 가능하다.
- `blk/net/serial/audio` 4개 채널을 단일 표준 이름과 인스턴스 규칙으로 통일하면 DomD 서비스 변경 시 DomA 영향 범위를 빠르게 식별할 수 있다.
- bring-up 게이트는 "디바이스 노출"이 아니라 "노출 + 핸드셰이크 완료 + 기본 I/O 성공" 3단계로 정의해야 한다.

## 코드 포인트

1. **채널 식별자 네이밍 고정**
   - 권장 규칙: `<type>.<domain>.<index>` (예: `net.doma.0`, `serial.doma.0`)
   - ad-hoc 별칭을 제거해 로그/스크립트/런북에서 동일 문자열을 사용한다.

2. **채널별 최소 헬스체크 추가**
   - `blk`: read-only 4KB read 성공 여부
   - `net`: link up + DHCP 또는 고정 IP ping 1회
   - `serial`: 부팅 배너 수신 확인
   - `audio`: PCM open/close 1회 성공
   - 각 체크는 `channel_name`, `peer_domain`, `result`, `latency_ms`를 공통 필드로 남긴다.

3. **DomD 서비스 경계 사전 명시**
   - 각 virtio 백엔드의 소유 서비스(systemd/unit 또는 런처)를 명시해 장애 시 책임 경계를 즉시 분리한다.
   - 동일 백엔드에서 다채널을 제공할 경우 인스턴스별 재시작 가능 여부를 분리 기록한다(전체 재시작 금지).

4. **부팅 시퀀스 의존성 표시**
   - `serial -> net -> blk -> audio`처럼 실제 의존 순서를 문서화하고, 선행 채널 실패 시 후속 채널 결과를 "참고치"로 라벨링한다.
   - 이 규칙으로 mixed 장애를 줄이고 1차 triage 시간을 단축한다.

## 리스크

- 채널 이름 통일 이전 로그와 이후 로그가 섞이면 추세 비교가 왜곡될 수 있다.
- DomD 백엔드 재시작 정책이 불명확하면 단일 채널 장애가 전체 서비스 재기동으로 확대될 수 있다.
- audio 채널은 초기 bring-up에서 비필수로 취급되기 쉬워, 출시 직전 통합 부하가 몰릴 가능성이 높다.

## 다음 액션

- Day16에서 DomD 서비스 단위별 책임 경계(소유 채널, 재시작 범위, 실패 전파 범위)를 표준 템플릿으로 고정한다.
- preflight에 4채널 최소 헬스체크를 넣어 merge 전에 "핸드셰이크 실패"를 차단한다.
- 최근 장애 로그를 채널 단위로 재태깅(`virtio-blk/net/serial/audio`)해 week3 기준선(metric baseline)을 만든다.

---
title: R사 X5H Day44 - Camera HAL/Provider/Stream Pipeline readiness 안정화 게이트
author: JaeHa
date: 2026-04-19 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day44, Camera, HAL, Provider, Stream, bringup]
---

Day44는 Day43의 suspend/resume 안정화 다음 단계로, **부팅은 끝났지만 카메라 open이 느리거나 첫 frame이 안 나오고, stream 전환 또는 resume 이후 preview/capture가 불안정한 상태**를 다룬다. X5H bring-up에서는 `camera provider 등록`, `HAL3 static metadata`, `sensor/CSI/ISP power-up 순서`, `buffer fence/stream format 정합` 중 하나만 어긋나도 앱에서는 단순히 "카메라가 가끔 안 뜬다"로 보이지만 실제 원인은 kernel/vendor/framework 경계 여러 곳에 걸쳐 있다.

## 핵심 요약

- 카메라 readiness 게이트는 `provider 등록 -> camera open -> configureStreams -> first frame -> stream switch/recovery`를 한 흐름으로 봐야 한다.
- bring-up 초기에는 고급 알고리즘보다 **첫 preview frame 도달 시간, stream 재설정 성공률, suspend/resume 이후 재오픈 안정성**이 더 중요하다.
- X5H처럼 Android/vendor/kernel 경계가 두꺼운 구조에서는 provider manifest mismatch, sensor power/reset 순서 불량, gralloc/format/fence 정합 실패가 가장 흔한 초기 실패축이다.

## 코드 포인트

1. **camera stack을 5단계로 절단해서 본다**
   - `framework/provider` : `cameraserver`가 provider service를 찾고 camera id, capability, static metadata를 수집하는 구간
   - `HAL entry` : `open()`/`initialize()`/`configureStreams()` 호출과 stream combination 검증 구간
   - `kernel sensor path` : sensor power rail, reset GPIO, clock, CSI lane, subdev bind, media graph 완성 구간
   - `ISP/buffer path` : pixel format, stride, gralloc usage, fence signal, ISP output node 연결 구간
   - `client lifecycle` : preview start, capture request, stream switch, suspend/resume 후 reopen 구간
   - 이 5단계를 분리해야 `camera open 실패`를 ownership 있는 결함으로 바꿀 수 있다.

2. **first-frame latency와 reopen latency를 readiness 지표로 고정**
   - 최소 계측 항목:
     - provider service 등록 시각과 `cameraserver` discovery 완료 시각
     - `open()` 호출 시각과 `configureStreams()` 완료 시각
     - 첫 유효 preview frame 도달 시각
     - preview -> capture 또는 rear/front 전환 시 stream tear-down/reconfigure 완료 시각
     - suspend/resume 뒤 첫 reopen 성공 시각과 실패 코드 분포
   - 핵심은 카메라 API 존재 여부가 아니라 **첫 usable frame과 재오픈이 정해진 시간 안에 끝나는가**를 관리하는 것이다.

3. **실패 taxonomy를 표준화**
   - `PROVIDER_REGISTRATION_FAIL` : VINTF/manifest 또는 service name mismatch로 provider가 안 뜸
   - `SENSOR_POWER_SEQUENCE_BROKEN` : regulator/reset/clock 순서 불량으로 sensor probe 또는 stream-on 실패
   - `MEDIA_GRAPH_INCOMPLETE` : CSI/ISP/video node bind 실패로 실제 capture path가 완성되지 않음
   - `FORMAT_FENCE_MISMATCH` : HAL stream format, gralloc usage, fence signal 정합 실패로 first frame 지연 또는 black frame 발생
   - `REOPEN_STATE_LOSS` : camera close/reopen, stream switch, suspend/resume 후 내부 상태가 정리되지 않아 busy/dead object가 남음
   - 이렇게 분류해야 framework/HAL/kernel/ISP triage가 짧아진다.

4. **pass/fail 기준을 데모 체감 기준으로 승격**
   - 예시 게이트:
     - cold boot 후 첫 preview frame latency 상한 고정
     - rear/front 또는 preview/capture stream switch 연속 20회 성공
     - suspend/resume 20회 반복 후 camera reopen failure 0건
     - black frame, green frame, fence timeout, provider death 0건
   - 그래야 `카메라 앱 실행됨`이 아니라 **반복 시나리오에서도 usable state를 유지하는가**를 판정할 수 있다.

## 리스크

- provider/VINTF 등록 불일치를 초기에 못 잡으면 앱/CTS 실패가 framework 문제처럼 보이며 분석 시간이 길어진다.
- sensor power-up/CSI bind 순서 문제를 로그 없이 두면 환경 온도나 부팅 타이밍에 따라 간헐 재현으로 바뀌어 양산 리스크가 커진다.
- reopen/state cleanup 불량이 남아 있으면 장시간 데모, 후방카메라 재진입, 절전 복귀 직후에 가장 먼저 무너진다.

## 다음 액션

- provider 등록, `open/configureStreams`, first frame, stream switch, suspend/resume reopen까지 하나의 타임라인으로 계측 항목을 고정.
- sensor regulator/reset/clock, CSI lane, media graph, ISP output node 상태를 한 번에 보는 bring-up 체크리스트를 만든다.
- fence timeout, black/green frame, provider death를 failure taxonomy에 연결해 로그 수집 위치(logcat/dmesg/media graph dump)를 표준화한다.

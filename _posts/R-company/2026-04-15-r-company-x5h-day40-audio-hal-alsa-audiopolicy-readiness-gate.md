---
title: R사 X5H Day40 - Audio HAL/ALSA/AudioPolicy readiness 안정화 게이트
author: JaeHa
date: 2026-04-15 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day40, Audio HAL, ALSA, AudioPolicy, bringup]
---

Day40은 Day39의 입력 readiness 다음 단계로, **UI는 정상인데 부팅 직후 미디어/알림/통화 계열 음성이 나오지 않는 상태**를 다룬다. X5H bring-up에서는 PCM device enumeration 순서, vendor audio HAL init timing, AudioPolicy route 선언, mixer path 적용 순서가 조금만 어긋나도 무음, 잘못된 output route, 첫 재생 실패 후 recover 불가 상태가 바로 드러난다.

## 핵심 요약

- audio bring-up은 `kernel ALSA -> audio HAL -> AudioPolicy -> mixer path`를 하나의 readiness gate로 묶어야 한다.
- 부팅 성공 후에도 **첫 재생 성공 시점, route 전환 성공 여부, underrun/xrun 빈도**를 못 잡으면 무음 불량이 늦게 발견된다.
- X5H처럼 vendor stack 경계가 두꺼운 구조에서는 PCM card/device 번호 drift와 policy XML 불일치가 가장 흔한 초기 실패축이다.

## 코드 포인트

1. **오디오 경로를 4단계로 절단해서 본다**
   - `kernel/ALSA` : card 등록, PCM device 생성, irq/xrun 상태
   - `audio HAL` : module load, stream open, device select, standby/resume
   - `AudioPolicy` : product strategy, output profile, route selection
   - `mixer path` : codec control 적용, speaker/headset/mic 실제 path 전환
   - 이 4단계를 섞어 보면 `무음` 하나로 뭉개지지만, 나눠 보면 ownership이 선다.

2. **첫 재생 readiness를 계측 포인트로 고정**
   - 최소 계측 항목:
     - boot complete 이후 첫 `AudioFlinger` output thread 생성 시각
     - 첫 PCM open 성공 시각
     - launcher 진입 후 notification 또는 test tone 첫 재생 성공 시각
     - route change 후 stable playback 도달 시간
   - 핵심은 사운드 기능 존재 여부가 아니라 **언제부터 안정적으로 소리가 나는가**다.

3. **실패 taxonomy를 표준화**
   - `PCM_NODE_DRIFT` : card/device 번호 변동으로 HAL profile mismatch
   - `HAL_OPEN_FAIL` : stream open 실패 또는 module init race
   - `POLICY_ROUTE_MISMATCH` : AudioPolicy profile/strategy 선언과 실제 target device 불일치
   - `MIXER_APPLY_FAIL` : mixer path 누락/순서 오류로 codec control 미적용
   - `XRUN_UNSTABLE` : 초기 underrun/xrun 반복으로 첫 재생 후 음 끊김
   - 이렇게 나눠야 kernel/vendor/framework triage가 짧아진다.

4. **pass/fail 기준을 사용자 체감으로 승격**
   - 예시 게이트:
     - boot complete 이후 첫 재생 성공 시간 상한 고정
     - speaker/headset/BT 전환 시 route settle 시간 상한 고정
     - 5분 안정화 구간 동안 fatal open failure 0건
     - 반복 재생 시 xrun/underrun 허용 한도 고정
   - 그래야 `오디오 장치 보임`이 아니라 **실사용 가능한 음성 출력 상태**를 판정할 수 있다.

## 리스크

- ALSA card/device 번호 drift를 방치하면 build마다 HAL 설정이 흔들려 재현 불량이 남는다.
- AudioPolicy XML과 vendor HAL capability가 어긋나면 첫 재생은 되더라도 route 전환에서 무음이 터질 수 있다.
- mixer path 적용 실패를 로그 없이 넘기면 codec 문제로 오인되어 디버깅 축이 잘못 잡힌다.

## 다음 액션

- boot/post-boot 수집 항목에 `AudioFlinger`, `audio_hw`, `tinyalsa` 계열 로그와 PCM open/route change 시점을 추가.
- card/device 번호, HAL profile, AudioPolicy route, mixer path를 한 표로 묶어 drift 검출 기준을 고정.
- notification/test tone/headset insert 기준의 첫 재생 성공 시간과 xrun 횟수를 Day38~Day39 readiness 지표와 함께 post-boot gate에 편입.

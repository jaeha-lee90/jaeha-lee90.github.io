---
title: R사 X5H Day12 - Display 성능 병목 후보 (Copy/Sync)
author: JaeHa
date: 2026-03-01 09:00:00 -0800
categories: [R사, X5H, Display, Bring-up]
tags: [R사, X5H, Display, Performance, Copy, Sync, Bring-up]
---

Day12는 Day11에서 고정한 단일 ownership 전제를 바탕으로, 화면이 나오긴 하지만 프레임 드랍/지연이 생기는 구간을 **copy 경로**와 **sync 경로**로 나눠 병목 후보를 압축한다. 목표는 “어디서 느려지는지 모르는 상태”를 끝내고, 계측 가능한 지점부터 우선순위를 세우는 것이다.

## 핵심 요약

- 초기 브링업 성능 이슈의 1순위는 불필요한 버퍼 복사(메모리 대역폭 낭비)와 fence/wait 과동기화(파이프라인 정지)다.
- copy 병목은 주로 **도메인 경계 버퍼 전달 방식**(zero-copy vs memcpy fallback)에서 발생하고, sync 병목은 **단일 프레임에 다중 wait**가 중첩될 때 폭발한다.
- 우선 계측 축은 `copy_bytes_per_frame`, `copy_count_per_frame`, `fence_wait_ms`, `queue_depth` 4개로 고정해 비교 가능성을 확보한다.

## 코드 포인트

1. **Copy 경로: fallback 복사 탐지 우선**
   - 점검: dmabuf export/import 실패 시 memcpy fallback으로 내려가는 분기 존재 여부
   - 구현 포인트: fallback 진입마다 reason code(권한/포맷/stride/캐시속성)를 로그로 남겨 원인별 빈도 집계

2. **Copy 경로: 포맷/stride 불일치에 따른 재패킹 비용**
   - 점검: producer 포맷과 consumer 요구 포맷이 달라 매 프레임 swizzle/repack이 발생하는지 확인
   - 구현 포인트: 협상 가능한 공통 포맷(예: NV12/RGBA 중 1개)으로 고정하고, 변환은 초기 1회 또는 오프라인 경로로 밀어낸다.

3. **Sync 경로: fence 대기 중복 제거**
   - 점검: acquire fence + compositor wait + display commit wait가 직렬로 누적되는지 확인
   - 구현 포인트: 동일 프레임 토큰 기준 중복 wait를 제거하고, timeout/slow-path만 경고 로그로 승격

4. **Sync 경로: queue depth와 backpressure 분리 관측**
   - 점검: producer queue가 먼저 차는지, display commit이 밀려 consumer queue가 차는지 분리
   - 구현 포인트: 도메인별 queue depth watermark를 별도 기록해 “복사 과다”와 “동기화 정체”를 혼동하지 않게 한다.

## 리스크

- zero-copy를 목표로 급하게 경로를 바꾸면 cache coherence 누락으로 간헐 artifact/tearing이 발생할 수 있다.
- sync 단순화 과정에서 timeout 정책을 약화하면 장애 탐지 시간이 늦어져 복구가 지연될 수 있다.
- 팀별로 “평균 FPS”만 공유하면 tail latency(최악 프레임) 병목이 가려져 실사용 체감 저하를 놓칠 위험이 있다.

## 다음 액션

- 이번 주 기준선 로그 포맷에 `copy_count_per_frame`, `copy_bytes_per_frame`, `fence_wait_ms_p95`, `queue_depth_max`를 추가한다.
- Day11 ownership 고정 시나리오 1개를 골라 10분 런에서 copy/sync 지표를 수집하고 병목 축을 1차 분류한다.
- Day13에서는 계획 순서대로 **디버깅 포인트와 로그 전략**을 지표 중심으로 정리해 재현/비교 루프를 완성한다.

---
title: R사 X5H Day45 - Wi-Fi/Bluetooth Firmware/NVRAM readiness 안정화 게이트
author: JaeHa
date: 2026-04-20 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day45, WiFi, Bluetooth, Firmware, NVRAM, bringup]
---

Day45는 Day44의 Camera readiness 다음 단계로, **부팅은 완료되지만 Wi-Fi가 가끔 attach되지 않거나 Bluetooth enable/pairing이 불안정하고, 재부팅·resume 이후 연결 상태가 흔들리는 문제**를 다룬다. X5H bring-up에서는 `chipset firmware download`, `board-specific NVRAM/calibration`, `wlan/bt driver bring-up 순서`, `supplicant/bluetooth stack state machine` 중 하나만 어긋나도 증상은 단순 연결 실패처럼 보이지만 실제 원인은 kernel/vendor/framework 경계 여러 곳에 흩어진다.

## 핵심 요약

- connectivity readiness 게이트는 `firmware/NVRAM load -> driver attach -> HIDL/AIDL or system service ready -> scan/pair/connect -> resume/reconnect`를 한 흐름으로 봐야 한다.
- bring-up 초기에는 throughput 최적화보다 **칩셋 attach 성공률, 첫 scan latency, pairing/reconnect 안정성, resume 후 auto-recovery**가 더 중요하다.
- X5H처럼 Android/vendor/kernel 분리가 두꺼운 구조에서는 firmware path mismatch, calibration/NVRAM 누락, MAC/address provisioning 오류, supplicant/bluetooth daemon 재시작 루프가 가장 흔한 초기 실패축이다.

## 코드 포인트

1. **Wi-Fi/Bluetooth 경로를 5단계로 절단해서 본다**
   - `firmware asset plane` : 칩셋 firmware, patchram, NVRAM/calibration 파일 경로와 버전 정합
   - `kernel attach plane` : SDIO/PCIe/UART attach, power/reset GPIO, IRQ, rfkill, netdev/hci 등록
   - `vendor control plane` : vendor daemon, firmware downloader, coex manager, country code/MAC provisioning
   - `Android service plane` : wificond/supplicant, bluetooth manager stack, adapter state transition
   - `runtime plane` : scan, pairing, DHCP, reconnect, suspend/resume 이후 auto-recover
   - 이 5단계를 분리해야 `와이파이가 가끔 안 잡힘`을 ownership 있는 결함으로 바꿀 수 있다.

2. **first-usable latency와 reconnect latency를 readiness 지표로 고정**
   - 최소 계측 항목:
     - boot 후 firmware/NVRAM load 완료 시각
     - wlan netdev up, hci attach, adapter ON 완료 시각
     - 첫 scan result 도달 시각과 첫 association/DHCP 완료 시각
     - BT enable 후 첫 inquiry/advertising/pairing 완료 시각
     - suspend/resume 또는 service restart 뒤 reconnect 성공 시각
   - 핵심은 interface 존재 여부가 아니라 **실제 연결 가능한 상태에 도달하는 시간과 반복성**을 관리하는 것이다.

3. **실패 taxonomy를 표준화**
   - `FIRMWARE_PATH_MISMATCH` : firmware/patchram/NVRAM 파일 경로 또는 버전 불일치로 attach 실패
   - `CALIBRATION_OR_MAC_INVALID` : board calibration 값 또는 MAC/address provisioning 오류로 불안정 동작
   - `ATTACH_SEQUENCE_BROKEN` : power/reset/IRQ/UART/SDIO/PCIe 초기화 순서 불량으로 wlan/hci 등록 실패
   - `SERVICE_STATE_MACHINE_STUCK` : supplicant/bluetooth daemon이 enable/disable 또는 reconnect 루프에서 멈춤
   - `RESUME_RECONNECT_LOSS` : 절전 복귀 후 scan은 되지만 association/pairing 재개가 깨지거나 시간이 과도하게 늘어남
   - 이렇게 분류해야 kernel/vendor/framework triage가 짧아진다.

4. **pass/fail 기준을 데모 체감 기준으로 승격**
   - 예시 게이트:
     - cold boot 후 Wi-Fi scan, association, DHCP 완료 시간 상한 고정
     - BT adapter enable, pair, reconnect 연속 20회 성공
     - suspend/resume 30회 반복 후 Wi-Fi/BT auto-recovery failure 0건
     - firmware load failure, hci attach failure, supplicant restart storm, DHCP timeout 0건
   - 그래야 `설정 화면에서 켜짐`이 아니라 **실사용 연결성이 반복 시나리오에서도 유지되는가**를 판정할 수 있다.

## 리스크

- firmware/NVRAM 경로 문제를 초기에 못 잡으면 환경 따라 재현율이 바뀌어 "가끔만 안 됨" 형태로 오래 남는다.
- MAC/address provisioning이나 country code 관리가 불안정하면 인증, 규제, coexistence 이슈가 뒤늦게 한꺼번에 터진다.
- resume/reconnect 게이트가 없으면 장시간 차량 대기 후 복귀 시 네트워크·전화 연동 데모가 가장 먼저 무너진다.

## 다음 액션

- firmware/NVRAM load, driver attach, service ready, first scan/pair/connect, resume reconnect를 한 타임라인으로 계측한다.
- Wi-Fi/BT 칩셋별 firmware 파일명, 경로, board calibration, MAC/address source를 보드 기준 체크리스트로 고정한다.
- supplicant/bluetooth daemon restart, hci attach, DHCP timeout, reconnect failure를 taxonomy에 연결해 logcat/dmesg/vendor log 수집 위치를 표준화한다.

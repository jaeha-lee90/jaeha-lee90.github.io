---
title: R사 X5H Day46 - USB Role Switch/ADB/Peripheral Enumeration readiness 안정화 게이트
author: JaeHa
date: 2026-04-21 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day46, USB, Type-C, ADB, Enumeration, bringup]
---

Day46은 Day45의 Wi-Fi/Bluetooth readiness 다음 단계로, **부팅은 되지만 USB device/host 전환이 불안정하고 ADB가 간헐적으로 안 붙거나, 부팅 후 특정 주변기기 인식이 흔들리는 상태**를 다룬다. X5H bring-up에서는 `Type-C role detection`, `extcon/USB controller 초기화`, `gadget/configfs 조립`, `PHY/power/clock`, `userspace function enable 타이밍` 중 하나만 틀어져도 증상은 단순히 "USB가 가끔 안 된다"로 보이지만 실제 원인은 bootloader/kernel/vendor/framework 경계에 분산된다.

## 핵심 요약

- USB readiness 게이트는 `role detect -> controller/PHY ready -> host or gadget enumerate -> data path usable -> reconnect/recovery`를 한 흐름으로 봐야 한다.
- bring-up 초기에는 고속 성능보다 **ADB attach 성공률, host/device role 전환 반복성, cold boot/reboot/resume 후 재인식 안정성**이 더 중요하다.
- X5H처럼 보드 전원 시퀀스와 Android userspace가 강하게 엮인 구조에서는 `VBUS/ID/CC 감지`, `dr_mode/extcon 설정`, `configfs gadget 조립`, `USB PHY reset/clock`, `SELinux/property gating`이 가장 흔한 초기 실패축이다.

## 코드 포인트

1. **USB stack을 5단계로 절단해서 본다**
   - `role detect plane` : Type-C/CC, ID pin, extcon, PD policy 결과로 host/device role이 결정되는 구간
   - `controller/phy plane` : xHCI/DWC3/PHY probe, reset, clock, regulator, OTG state machine이 살아나는 구간
   - `gadget or host plane` : gadget configfs 조립(ADB/MTP/RNDIS) 또는 host enumeration(HID/MSC/ethernet)이 실제로 일어나는 구간
   - `userspace policy plane` : `init.rc`, property, `adbd`, usb hal/service, permission/policy가 기능을 열어 주는 구간
   - `runtime recovery plane` : cable replug, warm reboot, suspend/resume, role swap 이후 다시 usable state로 복귀하는 구간
   - 이 5단계를 분리해야 `ADB가 가끔 안 붙음`을 ownership 있는 결함으로 바꿀 수 있다.

2. **first-enumeration latency와 role-swap recovery를 readiness 지표로 고정**
   - 최소 계측 항목:
     - cable insert 또는 boot 후 role 결정 시각
     - controller probe 완료, PHY ready, extcon state publish 시각
     - gadget bind 또는 host enumeration 완료 시각
     - `adbd` ready 시각과 host에서 실제 device node/serial 인식 완료 시각
     - replug, warm reboot, suspend/resume 뒤 재인식 성공 시각
   - 핵심은 포트가 존재하느냐가 아니라 **실제 데이터 경로가 살아나는 시간과 반복 안정성**을 관리하는 것이다.

3. **실패 taxonomy를 표준화**
   - `ROLE_DETECTION_MISMATCH` : Type-C/ID/extcon 상태가 실제 케이블 상태와 어긋나 host/device role이 잘못 결정됨
   - `PHY_OR_CONTROLLER_BRINGUP_FAIL` : PHY reset/clock/regulator 또는 controller probe 실패로 enumeration 자체가 시작되지 않음
   - `GADGET_COMPOSITION_BROKEN` : configfs function 조립, UDC bind, descriptor 구성 오류로 ADB/MTP/RNDIS가 안 뜸
   - `USERSPACE_POLICY_BLOCK` : property, SELinux, init trigger, usb hal state machine 문제로 kernel은 살아 있어도 기능이 안 열림
   - `RECOVERY_ON_REPLUG_FAIL` : replug/resume/role swap 뒤 이전 상태가 정리되지 않아 두 번째 attach부터 실패함
   - 이렇게 분류해야 bootloader/kernel/vendor/framework triage가 짧아진다.

4. **pass/fail 기준을 lab 체감 기준으로 승격**
   - 예시 게이트:
     - cold boot 후 ADB attach latency 상한 고정
     - host/device role swap 연속 20회 성공
     - HID/USB storage/ethernet class 최소 셋에 대해 enumeration success 100%
     - warm reboot, suspend/resume, replug 시 ADB loss 또는 ghost device 0건
   - 그래야 `USB icon이 보임`이 아니라 **개발·검증에 필요한 연결성이 반복 시나리오에서도 유지되는가**를 판정할 수 있다.

## 리스크

- role detect/extcon 불일치를 초기에 못 잡으면 케이블·허브·호스트 조합 따라 재현율이 바뀌어 장비 탓으로 오판하기 쉽다.
- gadget 조립과 userspace policy 경계가 흐리면 kernel 로그는 정상인데 ADB/MTP만 죽는 반쪽 장애가 오래 남는다.
- replug/resume recovery 게이트가 없으면 양산 이전까지는 멀쩡해 보여도 개발실·공장 flashing 환경에서 가장 먼저 무너진다.

## 다음 액션

- role detect, PHY/controller ready, gadget/host enumeration, `adbd` usable, replug/resume recovery까지 한 타임라인으로 계측 항목을 고정한다.
- `dr_mode`, extcon/typec state, UDC bind, configfs function 조합, `sys.usb.config` 전이를 한 번에 보는 체크리스트를 만든다.
- ADB attach 실패, enumeration timeout, role swap 실패, ghost device 재현 케이스를 failure taxonomy에 연결해 dmesg/logcat/sysfs snapshot 수집 위치를 표준화한다.

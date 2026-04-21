---
title: R사 X5H Day47 - Ethernet PHY/MAC/DHCP/Link Stability readiness 안정화 게이트
author: JaeHa
date: 2026-04-22 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day47, Ethernet, PHY, MAC, DHCP, Link, bringup]
---

Day47은 Day46의 USB readiness 다음 단계로, **보드는 뜨지만 유선 네트워크 링크가 늦게 올라오거나, DHCP가 불안정하거나, 부팅 후 몇 분 내 link flap이 반복되는 상태**를 다룬다. X5H bring-up에서는 `PHY reset/strap`, `MAC probe/clock`, `MDIO`, `device tree pinmux/regulator`, `userspace netd/dhcp client`, `suspend/resume 이후 carrier 복구` 중 하나만 흔들려도 현상은 단순히 "이더넷이 가끔 안 된다"로 보이지만 실제 원인은 bootloader/kernel/vendor/framework 경계에 분산된다.

## 핵심 요약

- Ethernet readiness 게이트는 `PHY detect -> MAC attach -> carrier up -> IP lease/static config -> sustained traffic -> resume/recovery`를 한 흐름으로 봐야 한다.
- bring-up 초기에는 throughput peak보다 **cold boot link-up 시간, DHCP lease 안정성, link flap 빈도, resume 후 carrier 복구 시간**이 우선이다.
- X5H처럼 보드별 PHY 전원/strap/clock 영향이 큰 구조에서는 `reset timing`, `RGMII/SGMII mode`, `MDIO visibility`, `pinmux/regulator`, `netd/dhcp sequencing`이 가장 흔한 초기 실패축이다.

## 코드 포인트

1. **Ethernet stack을 5단계로 절단해서 본다**
   - `hardware detect plane` : PHY power, reset GPIO, strap, reference clock, MDIO 응답이 정상인지 확인하는 구간
   - `kernel attach plane` : MAC driver probe, PHY attach, interface register, carrier state publish가 이뤄지는 구간
   - `network config plane` : DHCP/static IP, route, DNS, firewall policy가 실제 통신 가능 상태를 만드는 구간
   - `traffic validation plane` : ping, gateway reachability, sustained packet loss, duplex/speed negotiation을 검증하는 구간
   - `runtime recovery plane` : cable replug, warm reboot, suspend/resume 뒤 다시 carrier/IP가 복구되는 구간
   - 이 단계 분리가 되어야 `링크가 늦게 뜬다`를 ownership 있는 결함으로 바꿀 수 있다.

2. **first-link-up latency와 carrier flap budget을 readiness 지표로 고정**
   - 최소 계측 항목:
     - boot 시작 후 PHY detect 시각
     - MAC probe 완료, netdev 등록, carrier on 시각
     - DHCP discover/start, lease 획득, default route 활성 시각
     - 첫 gateway ping 성공 시각
     - suspend/resume, cable replug 뒤 carrier 재획득 시각
   - 핵심은 인터페이스가 존재하느냐가 아니라 **실제 통신 가능한 상태까지 가는 시간과 그 상태의 지속성**이다.

3. **실패 taxonomy를 표준화**
   - `PHY_POWER_OR_RESET_FAIL` : PHY 전원, reset, strap, clock 문제로 MDIO detect 자체가 실패함
   - `MAC_PHY_ATTACH_FAIL` : driver probe 또는 interface mode 불일치로 carrier state가 정상 publish되지 않음
   - `NEGOTIATION_OR_LINK_FLAP` : speed/duplex auto-negotiation, 케이블 민감도, EMI 영향으로 link up/down이 반복됨
   - `IP_CONFIG_FAIL` : carrier는 올라오지만 DHCP/static route/DNS 설정이 불완전해 실제 통신이 안 됨
   - `RESUME_RECOVERY_FAIL` : replug/resume 이후 이전 state가 정리되지 않아 두 번째부터 lease 재획득이 깨짐
   - 이렇게 분류해야 board/kernel/network userspace triage가 짧아진다.

4. **pass/fail 기준을 lab 운영 기준으로 승격**
   - 예시 게이트:
     - cold boot 후 carrier up 및 DHCP lease latency 상한 고정
     - 12시간 soak 동안 link flap 0건
     - cable replug 20회 연속 성공
     - suspend/resume 20회 뒤 carrier/IP/gateway reachability 100% 복구
   - 그래야 `ifconfig에 eth0가 보임`이 아니라 **개발/검증/OTA 환경에서 믿고 쓸 수 있는 네트워크인지**를 판정할 수 있다.

## 리스크

- PHY reset/strap 문제를 초기에 못 잡으면 보드 편차나 스위치/허브 차이로 재현율이 흔들려 HW와 SW가 서로 책임을 미루기 쉽다.
- carrier와 IP layer를 분리해 보지 않으면 `링크는 올라옴`과 `실통신 가능`이 섞여 DHCP/netd 문제를 PHY 문제로 오판할 수 있다.
- resume/replug 게이트가 없으면 개발실에서는 간헐 장애로만 보이다가 장시간 검증이나 OTA 회귀에서 크게 터진다.

## 다음 액션

- PHY detect, MAC attach, carrier on, DHCP lease, gateway ping 성공까지 한 타임라인으로 계측 항목을 고정한다.
- device tree의 reset GPIO, regulator, PHY mode, clock, MDIO address와 kernel probe 로그를 한 체크리스트에서 같이 본다.
- link flap, negotiation mismatch, DHCP timeout, resume 후 lease loss 재현 케이스를 failure taxonomy에 연결해 dmesg/logcat/ethtool(if available)/ip addr snapshot 수집 위치를 표준화한다.

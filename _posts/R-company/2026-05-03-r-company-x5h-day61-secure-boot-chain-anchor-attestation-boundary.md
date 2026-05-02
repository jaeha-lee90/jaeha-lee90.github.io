---
title: R사 X5H Day61 - Secure Boot Chain Anchor와 인증 경계
author: JaeHa
date: 2026-05-03 01:02:00 +0900
categories: [R사, X5H, Security, Bring-up, BSP]
tags: [R사, X5H, Day61, secure-boot, rsip, tf-a, op-tee, xen, bringup]
---

Day60에서 X5H 전체 아키텍처를 한 번에 펼쳤다면, Day61은 그중에서도 bring-up 초기에 가장 먼저 틀어지면 안 되는 **secure boot chain**만 따로 잘라 본다. X5H에서는 단순히 "커널이 올라오느냐"보다, **어느 이미지가 어느 저장매체에서 먼저 검증·적재되고, secure world가 어디까지 책임지는지**를 고정해야 이후의 rollback, crash evidence, OTA 판정이 흔들리지 않는다.

## 핵심 요약

- X5H의 boot anchor는 `HyperFlash` 쪽 **1st IPL / RSIP lane** 에서 시작되고, A-core 일반 실행 전 **BL31(TF-A) → OP-TEE → U-Boot → Xen** 순서로 secure boundary를 지난다.
- R52/SCP/NPU 펌웨어가 A-core Linux/Android보다 먼저 올라오기 때문에, secure boot 설계는 **A-core chain 검증**만이 아니라 **선행 RT firmware provenance**까지 묶어 봐야 한다.
- bring-up 관점의 핵심 게이트는 세 가지다: **(1) 이미지 슬롯/주소 정합**, **(2) secure hand-off 증거 확보**, **(3) boot failure를 secure stage까지 역추적 가능한 공통 watermark**.

## 코드 포인트

1. **실제 적재 순서는 `gen5-prebuilts/ipls/x5h_bootloaders.yaml` 이 단일 진실 원본이다**  
   Day60에서 확인한 주소 맵을 다시 보면 secure chain의 anchor가 분명하다.

   ```text
   HyperFlash @0x00000000  1st_ipl_normal_boot_certificate_a_b
   HyperFlash @0x00040000  1st_ipl_rsip_m
   HyperFlash @0x00440000  images_normal_boot_certificate_a
   HyperFlash @0x007C0000  2nd_ipl_rt_core_cluster2_core0

   UFS @0x05000000  bl31-ironhide
   UFS @0x05200000  tee-ironhide
   UFS @0x07200000  u-boot-elf-ironhide
   ```

   여기서 bring-up 팀이 먼저 확인해야 할 건 "커널 이미지가 맞는가"가 아니라, **certificate/RSIP lane/BL31/TEE/U-Boot slot 배치가 서로 같은 릴리즈 집합인가**다. 이 축이 어긋나면 Linux 로그는 거의 남지 않는데 현상은 단순 boot hang처럼 보인다.

2. **`1st_ipl_rsip_m → BL31 → OP-TEE` 는 단순 연쇄가 아니라 책임 경계선이다**  
   - `1st_ipl_rsip_m`: secure 이미지 적재의 출발점
   - `BL31(TF-A)`: EL3 monitor, 다음 world로의 hand-off 결정점
   - `OP-TEE`: secure service / key material / trusted storage 확장점
   - `U-Boot`: non-secure world로 넘어가기 직전의 boot policy 실행점

   즉 Day50~Day56에서 다룬 `boot success`, `rollback`, `failure_instance_id` 를 A-core userspace에서만 관리하면 절반짜리다. 최소한 **BL31 진입 여부**, **OP-TEE hand-off 성공 여부** 정도는 별도 boot watermark로 남겨야 secure lane에서 실패한 사건과 Linux 이후 실패를 구분할 수 있다.

3. **도메인 런타임이 Xen인 만큼 secure boot의 종료 조건을 `kernel boot`로 두면 부족하다**  
   `meta-xt-x5h-dev/prod-devel-rcar-gen5.yaml` 기준 최종 런타임 타깃은 `DomD + DomU + DomA(Android)` 다. 그래서 secure boot 관점의 실질 완료 조건은 아래처럼 잡는 편이 맞다.
   - `BL31_OK`
   - `OPTEE_OK`
   - `UBOOT_OK`
   - `XEN_LOADED`
   - `DOMD_BOOTSTRAP_OK`

   특히 X5H는 driver domain인 DomD가 실제 HW access를 쥐고 있으므로, **Xen까지 떴는데 DomD DTB/bootargs가 틀린 경우**도 사용자는 "부팅 실패"로 체감한다. secure boot와 domain bring-up gate를 분리하되, 판정 체계는 같은 timeline에 올려야 한다.

4. **R52 선행 부팅 때문에 "secure success / functional fail" 조합이 자주 나온다**  
   X5H는 2nd IPL 이후 R52 multifwk와 SCP가 먼저 살아난다. 그래서 아래 패턴이 가능하다.
   - secure chain은 정상 통과
   - Xen도 기동
   - 하지만 RT firmware ABI mismatch 때문에 RPMsg/disfwk/camfwk가 불능

   이 경우 secure boot는 성공이지만 제품 bring-up은 실패다. 따라서 release gate는 하나로 뭉뚱그리면 안 되고, **secure provenance green** 과 **cross-domain functionality green** 을 별도 상태로 유지해야 한다.

## 리스크

- **릴리즈 집합 불일치**: `x5h_bootloaders.yaml` 의 certificate/BL31/OP-TEE/U-Boot 배치와 실제 패키징 산출물이 어긋나면 부팅 초반 hang가 나도 Linux 증거가 거의 없다.
- **secure stage 가시성 부족**: BL31/OP-TEE hand-off 성공 여부를 별도 watermark로 남기지 않으면 Day49~Day56의 reboot reason/ledger 체계가 secure lane 앞에서 끊긴다.
- **RT firmware 검증 사각지대**: A-core secure chain만 보고 green 처리하면, 먼저 떠야 하는 R52/SCP/NPU 이미지의 버전·서명·조합 불일치가 늦게 발견된다.
- **Xen 이후 판정 누락**: secure boot 완료와 제품 usable 상태를 같은 뜻으로 취급하면 DomD 부팅 실패, passthrough 오류, Android 미기동 같은 현상이 release 직전까지 숨어든다.

## 다음 액션

다음 글에서는 Day61의 secure anchor 위에 이어서, **Xen DTB 분할과 IPMMU/passthrough 메모리 ownership** 을 정리해 DomD/DomA가 실제로 어떤 HW 경계를 갖는지 더 좁혀 보겠다.

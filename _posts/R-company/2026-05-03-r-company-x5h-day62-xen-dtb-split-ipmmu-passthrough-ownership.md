---
title: R사 X5H Day62 - Xen DTB 분할과 IPMMU/passthrough Ownership Ledger
author: JaeHa
date: 2026-05-03 06:05:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day62, xen, dtb, ipmmu, passthrough, reserved-memory, bringup]
---

Day61에서 secure boot chain의 anchor를 봤다면, 이제 부팅이 끝난 뒤 **누가 어떤 HW와 메모리를 실제로 소유하는지**를 좁혀 볼 차례다. X5H Xen 구성에서는 이 ownership이 문서 한 장에 적혀 있는 게 아니라, **Xen용 DTB / DomD·guest용 DTB / reserved-memory carveout / IPMMU binding** 에 분산돼 있다. 그래서 bring-up에서 자주 나오는 "buffer 주소는 맞는데 화면이 안 뜬다", "DomD에서는 보이는데 DomA에서는 안 된다" 같은 현상은 단순 드라이버 버그가 아니라 **ownership ledger 해석 실패**인 경우가 많다.

## 핵심 요약

- `prod-devel-rcar-gen5.yaml` 의 `XT_DEVICE_TREES` 는 단순 빌드 산출물 목록이 아니라, **Xen이 보는 ownership DTB와 각 도메인이 해석하는 런타임 DTB가 분리돼 있음을 선언**한다.
- `r8a78000-ironhide-xen.dts` 에서는 `xen,passthrough`, SMMU enable/disable, reserved-memory carveout을 한 자리에서 묶어 **어느 장치와 주소 대역을 어느 도메인 체계로 넘길지**를 정한다.
- guest DTS 의 `shared-dma-pool` 은 "이 메모리를 같이 쓰자"는 의도만 표현한다. **실제 DMA 가시성은 IPMMU/SMMU binding이 맞아야** 생긴다.
- 그래서 bring-up triage는 `device node → iommus → reserved-memory → guest memory-region` 순서로 ownership ledger를 따라 읽는 게 가장 빠르다.

## 코드 포인트

1. **DTB split 자체가 ownership ledger다**  
   `meta-xt-x5h-dev/prod-devel-rcar-gen5.yaml` 에서는 DomD 빌드가 여러 DTB를 한 번에 다룬다.

   ```yaml
   XT_DOMD_DTB_NAME: "%{SOC_FAMILY}-%{DOMD_MACHINE}-domd.dtb"
   XT_XEN_DTB_NAME: "%{SOC_FAMILY}-%{DOMD_MACHINE}-xen.dtb"
   ...
   XT_DEVICE_TREES = "%{XT_DOMD_DTB_NAME} %{XT_XEN_DTB_NAME} %{XT_DOMA_DTB_NAME}"
   ```

   이건 단순히 파일이 여러 개 나온다는 뜻이 아니다.
   - `xen.dtb`: 하이퍼바이저가 **무엇을 직접 예약/격리/패스스루할지** 결정
   - `domd.dtb` / `doma.dtb`: 각 도메인이 **그 자원을 어떤 이름과 정책으로 재해석할지** 결정

   즉, 같은 `0xa2000000` 대역이라도 Xen 입장에서는 reserve 대상이고, guest 입장에서는 `shared-dma-pool` 이며, 상위 SW 입장에서는 display/camera shared buffer일 수 있다.

2. **Xen DTB는 장치 ownership과 DMA visibility를 같이 고정한다**  
   `r8a78000-ironhide-xen.dts` 앞부분을 보면 SMMU부터 먼저 갈라 놓는다.

   ```dts
   &smmu_hcn  { status = "okay"; };
   &smmu_pere { status = "okay"; };
   &smmu_pv   { status = "okay"; };
   &smmu_hcs1 { status = "okay"; };

   &smmu_perw { status = "disabled"; };
   &smmu_dsp  { status = "disabled"; };
   &smmu_vi0  { status = "disabled"; };
   &smmu_vi1  { status = "disabled"; };
   ```

   그리고 바로 아래에서 bus master를 특정 SMMU에 묶는다.

   ```dts
   &rswitch3 { iommus = <&smmu_hcn ...>; };
   &mmc0     { iommus = <&smmu_pere 0x20000>; };
   gsx_osid0_domd { iommus = <&smmu_pv 0x00> ...; xen,passthrough; };
   gsx_osid1      { iommus = <&smmu_pv 0x01> ...; xen,passthrough; };
   &pciec60  {
       iommus = <&smmu_hcs1 0x20000>;
       iommu-map = <0x0 &smmu_hcs1 0x20000 0x10000>;
   };
   ```

   bring-up에서 중요한 건 `xen,passthrough` 만 보는 게 아니라, **그 장치가 어느 SMMU domain을 타는지까지 같이 봐야 한다**는 점이다. passthrough가 맞아도 IPMMU binding이 틀리면 주소만 있고 DMA는 실패한다.

3. **reserved-memory 는 공유 계약서이고, guest DTS 는 그 계약의 재해석본이다**  
   Xen DTB reserved-memory 블록은 ownership 경계를 크게 잘라 둔다.

   ```dts
   vdev_regions-domd {
       reg = <0x0 0xa0000000 0x0 0x1000000>;
       no-map;
   };

   vdev_region-domu {
       reg = <0x0 0xa1000000 0x0 0x1000000>;
       no-map;
   };

   rvgc_region-domu {
       reg = <0x0 0xa2000000 0x0 0x10000000>;
       no-map;
   };

   rvgc_region-domd {
       reg = <0x0 0xb2000000 0x0 0xe000000>;
       no-map;
   };
   ```

   그런데 guest 쪽 `doma-virtio.dts` 에서는 같은 공간이 이렇게 다시 의미를 얻는다.

   ```dts
   vdev0vring0 { compatible = "shared-dma-pool"; reg = <0x0 0xa1000000 0x0 0x8000>; no-map; };
   vdev0vring1 { compatible = "shared-dma-pool"; reg = <0x0 0xa1008000 0x0 0x8000>; no-map; };
   vdev0buffer { compatible = "shared-dma-pool"; reg = <0x0 0xa1010000 0x0 0xff0000>; no-map; };
   rvgc_region {
       compatible = "shared-dma-pool";
       reusable;
       linux,cma;
       reg = <0x0 0xa2000000 0x0 0x10000000>;
   };

   rvgc-memory {
       memory-region = <&rvgc_region>;
   };
   ```

   즉 Xen은 "이 대역은 여기까지 예약해 두고 Dom0가 건드리지 마"를 말하고, guest는 "그 예약분 중 이 조각은 vring, 이 조각은 buffer, 이 조각은 RVGC용 CMA"라고 다시 쪼갠다. 여기 둘이 어긋나면 버퍼 alloc은 돼도 기능 경로는 깨질 수 있다.

4. **소프트웨어 스택도 이미 IPMMU identity를 ownership의 일부로 본다**  
   `x5h-cr52-platform/.../r_osal_os_io.c` 에서는 단순 물리 주소가 아니라 AXI/IPMMU 이름 테이블을 등록한다.

   ```c
   osal_ret = r_osal_os_io_register_axi_bus_info(
       "mm(ipa)", "ipmmu_mm_00", OSAL_INVALID_UTLB);
   ```

   이건 중요한 힌트다. 상위 OSAL도 이미 "메모리가 있다"와 "그 메모리에 DMA로 접근 가능하다"를 같은 뜻으로 보지 않는다. bring-up 로그에서 fd, IPA, carveout 주소만 확인하고 green 처리하면 안 되는 이유다.

## 리스크

- `xen,passthrough` 누락/중복으로 **장치 ownership이 비거나 충돌**하면 DomD와 guest가 서로 장치를 못 잡거나 동시에 잡으려 한다.
- carveout 주소는 맞아도 guest 측 `shared-dma-pool` 분할이나 `memory-region` 연결이 틀리면 **buffer alloc 성공 + 기능 실패** 조합이 나온다.
- SMMU/IPMMU binding mismatch가 있으면 dmabuf fd나 shared memory 핸들은 살아 있어도 **실제 DMA read/write는 fault 또는 garbage** 가 된다.
- DomD 전용 HW access와 guest buffer sharing을 혼동하면 **black screen / camera freeze 원인**을 display policy 쪽으로 오판하기 쉽다.

## 다음 액션

다음 글에서는 이 ownership ledger 위에서, **DomD/CR52/RVGC/RPMsg가 실제 display·camera 경로에서 어떻게 hand-off 되는지**를 failure pattern 중심으로 더 잘라 보겠다.

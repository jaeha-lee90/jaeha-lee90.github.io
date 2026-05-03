---
title: R사 X5H Day63 - DomD/CR52/RVGC/RPMsg Display Hand-off 실패 패턴
author: JaeHa
date: 2026-05-04 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day63, domd, cr52, rvgc, rpmsg, display, bringup]
---

Day62에서 Xen DTB 분할과 memory ownership ledger를 정리했다면, 그 다음 bring-up 핵심은 **그 ownership이 실제 hand-off로 이어지는지**다. X5H display 경로는 단순히 Android가 framebuffer를 그리는 구조가 아니라, **DomD/guest의 carveout + CR52 remoteproc + RPMsg vring + RVGC display map** 이 순서대로 붙어야 화면이 뜬다. 그래서 black screen, stale frame, RPMsg timeout은 서로 다른 버그처럼 보여도 실제로는 같은 hand-off 체인의 다른 절단면인 경우가 많다.

## 핵심 요약

- `r8a78000-ironhide-domd.dts` 와 `doma-virtio.dts` 는 각각 **DomD용 / guest용 RPMsg vring·buffer·CR52 RAM 세트**를 따로 잡고, `cr52_rproc1/2` 에 연결한다.
- `r8a78000-ironhide-xt.dts` 는 MFIS ch2 와 `cr52_rproc2` 를 별도 분리해 **DomU/Android display path를 독립 hand-off 체인**으로 만든다.
- `rvgc-memory` 와 `display-map/layer-map` 이 맞아도, RPMsg vring 또는 CR52 RAM binding 이 틀리면 **buffer alloc 성공 + frame 미표시** 패턴이 나온다.
- bring-up triage는 `remoteproc attach → rpmsg vring → rvgc carveout → layer map` 순서로 잘라 보는 게 가장 빠르다.

## 코드 포인트

1. **DomD와 guest는 같은 구조를 쓰지만, 같은 버퍼를 직접 공유하지 않는다**  
   `r8a78000-ironhide-domd.dts` 에서는 DomD용 hand-off 메모리가 `0xa0000000` 대역에 잡혀 있다.

   ```dts
   vdev0vring0 { reg = <0x0 0xa0000000 0x0 0x8000>; };
   vdev0vring1 { reg = <0x0 0xa0008000 0x0 0x8000>; };
   vdev0buffer { reg = <0x0 0xa0010000 0x0 0xff0000>; };
   cr52_ram1@96650000 { reg = <0x0 0x96650000 0x0 0x50000>; };
   ...
   &cr52_rproc1 {
       memory-region = <&vdev0buffer>, <&vdev0vring0>, <&vdev0vring1>, <&cr52_ram1>;
   };
   ```

   반면 `doma-virtio.dts` 에서는 guest용으로 같은 형태를 `0xa1000000` / `0x966a0000` 쪽에 다시 잡는다.

   ```dts
   vdev0vring0 { reg = <0x0 0xa1000000 0x0 0x8000>; };
   vdev0vring1 { reg = <0x0 0xa1008000 0x0 0x8000>; };
   vdev0buffer { reg = <0x0 0xa1010000 0x0 0xff0000>; };
   cr52_ram2@966a0000 { reg = <0x0 0x966a0000 0x0 0x50000>; };
   ...
   cr52_rproc2 {
       memory-region = <&vdev0buffer>, <&vdev0vring0>, <&vdev0vring1>, <&cr52_ram2>;
   };
   ```

   즉 증상이 비슷해 보여도 **DomD RPMsg chain 문제인지, Android guest chain 문제인지** 먼저 분리해야 한다.

2. **`r8a78000-ironhide-xt.dts` 는 display용 CR52 경로를 일부러 분리한다**  
   base XT DTS 에서는 MFIS를 쪼개서 channel 2를 별도 노드로 떼고, `cr52_rproc2` 를 활성화한다.

   ```dts
   &mfis {
       reg-names = "ch0", "ch1", "ch3", "ch4", "ch5", "ch6", "ch7", "unlock_reg";
       renesas,mfis-channels = <0 1>;
   };

   mfis_ch2: mfis_ch2@18802000 {
       reg-names = "ch2";
       renesas,mfis-channels = <2>;
       status = "okay";
   };

   &cr52_rproc2 {
       status = "okay";
   };
   ```

   이건 display 쪽 hand-off 를 일반 CR52 path와 분리하려는 의도다. 그래서 bring-up 때 `mfis` 전체가 보인다고 안심하면 안 되고, **ch2 interrupt/remoteproc attach가 실제로 guest display path까지 이어졌는지** 봐야 한다.

3. **RVGC는 최종 표시 정책이지만, 그 앞단 hand-off가 살아야 의미가 있다**  
   `doma-virtio.dts` 의 RVGC 블록은 layer policy를 꽤 명시적으로 가진다.

   ```dts
   rvgc-memory {
       memory-region = <&rvgc_region>;
   };

   display-0 {
       display-map = <0x00>;
       layers {
           primary   { layer-map = <0x00>; };
           overlay-0 { layer-map = <0x01>; };
           overlay-1 { layer-map = <0x02>; };
           overlay-2 { layer-map = <0x03>; };
           overlay-3 { layer-map = <0x04>; };
       };
   };
   ```

   하지만 이건 어디까지나 **마지막 소비자 쪽 배치 규칙**이다. RPMsg vring 이 안 돈다면 RVGC는 그릴 frame 자체를 못 받는다. 그래서 `display-map` 만 보고 UI policy 문제로 몰아가면 원인 절반을 놓친다.

4. **Xen passthrough는 "장치 접근권"만 주고, hand-off 성공까지 보장하진 않는다**  
   Day62에서 본 `r8a78000-ironhide-xen.dts` 는 아래 자원들을 guest display path에 넘긴다.

   ```dts
   &cr52_rproc2 { xen,passthrough; };
   &mfis_ch2    { xen,passthrough; };
   &rpmsg_ipi0  { xen,passthrough; };
   &rpmsg_shm0  { xen,passthrough; };
   &rvgc        { xen,passthrough; };
   ```

   여기서 passthrough가 있다는 건 **접근 경로가 열렸다**는 뜻이지, vring 주소·interrupt·firmware ABI가 맞다는 뜻은 아니다. 실무에서는 이 단계를 green으로 착각해서, 실제 RPMsg timeout을 RVGC/GSX black screen으로 오판하는 경우가 많다.

5. **상위 SW도 이미 "메모리 존재"와 "DMA 가능"을 분리해서 본다**  
   `r_osal_os_io.c` 는 `mm(ipa)` 를 `ipmmu_mm_00` 에 등록한다.

   ```c
   r_osal_os_io_register_axi_bus_info("mm(ipa)", "ipmmu_mm_00", OSAL_INVALID_UTLB);
   ```

   이건 display/camera buffer hand-off에서도 그대로 적용된다. carveout 주소가 맞고 fd가 살아 있어도, 실제 DMA visibility 경로가 안 맞으면 frame consumer는 쓰레기 데이터를 받거나 멈춘다.

## 리스크

- `cr52_rproc1/2` 의 `memory-region` 순서나 주소 대역이 어긋나면 **remoteproc boot는 된 것처럼 보여도 RPMsg channel init 이 깨질 수 있다**.
- `mfis_ch2` interrupt 또는 passthrough 누락 시 guest display path만 죽어서 **DomD는 정상인데 Android만 black screen** 이 날 수 있다.
- `rvgc_region` 은 살아 있고 layer map 도 맞는데 vring/buffer hand-off 가 깨지면 **alloc success + stale frame/frozen overlay** 조합이 나온다.
- RPMsg/CR52 ABI mismatch를 display policy 버그로 오판하면 디버깅이 GSX/HWC 쪽으로 새서 복구 시간이 길어진다.

## 다음 액션

다음 글에서는 Day63의 hand-off 체인 위에서, **GSX/RVGC/display plane commit 경로와 black-screen signature 분리법**을 더 좁혀 보겠다.

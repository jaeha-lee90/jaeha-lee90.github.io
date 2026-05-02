---
title: R사 X5H Day60 - 전체 HW/SW 아키텍처 정리 (Boot, Xen 멀티도메인, CR52 rcar-xos, Android Trout-XenVM)
author: JaeHa
date: 2026-05-02 02:00:00 +0900
categories: [R사, X5H, Architecture, BSP, Bring-up]
tags: [R사, X5H, Day60, architecture, boot, xen, cr52, rcar-xos, android, trout, disfwk, camfwk, powervr, rpmsg]
---

Day50~Day59에서 boot success → state retention → 서비스 deadline → restart budget → criticality class → cross-layer recovery ledger → camera/display 아키텍처/bring-up 까지 한 사이클로 묶었다. 이번 Day60에서는 그 시리즈가 깔린 **밑그림 자체**, 즉 **R-Car X5H 전체 HW/SW 아키텍처**를 한 번에 정리한다.

이번 글의 목적은 단순한 "보드 소개"가 아니다. 새로운 X5H 레포 11개를 받았을 때 **각 레포가 어느 레이어/도메인을 담당하는지, 누가 누구한테 데이터를 넘기는지**를 한 번에 잡을 수 있는 reference map을 만드는 것이다. NVIDIA Automotive / DRIVE OS 쪽 BSP 통합 관점에서 다시 들여다봐도 똑같이 통하는 구조라, 이후 글들의 공용 베이스로 둔다.

## 핵심 요약

- X5H = R-Car gen5 `r8a78000`, 보드 코드네임 `ironhide`. **A-core(Cortex-A72) + R-core(Cortex-R52 ×3) + M-core(Cortex-M33 SCP) + NPU0/NPU1 + PowerVR Rogue GPU** 의 비대칭 멀티코어 SoC다.
- 부팅은 **HyperFlash → 1st IPL → 2nd IPL(R52 c2/core0) → SCP / multifwk(R52 c0/c1) / NPU FW** 가 먼저 깨어나고, 그 다음 **BL31(TF-A) → OP-TEE → U-Boot → Xen** 으로 A-core가 올라온다.
- 런타임은 **Xen 4.21 + 4도메인** 구조: `Dom0(컨트롤)` / `DomD(드라이버 도메인 Linux)` / `DomU(Linux ADAS 게스트)` / `DomA(AOSP main, Trout-XenVM)`.
- CR52 쪽은 **`rcar-xos` SDK v4.31 (FreeRTOS / baremetal / Linux 백엔드 OSAL)** 위에서 **camfwk(카메라) / disfwk(디스플레이) / MCAL / SCP / ADAS ref-app** 펌웨어가 동작한다.
- Android 디스플레이는 **HWC3-DRM → `disfwk_fe.ko`(DRM_RCAR_RVGC) → RPMsg(Taurus RVGC 프로토콜) → CR52 disfwk → VSPD/DPTX → Panel** 경로다.
- 카메라는 **Sensor → GMSL SerDes → CSI-2/VIN → CR52 camfwk → RPMsg → DomD V4L2 → virtio + IPMMU 보호 공유 버퍼 → Android 카메라 HAL** 경로다.
- 11개 레포는 **(1) HW + 펌웨어 prebuilt, (2) Xen/Yocto 빌드 메타, (3) Android 디바이스/매니페스트, (4) GPU DDK, (5) Display 프론트엔드, (6) CR52 SDK** 6박스로 깔끔하게 떨어진다.

## 0. 먼저 그림으로 보는 X5H 전체 아키텍처

### 0-1. HW SoC 자체가 비대칭 멀티코어

![X5H Day60 diagram 1](/assets/img/r-company/x5h-day60-diagram-01.png)


### 0-2. Boot chain — 누가 먼저 깨어나는가

![X5H Day60 diagram 2](/assets/img/r-company/x5h-day60-diagram-02.png)


### 0-3. Xen 4도메인 분할

![X5H Day60 diagram 3](/assets/img/r-company/x5h-day60-diagram-03.png)


### 0-4. CR52 SW 스택 (rcar-xos SDK v4.31)

![X5H Day60 diagram 4](/assets/img/r-company/x5h-day60-diagram-04.png)


### 0-5. Cross-domain 데이터 패스 (Display / Camera / GPU)

![X5H Day60 diagram 5](/assets/img/r-company/x5h-day60-diagram-05.png)


### 0-6. 11개 레포 → 6개 역할 박스

![X5H Day60 diagram 6](/assets/img/r-company/x5h-day60-diagram-06.png)


## 코드 포인트

### 1) Boot 순서의 진짜 근거: `gen5-prebuilts/ipls/x5h_bootloaders.yaml`

이 yaml에 실제 flash 주소와 srec 적재 순서가 박혀 있어서, 한 번 읽으면 X5H 부팅 시퀀스가 그대로 잡힌다.

```text
HyperFlash @0x00000000   1st_ipl_normal_boot_certificate_a_b
HyperFlash @0x00040000   1st_ipl_rsip_m                     ← 보안 IP
HyperFlash @0x00440000   images_normal_boot_certificate_a
HyperFlash @0x004C0000   App_SCP_X5H_Sample                 ← CM33 SCP
HyperFlash @0x007C0000   2nd_ipl_rt_core_cluster2_core0     ← R52 c2/core0 부팅 시작

UFS        @0x00E00000   3x4K_multifwk_freertos_sample_app_x5h  ← R52 c0/c1 (camfwk + disfwk)
UFS        @0x05000000   bl31-ironhide                         ← TF-A (EL3)
UFS        @0x05200000   tee-ironhide                          ← OP-TEE (S-EL1)
UFS        @0x07200000   u-boot-elf-ironhide                   ← BL33
UFS        @0x07300000   X5H_NPU0_FW
UFS        @0x08000000   X5H_NPU1_FW
```

여기서 절대 놓치면 안 되는 두 가지.

- **R-core(Safety/RT)와 SCP가 A-core보다 먼저 깨어난다.** Day50~Day55에서 다룬 *boot stage watermark*, *service criticality class*, *reboot authority matrix* 는 이 순서가 전제다. A-core 쪽 bootcount/Active slot이 결정될 때 R-core는 이미 한참 전에 camfwk/disfwk가 떠 있다.
- **A-core 부팅은 secure lane을 한 번 통과한다.** `1st_ipl_rsip_m → BL31 → OP-TEE → U-Boot → Xen` 순서로, EL3(Secure)에서 TF-A(BL31)가 secure monitor 역할을 잡고 OP-TEE를 띄운 다음 BL33으로 빠진다. Day56의 `failure_instance_id`를 secure-world까지 끌고 가려면 이 단계의 디자인이 필요하다.

### 2) Display IPC 계약: `disfwk_fe/r_taurus_rvgc_protocol.h`

Day59에서 본 black-screen 절단 순서의 핵심 계약이 이 헤더 한 장에 들어있다. Linux DRM 프론트엔드(`disfwk_fe`)와 R52 disfwk 펌웨어가 RPMsg 위에서 주고받는 IOCTL/이벤트 정의다.

```c
/* disfwk_fe/r_taurus_rvgc_protocol.h */

#define RVGC_PROTOCOL_IOC_LAYER_SET_ADDR    /* plane 주소 변경 */
#define RVGC_PROTOCOL_IOC_LAYER_SET_POS     /* 위치 */
#define RVGC_PROTOCOL_IOC_LAYER_SET_SIZE    /* 크기 */
#define RVGC_PROTOCOL_IOC_DISPLAY_FLUSH     /* commit (blocking 옵션) */
#define RVGC_PROTOCOL_IOC_DISPLAY_INIT      /* W/H/refresh */
#define RVGC_PROTOCOL_IOC_DISPLAY_GET_INFO  /* capability */

#define RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY0 ... DISPLAY9   /* per-display VBLANK */
```

여기서 보이는 핵심:

- **Display 0~9** 까지 VBLANK 이벤트를 fan-out 한다 → 멀티 디스플레이/멀티 게스트가 native 설계.
- IOCTL은 **layer 단위**(plane 단위가 아니라 layer)이고, **address/pos/size를 따로 설정한 뒤 FLUSH**로 atomic하게 적용 → Linux DRM atomic의 plane property와 1:1 매칭된다.
- DRM_RCAR_RVGC가 결국 RVGC = "R-Car Virtual Graphics Card" 의 의미라는 것도 이 헤더에서 드러난다.

### 3) Android 모듈 결선: kernel manifest 두 줄

`android-kernel-manifest/proprietary.xml`:

```xml
<project path="modules/imagination/ddk"
         name="ts-kefico-gen5-demo/x5h-fusion-poc/ddk-km" .../>      <!-- 24.2_6923851 -->
<project path="modules/renesas/disfwk_fe"
         name="ts-kefico-gen5-demo/x5h-fusion-poc/disfwk_fe" .../>   <!-- disfwk_multiguest_4.31 -->
```

이 두 줄이 X5H Android 그래픽 스택의 전부다. **`pvrsrvkm` (PowerVR KM)** 와 **`disfwk_fe.ko` (DRM_RCAR_RVGC)** 만 vendor 모듈로 들어가고, 사용자공간은 `ddk-um-bin`의 `libEGL/GLES/Vulkan_powervr` + `gralloc/mapper.powervr` 가 담당한다. 빌드 타깃은 `xen_virtual_device_aarch64` 로, kleaf/bazel 경로를 탄다.

### 4) 도메인 구성의 진실: `meta-xt-x5h-dev/prod-devel-rcar-gen5.yaml`

moulin yaml의 핵심 변수만 보면 도메인이 어떻게 깔리는지 다 보인다.

```yaml
SOC_FAMILY:    r8a78000
DOMD_MACHINE:  ironhide
DOMU_MACHINE:  ironhide
XT_DOMD_DTB_NAME: r8a78000-ironhide-domd.dtb
XT_XEN_DTB_NAME:  r8a78000-ironhide-xen.dtb
BUILD_TARGET_DOMD: rcar-image-adas
BUILD_TARGET_DOMU: rcar-image-adas
ANDROID_DEVICE: xenvm_trout_arm64
ANDROID_LUNCH_TARGET: aosp_xenvm_trout_arm64-trunk_staging-userdebug
```

특히 **DomD는 단순 Linux가 아니라 driver domain** 이다. PCIe/디스크/네트워크/디스플레이 백엔드가 여기 붙고, 다른 도메인(Android 포함)은 virtio/pv-driver를 통해 DomD를 거쳐 HW에 접근한다. 즉 **DomD가 죽으면 Android UI도 같이 죽는다.** Day54~Day55의 *restart budget / criticality class* 가 도메인 단위로도 그대로 적용돼야 하는 이유.

### 5) Multi-display 정책: `android_device/display_layout_configuration.xml`

```xml
<layouts>
  <layout>
    <display enabled="true" defaultDisplay="true">
      <address>0</address>
    </display>
    <display enabled="true" defaultDisplay="false" displayGroup="passenger_display">
      <address>1</address>
    </display>
  </layout>
</layouts>
```

`display_settings.xml` 에서는 `local:1` 에 `shouldShowSystemDecors=true`, `shouldShowIme=true`, `forcedDensity=160` 같은 보조 디스플레이 정책을 걸어둔다. Day59에서 본 "commit은 성공했는데 다른 display로 라우팅된다" 문제가 바로 이 두 XML과 `CarServiceOverlayXenVm` 의 합작이다.

## 리스크

### Stage 1 — 부팅 순서 오해로 R-core 의존성 깨짐

증상:
- A-core 쪽에서 RPMsg endpoint discover 실패
- camfwk/disfwk 응답이 도착하기 전에 Linux DRM init이 timeout

리스크 해석:
- A-core 입장에서는 "내가 먼저 init했는데 응답이 없다"로 보이지만, 사실은 **R52 c0/c1의 multifwk가 아직 안 떴거나**, 2nd_ipl(R52 c2/core0) 이후 chain이 깨진 경우다.
- 부팅 단계 watermark를 R-core/A-core 양쪽에서 같은 timeline에 찍어둬야 디버깅이 가능해진다.

### Stage 2 — 도메인 ownership 불명확

증상:
- 같은 IP 블록(예: VSPD)을 두 도메인이 동시에 만지려고 시도
- DTB의 `xen,passthrough` 노드 누락 / 중복

리스크 해석:
- moulin yaml + DTB(`r8a78000-ironhide-{xen,domd}.dtb`)에서 **어떤 IP가 어느 도메인에 passthrough 되는지**가 단일 진실 원본이다.
- 여기서 어긋나면 commit은 성공해도 화면이 안 뜨거나, 한쪽 도메인에서만 깜빡거리는 패턴이 나온다.

### Stage 3 — IPMMU 매핑 누락 (DMA 격리 위반)

증상:
- V4L2에서 dmabuf는 받아왔지만 GPU/Display 쪽에서 읽으면 0 또는 noise
- abort/IPMMU fault 로그

리스크 해석:
- 카메라 데이터가 DomD → DomA로 넘어갈 때 **IPMMU mapping이 양쪽 도메인에 모두 있어야** 한다.
- "공유 dmabuf" 는 IPMMU 없이는 그냥 다른 도메인 입장에서 의미 없는 주소다.

### Stage 4 — driver domain crash → Android UI freeze

증상:
- DomD 쪽 V4L2/DRM 백엔드 프로세스가 crash
- Android는 살아있는데 SurfaceFlinger commit이 hang
- HWC3에서 fence timeout

리스크 해석:
- Android는 `disfwk_fe → RPMsg → R52` 경로라서 일견 DomD 와 무관해 보이지만, **공통 dmabuf 와 일부 백엔드는 DomD 가 들고 있다**.
- DomD 단독 재시작이 가능한지/Android UI가 회복되는지가 *restart budget* 의 실제 평가 기준이다.

### Stage 5 — CR52 펌웨어 버전 mismatch

증상:
- `disfwk_multiguest_4.31` 과 disfwk_fe 헤더 버전 불일치
- IOCTL 패킷 사이즈/필드 mismatch로 commit이 무시됨

리스크 해석:
- `disfwk_fe/r_taurus_rvgc_protocol.h` 의 struct가 R52 펌웨어 빌드와 ABI 호환이어야 한다.
- gen5-prebuilts 의 `multifwk_freertos_sample_app_x5h.elf` 와 android-kernel-manifest 의 `disfwk_fe` revision 을 같은 릴리즈 태그(`ren_v2.0.0`)로 묶어두는 게 안전하다.

## 다음 액션

이 베이스 위에서 깊이 파고들 글을 5개 후보로 둔다.

1. **X5H secure boot chain** — `1st_ipl_rsip_m → BL31 → OP-TEE → Xen` 까지 anchor와 attestation 흐름
2. **Xen DTB 분할 + IPMMU 도메인 메모리 맵** — DomD/DomU/DomA의 passthrough 범위
3. **Taurus RVGC 프로토콜 디테일** — IOCTL payload/VBLANK fan-out/multi-guest 재진입
4. **HWC3-DRM 합성 경로** — Android plane → DRM atomic → disfwk_fe layer 매핑까지
5. **DomD restart budget vs Android UI freeze** — driver domain crash → 화면 freeze 까지의 timing 분석

새 X5H 레포가 들어왔을 때는 이 글의 0-6 그림(6개 역할 박스)을 먼저 보고, 어디에 떨어지는지를 결정한 다음 1~5의 깊이 글로 넘어가는 흐름을 권장한다.

## 부록: 11개 레포 cheat sheet

| 레포 | 역할 | 어디에 들어가나 |
|---|---|---|
| `gen5-prebuilts` | IPL · BL31 · OP-TEE · U-Boot · multifwk · NPU FW · GPU prebuilt | 부팅 + 펌웨어 (HyperFlash + UFS slot) |
| `x5h-cr52-platform` | rcar-xos SDK v4.31 (OSAL · drivers · MW · 샘플 — camfwk/disfwk) | R-core (CR52) SW 본체 |
| `x5h-cr52-dev` | CR52 SDK 빌드 orchestration (`setup_and_build.sh`) | CR52 빌드 환경 |
| `meta-xt-gen5-platform` | `meta-xt-{dom0,domd,domu,domx}-gen5` Yocto 레이어 | Xen + Linux 도메인 빌드 |
| `meta-xt-x5h-dev` | moulin 통합 yaml — Xen+kernel+Android+CR52 결합 | 루트 빌드 매니페스트 |
| `android-manifest` | AOSP repo 매니페스트 (Trout XenVM, mesa3d, lisot) | Android 사용자공간 |
| `android-kernel-manifest` | Android 커널 매니페스트 (xen-troops/linux + ddk-km + disfwk_fe) | Android 커널 |
| `android_device` | `device/epam/aosp-xenvm-trout` (HWC3, audio, sepolicy, multi-display XML) | Android 디바이스 정의 |
| `ddk-km` | Imagination PowerVR Rogue KM 드라이버 (`pvrsrvkm`) | Android 커널 모듈 |
| `ddk-um-bin` | PowerVR Android 사용자공간 (libEGL/GLES/Vulkan, gralloc, mapper, vintf) | Android vendor 파티션 |
| `disfwk_fe` | Linux DRM 프론트엔드 (`DRM_RCAR_RVGC`) — RPMsg ↔ CR52 disfwk | Android/DomD 커널 모듈 |

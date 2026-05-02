---
title: R사 X5H Day60 - 전체 HW/SW 아키텍처 정리 (Boot, Xen 멀티도메인, CR52 rcar-xos, Android Trout-XenVM, RPMsg)
author: JaeHa
date: 2026-05-02 02:00:00 +0900
categories: [R사, X5H, Architecture, BSP, Bring-up]
tags: [R사, X5H, Day60, architecture, boot, xen, cr52, rcar-xos, android, trout, disfwk, camfwk, powervr, rpmsg]
---

Day50~Day59에서 boot success → state retention → 서비스 deadline → restart budget → criticality class → cross-layer recovery ledger → camera/display 아키텍처/bring-up 까지 한 사이클로 묶었다. 이번 Day60에서는 그 시리즈가 깔린 **밑그림 자체**, 즉 **R-Car X5H 전체 HW/SW 아키텍처**를 한 장으로 정리한다.

이번 글의 목적은 단순한 “보드 소개”가 아니다. 실무에서 새로운 X5H 레포(11개)를 받았을 때 **각 레포가 어느 레이어/도메인을 담당하는지, 누가 누구한테 데이터를 넘기는지**를 한 번에 잡을 수 있는 reference map을 만드는 것이 목표다. NVIDIA Automotive / DRIVE OS 쪽 BSP 통합 관점에서 다시 들여다봐도 똑같이 통하는 구조라, 이후 글을 위한 공용 베이스로 둔다.

## 핵심 요약

- X5H = R-Car gen5 `r8a78000`, 보드 코드네임 `ironhide`. **A-core(Cortex-A72) + R-core(Cortex-R52 ×3 클러스터) + M-core(Cortex-M33 SCP) + NPU0/NPU1 + PowerVR Rogue GPU** 의 비대칭 멀티코어 SoC다.
- 부팅은 **HyperFlash → 1st IPL → 2nd IPL(R52 c2/core0) → SCP / multifwk(R52 c0/c1) / NPU FW** 가 먼저 깨어나고, 그 다음 **BL31(TF-A) → OP-TEE → U-Boot → Xen** 으로 A-core가 올라온다.
- 런타임은 **Xen 4.21 + 4도메인** 구조: `Dom0(컨트롤)` / `DomD(드라이버 도메인 Linux)` / `DomU(Linux ADAS 게스트)` / `DomA(AOSP main, Trout-XenVM)`.
- CR52 쪽은 **`rcar-xos` SDK v4.31 (FreeRTOS/baremetal/Linux 백엔드 OSAL)** 위에서 **camfwk(카메라) / disfwk(디스플레이) / MCAL / SCP / ADAS ref-app** 펌웨어가 동작한다.
- Android 쪽 디스플레이는 **HWC3-DRM → `disfwk_fe.ko`(DRM_RCAR_RVGC) → RPMsg(Taurus RVGC 프로토콜) → CR52 disfwk → VSPD/DPTX → Panel** 경로로 흐른다.
- 카메라는 **Sensor → GMSL SerDes → CSI-2/VIN → CR52 camfwk → RPMsg → DomD V4L2 → virtio + IPMMU 보호 공유 버퍼 → Android 카메라 HAL** 경로다.
- 11개 레포는 “HW/펌웨어 prebuilt — Yocto/Xen 빌드 메타 — Android 디바이스/매니페스트 — GPU DDK — Display 프론트엔드 — CR52 SDK” 의 6가지 역할로 깔끔하게 분리된다.

## 0. 먼저 그림으로 보는 전체 아키텍처

![X5H Day60 full HW/SW architecture](/assets/img/r-company/x5h-day60-diagram-01.png)

> 같은 그림 SVG: [x5h-day60-diagram-01.svg](/assets/img/r-company/x5h-day60-diagram-01.svg)

색 구분만 보고 가도 큰 그림이 잡힌다.

- 파란 영역 — **HW (X5H SoC + ironhide 보드)**
- 노란 영역 — **부트 체인 / 펌웨어 (gen5-prebuilts)**
- 초록 영역 — **R-core / CR52 SW 스택 (x5h-cr52-platform / -dev)**
- 보라 영역 — **Xen 하이퍼바이저 + Linux 도메인들 (meta-xt-\*)**
- 분홍 영역 — **Android DomA (AOSP Trout-XenVM)**
- 화살표 색은 데이터 패스 — **초록=카메라, 분홍=디스플레이, 빨강=GPU, 청록=메일박스/IPMMU**

## 1. HW 레이어 — X5H SoC 자체가 “비대칭 멀티코어”

X5H에서 처음 헷갈리는 게 “이게 단순히 A72 SoC가 아니다”라는 점이다. 실제 보드 위에서 동시에 깨어 있는 코어/엔진은 다음과 같다.

| 영역 | 구성 |
|---|---|
| Compute | **A-core Cortex-A72 클러스터**, **R-core Cortex-R52 ×3 클러스터** (c0/c1/c2), **M-core Cortex-M33 (SCP)**, **NPU0 / NPU1**, **PowerVR Rogue GPU (Imagination)** |
| Video / Display | CSI-2 RX, VIN, ISP/CISP, IMR (resize/LDC), VSPD (composer), DPTX (DisplayPort TX), VCON, UVCS |
| IPC / 시스템 | **MFIS (HW 메일박스)**, **IPMMU (DMA 격리)**, RSIP-M (보안 IP) |
| 스토리지/IO | UFS, HyperFlash, Ethernet switch, CAN-FD, USB Type-C |
| Off-chip | GMSL2 SerDes + 카메라 센서 (IMX390 / IMX623 / IMX728 계열), Display 0(메인 cluster) + Display 1(Passenger / IVI) |

이 비대칭 구조 때문에 BSP 작업의 90%가 **“어느 코어가 어느 IP 블록의 master 인지”** 를 정의하는 일이 된다. 예를 들어 disfwk(VSPD/DPTX)는 **R52 master**, GPU(PowerVR)는 **A-core(Android) master**, NPU FW는 **A-core가 띄우지만 NPU 엔진 위에서 도는** 식이다. Day57~58에서 본 multi-domain 분할이 바로 이 hardware boundary를 그대로 따라간다.

## 2. Boot chain — 누가 먼저 깨어나는가

`gen5-prebuilts/ipls/x5h_bootloaders.yaml` 에 실제 flash address와 srec 순서가 박혀 있어서, 이걸 읽으면 X5H boot sequence가 한 번에 잡힌다.

```text
BootROM
 ├─ HyperFlash : 1st_ipl_normal_boot_certificate_a_b @0x00000000
 │              1st_ipl_rsip_m                        @0x00040000
 │              images_normal_boot_certificate_a      @0x00440000
 │              App_SCP_X5H_Sample                    @0x004C0000   → CM33 SCP
 │              2nd_ipl_rt_core_cluster2_core0        @0x007C0000   → R52 c2/core0 부팅 시작
 │
 └─ UFS       : 3x4K_multifwk_freertos_sample_app_x5h @0x00E00000   → R52 c0/c1 (camfwk + disfwk)
                test_bare_metal_rt_core_cluster0_core1
                test_bare_metal_rt_core_cluster1_core0
                bl31-ironhide                         @0x05000000   → ARM TF-A (EL3)
                tee-ironhide                          @0x05200000   → OP-TEE (S-EL1)
                u-boot-elf-ironhide                   @0x07200000   → BL33
                X5H_NPU0_FW                           @0x07300000
                X5H_NPU1_FW                           @0x08000000
```

여기서 중요한 두 가지.

1. **R-core(Safety/RT)와 SCP가 A-core보다 먼저 깨어난다.** 그래서 Day50~Day55에서 다룬 *boot stage watermark*, *service criticality class*, *reboot authority matrix* 는 이 순서를 전제로만 의미가 있다. A-core 쪽 bootcount/Active slot 결정이 끝났을 때 R-core는 이미 한참 전에 camfwk/disfwk 가 떠 있는 상태다.
2. **A-core 부팅은 보안/시큐어 레인을 통과한다.** `1st_ipl_rsip_m → BL31 → OP-TEE → U-Boot → Xen` 순서로, EL3(Secure)에서 **TF-A (BL31)** 가 secure monitor 역할을 잡고 OP-TEE를 띄운 다음 BL33(U-Boot)으로 빠진다. `failure_instance_id` (Day56) 같은 cross-layer 식별자를 secure-world까지 끌고 가려면 이 단계에서 디자인이 필요하다.

## 3. R-core / CR52 SW — `rcar-xos` SDK v4.31

`x5h-cr52-platform`의 `sw_src/renesas/` 트리를 보면 **AUTOSAR-스러운 정통 R-core 플랫폼**이 그대로 있다.

```text
sw_src/renesas/
  os/
    osal/                  ← OS 추상화 인터페이스
    osal_wrapper/          ← FreeRTOS / baremetal / Linux 백엔드
    osal_configuration/    ← per-target 설정 (freertos / linux / simulation)
  driver/soc/platform/     ← csi2, vin, vspd, dptx, vcon, imr, uvcsdrv
  middleware/libraries/    ← canrout, canethconv, ethswtcont,
                             rivp_codec, memory_allocator
  applications/platform/   ← ipl, ldr_daemon
  tools/                   ← mcal, scp, dbg_ssl_ramdump
```

빌드는 `x5h-cr52-dev/setup_and_build.sh` 가 SDK 압축을 풀고 위 platform 트리를 cmake로 묶는 구조다. 빌드 결과는 `samples/` 의 펌웨어 srec/elf로 나오고, 실 보드에서는 `gen5-prebuilts/ipls/` 의 srec 슬롯을 그대로 교체해서 올린다.

핵심 펌웨어 두 개:

- **camfwk** (`samples/camfwk_freertos_sample_app`) — 카메라 라우팅(Deser / Sensor / VIN map), `RPMsg` 위에서 frame-ready / FEED_BUFFER 이벤트 발생
- **disfwk** (`samples/disfwk_freertos_sample_app`) — VSPD layer compose + DPTX scanout, `RPMsg + Taurus RVGC` 프로토콜로 Linux/Android DRM과 통신

Day53~Day55에서 이야기한 *service deadline / restart budget / reboot authority* 는, 실제로 이 계층의 **OSAL task scheduling + watchdog + criticality_class**에서 구현된다. `OS / OSAL_wrapper` 의 backend가 FreeRTOS인지 baremetal인지에 따라 watchdog 설정 패턴이 달라진다는 점만 기억해두면 좋다.

## 4. 하이퍼바이저 + Linux 도메인 — Xen 4.21 멀티게스트

`meta-xt-x5h-dev/prod-devel-rcar-gen5.yaml` (moulin) 과 `meta-xt-gen5-platform/layers/meta-xt-{dom0,domd,domu,domx}-gen5/` 가 도메인 구성을 정의한다.

| Domain | 역할 | 커널 / 이미지 | 주요 layer |
|---|---|---|---|
| Dom0 | 컨트롤 도메인. xl/xenstore/toolstack | Linux 6.1.102 (`renesas-rcar/linux-bsp` v6.1.102/rcar-6.0.0.rc8) | `meta-xt-dom0-gen5` |
| DomD | **드라이버 도메인** (실제 HW 백엔드) | Linux 6.1.102 (`xen-troops/linux` xt 포크) + Yocto `rcar-image-adas` | `meta-xt-domd-gen5`, `meta-xt-domx-gen5` |
| DomU | Linux ADAS 게스트 | Linux 6.1.102 (xt) + `rcar-image-adas` | `meta-xt-domu-gen5` |
| DomA | **AOSP main, Trout-XenVM** | `aosp_xenvm_trout_arm64-trunk_staging-userdebug` + `common-android14-6.1` (xen-troops 포크) | `android-manifest` + `android-kernel-manifest` + `android_device` |

핵심 설정 값 몇 개:

- `SOC_FAMILY: r8a78000`, `DOMD_MACHINE: ironhide`, `DOMU_MACHINE: ironhide`
- `XT_DOMD_DTB_NAME: r8a78000-ironhide-domd.dtb`
- `XT_XEN_DTB_NAME: r8a78000-ironhide-xen.dtb`
- `BUILD_TARGET_DOMD/DOMU: rcar-image-adas`
- 소스 베이스: poky `scarthgap`, meta-virtualization `scarthgap`, xen-troops/meta-xt-common, renesas-rcar/meta-renesas

특히 **DomD는 단순한 Linux가 아니라 driver domain** 이다. 실제 PCIe/디스크/네트워크/디스플레이 백엔드가 여기 붙고, 다른 도메인(Android 포함)은 virtio / pv-driver를 통해 DomD를 거쳐 HW에 접근한다. DomD가 죽으면 Android의 화면도 사실상 같이 죽는 구조라서 Day54~Day55에서 본 *restart budget / criticality class* 가 도메인 단위로도 그대로 적용돼야 한다.

## 5. Android (DomA) — Trout-XenVM 위의 AOSP

`android-manifest/aosp-device-xenvm-trout.xml` 의 핵심:

```xml
<project path="device/epam/aosp-xenvm-trout"
         name="ts-kefico-gen5-demo/x5h-fusion-poc/android_device" .../>
<project path="external/mesa3d_upstream" name="mesa/mesa" .../>
<project path="vendor/epam/lisot" name="xen-troops/lisot" .../>
```

`android-kernel-manifest/proprietary.xml` 쪽:

```xml
<project path="modules/imagination/ddk"
         name="ts-kefico-gen5-demo/x5h-fusion-poc/ddk-km" .../>   <!-- 24.2_6923851 -->
<project path="modules/renesas/disfwk_fe"
         name="ts-kefico-gen5-demo/x5h-fusion-poc/disfwk_fe" .../>  <!-- disfwk_multiguest_4.31 -->
```

즉 Android 커널은 `common-android14-6.1` (xen-troops 포크) 베이스에 **`disfwk_fe.ko` (DRM_RCAR_RVGC)** 와 **PowerVR `pvrsrvkm`** 두 vendor 모듈이 핵심이다. 빌드는 **kleaf / bazel** (`xen_virtual_device_aarch64`).

Android 사용자공간은 `ddk-um-bin/x5h/vendor/`:

- `lib64/egl/libEGL_powervr.so`, `libGLESv1_CM_powervr.so`, `libGLESv2_powervr.so`
- `lib64/hw/gralloc.xenvm_trout_arm64.so`, `mapper.powervr.so`, `vulkan.powervr.so`
- `bin/pvrsrvctl`, `pvrhwperf`, `pvrdebug` …

그리고 디스플레이/HAL 쪽 핵심은 `android_device/`:

- `hals/hwc3` — HWC3 DRM HAL (`hwc3-drm-xt.rc`)
- `display_layout_configuration.xml` — 두 개 display (`address=0` 메인, `address=1` `passenger_display` 그룹)
- `display_settings.xml` — 보조 디스플레이 IME/system decor 정책
- `aosp_xenvm_trout_*.mk` — `ro.hardware.egl=powervr`, `ro.hardware.vulkan=powervr`, automotive vehicle HAL의 **virtualization-service** 변형 채택
- `sepolicy/` — `vendor`, `private`, 도메인별 .te (`hal_bluetooth_default`, `kernel`, `system_suspend`, `vold` 등)

여기서 자주 놓치는 포인트: **CarServiceOverlayXenVm + automotive vehicle HAL virtualization-service** 조합이 들어간다는 건, 차량 신호(VHAL)도 실제 ECU/CAN과 직결되는 게 아니라 Xen 경계를 한 번 거친다는 뜻이다. 현실 차량 통합 시점에서 latency 예산을 잡을 때 이 부분이 의외로 발목을 잡는다.

## 6. 데이터 패스 — Cross-domain flow 3개

### 6-1. Display (Android → Panel)

```text
SurfaceFlinger / CarUI
  → HWC3-DRM HAL (android_device/hals/hwc3, hwc3-drm-xt)
  → DRM atomic commit
  → disfwk_fe.ko  (DRM_RCAR_RVGC, Linux DRM 프론트엔드)
  → RPMsg + Taurus RVGC protocol
       (RVGC_PROTOCOL_IOC_LAYER_SET_ADDR / POS / SIZE,
        DISPLAY_FLUSH / INIT / GET_INFO,
        VBLANK_DISPLAY0~9 events)
  → MFIS HW 메일박스
  → CR52 disfwk firmware
  → VSPD compose → DPTX → Panel0 (메인) / Panel1 (passenger)
```

이 경로의 “계약”이 박혀 있는 헤더가 `disfwk_fe/r_taurus_rvgc_protocol.h` 다. IOCTL ID와 페이로드 struct들이 R52 disfwk와 1:1로 매칭된다. Day59에서 본 **black screen 절단 순서**는 이 stack 어느 층에서 commit이 도달하지 못했는지를 찾는 작업과 같다.

### 6-2. Camera (Sensor → Android)

```text
Sensor (IMX390 / IMX623 / IMX728 …)
  → GMSL SerDes → CSI-2 RX → VIN
  → CR52 camfwk firmware  (Deser / Sensor / VIN routing,
                            optional ISP / IMR preprocess)
  → RPMsg (frame-ready / FEED_BUFFER)
  → DomD V4L2 (/dev/video*)
  → virtio + IPMMU 보호 공유 버퍼
  → Android 카메라 HAL
```

Day57~58에서 그린 그림과 동일한 경로지만, 한 가지 강조해 둘 것: **IPMMU가 도메인 간 DMA 격리 경계를 만든다**. 즉 “DomD가 만든 V4L2 dmabuf를 Android가 공유”한다는 이야기는, IPMMU 매핑/공유 정책이 미리 셋업돼 있다는 전제 위에서만 성립한다. 카메라가 안 보일 때 IPMMU mapping을 같이 의심해야 하는 이유다.

### 6-3. GPU (Android EGL/Vulkan → 합성)

```text
Android EGL / GLES / Vulkan_powervr  (ddk-um-bin/x5h/vendor/lib64/...)
  → pvrsrvkm.ko  (ddk-km, DDK 24.2_6923851 베이스)
  → PowerVR Rogue GPU
  → 렌더링 결과 → gralloc/mapper.powervr 버퍼
  → DRM/HWC3 합성 → 6-1 Display 경로로 합류
```

GPU는 **Android 도메인이 직접 master** 다. DomD를 안 거친다. 그래서 GPU가 죽어도 다른 도메인의 디스플레이/카메라는 살아있는 것처럼 보일 수 있고, 반대로 DomD가 죽었을 때 GPU rendering은 잘 되는데 화면이 안 뜨는 이상 현상이 생길 수 있다. *failure_instance_id* 기반 cross-layer ledger(Day56)가 필요한 가장 분명한 케이스다.

## 7. 11개 레포 → 역할 매핑 (실무용 cheat sheet)

| 레포 | 역할 | 어디에 들어가나 |
|---|---|---|
| `gen5-prebuilts` | IPL / BL31 / OP-TEE / U-Boot / multifwk FW / NPU FW / GPU prebuilt | **부팅 + 펌웨어** (HyperFlash + UFS slot) |
| `x5h-cr52-platform` | rcar-xos SDK v4.31 (OSAL, drivers, MW, samples — camfwk/disfwk) | **R-core (CR52) SW 본체** |
| `x5h-cr52-dev` | CR52 SDK 빌드 orchestration (`setup_and_build.sh`) | CR52 빌드 환경 |
| `meta-xt-gen5-platform` | `meta-xt-{dom0, domd, domu, domx}-gen5` Yocto 레이어 | **Xen + Linux 도메인 빌드** |
| `meta-xt-x5h-dev` | moulin 통합 yaml (`prod-devel-rcar-gen5.yaml`) — Xen+kernel+Android+CR52 결합 | **루트 빌드 매니페스트** |
| `android-manifest` | AOSP repo 매니페스트 (Trout XenVM, mesa3d, lisot) | Android 사용자공간 |
| `android-kernel-manifest` | Android 커널 매니페스트 (xen-troops/linux + ddk-km + disfwk_fe) | Android 커널 |
| `android_device` | `device/epam/aosp-xenvm-trout` (HWC3, audio, sepolicy, multi-display XML) | Android 디바이스 정의 |
| `ddk-km` | Imagination PowerVR Rogue KM 드라이버 (`pvrsrvkm`) | Android 커널 모듈 |
| `ddk-um-bin` | PowerVR Android 사용자공간 (libEGL/GLES/Vulkan, gralloc, mapper, vintf) | Android vendor 파티션 |
| `disfwk_fe` | Linux DRM 프론트엔드 (`DRM_RCAR_RVGC`) — RPMsg ↔ CR52 disfwk | Android/DomD 커널 모듈 |

11개 레포가 늘어 보이지만, 실제로는 **(1) HW + 펌웨어 prebuilt, (2) Xen/Yocto 빌드 메타, (3) Android 디바이스 정의 + 매니페스트, (4) GPU DDK, (5) Display 프론트엔드, (6) CR52 SDK** 의 6가지 박스로만 잘 분리된다. 새 레포가 추가됐을 때도 이 6박스 중 어디에 떨어지는지를 먼저 보면 된다.

## 8. NVIDIA Automotive / DRIVE OS 관점에서 다시 보기

이 구조는 결국 다음과 같이 추상화된다.

| 역할 | X5H | DRIVE OS 대응 |
|---|---|---|
| Safety/RT MCU domain | **R52 + FreeRTOS + camfwk/disfwk** | DRIVE OS Safety / Tegra Safety MCU + RTOS |
| Hypervisor + driver domain | **Xen 4.21 + DomD Linux** | NVIDIA Hypervisor + driver VM (Linux) |
| Linux ADAS guest | **DomU + rcar-image-adas** | DRIVE OS Linux (CUDA/TensorRT/DriveWorks) |
| Android IVI guest | **DomA Trout-XenVM (AOSP main)** | DRIVE Concierge / Android Automotive on DRIVE |
| Camera capture path | **CSI/VIN → camfwk → V4L2 → virtio** | DRIVE Camera (NvSciStream/NvSciBuf 기반) |
| Display path | **disfwk → VSPD/DPTX, disfwk_fe DRM** | DRIVE Display / Tegra Display Controller |
| GPU stack | **PowerVR Rogue (Imagination)** | NVIDIA Tegra GPU (CUDA/Vulkan) |
| 보안/secure world | **TF-A BL31 + OP-TEE** | NVIDIA Secure OS / Trusty 변형 |

코어 분리 / 도메인 격리 / safety MCU 선부팅 / driver VM 게이트키퍼 / 카메라-디스플레이의 RPMsg-스러운 IPC — 이 다섯 패턴은 **벤더가 바뀌어도 그대로** 유지된다. X5H에서 익혀둔 layering이 그대로 DRIVE 쪽 디버깅 메탈모델로 옮겨가는 이유.

## 9. 다음 글 후보

이번 Day60은 **베이스 맵**이라서, 여기서 하나씩 깊이 파고드는 글이 자연스럽게 따라온다.

- (a) X5H boot security chain — `1st_ipl_rsip_m / BL31 / OP-TEE / Xen` 까지 secure boot anchor와 attestation 흐름
- (b) Xen DTB 분할과 IPMMU 도메인 메모리 맵 — DomD/DomU/DomA가 어떤 범위를 들고 가는지
- (c) `Taurus RVGC` 프로토콜 디테일 — IOCTL 별 payload, VBLANK fan-out, multi-guest 재진입
- (d) HWC3-DRM 합성 경로 — Android plane → DRM atomic → disfwk_fe 매핑까지의 layer-by-layer
- (e) DomD restart budget vs Android UI freeze — driver domain 크래시 → 화면 freeze까지의 timing 분석

기본 베이스가 깔렸으니, 다음부터는 “어디서 막혔을 때 어느 글로 돌아가서 봐야 하는가” 가 명확해질 것이다.

---
title: R사 X5H Day57 - Camera to Android Display 가상화 아키텍처 분해 (R-core, A-core, Xen, Linux, Android)
author: JaeHa
date: 2026-05-01 13:45:00 +0900
categories: [R사, X5H, Camera, Display, Xen, Android]
tags: [R사, X5H, Day57, camera, display, xen, android, linux, freertos, cr52, bringup]
---

이번 글은 X5H 레포를 실제로 뒤져가며 **카메라 입력부터 Android 화면 출력까지**의 흐름을 정리한 기록이다. 포인트는 단순히 `camera -> Linux -> Android`로 보면 안 된다는 점이다. 이 플랫폼은 **R-core(CR52, FreeRTOS)**, **A-core(Linux)**, **Xen virtualization**, **Android guest**가 서로 역할을 분담하는 구조이고, camera path와 display path 역시 각각 독립적인 계층과 프레임워크를 갖는다.

특히 NVIDIA DRIVE OS류 포지션을 준비하는 관점에서도 이런 식의 구조 이해가 중요하다. 결국 실제 시스템 소프트웨어 역할은 “센서가 어떻게 들어와서 어느 코어/도메인에서 처리되고, 어떤 display path를 통해 최종 UI까지 연결되는지”를 설명할 수 있어야 하기 때문이다.

## 한 줄 요약

- 이 X5H 구조는 **카메라/디스플레이 파이프라인을 CR52 펌웨어, Linux driver domain, Xen guest, Android UI stack이 나눠 맡는 multi-core / multi-domain 구조**다.
- 카메라는 sensor/serdes/CSI/VIN에서 시작해 **camfwk**, **RPMsg/virt bridge**, **V4L2**, 필요 시 **ISP/IMR/AI/VOUT**로 확장된다.
- 디스플레이는 FreeRTOS 쪽 native display framework와 Linux DRM/KMS 쪽 display framework가 둘 다 보이며, Android는 그 위에서 **HWC3 / SurfaceFlinger / multi-display policy**를 담당하는 consumer에 가깝다.

---

## 0. 한눈에 보는 SW/HW 아키텍처 그림

### 0-1. 전체 시스템 그림

![X5H Day57 diagram 1](/assets/img/r-company/x5h-day57-diagram-01.png)


### 0-2. 코어/도메인 ownership 그림

![X5H Day57 diagram 2](/assets/img/r-company/x5h-day57-diagram-02.png)


### 0-3. 데이터 경로 vs 제어 경로 그림

![X5H Day57 diagram 3](/assets/img/r-company/x5h-day57-diagram-03.png)


---

## 1. 레포에서 확인한 핵심 근거

이번 해석의 주 근거는 아래 파일들이다.

### 카메라 쪽

- `x5h-cr52-platform/.../r_camfwk_hal_camera_system.h`
- `x5h-cr52-platform/.../camera_protocol.hpp`
- `x5h-cr52-platform/.../camera_manager.h`
- `x5h-cr52-platform/.../camera_config.h`
- `x5h-cr52-platform/.../camera.c`
- `x5h-cr52-platform/.../frontcamera_customize.cpp`

### 디스플레이 쪽

- `x5h-cr52-platform/.../display_drm.h`
- `x5h-cr52-platform/.../display_drm.c`
- `x5h-cr52-platform/.../display.c`
- `x5h-cr52-platform/.../display_device.hpp`

### Xen / Android / domain 구조

- `meta-xt-gen5-platform/prod-devel-rcar-gen5.yaml`
- `meta-xt-gen5-platform/.../doma-virtio.dts`
- `android_device/aosp_xenvm_trout_arm64.mk`
- `android_device/display_settings.xml`
- `android_device/display_layout_configuration.xml`
- `android_device/init/xenvm_trout.init.rc`

즉 이 글은 개념 추정이 아니라, 실제 레포에 남아 있는 구조물들을 이어 붙여 해석한 내용이다.

---

## 2. 먼저 시스템을 코어/OS/도메인별로 나누기

이 플랫폼을 이해할 때 제일 중요한 건 “카메라가 어디서 처리되고, 디스플레이가 어디서 제어되며, Android가 어디까지 책임지는가”를 분리해서 보는 것이다.

### 2-1. R-core / CR52 / FreeRTOS

이쪽은 하드웨어에 가장 가까운 층이다.

`r_camfwk_hal_camera_system.h`를 보면 camera HAL config 구조체에 아래 같은 정보가 직접 들어 있다.

- `sensor_type`
- `active_link`
- `i2c_addr`
- `vin_channel`
- `isp_unit`
- `csi2_channel`
- `num_lanes`
- `mipi_lane_speed`

이건 곧 CR52 / FreeRTOS 쪽이 단순 control task만 하는 게 아니라,

- 어떤 sensor를 붙일지
- 어떤 deserializer link를 활성화할지
- 어느 VIN 채널로 받는지
- 어느 ISP unit을 쓰는지
- CSI lane과 speed를 어떻게 둘지

같은 **카메라 하드웨어 토폴로지와 초기화 책임**을 가지고 있다는 뜻이다.

또 sample 경로를 보면 camera framework(`camfwk`)와 display framework(`disfwk`) 모두 FreeRTOS 쪽 샘플이 존재한다.

- `samples/camfwk_freertos_sample_app/...`
- `samples/disfwk_freertos_sample_app/...`
- `samples/multifwk_freertos_sample_app/...`

즉 R-core는 이 플랫폼에서 다음 역할을 맡는다고 보는 게 자연스럽다.

1. camera low-level init / route 설정
2. sensor / serializer / deserializer 제어
3. capture path backend 제어
4. inter-core bridge endpoint 제공
5. 경우에 따라 display fw 역할 수행

### 2-2. A-core Linux

반면 A-core Linux 쪽은 실제 user/kernel 인터페이스를 통해 frame을 받고 display에 싣는 **실행 계층**이 분명히 존재한다.

`camera.c`는 전형적인 V4L2 streaming path를 보여준다.

- `/dev/videoX` open
- capability query
- input select
- crop / fps / format set
- mmap init
- `QBUF / DQBUF`
- `stream_on`
- `select()` loop
- frame processing 후 재queue

즉 Linux 쪽은 명확하게 **V4L2 capture consumer**다.

같은 식으로 `display_drm.c`는 Linux 쪽 display path가 **DRM/KMS atomic commit** 기반이라는 걸 보여준다.

- `drmOpen("disfwk_fe")`
- connector / CRTC / plane 탐색
- dumb buffer 생성
- framebuffer 등록
- property set
- `drmModeAtomicCommit`

즉 Linux 쪽은

- camera에 대해서는 `/dev/video*` 기반 V4L2 executor
- display에 대해서는 `disfwk_fe` 기반 DRM/KMS executor

라고 요약할 수 있다.

### 2-3. Xen domain 구조

`prod-devel-rcar-gen5.yaml`를 보면 domain 구성이 명확하다.

- `dom0`
- `domd`
- `domu`
- `doma` (Android guest)

여기서 중요한 건 Android가 bare-metal OS처럼 하드웨어를 직접 다 쥐는 구조가 아니라는 점이다. 중간에 **driver domain(DomD)**가 있고, Xen 아래에서 각 domain이 역할을 나눠 갖는다.

또 `doma-virtio.dts`에는

- `vdev0vring0`
- `vdev0vring1`
- `vdev0buffer`
- `rvgc_region`
- `cr52_rproc2`

같은 shared memory / remoteproc / virtio 계열 흔적이 보인다. 즉 camera/display data/control이 단일 프로세스 내부에서만 움직이는 게 아니라, **inter-core + inter-domain 경로**를 탄다는 이야기다.

### 2-4. Android guest

Android 쪽은 lower-level camera bring-up 주체라기보다, **guest UI stack** 역할이 강하다.

`aosp_xenvm_trout_arm64.mk`에서 확인되는 포인트:

- `android.hardware.composer.hwc3-service.drm.xt`
- `disfwk.elf`
- `ro.surface_flinger.max_frame_buffer_acquired_buffers=3`
- `display_settings.xml`
- `display_layout_configuration.xml`
- `CarServiceOverlayXenVm`
- `com.android.car.carlauncher`
- `fw.visible_bg_users=true`

즉 Android는 여기서

- HWC3
- SurfaceFlinger
- CarService
- multi-display policy
- passenger display UX

를 담당하는 **상위 consumer / composer** 쪽이다.

---

## 3. 카메라 경로를 하드웨어부터 끝까지 따라가기

이제 camera path만 따로 풀어보자.

### 3-1. Hardware ingress: Sensor -> SerDes -> CSI/VIN

`r_camfwk_hal_camera_system.h`는 camera hardware ingress를 설명하는 데 핵심이다.

구조체 레벨에서 이미 다음 요소들이 정의돼 있다.

- deserializer index
- bus id
- number of cameras
- `use_cphy`
- `csi2_channel`
- `num_lanes`
- `mipi_lane_speed`
- per-camera `sensor_type`
- `vin_channel`
- `isp_unit`

즉 이 계층은 “카메라가 있다” 수준이 아니라,

- 어떤 sensor가 어떤 deser에 붙는지
- 어떤 CSI 채널로 들어오는지
- 몇 lane인지
- lane speed가 얼마인지
- VIN/ISP 자원에 어떻게 연결되는지

를 시스템 구성으로 표현한다.

이 구조는 bring-up 시 다음 질문을 바로 떠올리게 만든다.

- sensor register init은 누가 하는가?
- deserializer link lock은 어디서 확인하는가?
- CSI lane mismatch가 나면 어느 레이어에서 터지는가?
- VIN channel mapping이 잘못되면 어떤 `/dev/videoX`가 비정상인가?

즉 NVIDIA류 면접에서 묻는 **sensor/CSI/SerDes path** 설명에 바로 대응 가능한 구조다.

### 3-2. CR52 / FreeRTOS 카메라 framework

카메라가 들어왔다고 바로 Linux `/dev/video`가 되는 게 아니라, 중간에 framework 계층이 있다.

`camera_protocol.hpp`를 보면 camera bridge protocol이 존재한다.

- `CAMERA_PROTOCOL_IOC_CHANNEL_INIT`
- `CAMERA_PROTOCOL_IOC_CHANNEL_START`
- `CAMERA_PROTOCOL_IOC_CHANNEL_STOP`
- `CAMERA_PROTOCOL_IOC_CHANNEL_FEED_BUFFER`
- `CAMERA_PROTOCOL_IOC_CHANNEL_RELEASE`

그리고 event도 있다.

- `TAURUS_CAMERA_EVT_FRAME_READY`
- `TAURUS_CAMERA_EVT_FEED_ME`

이건 곧 camera framework가 상위 계층에 대해 **buffer-driven service interface**를 제공하고 있다는 뜻이다.

해석하면 이런 그림이다.

1. 상위가 channel을 init한다.
2. buffer를 등록한다.
3. channel을 start한다.
4. frame ready event를 받는다.
5. 소비한 buffer를 다시 feed한다.

즉 CR52/FreeRTOS 쪽은 단순 background firmware가 아니라,
**카메라 서비스를 제공하는 low-level producer endpoint** 역할을 한다.

### 3-3. Linux V4L2 capture layer

`camera.c`는 Linux 쪽 capture executor를 보여준다.

핵심 흐름:

1. `open_device()`에서 `/dev/videoX` open
2. capability 확인
3. input select
4. crop / fps / format 설정
5. MMAP 버퍼 초기화
6. `start_capturing()`에서 버퍼 queue 후 `stream_on`
7. `mainloop()`에서 `select()`로 frame 대기
8. `DQBUF`로 frame dequeue
9. frame 처리 후 `QBUF`

즉 Linux 카메라 path는 **classical V4L2 streaming capture**다.

`camera_config.h`에서는

- `DEV_NAME_0 ~ DEV_NAME_7`
- `MAX_CAMERAS`
- pixel format(`SBGGR16`, `SBGGR12`, `ARGB32` 등)
- 기본 해상도 `1920x1080`
- 실행할 camera ID 목록

같은 정보가 나온다.

이건 결국 이 시스템이 multi-camera 또는 at least multi-channel 구조를 염두에 두고 있고, `/dev/video0..7` 수준의 Linux device model을 갖고 있다는 뜻이다.

### 3-4. Camera pipeline variants: Capture / ISP / IMR / AI / VOUT

`frontcamera_customize.cpp`는 이 플랫폼이 단순 capture sample이 아니라 **ADAS front camera pipeline** 성격을 가진다는 걸 보여준다.

거기서 보이는 pipeline 조합은 아래처럼 다양하다.

- `CAPTURE`
- `CAPTURE -> ISP`
- `CAPTURE -> IMR`
- `CAPTURE -> ISP -> IMR`
- `CAPTURE -> IMR -> AI`
- `CAPTURE -> ISP -> IMR -> AI`
- `CAPTURE -> VOUT`
- `CAPTURE -> ISP -> VOUT`
- `CAPTURE -> IMR -> VOUT`
- `CAPTURE -> ISP -> IMR -> VOUT`
- `CAPTURE -> IMR -> AI -> VOUT`
- `CAPTURE -> ISP -> IMR -> AI -> VOUT`

이 말은 이 camera stack이 단순 sensor validation이 아니라,

- capture
- optional ISP
- image manipulation / resize / LDC
- AI
- display output

까지 포함하는 **end-to-end application pipeline**으로 설계됐다는 뜻이다.

또 customize parameter에서 다음 항목들이 보인다.

- `VIN_Device`
- `VIN_Capture_Format`
- `Capture_Width/Height`
- `ISP_Enable`
- `IMR_LDC_Enable`
- `IMR_RZ_PipelineX`
- `OBJ_DET_Enable`
- `SEM_SEG_Enable`
- `VOUT_Enable`
- `DRM_Module`

즉 실제 bring-up/앱 통합에서

- camera input mode
- ISP on/off
- resize pipeline
- AI on/off
- display output

을 구성 가능한 구조다.

---

## 4. 디스플레이 경로를 별도로 분해하기

이제 display path만 보자.

### 4-1. FreeRTOS native display path

`display.c`는 FreeRTOS 쪽 display path가 존재한다는 증거다.

주요 함수:

- `R_WM_DisplayConfig`
- `R_WM_DisplayInit`
- `R_WM_DisplayEnable`
- `R_WM_LayerInit`
- `R_WM_LayerFlush`
- `R_WM_LayerFlushMultiPlane`
- `R_WM_LayerShow`

즉 R-core 쪽에는 native display output API 계층이 있고,
`R_WM_*` 류 API를 사용해

- display device init
- pixel format 설정
- background 설정
- layer attach
- frame buffer flush
- vsync 대기 flush

를 수행한다.

이건 Linux DRM path와 별개로,
**FreeRTOS 기반 display framework path가 따로 존재한다**는 뜻이다.

### 4-2. Linux DRM/KMS display path

`display_drm.c`는 Linux 쪽 display path의 핵심이다.

여기서 보이는 것:

- DRM device 이름: `disfwk_fe`
- atomic mode setting 사용
- connector / CRTC / plane 탐색
- framebuffer 생성 (`drmModeCreateDumbBuffer`, `drmModeAddFB2`)
- property 세팅
- `drmModeAtomicCommit`

즉 이 display path는 아주 명확하게

> **camera or image source -> Linux buffer -> DRM framebuffer -> plane -> CRTC -> connector -> display scanout**

형태다.

또 `display_drm.h`를 보면

- `camera_index`
- `display_index`
- `plane_arg`
- `display_arg`

같은 구조가 있고,
thread info에 camera index와 display index가 같이 들어간다. 이건 결국 **특정 camera output을 특정 display output에 싣는 path**를 의도하고 있다는 의미다.

`disfwk_main()`에서는 실제로:

1. camera output queue에서 frame을 받음
2. 필요시 format conversion 수행
3. DRM framebuffer로 `memcpy`
4. atomic commit
5. FPS / processing time 로그 측정

을 한다.

즉 이 Linux display path는 단순 init만 하는 게 아니라 **camera frame display executor** 역할을 직접 수행한다.

### 4-3. Android UI display path

Android 쪽은 lower-level DRM과는 다른 층이다.

`aosp_xenvm_trout_arm64.mk`에서 다음이 보인다.

- `android.hardware.composer.hwc3-service.drm.xt`
- Powervr graphics stack
- `ro.surface_flinger.max_frame_buffer_acquired_buffers=3`
- `vendor.hwc.backend_override=client`
- `display_settings.xml`
- `display_layout_configuration.xml`
- `CarServiceOverlayXenVm`
- secondary display launcher

즉 Android 쪽은

- graphics composer(HWC3)
- SurfaceFlinger
- UI composition
- multi-display routing
- automotive UX policy

를 맡는다.

여기서 중요한 건 Android가 **하드웨어 camera bring-up owner**가 아니라,
**상위 display consumer / UI stack**이라는 점이다.

---

## 5. Android multi-display 쪽 레포 근거

이건 이번 분석에서 매우 명확하게 보인다.

### `display_layout_configuration.xml`

- display 0 = defaultDisplay
- display 1 = `passenger_display`

즉 시스템은 기본적으로 **두 개의 디스플레이 논리 구조**를 가진다.

### `display_settings.xml`

secondary display에 대해:

- system decors 표시
- IME 표시
- insecure keyguard 허용
- density 강제

즉 passenger display가 단순 mirror가 아니라, **실제 별도 정책이 적용되는 secondary display**라는 뜻이다.

### `aosp_xenvm_trout_arm64.mk`

- `CarServiceOverlayXenVm`
- `com.android.car.carlauncher`
- `fw.visible_bg_users=true`

이건 Android Automotive의 multi-user / multi-display 사용 시나리오까지 염두에 둔 구조다.

즉 이 플랫폼은 단순 display bring-up을 넘어,
**virtualized Android automotive multi-display UX stack**으로 읽는 게 맞다.

---

## 6. Xen virtualization이 camera/display에 끼치는 의미

이제 이걸 virtualization 관점으로 다시 보면 훨씬 흥미롭다.

### `prod-devel-rcar-gen5.yaml`

이 빌드 설정은 아래를 명시한다.

- Dom0 image
- DomD image
- DomU image
- Android guest(DomA)
- Android kernel build
- Xen DTB / domd DTB / Android DTB 연계

즉 이 시스템은 처음부터 guest 분리를 전제로 한다.

### `doma-virtio.dts`

여기서 보이는 중요 포인트:

- `reserved-memory`
- `vdev0vring0`
- `vdev0vring1`
- `vdev0buffer`
- `rvgc_region`
- `cr52_rproc2`
- Android firmware node

이건 결국 다음을 시사한다.

1. **remoteproc(CR52)**가 있다.
2. **shared DMA pool**이 있다.
3. **virtio/vring**을 타는 인터-domain communication이 있다.
4. display/camera가 쓸 shared region이 있다.
5. Android는 guest firmware node로 연결된다.

즉 이 구조는 bare-metal Android IVI가 아니라,
**camera/display 자원을 여러 코어/도메인이 공유하는 virtualized automotive platform**이다.

---

## 7. 전체 스토리: Camera에서 Android 화면까지

이제 이걸 사람 말로 다시 정리하면 다음 순서가 된다.

### 7-1. Camera ingest 스토리

1. front camera sensor가 영상 생성
2. serializer/deserializer를 통해 SoC로 유입
3. CR52/FreeRTOS camera framework가
   - sensor/deser 초기화
   - active link 설정
   - VIN/CSI route 매핑
   - ISP 자원 연결
   을 수행
4. frame/event는 RPMsg/bridge 또는 shared mechanism으로 상위 계층과 연결
5. A-core Linux가 `/dev/videoX`를 통해 V4L2 방식으로 frame capture
6. 필요 시 ISP, IMR, AI, VOUT 단계로 이어짐

### 7-2. Display output 스토리

1. camera frame 또는 processed frame이 display path로 전달
2. FreeRTOS 경로라면 R_WM 계열 display framework로 flush 가능
3. Linux 경로라면 `disfwk_fe` 기반 DRM/KMS atomic commit 수행
4. Android guest는 상위 HWC3 / SurfaceFlinger / CarService에서 UI와 multi-display 정책을 결정
5. 최종적으로 main display / passenger display에 출력

### 7-3. Virtualization 스토리

1. Xen 아래 Dom0 / DomD / DomA / DomU가 분리됨
2. hardware-facing 책임은 DomD와 CR52가 상대적으로 많이 가진다.
3. Android는 guest로서 상위 UI/UX와 graphics stack을 담당한다.
4. inter-core / inter-domain communication은 shared memory, virtio, RPMsg 류 메커니즘을 사용한다.

---

## 8. PNG 구조도

### 8-1. 코어/도메인 분리 전체 구조

![X5H Day57 diagram 4](/assets/img/r-company/x5h-day57-diagram-04.png)


### 8-2. Camera path 전용 구조도

![X5H Day57 diagram 5](/assets/img/r-company/x5h-day57-diagram-05.png)


### 8-3. Display path 전용 구조도

![X5H Day57 diagram 6](/assets/img/r-company/x5h-day57-diagram-06.png)


---

## 9. 왜 이 구조가 중요한가

이런 레포는 단순 샘플 코드 모음이 아니라, 실제로 아래 질문에 답할 수 있게 해준다.

- camera bring-up은 어느 코어에서 시작되는가?
- Linux는 camera owner인가 consumer인가?
- Android는 sensor bring-up 주체인가 UI consumer인가?
- Xen이 끼면 camera/display buffer는 어떻게 이동하는가?
- multi-display는 어디서 정책이 결정되는가?
- CR52 firmware와 Linux V4L2 path는 어떻게 공존하는가?

그리고 이건 그대로 NVIDIA DRIVE OS류 역할에서 중요한 질문이기도 하다. 단순 앱 수준이 아니라,
**multi-core, virtualized, hardware-facing automotive software stack을 구조적으로 설명할 수 있는가**를 보기 때문이다.

---

## 10. 정리

이번 X5H 레포를 통해 확인한 가장 중요한 사실은 아래다.

1. camera path는 sensor/serdes/CSI/VIN부터 CR52 camfwk, Linux V4L2, optional ISP/IMR/AI/VOUT까지 이어지는 다층 구조다.
2. display path는 FreeRTOS native display path와 Linux DRM/KMS path가 모두 존재한다.
3. Android는 guest로서 HWC3 / SurfaceFlinger / multi-display policy를 담당하는 상위 consumer 역할이 강하다.
4. Xen / DomD / shared memory / virtio / remoteproc가 중간에 끼기 때문에, 이 시스템은 단일 OS 파이프라인이 아니라 **multi-domain virtualized pipeline**으로 봐야 한다.

내가 앞으로 이 주제를 더 파면, 단순히 “카메라가 나온다 / 화면이 뜬다”가 아니라,
**어느 코어가 어떤 자원을 소유하고, 어느 계층에서 병목과 책임이 생기는지**까지 설명할 수 있게 된다.

## 다음 액션

다음 글에서는 이 구조를 기반으로 아래 둘 중 하나를 이어갈 수 있다.

1. **X5H camera bring-up debug 체크리스트**
   - sensor / serdes / CSI / VIN / V4L2 / buffer / frame drop 관점
2. **X5H display bring-up debug 체크리스트**
   - disfwk_fe / DRM atomic / HWC3 / SurfaceFlinger / multi-display 관점

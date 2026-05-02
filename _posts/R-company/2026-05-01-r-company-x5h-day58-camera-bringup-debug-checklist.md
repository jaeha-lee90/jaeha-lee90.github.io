---
title: R사 X5H Day58 - Camera Bring-up Debug Checklist (Sensor, SerDes, CSI, VIN, V4L2, Virtualization)
author: JaeHa
date: 2026-05-01 16:28:00 +0900
categories: [R사, X5H, Camera, Bring-up, Debug]
tags: [R사, X5H, Day58, camera, bringup, debug, v4l2, serdes, csi, vin, xen]
---

Day57에서는 X5H의 camera → virtualization → Android display 구조를 **R-core / A-core / Xen / Android**로 나눠서 봤다. 이번 Day58은 그 구조를 실제 bring-up/debug 관점으로 바꿔 본다. 목표는 단순히 “카메라가 안 나온다”가 아니라, **어느 계층에서 막혔는지 빠르게 좁히는 checklist**를 만드는 것이다.

이 글은 실제 레포에 보이는 구조를 기준으로 정리했다.

- CR52 / FreeRTOS camera framework
- sensor / deserializer / CSI / VIN routing config
- Linux `/dev/videoX` V4L2 path
- RPMsg / shared buffer / virtualization bridge
- ISP / IMR / optional downstream pipeline

즉 이 checklist는 sensor bring-up부터 Linux streaming, virtualization boundary, frame-ready event까지를 한 번에 보는 데 목적이 있다.

## 한 줄 요약

카메라 bring-up 문제는 대부분 아래 6단계 중 하나에서 끊긴다.

1. **sensor / serdes 전원·링크 초기화 실패**
2. **CSI / VIN route 매핑 실패**
3. **CR52 camfwk config mismatch**
4. **Linux V4L2 노드 생성/format/streaming 실패**
5. **buffer/event/RPMsg 연계 실패**
6. **후단 ISP/IMR/AI/VOUT 기대와 실제 입력 mismatch**

---

## 0. 먼저 그림으로 보는 Camera Bring-up Debug Map

### 0-1. 카메라 SW/HW 전체 경로

![X5H Day58 diagram 1](/assets/img/r-company/x5h-day58-diagram-01.png)


### 0-2. 어느 계층에서 문제를 좁히는지

![X5H Day58 diagram 2](/assets/img/r-company/x5h-day58-diagram-02.png)


### 0-3. 증상별 최초 의심 영역

![X5H Day58 diagram 3](/assets/img/r-company/x5h-day58-diagram-03.png)


---

## 1. 먼저 구조를 다시 짧게 잡고 시작

이 레포를 기준으로 camera path를 아주 단순화하면 아래 순서다.

![X5H Day58 diagram 4](/assets/img/r-company/x5h-day58-diagram-04.png)


즉 bring-up할 때는 “한 번에 전체를 본다”가 아니라,
**이 체인을 한 칸씩 분해해서 어느 경계에서 끊겼는지 보는 게 핵심**이다.

또 실제 디버그는 아래처럼 **HW → FW → Linux → Bridge → Post-process** 순서로 끊어 보는 게 가장 빠르다.

![X5H Day58 diagram 5](/assets/img/r-company/x5h-day58-diagram-05.png)


### 1-1. 증상에서 root cause로 들어가는 절단 트리

![X5H Day58 diagram 6](/assets/img/r-company/x5h-day58-diagram-06.png)


---

## 2. 레포 기준으로 특히 중요한 구조물

### 2-1. CR52 / FreeRTOS 쪽 config 근거

`r_camfwk_hal_camera_system.h`에는 아래 정보가 들어 있다.

- `sensor_type`
- `active_link`
- `i2c_addr`
- `vin_channel`
- `isp_unit`
- `csi2_channel`
- `num_lanes`
- `mipi_lane_speed`

이건 결국 camera bring-up에서 제일 중요한 표다. 즉 문제를 볼 때 무조건 먼저 확인해야 할 것은:

- 지금 sensor type이 맞는가?
- 지금 active link가 실제 보드 wiring과 맞는가?
- 지금 VIN channel이 올바른가?
- CSI channel / lane count / speed가 맞는가?
- ISP unit 기대와 실제 연결이 맞는가?

### 2-2. Linux capture 근거

`camera.c`, `camera_config.h`

보이는 핵심:

- `/dev/video0 ~ /dev/video7`
- `V4L2_PIX_FMT_SBGGR16`, `SBGGR12`, `ARGB32` 등
- `1920x1080`
- `open -> set_input -> set_crop -> set_fps -> set_format -> mmap -> qbuf/dqbuf -> stream_on`

즉 Linux 쪽 이슈는 대부분

- video node 생성 문제
- format mismatch
- streaming start 실패
- dqbuf timeout
- buffer 재queue 불량

으로 나뉜다.

### 2-3. virtualization / protocol bridge 근거

`camera_protocol.hpp`

- `CHANNEL_INIT`
- `CHANNEL_START`
- `CHANNEL_STOP`
- `CHANNEL_FEED_BUFFER`
- `FRAME_READY`
- `FEED_ME`

즉 이 레이어에서는 다음 질문이 중요하다.

- channel init은 성공했는가?
- start 이후 frame ready가 오는가?
- feed buffer를 누가 책임지는가?
- buffer가 고갈되어 feed-me 상태만 반복되는가?

---

## 3. Camera bring-up debug를 6단계로 나누기

---

## 3-1. Stage 1 — Sensor / SerDes / Link 살아있는가?

이 단계는 **아예 영상 입력이 SoC에 들어오기 전**이다.

### 증상
- `/dev/videoX`는 뜨는데 frame이 안 옴
- 아예 downstream이 조용함
- 간헐적 frame drop이 아니라 완전 no-frame
- 특정 camera channel만 죽음

### 먼저 볼 것
- sensor 전원 sequencing
- reset release 여부
- serializer / deserializer link lock
- sensor I2C access 가능 여부
- sensor ID read 가능 여부
- active link 선택이 보드 wiring과 일치하는지

### 레포 관점 체크 포인트
`r_camfwk_hal_camera_system.h` 기준으로:

- `active_link`
- `i2c_addr`
- `sensor_type`
- `num_cameras`
- `deser_index`

### 여기서 자주 나는 실수
1. **sensor type만 바꾸고 i2c 주소 안 맞춤**
2. board는 2-lane인데 config는 4-lane 가정
3. active link가 실제 연결된 port와 다름
4. SerDes는 lock인데 sensor register init이 빠짐

### 질문형 점검
- sensor가 진짜 ACK를 주는가?
- deserializer link status가 stable한가?
- power/reset sequence를 board bring-up 문서대로 탔는가?
- 한 channel만 죽으면 link map 문제인가, sensor 문제인가?

---

## 3-2. Stage 2 — CSI / VIN route가 맞는가?

sensor가 살아있어도 SoC capture route가 틀리면 프레임은 안 보인다.

### 증상
- link는 살아있는데 frame 없음
- wrong channel에서만 노이즈/timeout
- 특정 `/dev/videoX`만 비정상

### 레포 기준 체크 포인트
`r_camfwk_hal_camera_system.h`

- `vin_channel`
- `csi2_channel`
- `num_lanes`
- `mipi_lane_speed`
- `use_cphy`
- `isp_unit`

### 핵심 질문
- sensor가 내보내는 lane 수와 config lane 수가 일치하는가?
- sensor output speed와 receiver side speed가 맞는가?
- VIN channel mapping이 device node mapping과 맞는가?
- ISP를 거칠 sensor인데 bypass처럼 구성한 건 아닌가?

### 디버그 관점
이 단계는 보통 “카메라가 안 된다”가 아니라,
**routing / topology mismatch** 문제다.

즉 software bug처럼 보이지만 실은 아래 중 하나인 경우가 많다.

- 잘못된 VIN 채널 선택
- 잘못된 CSI 포트 선택
- lane count / speed mismatch
- RAW / YUV 기대 format mismatch

---

## 3-3. Stage 3 — CR52 camfwk init / route가 끝났는가?

하드웨어 route가 맞아도 framework init이 안 끝나면 상위 계층은 아무것도 못 한다.

### 관련 레포 흔적
- `r_camfwk_hal_camera_system.h`
- `camera_manager.h`
- FreeRTOS sample app 경로들

`camera_manager.h`를 보면 camera를 logical device처럼 다룬다.

- `camera_manager_init_device()`
- `camera_manager_capture_device()`
- `camera_manager_init_all_device()`

즉 CR52 쪽에서 이미 “device init -> capture start” 개념이 있다.

### 체크 포인트
- init 호출이 실제로 수행되는가?
- 특정 cam_id만 init되는가, all_device init인가?
- capture start 전에 route init이 끝났는가?
- CR52 firmware 이미지 자체가 현재 build에 맞는 버전인가?

### 흔한 문제
1. firmware 버전 mismatch
2. config는 4 cameras인데 실제 init은 1 camera만 수행
3. frontcam/rearview 샘플 설정이 섞임
4. camera db와 sensor wiring이 안 맞음

---

## 3-4. Stage 4 — Linux `/dev/videoX`와 V4L2 streaming이 되는가?

여기부터는 Linux camera executor의 문제다.

### `camera.c` 기준 핵심 흐름

- `open_device()`
- `v4l2_querry_cap()`
- `v4l2_set_input()`
- `v4l2_set_crop()`
- `v4l2_set_fps()`
- `v4l2_set_format()`
- `v4l2_init_mmap()`
- `v4l2_qbuf()`
- `v4l2_stream_on()`
- `select()`
- `v4l2_dqbuf()`

### 증상별 해석

#### A. `/dev/videoX`가 없다
- driver probe 안 됨
- media path 등록 실패
- VIN route 미완성
- domain/device tree 문제 가능

#### B. open은 되는데 set_format 실패
- pixel format mismatch
- unsupported resolution
- sensor output format과 capture 기대 format 불일치

#### C. stream_on은 되는데 dqbuf가 안 돌아옴
- 실제 프레임 ingress 없음
- buffer queue 문제
- interrupt path 문제
- upstream sensor output 없음

#### D. 몇 frame 오다가 멈춤
- buffer starvation
- FEED_BUFFER/queue replenishment 문제
- timeout handling 부재
- downstream consumer가 막힘

### 꼭 확인할 질문
- `/dev/videoX`와 `vin_channel` 매핑이 맞는가?
- format이 RAW인지 YUV인지 ARGB인지 명확한가?
- capture width/height가 sensor 최대 해상도 범위 안인가?
- mmap buffer count가 충분한가?

---

## 3-5. Stage 5 — Buffer / RPMsg / Frame-ready 연계가 맞는가?

이 단계는 virtualization / inter-core bridge의 핵심이다.

### `camera_protocol.hpp` 기준 체크 포인트

- `CHANNEL_INIT`
- `CHANNEL_START`
- `CHANNEL_FEED_BUFFER`
- `FRAME_READY`
- `FEED_ME`

### 해석
정상적인 흐름은 대략 이렇다.

1. channel init
2. buffer 등록
3. start
4. frame ready event 수신
5. 소비 후 buffer feed
6. 반복

### 문제 패턴

#### A. init은 되는데 frame ready가 안 온다
- upstream capture가 실제로 안 돌고 있음
- start sequence 누락
- channel 매핑 mismatch

#### B. frame ready는 한 번 오는데 이후 멈춘다
- feed buffer가 안 들어감
- empty buffer pool 고갈
- consumer가 buffer release를 안 함

#### C. feed-me만 계속 나온다
- producer는 살아있지만 guest/consumer가 buffer replenishment를 못 함

#### D. frame size / channel info mismatch
- shared buffer 크기와 실제 frame size 불일치
- width/height 합의가 안 맞음

### 이 단계 핵심
이건 단순 driver debug가 아니라,
**producer-consumer contract debug**다.

즉 “frame이 오냐 마냐”보다,

- 누가 buffer를 ownership 하는지
- 언제 recycle 되는지
- ready event와 feed event가 짝이 맞는지

를 봐야 한다.

---

## 3-6. Stage 6 — ISP / IMR / AI / VOUT 후단 기대가 맞는가?

camera ingress가 살아도 후단 기대가 틀리면 사용자는 “카메라 안 됨”처럼 느낀다.

### `frontcamera_customize.cpp` 기준
지원 pipeline 예:

- `CAPTURE`
- `CAPTURE -> ISP`
- `CAPTURE -> IMR`
- `CAPTURE -> IMR -> AI`
- `CAPTURE -> VOUT`
- `CAPTURE -> ISP -> IMR -> AI -> VOUT`

### 체크 포인트
- 현재 pipeline이 어떤 조합으로 켜졌는가?
- ISP가 enable인데 실제 input이 RAW인가?
- IMR resize path가 frame size와 맞는가?
- AI resize 채널이 잘못되어 downstream만 죽는가?
- VOUT만 안 보이면 capture는 살았는가?

### 흔한 오해
- “화면에 안 뜬다 = 카메라 안 된다”는 아니다.
- 실제로는 capture는 정상인데 VOUT path만 죽어 있을 수 있다.
- 또는 AI stage가 invalid config라서 application pipeline 전체가 fail될 수 있다.

즉 반드시 구분해야 한다.

1. **no-frame 문제**
2. **frame captured but not displayed 문제**
3. **frame displayed but AI path broken 문제**

---

## 4. 실제 디버그 순서 추천

내가 이 구조라면 아래 순서로 본다.

### Step 1 — 하드웨어 link 확인
- sensor 응답
- serdes lock
- power/reset sequence

### Step 2 — topology 확인
- sensor type
- active link
- CSI/VIN mapping
- lane/speed

### Step 3 — CR52 framework init 확인
- route init
- device init
- capture start sequence

### Step 4 — Linux node 확인
- `/dev/videoX`
- open / format / stream_on
- dqbuf 여부

### Step 5 — event/buffer 확인
- init/start success
- frame_ready 수신
- feed_buffer 반복 여부

### Step 6 — 후단 pipeline 확인
- ISP/IMR config
- AI resize / model path
- VOUT / display path

이 순서를 뒤집으면 시간이 많이 날아간다. 예를 들어 display가 안 보인다고 Android/HWC부터 보면 안 되고, frame_ready도 안 오는데 SurfaceFlinger를 보는 것도 순서가 틀린다.

---

## 5. bring-up에서 특히 잘 터지는 함정

### 함정 1 — format mismatch를 routing 문제로 오해
RAW sensor인데 YUV capture처럼 보면 format set부터 꼬인다.

### 함정 2 — `/dev/video`가 뜬다고 ingress 정상이라 착각
node가 떠도 실제 sensor frame이 안 들어오면 dqbuf는 timeout 난다.

### 함정 3 — 한 번 frame 나오고 멈추는 문제를 sensor issue로 오해
이건 buffer recycle / feed path 문제일 때가 많다.

### 함정 4 — VOUT 실패를 capture 실패로 오해
capture, AI, VOUT는 분리해서 봐야 한다.

### 함정 5 — virtualization boundary를 빼먹음
shared buffer / rpmsg / virt queue가 들어가면, 단순 Linux local pipeline처럼 생각하면 안 된다.

---

## 6. 내가 이 시스템에서 제일 먼저 만들고 싶은 로그 체계

이 구조는 계층이 많아서, 로그가 한 군데만 있어도 문제를 못 좁힌다. 그래서 최소한 아래 태그는 구분되어야 한다.

- `CAM-HW` : sensor/serdes/link
- `CAM-ROUTE` : csi/vin/isp route
- `CAM-RT` : CR52 camera fw init/start
- `CAM-V4L2` : Linux capture state
- `CAM-BUF` : buffer lifecycle
- `CAM-RPMSG` : frame_ready / feed_me / init/start
- `CAM-PIPE` : ISP/IMR/AI/VOUT pipeline stage

이렇게만 나눠도 “지금 no-frame이 어디서 생긴 건지”를 훨씬 빨리 좁힐 수 있다.

---

## 7. 면접/실무에서 이렇게 말하면 좋다

이 구조를 설명할 때는 아래 식으로 요약 가능하다.

> In this platform, camera bring-up is not just a Linux V4L2 issue. It spans sensor and SerDes initialization, CSI/VIN routing, CR52 camera framework configuration, Linux streaming setup, virtualization-aware buffer/event handling, and optional downstream ISP/IMR/AI/VOUT stages. The key to fast debug is to separate the problem by stage and verify ownership at each boundary.

즉 중요한 건 “카메라를 안다”보다,
**bring-up failure를 계층별로 분해해서 책임 경계를 자를 수 있다**는 점이다.

---

## 8. 정리

Day58의 결론은 간단하다.

- camera bring-up은 단일 드라이버 문제가 아니다.
- 이 X5H 구조에서는 **sensor -> route -> CR52 fw -> Linux V4L2 -> buffer bridge -> downstream pipeline**로 끊어서 봐야 한다.
- 증상은 비슷해도 root cause는 completely different layer에 있을 수 있다.
- 그래서 가장 중요한 역량은 “많이 아는 것”보다 **빠르게 어느 계층 문제인지 가르는 것**이다.

## 다음 액션

다음 Day59에서는 반대로 **display bring-up debug checklist**를 정리해보겠다.

- `disfwk_fe` DRM path
- plane / CRTC / connector
- framebuffer / atomic commit
- Android HWC3 / SurfaceFlinger / multi-display 경계

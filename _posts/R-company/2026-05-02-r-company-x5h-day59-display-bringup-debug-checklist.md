---
title: R사 X5H Day59 - Display Bring-up Debug Checklist (DRM/KMS, disfwk_fe, HWC3, SurfaceFlinger, Multi-display)
author: JaeHa
date: 2026-05-02 01:00:00 +0900
categories: [R사, X5H, Display, Bring-up, Debug]
tags: [R사, X5H, Day59, display, bringup, debug, drm, kms, hwc3, surfaceflinger, xen]
---

Day57~58에서 camera 입력과 virtualization 경계를 먼저 정리했으니, 이번에는 그 다음 단계인 **display bring-up**을 체크리스트 형태로 다시 묶는다. 이전 표현만 보면 Linux가 디스플레이를 전부 직접 올리는 것처럼 읽힐 수 있는데, X5H는 그렇게 단순하지 않다. 실제 경로는 **CR52/FreeRTOS 쪽 disfwk 백엔드가 먼저 살아 있고**, Linux/Android 쪽은 그 위에서 **`disfwk_fe` DRM 프런트엔드와 HWC3/SurfaceFlinger** 로 제어·합성·라우팅을 담당한다.

즉 X5H의 black screen은 그냥 “Linux 화면 안 나옴”이 아니라, 대체로 아래 5층 중 어디서 끊기는지의 문제다.

1. **CR52 disfwk / panel backend 초기화** 자체가 안 됨
2. **Linux `disfwk_fe` DRM 진입**이 안 됨
3. **DRM resource / framebuffer / atomic commit 구성**이 틀어짐
4. **RPMsg/RVGC contract 또는 layer flush** 쪽 hand-off가 틀어짐
5. **Android HWC3 / SurfaceFlinger / multi-display policy** 에서 다른 logical display로 감

이번 글의 목적은 “검은 화면”을 하나의 증상으로 뭉개지 않고, **누가 실제 owner인지부터 분리해서 어느 레이어에서 막혔는지 빠르게 절단**하는 기준을 만드는 것이다.

## 핵심 요약

- display bring-up은 최소한 **CR52/FreeRTOS backend**, **Linux `disfwk_fe` DRM frontend**, **framebuffer/atomic commit**, **Android composer/UI policy** 네 층으로 나눠서 봐야 한다.
- `display.c`를 보면 FreeRTOS 쪽에는 `R_WM_DisplayInit`, `R_WM_DisplayEnable`, `R_WM_LayerFlush` 기반의 **native display framework** 가 실제로 존재한다.
- `display_drm.c`를 보면 Linux 쪽에는 `drmOpen("disfwk_fe") → drmModeAddFB2() → drmModeAtomicCommit()` 경로가 존재한다. 다만 이건 **백엔드를 직접 소유한다기보다 CR52 disfwk를 향해 요청을 보내는 프런트엔드/실행자** 로 읽는 게 더 정확하다.
- `r_taurus_rvgc_protocol.h`를 보면 `DISPLAY_INIT`, `LAYER_SET_ADDR`, `DISPLAY_FLUSH`, `VBLANK_DISPLAY0~9` 계약이 이미 정의돼 있다. 즉 Linux atomic commit 뒤에는 **RPMsg 기반 RVGC hand-off** 가 한 번 더 있다.
- “화면이 안 뜬다”는 증상은 아래 다섯 부류로 분류하는 게 빠르다.
  1. CR52 disfwk / panel backend 자체가 안 살아남
  2. `disfwk_fe` open 또는 DRM 리소스 탐색(connector/CRTC/plane) 실패
  3. framebuffer format/stride/dumb buffer 설정 실패
  4. atomic commit 또는 flush hand-off는 갔지만 올바른 layer/target에 안 올라감
  5. Android HWC3 / SurfaceFlinger / multi-display policy에서 다른 display로 라우팅됨

## 0. 먼저 그림으로 보는 Display Bring-up Debug Map

### 0-1. display 전체 계층 구조

<object data="/assets/img/r-company/x5h-day59-diagram-01.png" type="image/png" aria-label="X5H Day59 diagram 1" style="width:100%; height:auto; border-radius:12px; margin:12px 0 20px;"></object>


### 0-2. bring-up 절단 순서

<object data="/assets/img/r-company/x5h-day59-diagram-02.png" type="image/png" aria-label="X5H Day59 diagram 2" style="width:100%; height:auto; border-radius:12px; margin:12px 0 20px;"></object>


### 0-3. 증상별 최초 의심 영역

<object data="/assets/img/r-company/x5h-day59-diagram-03.png" type="image/png" aria-label="X5H Day59 diagram 3" style="width:100%; height:auto; border-radius:12px; margin:12px 0 20px;"></object>


## 코드 포인트

### 0-4. Linux direct scanout과 Android policy의 관계

<object data="/assets/img/r-company/x5h-day59-diagram-04.png" type="image/png" aria-label="X5H Day59 diagram 4" style="width:100%; height:auto; border-radius:12px; margin:12px 0 20px;"></object>


### 1) Linux display 실행 경로: `display_drm.c`

레포에서 먼저 눈에 띄는 건 Linux sample이 `disfwk_fe`를 DRM 디바이스처럼 열고 atomic path를 돈다는 점이다.

핵심 흐름은 대략 아래 순서다.

1. `drmOpen("disfwk_fe")`
2. connector / CRTC / plane 탐색
3. dumb buffer 생성 (`drmModeCreateDumbBuffer`)
4. `drmModeAddFB2()`로 framebuffer 등록
5. plane property / CRTC property 세팅
6. `drmModeAtomicCommit()` 수행

여기까지만 보면 “아, Linux가 display를 직접 다 올리네”라고 읽기 쉽다. 그런데 이 해석은 반만 맞다. 이 경로는 **Linux 프런트엔드 관점의 실행 경로**이고, 실제 backend owner가 어디인지는 다음 레이어를 같이 봐야 한다.

즉 first bring-up에서 먼저 확인할 질문은 아래처럼 두 묶음이다.

- Linux frontend 자체는 정상인가?
  - `disfwk_fe`가 실제로 open 되는가?
  - connector가 connected 상태인가?
  - 사용할 CRTC/plane 조합을 찾는가?
  - framebuffer 생성과 `AddFB2`가 성공하는가?
  - atomic commit return code가 0인가?
- 그런데 commit 뒤 hand-off는 정상인가?
  - commit 성공 뒤 실제 panel update/VBLANK가 오는가?
  - display index / layer target이 CR52 backend 기대와 맞는가?

### 2) Display ownership 경계: FreeRTOS backend vs Linux frontend

이 플랫폼은 FreeRTOS 쪽 native display API(`R_WM_DisplayInit`, `R_WM_DisplayEnable`, `R_WM_LayerFlush`)도 있고, Linux DRM/KMS path도 같이 보인다. 여기서 핵심은 둘 중 하나가 가짜라는 뜻이 아니라, **둘이 서로 다른 층의 owner** 라는 점이다.

`display.c` 쪽에서 보이는 사실:

- `R_WM_DisplayInit()` → `R_WM_DevInit()` + screen property set
- `R_WM_DisplayEnable()` → screen enable/disable
- `R_WM_LayerFlush()` / `R_WM_ScreenSurfacesFlush()` → 실제 layer update flush

즉 CR52 / FreeRTOS 쪽에는 **native display backend API** 가 실제로 있다. 단순 더미 stub가 아니다.

반대로 `display_drm.c` 쪽에서 보이는 사실:

- Linux는 `disfwk_fe`를 열어 DRM 리소스를 잡고
- dumb buffer / FB / atomic req를 구성한 뒤
- commit을 날리는 **DRM frontend executor** 역할을 한다.

그리고 `r_taurus_rvgc_protocol.h`를 보면 그 아래에 더 중요한 계약이 있다.

- `RVGC_PROTOCOL_IOC_DISPLAY_INIT`
- `RVGC_PROTOCOL_IOC_LAYER_SET_ADDR`
- `RVGC_PROTOCOL_IOC_LAYER_SET_POS / SIZE / FMT`
- `RVGC_PROTOCOL_IOC_DISPLAY_FLUSH`
- `RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY0~9`

이건 결국 **Linux `disfwk_fe` ↔ CR52 disfwk** 사이에 RPMsg/RVGC 프로토콜이 있고, Linux atomic commit이 최종 물리 scanout 그 자체라기보다 **CR52 backend에 layer/flush 요청을 전달하는 상위 제어 경로** 라는 뜻이다.

그래서 ownership을 이렇게 정리하는 게 제일 덜 헷갈린다.

- **CR52 / FreeRTOS disfwk**: display backend init, layer flush, panel 쪽 실제 구동 주체
- **Linux `disfwk_fe` DRM**: frontend 제어/버퍼/atomic 모델 제공
- **Android HWC3 / SurfaceFlinger**: composition + logical display policy

따라서 문제를 볼 때도 반드시 이렇게 잘라야 한다.

- 지금 black screen이 **CR52 backend 초기화 이전** 문제인가?
- Linux atomic commit까지는 성공했지만 **RVGC flush/VBLANK hand-off** 가 안 되는가?
- 혹은 backend는 살았는데 **Android가 다른 display로 라우팅** 했는가?

### 3) Android 쪽 policy 근거

레포에서 Android display policy를 읽게 해주는 포인트는 아래 파일들이다.

- `android_device/display_layout_configuration.xml`
- `android_device/display_settings.xml`
- `aosp_xenvm_trout_arm64.mk`

실제 XML을 보면:

- `display_layout_configuration.xml` 에서 `address 0 = defaultDisplay`, `address 1 = passenger_display`
- `display_settings.xml` 에서 `local:1` 에 `shouldShowSystemDecors`, `shouldShowIme`, `forcedDensity=160` 같은 별도 정책 적용

즉 여기서 확인되는 건:

- default display와 passenger display가 실제로 분리돼 있음
- secondary display에 별도 settings/policy가 적용됨
- HWC3 + SurfaceFlinger + CarService overlay가 display routing에 개입함

그래서 화면이 “안 나온다”가 아니라 **Linux/CR52 쪽 scanout은 살았는데 다른 logical display target에 올라간 것**일 가능성도 항상 열어둬야 한다.

## 리스크

### Stage 1 — CR52 disfwk / panel backend 생존 확인 실패

증상:
- connector가 disconnected로 보임
- backlight는 켜지는데 실제 scanout 없음
- 특정 display port만 죽음
- Linux 쪽 commit 시도 전부터 panel 쪽 반응이 이상함

리스크 해석:
- panel power sequence, bridge IC init, link training, connector detect 중 하나가 안 맞을 수 있다.
- 더 아래쪽으로는 CR52 disfwk firmware 초기화, display device init, layer reserve/flush 이전 단계가 막혔을 가능성도 있다.
- 이 단계가 안 풀리면 HWC3나 SurfaceFlinger를 아무리 봐도 시간만 버린다.

### Stage 2 — Linux DRM frontend 또는 RPMsg hand-off 실패

증상:
- `drmOpen("disfwk_fe")`는 되지만 connector/CRTC/plane 선택 실패
- atomic req 구성 전 단계에서 abort
- 또는 atomic commit은 성공처럼 보이는데 실제 update/vblank가 안 옴

리스크 해석:
- device tree / domain assignment / display index 매핑이 실제 HW와 어긋났을 수 있다.
- `disfwk_fe`가 잡은 display/layer 정보와 CR52 disfwk backend 기대가 어긋났을 수 있다.
- virtualized platform에서는 단순 DomD/guest ownership뿐 아니라, **Linux frontend ↔ CR52 backend RPMsg 계약 불일치** 도 함께 의심해야 한다.

### Stage 3 — Framebuffer format/stride mismatch

증상:
- commit은 성공하지만 화면이 깨지거나 녹색/보라색 noise가 남
- 일부 해상도에서만 blank 또는 partial update 발생

리스크 해석:
- pixel format, pitch/stride, plane modifier 기대가 다르면 commit 성공만으로는 정상 표시를 보장하지 못한다.
- camera/display 연계 시 producer buffer format과 display plane 기대 format이 다르면 이 레이어에서 터진다.

### Stage 4 — Atomic commit success, but wrong plane / z-order / target

증상:
- commit은 success
- 화면은 그대로 black
- cursor/UI 일부만 보이거나 완전히 다른 output에만 나감

리스크 해석:
- plane attach target, CRTC association, source/destination rectangle, alpha/z-order 설정이 틀렸을 수 있다.
- 특히 multi-display 환경에서는 commit이 성공해도 의도한 panel이 아닌 다른 logical display에 올라갈 수 있다.

### Stage 5 — Android composer/policy mismatch

증상:
- Linux 레벨 test pattern이나 direct scanout은 보이는데 Android UI만 안 보임
- bootanimation 또는 launcher가 passenger display로 감
- SurfaceFlinger는 살아있는데 user-visible UI가 없음

리스크 해석:
- 이건 하드웨어 bring-up 실패가 아니라 HWC3 policy, display layout, CarService overlay, default display assignment 문제일 가능성이 높다.

## 다음 액션

내가 실제 bring-up을 맡았다면 아래 순서로 자른다.

1. **CR52 backend / panel state 먼저 고정**
   - panel power, detect, link, backlight, connected state 확인
   - FreeRTOS/disfwk 쪽 display init / enable / flush 가능 상태인지 먼저 확인
2. **Linux DRM frontend 검증**
   - `disfwk_fe` open
   - connector/CRTC/plane 탐색
   - dumb buffer + `AddFB2`
   - atomic commit success 확인
3. **RPMsg / RVGC hand-off 확인**
   - display index / layer / format / flush 요청이 backend 기대와 맞는지 점검
   - commit 이후 실제 vblank/update 반응이 오는지 확인
4. **Known-good test pattern 확보**
   - camera/Android 없이 단색 또는 color-bar scanout 먼저 고정
5. **Buffer contract 검증**
   - format, stride, width/height, plane target 일치 여부 확인
6. **Android policy 검증**
   - HWC3, SurfaceFlinger, default/passenger display mapping 점검
7. **최종적으로 camera/display end-to-end 연결**
   - capture가 살았더라도 display plane target이 맞지 않으면 user 입장에선 여전히 실패다.

실무적으로는 여기서 한 단계 더 들어가서 **Day60에 camera buffer가 display plane까지 넘어오는 ownership/latency contract**를 정리하는 게 좋다. 그 주제를 잡으면 camera bring-up과 display bring-up 사이의 실제 통합 경계를 더 날카롭게 설명할 수 있다.

## 부록: 최종 display ownership 그림

<object data="/assets/img/r-company/x5h-day59-diagram-05.png" type="image/png" aria-label="X5H Day59 diagram 5" style="width:100%; height:auto; border-radius:12px; margin:12px 0 20px;"></object>


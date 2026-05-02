---
title: R사 X5H Day59 - Display Bring-up Debug Checklist (DRM/KMS, disfwk_fe, HWC3, SurfaceFlinger, Multi-display)
author: JaeHa
date: 2026-05-02 01:00:00 +0900
categories: [R사, X5H, Display, Bring-up, Debug]
tags: [R사, X5H, Day59, display, bringup, debug, drm, kms, hwc3, surfaceflinger, xen]
---

Day57~58에서 camera 입력과 virtualization 경계를 먼저 정리했으니, 이번에는 그 다음 단계인 **display bring-up**을 체크리스트 형태로 묶는다. X5H 계열에서 화면이 안 뜨는 문제는 단순히 Android UI 문제로 보면 안 된다. 실제로는 **Linux DRM/KMS scanout**, **`disfwk_fe` 프런트엔드**, **buffer format/stride**, **Android HWC3/SurfaceFlinger**, **multi-display policy**가 각각 다른 실패 지점을 만든다.

이번 글의 목적은 “검은 화면”을 하나의 증상으로 뭉개지 않고, **어느 레이어에서 막혔는지 빠르게 절단**하는 기준을 만드는 것이다.

## 핵심 요약

- display bring-up은 최소한 **panel/link**, **DRM device/connector/CRTC/plane**, **framebuffer/atomic commit**, **Android composer/UI policy** 네 층으로 나눠서 봐야 한다.
- X5H 레포 기준으로 Linux 쪽 핵심 실행 경로는 `disfwk_fe` 기반 DRM/KMS atomic commit이고, Android는 그 위에서 HWC3/SurfaceFlinger consumer 역할을 맡는다.
- “화면이 안 뜬다”는 증상은 아래 다섯 부류로 분류하는 게 빠르다.
  1. display HW 또는 panel link 자체가 안 살아남
  2. DRM 리소스 탐색(connector/CRTC/plane) 실패
  3. framebuffer format/stride/dumb buffer 설정 실패
  4. atomic commit은 성공하지만 올바른 plane에 안 올라감
  5. Android HWC3 / SurfaceFlinger / multi-display policy에서 다른 display로 라우팅됨

## 0. 먼저 그림으로 보는 Display Bring-up Debug Map

### 0-1. display 전체 계층 구조

![X5H Day59 diagram 1](/assets/img/r-company/x5h-day59-diagram-01.png)


### 0-2. bring-up 절단 순서

![X5H Day59 diagram 2](/assets/img/r-company/x5h-day59-diagram-02.png)


### 0-3. 증상별 최초 의심 영역

![X5H Day59 diagram 3](/assets/img/r-company/x5h-day59-diagram-03.png)


## 코드 포인트

### 0-4. Linux direct scanout과 Android policy의 관계

![X5H Day59 diagram 4](/assets/img/r-company/x5h-day59-diagram-04.png)


### 1) Linux display 실행 경로: `display_drm.c`

레포에서 제일 중요한 근거는 Linux display sample이 `disfwk_fe`를 DRM 디바이스처럼 열고 atomic path를 돈다는 점이다.

핵심 흐름은 대략 아래 순서다.

1. `drmOpen("disfwk_fe")`
2. connector / CRTC / plane 탐색
3. dumb buffer 생성
4. `drmModeAddFB2()`로 framebuffer 등록
5. plane property / CRTC property 세팅
6. `drmModeAtomicCommit()` 수행

즉 first bring-up에서 먼저 확인할 질문은 아주 명확하다.

- `disfwk_fe`가 실제로 open 되는가?
- connector가 connected 상태인가?
- 사용할 CRTC/plane 조합을 찾는가?
- framebuffer 생성과 `AddFB2`가 성공하는가?
- atomic commit return code가 0인가?

### 2) Display ownership 경계: FreeRTOS path vs Linux path

이 플랫폼은 FreeRTOS 쪽 native display API(`R_WM_DisplayInit`, `R_WM_LayerFlush` 계열)도 있고, Linux DRM/KMS path도 같이 보인다. 그래서 bring-up 중 가장 위험한 오해는 **누가 최종 scanout owner인지 불명확한 상태**다.

정리하면:

- FreeRTOS 쪽은 low-level/native display framework path
- Linux 쪽은 `disfwk_fe` 기반 DRM/KMS executor
- Android는 guest에서 HWC3/SurfaceFlinger로 composition/policy 담당

따라서 문제를 볼 때도 반드시 잘라야 한다.

- 지금 black screen이 Linux DRM 이전 문제인가?
- Linux atomic commit까지는 성공했는데 Android UI만 안 보이는가?
- 혹은 Android는 정상인데 main/passenger display route가 바뀐 것인가?

### 3) Android 쪽 policy 근거

레포에서 Android display policy를 읽게 해주는 포인트는 아래 파일들이다.

- `android_device/display_layout_configuration.xml`
- `android_device/display_settings.xml`
- `aosp_xenvm_trout_arm64.mk`

여기서 확인되는 건:

- default display와 passenger display가 분리돼 있음
- secondary display에 별도 settings/policy가 적용됨
- HWC3 + SurfaceFlinger + CarService overlay가 display routing에 개입함

즉 화면이 “안 나온다”가 아니라 **다른 display logical target에 올라간 것**일 가능성도 항상 열어둬야 한다.

## 리스크

### Stage 1 — Panel/link/HW 생존 확인 실패

증상:
- connector가 disconnected로 보임
- backlight는 켜지는데 실제 scanout 없음
- 특정 display port만 죽음

리스크 해석:
- panel power sequence, bridge IC init, link training, connector detect 중 하나가 안 맞을 수 있다.
- 이 단계가 안 풀리면 HWC3나 SurfaceFlinger를 아무리 봐도 시간만 버린다.

### Stage 2 — DRM resource binding 실패

증상:
- `drmOpen("disfwk_fe")`는 되지만 connector/CRTC/plane 선택 실패
- atomic req 구성 전 단계에서 abort

리스크 해석:
- device tree / domain assignment / display index 매핑이 실제 HW와 어긋났을 수 있다.
- virtualized platform에서는 DomD와 guest 사이 ownership 충돌도 의심해야 한다.

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

1. **HW / connector state 고정**
   - panel power, detect, link, backlight, connected state 확인
2. **DRM direct path 검증**
   - `disfwk_fe` open
   - connector/CRTC/plane 탐색
   - dumb buffer + `AddFB2`
   - atomic commit success 확인
3. **Known-good test pattern 확보**
   - camera/Android 없이 단색 또는 color-bar scanout 먼저 고정
4. **Buffer contract 검증**
   - format, stride, width/height, plane target 일치 여부 확인
5. **Android policy 검증**
   - HWC3, SurfaceFlinger, default/passenger display mapping 점검
6. **최종적으로 camera/display end-to-end 연결**
   - capture가 살았더라도 display plane target이 맞지 않으면 user 입장에선 여전히 실패다.

실무적으로는 여기서 한 단계 더 들어가서 **Day60에 camera buffer가 display plane까지 넘어오는 ownership/latency contract**를 정리하는 게 좋다. 그 주제를 잡으면 camera bring-up과 display bring-up 사이의 실제 통합 경계를 더 날카롭게 설명할 수 있다.

## 부록: 최종 display ownership 그림

![X5H Day59 diagram 5](/assets/img/r-company/x5h-day59-diagram-05.png)


---
title: R사 X5H Day72 - DISPLAY_GET_INFO 와 panel mode/format mismatch가 black screen으로 번지는 경로
author: JaeHa
date: 2026-05-13 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day72, display, panel, timing, format, pitch, bpp, black-screen, bringup]
---

Day71에서 `flush ok + no-VBLANK` 를 `misroute / enable / timing` 으로 10분 안에 자르는 체크리스트를 정리했다면, 그다음은 **timing 쪽에서도 특히 `DISPLAY_GET_INFO` 와 실제 panel mode/format이 어떻게 어긋나야 black screen으로 이어지는지** 를 더 좁혀야 한다. bring-up 초반에는 해상도 숫자만 맞으면 안심하기 쉬운데, 실제로는 `pitch`, bpp, pixel format, pixel clock 중 하나만 틀려도 scanout은 조용히 죽는다.

## 핵심 요약

- `DISPLAY_GET_INFO width/height` 가 panel 해상도와 같아도 **`pitch`, bpp, format, pixelclock` 중 하나가 어긋나면** visible output은 실패할 수 있다.
- `RVGC_FLUSH=OK` 와 `COMPLETE=OK` 는 command lane 성공일 뿐이고, **final fetch stride 해석과 panel timing generator lock** 까지는 보장하지 않는다.
- bring-up 현장에서는 `DISPLAY_GET_INFO` 와 `PANEL_MODE` 를 한 줄 비교하는 것보다, **`pitch = width * bytes_per_pixel` 가 맞는지** 부터 보는 편이 가장 빠르다.
- `no-VBLANK` 가 아니라 `VBLANK는 있는데 화면이 까맣다` 면 timing보다 먼저 **format/stride mismatch** 를 의심해야 한다.

## 코드 포인트

1. **`DISPLAY_GET_INFO` 는 backend가 믿는 fetch 계약이다**

   Day70~71에서 본 capability 응답은 단순 참고값이 아니라 scanout 계약의 기준점이다.

   ```c
   uint32_t width;
   uint32_t height;
   uint32_t pitch;
   uint32_t layers;
   ```

   여기서 `width=1920`, `height=1080` 만 맞고 `pitch=3840` 인데 실제 buffer format이 ARGB8888(4 Bpp)라면 fetch 쪽 기대 pitch는 보통 `7680` 이어야 한다. 이 차이는 panel dead처럼 보이지만 실체는 **line stride 해석 mismatch** 다.

2. **format mismatch는 flush 성공 이후에야 드러날 수 있다**

   backend는 앞단에서 format property를 받는다.

   ```c
   surf->Fmt = in->format;
   wm_ret = R_WM_SurfacePropertySet(..., R_WM_SURF_PROP_COLORFMT, surf);
   ```

   이 호출이 `OK` 라도 panel path가 실제로 기대하는 fetch format과 다르면, command는 수락됐지만 final visible output이 깨지거나 완전히 까매질 수 있다. 즉 `set_fmt ok` 는 **정합성 증명** 이 아니라 **요청 접수 성공** 에 가깝다.

3. **현장에서는 stride 계산식 한 줄이 가장 강력하다**

   아래처럼 비교 로그를 바로 남기면 된다.

   ```text
   DISPLAY_GET_INFO display_slot=0 width=1920 height=1080 pitch=3840
   PANEL_EXPECT panel=cluster format=ARGB8888 bpp=4 expected_pitch=7680
   ```

   이 조합이면 panel glass fault보다 먼저 `format/pitch mismatch` 를 봐야 한다. 반대로 `RGB565`(2 Bpp) panel path라면 `3840` 이 맞을 수도 있으니, 핵심은 **해상도 숫자 자체가 아니라 bytes-per-pixel까지 포함한 계약 비교** 다.

4. **VBLANK 유무에 따라 절단면이 달라진다**

   - `flush ok + no-VBLANK` : timing generator / enable / route 우선
   - `flush ok + VBLANK alive + black screen` : format / stride / plane content 우선

   예를 들면 아래 로그는 timing보다 format 쪽이다.

   ```text
   RVGC_FLUSH seq=126 display_slot=0 result=OK
   RVGC_COMPLETE seq=126 display_slot=0 result=OK
   RVGC_VBLANK_SUMMARY window=500ms display_slot=0 count=30
   SCREEN_STATE panel=cluster black
   ```

   VBLANK가 살아 있으면 refresh는 돌고 있으므로, 완전 무출력보다 **잘못된 buffer fetch 또는 unsupported format** 가능성이 더 높다.

5. **pixelclock mismatch는 `간헐적 표시 후 소실` 패턴으로도 나타난다**

   ```text
   PANEL_MODE panel=cluster hactive=1920 vactive=1080 pixelclock=74250
   PANEL_EXPECT panel=cluster target_pixelclock=148500
   ```

   이런 경우는 완전 무반응만이 아니라, 부팅 로고 직후 소실·깜빡임·불안정 sync로 보일 수 있다. 그래서 bring-up 초반에는 black screen만 보지 말고 **잠깐이라도 켜졌는지** 를 같이 기록해야 timing 축을 더 빨리 자를 수 있다.

## 리스크

- `width/height` 만 맞다고 보고 `pitch` 비교를 생략하면 format mismatch를 panel dead로 오판하기 쉽다.
- `set_fmt ok` 를 실제 scanout 정합성으로 받아들이면 backend acceptance와 visible output failure를 구분하지 못한다.
- VBLANK가 살아 있는 black screen을 timing 문제로만 밀면 route/format/plane content 버그를 오래 파게 된다.
- pixelclock 불일치를 놓치면 `가끔 보임` 류의 비결정적 현상을 software race로 오해할 수 있다.

## 다음 액션

다음 글에서는 Day72를 이어서, **`VBLANK는 살아 있는데 black screen` 인 경우를 plane content/address/stride 관점에서 1차 절단하는 체크리스트** 로 더 압축해 보겠다.

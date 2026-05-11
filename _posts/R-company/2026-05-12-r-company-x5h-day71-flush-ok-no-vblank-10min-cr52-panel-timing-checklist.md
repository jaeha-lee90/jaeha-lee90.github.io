---
title: R사 X5H Day71 - flush 성공 후 no-VBLANK를 10분 안에 자르는 CR52/panel timing 체크리스트
author: JaeHa
date: 2026-05-12 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day71, display, flush, vblank, cr52, panel, timing, checklist, bringup]
---

Day70에서 `flush ok + no-VBLANK` 를 **final scanout 문제** 로 잘랐다면, 현장에서는 그다음 10분 안에 `misroute / panel enable 누락 / timing 불일치` 셋 중 어디인지 결론내야 한다. 이 단계에서 오래 끌면 Android/HWC 로그만 늘어나고 복구 속도는 거의 안 좋아진다.

## 핵심 요약

- `RVGC_FLUSH=OK` 와 `COMPLETE=OK` 가 이미 있으면, 우선순위는 userspace가 아니라 **CR52 route → panel enable → timing/format** 순서다.
- 같은 창에서 **반대 panel VBLANK가 살아 있으면 misroute**, 둘 다 죽어 있으면 **global enable/clock/timing** 쪽을 먼저 의심한다.
- `DISPLAY_GET_INFO` 의 `width/height/pitch/layers` 와 panel timing 기대치가 다르면, flush는 성공해도 visible refresh는 죽을 수 있다.
- 현장 최소 증거는 `flush/complete`, display별 `VBLANK_SUMMARY`, panel enable 로그, mode/timing 덤프 네 묶음이면 충분하다.

## 코드 포인트

1. **첫 2분: target/other panel VBLANK를 바로 비교한다**

   ```text
   RVGC_FLUSH seq=118 display_slot=0 result=OK
   RVGC_COMPLETE seq=118 display_slot=0 result=OK
   RVGC_VBLANK_SUMMARY window=500ms display_slot=0 count=0
   RVGC_VBLANK_SUMMARY window=500ms display_slot=1 count=31
   ```

   이 패턴이면 panel glass fault보다 먼저 **route mismatch** 를 본다. Day65~67에서 정리한 `display_mapping` 과 실제 scanout 번호가 엇갈렸다는 뜻이기 때문이다.

2. **둘 다 VBLANK가 죽어 있으면 route보다 enable/timing을 먼저 본다**

   ```text
   RVGC_VBLANK_SUMMARY window=500ms display_slot=0 count=0
   RVGC_VBLANK_SUMMARY window=500ms display_slot=1 count=0
   ```

   이 경우는 특정 panel 미스라우트보다 **CR52 display block enable, clock, bridge, timing generator** 공통 축이 더 유력하다. 즉 `flush` command lane은 살았지만 scanout engine이 실제 refresh를 못 만든 상태다.

3. **backend capability와 panel mode 기대치를 한 줄로 맞춰 둔다**

   Day70에서 본 capability 응답은 여기서 다시 중요하다.

   ```c
   uint32_t width;
   uint32_t height;
   uint32_t pitch;
   uint32_t layers;
   ```

   여기에 panel 쪽 기대치를 바로 붙여 비교한다.

   ```text
   DISPLAY_GET_INFO display_slot=0 width=1920 height=1080 pitch=7680
   PANEL_MODE panel=cluster hactive=1920 vactive=1080 pixelclock=148500
   ```

   `width/height` 는 맞아도 `pitch`, pixel format, bpp 가 실제 panel path와 어긋나면 blank가 난다. bring-up 초반에는 해상도만 맞는다고 안심하면 안 된다.

4. **panel enable 순서는 flush보다 앞뒤 관계가 보이게 남겨야 한다**

   ```text
   PANEL_PWR_EN panel=cluster state=1
   PANEL_BRIDGE_EN panel=cluster state=1
   PANEL_TIMING_EN panel=cluster state=1
   RVGC_FLUSH seq=118 display_slot=0 result=OK
   ```

   또는 반대로 아래처럼 나오면 원인은 훨씬 좁아진다.

   ```text
   RVGC_FLUSH seq=118 display_slot=0 result=OK
   PANEL_TIMING_EN panel=cluster state=0
   ```

   이 경우는 userspace/HWC가 아니라 **flush 이후에도 panel side enable chain이 완성되지 않은 것** 이다.

5. **10분 체크리스트는 네 묶음으로 끝낸다**

   - `RVGC_FLUSH` / `RVGC_COMPLETE` 존재 확인
   - target vs other panel `VBLANK_SUMMARY` 비교
   - `PANEL_PWR_EN` / `BRIDGE_EN` / `TIMING_EN` 순서 확인
   - `DISPLAY_GET_INFO` vs 실제 panel mode/timing 비교

   이 네 묶음으로도 안 갈리면 그때서야 PHY, lane swap, 실제 panel HW fault 쪽으로 내려가는 편이 맞다.

## 리스크

- 반대 panel VBLANK를 안 보면 misroute를 panel dead로 오판하기 쉽다.
- enable 로그 없이 `flush ok` 만 믿으면 scanout 이후 절단면을 놓친다.
- `width/height` 만 보고 `pitch`·format·clock 비교를 생략하면 timing mismatch를 오래 못 잡는다.
- target/other panel 비교 창을 같은 500ms 윈도로 맞추지 않으면 로그 해석이 흔들린다.

## 다음 액션

다음 글에서는 Day71 체크리스트를 이어서, **`DISPLAY_GET_INFO`/panel mode/pixel format mismatch가 실제 black screen으로 번지는 경로** 를 `stride`, bpp, timing 관점에서 더 좁혀 보겠다.

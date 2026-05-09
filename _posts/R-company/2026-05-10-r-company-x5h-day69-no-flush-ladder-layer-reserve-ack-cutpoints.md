---
title: R사 X5H Day69 - no-flush 가지 좁히기: layer reserve와 backend ACK 절단면
author: JaeHa
date: 2026-05-10 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day69, display, no-flush, rvgc, layer-reserve, bringup]
---

Day68에서 운전석 black screen을 `routing mismatch / panel dead / no-flush` 로 먼저 잘랐다면, 그다음 가장 빨리 해야 할 일은 `no-flush` 가지를 더 짧게 자르는 것이다. 이 경우 현장에서 자주 헷갈리는 포인트는 panel이 죽은 게 아니라, **RVGC backend가 flush까지 갈 자격을 주지 않은 상태** 인데도 상단 UI만 보고 panel 문제처럼 받아들이는 패턴이다.

## 핵심 요약

- `no-flush` 는 panel fault보다 먼저 **`DISPLAY_INIT`/`LAYER_RESERVE`/`LAYER_SET_*` 선행 단계 실패** 를 의심하는 편이 맞다.
- backend는 `s_validate_input_params()` 에서 **display initialized / layer 범위 / layer initialized** 를 모두 통과해야만 `set_addr`·`set_size`·`set_fmt` 를 받는다.
- `LAYER_RESERVE` 의 실체는 단순 bookkeeping이 아니라 `R_WM_SurfaceGet()` + `PosZ` 기반 HW layer 선점이다. 여기서 실패하면 이후에는 flush가 안 나와도 이상한 게 아니다.
- 그래서 `UI_ROUTE` 는 있는데 `RVGC_FLUSH` 가 없을 때의 1차 절단 순서는 **reserve 성공 여부 → set_fmt/set_size/set_addr 성공 여부 → flush 호출 여부** 가 가장 빠르다.

## 코드 포인트

1. **backend는 initialized layer가 아니면 다음 단계 자체를 막는다**

   `r_disfwk_rvgc_drv.c` 의 검증 함수는 생각보다 엄격하다.

   ```c
   if (disp_inst->initialized == false) { ... }
   if (input_layer >= disp_inst->num_layers) { ... }
   if (layer->intialized == false) { ... }
   ```

   즉 `UI_ROUTE(local:0)` 가 보이더라도 `LAYER_RESERVE` 가 아직 안 끝났으면, 그 뒤의 `LAYER_SET_ADDR`/`SIZE`/`FMT` 는 backend 입장에서 전부 invalid다. 현상은 black screen이지만 실제 절단면은 panel 이전이다.

2. **`LAYER_RESERVE` 는 HW layer 점유 단계다**

   reserve 경로는 surface를 하나 받고, `PosZ` 로 원하는 layer를 지정한 뒤 property set으로 확정한다.

   ```c
   wm_ret = R_WM_SurfaceGet(disp_inst->disp, surf);
   layer->surf.PosZ = in->layer;
   wm_ret = R_WM_SurfacePropertySet(disp_inst->disp, R_WM_SURF_PROP_POS, surf);
   ```

   코드 주석도 이 실패를 사실상 **"원하는 HW layer가 이미 taken"** 인 경우로 해석한다. 그래서 `no-flush` 상황에서 reserve 실패가 숨어 있으면, HWC 쪽은 frame을 만들고 있어도 backend는 scanout 자원을 못 잡은 상태일 수 있다.

3. **flush 전 단계는 상태 누적식이라 하나만 틀려도 마지막 flush가 의미를 잃는다**

   backend는 주소·크기·포맷을 각각 따로 받는다.

   ```c
   surf->FBuf = in->paddr;
   wm_ret = R_WM_SurfacePropertySet(..., R_WM_SURF_PROP_BUFFER, surf);

   surf->Width = in->size_w;
   surf->Height = in->size_h;
   surf->StrideY = in->stride;
   wm_ret = R_WM_SurfacePropertySet(..., R_WM_SURF_PROP_SIZE, surf);

   surf->Fmt = in->format;
   wm_ret = R_WM_SurfacePropertySet(..., R_WM_SURF_PROP_COLORFMT, surf);
   ```

   여기서 하나라도 `NG` 면 panel fault를 볼 단계가 아니다. bring-up 초기에는 `RVGC_FLUSH` 유무만 보지 말고, 바로 직전 `set_fmt/set_size/set_addr` return을 같은 seq로 묶어 두는 편이 훨씬 낫다.

4. **실제 flush는 마지막에 `R_WM_ScreenSurfacesFlush()` 한 번으로 닫힌다**

   flush 구현은 의외로 단순하다.

   ```c
   wm_ret = R_WM_ScreenSurfacesFlush(disp_inst->disp, in->blocking);
   if (R_WM_ERR_SUCCESS != wm_ret) { ... }
   ```

   그래서 `no-flush` 는 두 경우로 다시 나뉜다.
   - frontend가 여기까지 아예 못 내려옴
   - 내려왔지만 reserve/set 단계가 앞에서 깨져 flush 호출 직전 탈락함

   현장에서는 이 둘을 분리하지 않으면 SurfaceFlinger/HWC를 오래 뒤지게 된다.

5. **실전 로그는 reserve 중심으로 최소 세 줄이면 충분하다**

   ```text
   RVGC_RESERVE seq=91 display_slot=0 layer=0 result=OK
   RVGC_SET_SIZE seq=91 display_slot=0 layer=0 w=1920 h=1080 stride=7680 result=OK
   RVGC_FLUSH seq=91 display_slot=0 blocking=1 result=OK
   ```

   반대로 아래 조합이면 panel 문제로 가면 안 된다.

   ```text
   UI_ROUTE seq=91 ui_target=local:0 package=com.android.systemui
   RVGC_RESERVE seq=91 display_slot=0 layer=0 result=NG reason=layer_busy
   RVGC_FLUSH seq=91 missing
   ```

   이 패턴의 본질은 black screen이 아니라 **backend layer admission 실패** 다.

## 리스크

- `LAYER_RESERVE` 실패를 기록하지 않으면 `no-flush` 가 HWC 무출력처럼 보여 원인 축이 잘못 잡힌다.
- `set_addr/set_size/set_fmt` 중 한 단계만 깨져도 panel dead처럼 보일 수 있다.
- display mapping이 맞아 보여도 reserve된 layer가 없으면 scanout 경로는 실제로 비어 있다.
- flush 로그만 마지막에 추가하면, 이미 앞단에서 잘린 commit을 놓쳐 같은 black screen을 반복 분석하게 된다.

## 다음 액션

다음 글에서는 `no-flush` 반대편인 **`flush는 성공했는데 driver panel VBLANK가 비는 경우`** 를 이어서, CR52 display enable 순서·route·final scanout 절단면으로 더 좁혀 보겠다.

---
title: R사 X5H Day68 - Driver panel black screen decision tree
author: JaeHa
date: 2026-05-09 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day68, display, black-screen, routing, rvgc, vblank, triage]
---

Day67에서 `ui_target`/`flush_target`/`vblank_source` 를 같은 사건 번호로 묶는 SOP를 만들었다면, 이제 현장에서는 그 타임라인으로 **5분 안에 무엇을 결론낼지** 가 중요하다. X5H bring-up에서 운전석 panel black screen은 증상은 같아 보여도 실제로는 `routing mismatch`, `panel dead`, `no-flush` 세 갈래로 먼저 잘라야 복구 축이 짧아진다.

## 핵심 요약

- driver panel black screen은 먼저 **UI route 존재 여부 → backend flush 존재 여부 → 해당 panel VBLANK 존재 여부** 순서로 본다.
- `UI_ROUTE(local:1)` 와 `RVGC_FLUSH(display_slot=0)` 가 엇갈리면 `routing mismatch` 다.
- `RVGC_FLUSH(display_slot=0)` 는 반복되는데 `VBLANK_DISPLAY0` 만 비어 있으면 `panel dead / final scanout dead` 쪽을 먼저 본다.
- `UI_ROUTE` 는 있는데 `RVGC_FLUSH` 자체가 없으면 SurfaceFlinger/HWC/driver commit chain 이전 절단면, 즉 `no-flush` 로 분류하는 편이 맞다.

## 코드 포인트

1. **첫 분기점은 UI가 어디로 라우팅됐는지다**

   Android 쪽 최소 증거는 Day67 스키마 그대로면 충분하다.

   ```text
   UI_ROUTE seq=77 ui_target=local:0 package=com.android.systemui
   UI_ROUTE seq=78 ui_target=local:1 package=com.android.car.carlauncher
   ```

   driver panel black screen 상황에서 `local:1` 쪽 이벤트만 있고 `local:0` 에는 launcher/decor가 없다면, 처음부터 driver UX가 다른 panel로 간 가능성을 먼저 열어야 한다.

2. **둘째 분기점은 commit이 실제 backend flush까지 내려왔는지다**

   `disfwk_fe` 경로의 최종 truth source는 여전히 flush 호출이다.

   ```c
   display_idx = rvgc_pipe->display_mapping;
   ret = rvgc_taurus_display_flush(rcrvgc, display_idx, ...);
   ```

   현장 로그는 아래처럼 남기는 편이 가장 빠르다.

   ```text
   RVGC_FLUSH seq=78 display_slot=0 plane=1 fence=442
   ```

   `UI_ROUTE` 는 있는데 같은 `seq` 주변에 `RVGC_FLUSH` 가 아예 없다면, panel 자체보다 **HWC atomic path / plane programming / flush trigger 누락** 을 먼저 봐야 한다.

3. **셋째 분기점은 flush 대상 panel의 VBLANK 생존 여부다**

   ```c
   RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY0
   RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY1
   ```

   실전에서는 raw interrupt보다 요약 로그가 낫다.

   ```text
   RVGC_VBLANK_SUMMARY window=500ms display_slot=0 count=0 last_seq=78
   RVGC_VBLANK_SUMMARY window=500ms display_slot=1 count=31 last_seq=78
   ```

   이 조합이면 flush는 0번으로 가지만 실제 refresh는 1번만 뛰는 것이므로, userspace 무출력보다 **panel enable 순서 / backend numbering / scanout route** 를 먼저 의심해야 한다.

4. **결정 트리는 세 줄 비교로 끝내는 편이 좋다**

   - `ui_target != flush_target` → `routing mismatch`
   - `ui_target == flush_target` 이고 `flush_target != vblank_source` → `panel dead` 또는 final scanout 문제 우선
   - `ui_target` 은 있는데 `flush_target` 로그 부재 → `no-flush`

   여기서 `panel dead` 는 패널 전원만 뜻하지 않는다. timing, bridge, enable 순서, CR52/backend display route 누락까지 포함한 **final visible output dead** 로 해석하는 편이 실무에 맞다.

5. **오판 방지용 보조 증거도 같이 묶어둔다**

   `display_settings.xml` 기준 `local:1` 은 secondary decor/IME 흔적이 남기 쉽다. 그래서 아래 보조 증거가 있으면 `routing mismatch` 쪽 확신이 빨리 올라간다.

   ```text
   UI_DECOR seq=78 ui_target=local:1 decor=system ime=enabled
   PASSENGER_PANEL_OBS seq=78 launcher=visible picker=visible
   ```

   반대로 두 panel 모두 decor 흔적이 없고 flush도 없다면, panel fault보다 상단 composition 절단면일 가능성이 크다.

## 리스크

- driver panel만 육안으로 보고 passenger panel 관측을 빼먹으면 `routing mismatch` 를 `panel dead` 로 오판할 수 있다.
- `RVGC_FLUSH` 존재 여부를 분리하지 않으면 HWC/driver commit 누락과 panel scanout 문제를 같은 black screen으로 뭉개게 된다.
- VBLANK raw 이벤트만 길게 모으면 타임라인 비교가 어려워져 오히려 결정 시간이 늦어진다.
- `panel dead` 를 물리 패널 고장으로만 좁게 해석하면, 실제 backend numbering/enable 순서 문제를 놓칠 수 있다.

## 다음 액션

다음 글에서는 Day68 decision tree를 이어서, **`no-flush` 가지에 한정해 HWC atomic commit·plane reserve·RVGC ACK/COMPLETE 절단면을 어디서 먼저 볼지** 를 더 좁혀 보겠다.

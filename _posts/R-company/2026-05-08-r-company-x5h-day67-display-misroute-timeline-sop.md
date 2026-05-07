---
title: R사 X5H Day67 - Display misroute 타임라인 로그 SOP
author: JaeHa
date: 2026-05-08 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day67, display, multidisplay, rvgc, vblank, launcher, timeline, sop]
---

Day66에서 `launcher 위치 + flush target + VBLANK source` 를 같이 보라고 정리했다면, 이번에는 그 세 축을 **같은 사건 번호와 같은 시간축으로 묶는 방법** 을 정리해야 한다. X5H display bring-up에서 시간을 가장 많이 태우는 경우는 black screen 자체가 아니라, Android logcat·RVGC backend·패널 관측 메모가 서로 다른 시간축에 흩어져 있어서 원인 절단이 늦어지는 상황이다.

## 핵심 요약

- display misroute는 단일 로그 한 줄로 닫히지 않는다. **`UI target` / `flush target` / `physical refresh`** 를 한 타임라인으로 묶어야 한다.
- 최소 공통 키는 `boot_id`, `display_slot`, `seq` 3개면 충분하다.
- `local:0|1` 과 RVGC `display_mapping=0|1`, `VBLANK_DISPLAY0|1` 를 같은 `display_slot` 으로 정규화하면 black-screen과 misroute를 바로 분리할 수 있다.
- bring-up 초기에 이 규칙이 없으면, 정상 렌더링된 secondary panel 증거가 있어도 SurfaceFlinger/HWC failure로 잘못 escalate된다.

## 코드 포인트

1. **타임라인 공통 키를 먼저 고정한다**

   추천 최소 스키마는 아래처럼 잡는다.

   ```text
   boot_id=<uuid>
   seq=<monotonic counter>
   display_slot=<0|1>
   ui_target=<local:0|local:1>
   flush_target=<0|1>
   vblank_source=<0|1>
   ```

   여기서 핵심은 component마다 자기 용어를 그대로 쓰지 말고, 최종 분석용으로는 `display_slot` 하나에 맵핑하는 것이다.

2. **Android 쪽은 UI target 관측점을 남긴다**

   최소한 아래 이벤트는 한 줄씩 남겨야 한다.

   ```text
   UI_ROUTE seq=41 ui_target=local:1 package=com.android.car.carlauncher
   UI_DECOR seq=42 ui_target=local:1 decor=system ime=enabled
   ```

   이 두 줄이 있으면, "아무것도 안 그려짐" 과 "secondary panel로 정상 라우팅됨" 을 바로 가를 수 있다.

3. **RVGC 쪽은 flush 호출 시점에 display index를 붙인다**

   Day65~66에서 본 핵심 backend truth source는 여전히 `display_mapping` 이다.

   ```c
   rvgc_pipe->display_mapping = index;
   ret = rvgc_taurus_display_flush(rcrvgc, rvgc_pipe->display_mapping, ...);
   ```

   따라서 flush 로그는 적어도 아래 형태가 되어야 한다.

   ```text
   RVGC_FLUSH seq=42 display_slot=0 plane=1 fb=0x8c000000 fence=391
   ```

   여기서 `seq` 를 Android route 이벤트와 맞출 수 있으면, UI가 `local:1` 로 향했는데 backend flush는 `display_slot=0` 으로 가는지 즉시 보인다.

4. **물리 panel 생존 판정은 VBLANK를 마지막 truth로 둔다**

   ```c
   RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY0
   RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY1
   ```

   VBLANK 로그는 noise가 많으니 raw dump보다 구간 요약이 낫다.

   ```text
   RVGC_VBLANK_SUMMARY window=500ms display_slot=1 count=29 last_seq=42
   ```

   이렇게 요약하면 flush는 0번으로 가는데 실제 refresh는 1번만 뛰는 상황을 바로 잡아낼 수 있다.

5. **판정 규칙은 세 가지로 단순화한다**

   - `ui_target == flush_target == vblank_source` → routing 정상, 다른 원인 추적
   - `ui_target != flush_target` → Android ↔ backend mapping mismatch
   - `flush_target != vblank_source` → backend numbering/panel wiring/enable 순서 문제 우선

   이 규칙을 SOP로 고정하면, black screen 보고가 들어와도 5분 안에 escalation 방향을 정할 수 있다.

## 리스크

- `seq` 없이 시간 문자열만 비교하면 logcat, kernel, RTOS 로그 clock drift 때문에 오판하기 쉽다.
- VBLANK raw event를 전부 저장하면 bring-up 중 log volume이 커져 정작 flush/UI route 이벤트를 놓칠 수 있다.
- Android `local:N` 과 backend panel 번호를 별도 표기로 방치하면, 팀마다 "0번 화면" 의미가 달라져 재현 회의가 길어진다.
- 이 규칙 없이 증상 캡처만 공유하면, 같은 misroute를 팀마다 black screen / HWC fail / panel dead로 다르게 부르게 된다.

## 다음 액션

다음 글에서는 Day67의 타임라인 SOP를 바탕으로, **driver panel black screen을 5분 안에 `routing mismatch / panel dead / no-flush` 세 갈래로 자르는 decision tree** 를 정리하겠다.

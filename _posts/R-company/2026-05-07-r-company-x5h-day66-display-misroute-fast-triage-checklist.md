---
title: R사 X5H Day66 - Display misroute 빠른 트리아지 체크리스트
author: JaeHa
date: 2026-05-07 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day66, display, multidisplay, rvgc, vblank, bringup, triage]
---

Day65에서 `local:0`/`local:1` 과 RVGC `display_mapping` mismatch가 **정상 flush + 다른 화면 출력** 을 만든다는 점을 봤다면, 다음 bring-up 포인트는 그걸 **몇 분 안에 잘라내는 관측점** 을 갖추는 것이다. X5H에서는 Android boot failure처럼 보이는 black screen도, 실제로는 launcher·system decor·VBLANK가 전부 passenger panel로 가 있는 경우가 있다.

## 핵심 요약

- misroute triage의 1차 질문은 "안 그렸나?" 가 아니라 **"어느 display가 실제로 살아 있나?"** 다.
- Android 쪽에서 `local:1` 은 secondary home/system decor 대상이고, CarService overlay는 `displayPort=1` 을 별도 occupant zone으로 본다.
- RVGC 쪽에서는 `display_mapping` 과 `RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY0/1` 이 물리 panel 관측점이므로, **launcher 위치 + VBLANK source + flush target** 을 함께 봐야 한다.
- bring-up 초기에 이 세 축을 동시에 보지 않으면, display routing 문제를 SurfaceFlinger/HWC crash로 오판하기 쉽다.

## 코드 포인트

1. **Android 증상은 launcher 위치로 가장 빨리 갈린다**

   Day65에서 확인한 overlay는 secondary display를 별도 UX target으로 만든다.

   ```xml
   <string-array name="config_displayUniqueIdArray" translatable="false">
       <item>"local:0"</item>
       <item>"local:1"</item>
   </string-array>
   <string name="config_secondaryHomePackage" translatable="false">com.android.car.carlauncher</string>
   ```

   따라서 bring-up 직후 **driver panel은 검은데 passenger panel에 car launcher나 system decor가 보이면**, UI stack이 죽은 게 아니라 display routing이 틀린 가능성이 더 크다.

2. **CarService zone mapping은 misroute를 사용자 증상으로 증폭한다**

   ```xml
   <item>displayPort=0,displayType=MAIN,occupantZoneId=0,...</item>
   <item>displayPort=1,displayType=MAIN,occupantZoneId=1,...</item>
   ```

   `displayPort=0`/`1` 이 occupant zone까지 갈라놓기 때문에, wrong panel output은 단순 미러링 문제가 아니다. user picker, home, IME, decor가 **아예 다른 좌석 UX로 분기** 된다.

3. **`display_settings.xml` 성격상 `local:1` 에 흔적이 더 많이 남는다**

   기존 분석 기준 `local:1` 은 `shouldShowSystemDecors`, `shouldShowIme`, `forcedDensity=160` 같은 별도 정책을 받는다. 그래서 bring-up 때는 "아무것도 안 보인다" 보다 **"보조 화면에만 decor/IME 흔적이 남는다"** 가 더 강한 misroute signal이다.

4. **backend truth source는 여전히 RVGC display index와 VBLANK다**

   ```c
   rvgc_pipe->display_mapping = index;
   ...
   ret = rvgc_taurus_display_flush(rcrvgc, rvgc_pipe->display_mapping, ...);
   ```

   ```c
   #define RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY0 ...
   #define RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY1 ...
   ```

   flush target이 `display_mapping=0` 인데 살아 있는 VBLANK가 `DISPLAY1` 쪽에서만 관측되면, panel numbering/pipe enable 순서/backend mapping 중 하나가 엇갈렸다고 보는 편이 빠르다.

5. **가장 빠른 현장 체크 순서는 세 줄이면 된다**

   - **화면 관측:** launcher/system decor/user picker가 어느 panel에 보이는가
   - **flush 관측:** `display_mapping` 이 몇 번으로 호출됐는가
   - **event 관측:** `VBLANK_DISPLAY0/1` 중 어느 쪽이 주기적으로 살아 있는가

   이 셋이 서로 같은 번호를 가리키지 않으면, userspace boot failure 추적 전에 routing mismatch부터 닫아야 한다.

## 리스크

- driver panel black screen만 보고 판단하면, 실제로는 passenger panel에 정상 부팅된 상태를 놓칠 수 있다.
- VBLANK source를 보지 않으면 "flush 성공 후 panel dead" 와 "flush가 다른 panel로 감" 을 구분하지 못한다.
- occupant zone 분기까지 걸린 상태에서 panel mapping이 틀어지면, 단순 display bug가 아니라 UX/product 결함으로 바로 번진다.
- HWC3/SurfaceFlinger 로그만 과하게 좇으면, backend numbering mismatch 같은 더 낮은 원인을 오래 놓칠 수 있다.

## 다음 액션

다음 글에서는 Day66을 이어서, **display index·VBLANK·launcher 위치를 한 타임라인에 묶는 로그 규칙** 을 정리해 black-screen triage를 한 번에 재현 가능한 SOP 형태로 좁히겠다.

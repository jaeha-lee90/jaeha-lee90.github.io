---
title: R사 X5H Day65 - Android multi-display routing과 RVGC display index mismatch
author: JaeHa
date: 2026-05-06 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day65, android, multi-display, hwc3, rvgc, display-index, bringup]
---

Day64에서 commit chain의 ACK/COMPLETE/VBLANK 절단면을 봤다면, 다음 bring-up 핵심은 **그 commit이 의도한 화면으로 갔는지** 를 확인하는 것이다. X5H 경로에서는 backend flush 자체는 성공해도, Android multi-display policy의 `local:0`/`local:1` 과 RVGC의 `display_mapping` 이 어긋나면 **정상 commit + 다른 화면 출력** 이라는 까다로운 패턴이 나온다.

## 핵심 요약

- Android 쪽 기본 레이아웃은 `address 0 -> defaultDisplay`, `address 1 -> passenger_display` 로 고정돼 있다.
- CarService overlay도 `displayPort=0` 을 driver zone, `displayPort=1` 을 passenger zone에 매핑하므로, **logical display routing이 이미 userspace에서 갈라진다**.
- `disfwk_fe` 는 pipe 생성 시 `rvgc_pipe->display_mapping = index` 로 RVGC display index를 결정하므로, **kernel/backend index와 Android logical address가 1:1로 맞는지** 반드시 검증해야 한다.
- 증상이 black screen처럼 보여도 실제로는 **화면이 꺼진 게 아니라 다른 panel로 성공적으로 그려진 것** 일 수 있다.

## 코드 포인트

1. **Android display layout은 시작부터 0번/1번 역할을 분리한다**

   `display_layout_configuration.xml` 은 default display와 passenger display를 명시적으로 나눈다.

   ```xml
   <display enabled="true" defaultDisplay="true">
     <address>0</address>
   </display>

   <display enabled="true" defaultDisplay="false" displayGroup="passenger_display">
     <address>1</address>
   </display>
   ```

   즉 SurfaceFlinger/HWC3 위에서 같은 frame을 그려도, 어느 logical display에 붙였는지가 먼저 갈린다.

2. **CarService occupant mapping도 display port 기준으로 분기한다**

   overlay 설정은 zone assignment를 이렇게 둔다.

   ```xml
   <item>displayPort=0,displayType=MAIN,occupantZoneId=0,...</item>
   <item>displayPort=1,displayType=MAIN,occupantZoneId=1,...</item>
   ```

   여기서 0번은 driver, 1번은 passenger다. 그래서 앱/런처가 정상 구동돼도 port mapping이 엇갈리면 **운전석 기준 black screen + 조수석에만 UI 표시** 같은 현상이 생길 수 있다.

3. **Framework overlay도 built-in display unique id를 `local:0`, `local:1` 로 고정한다**

   ```xml
   <string-array name="config_displayUniqueIdArray" translatable="false">
       <item>"local:0"</item>
       <item>"local:1"</item>
   </string-array>
   <string name="config_secondaryHomePackage" translatable="false">com.android.car.carlauncher</string>
   ```

   여기에 `display_settings.xml` 까지 합쳐지면 `local:1` 은 secondary home/system decor 대상이 된다. 즉 Android는 이미 **두 번째 화면을 별도 UX target** 으로 취급한다.

4. **RVGC backend는 별도 규칙으로 display index를 잡는다**

   `disfwk_fe-kms.c` 에서는 pipe 생성 시 index를 그대로 backend display 번호로 사용한다.

   ```c
   /* mode1 is match with display 1 and display_mapping 0 */
   /* mode2 is match with display 2 and display_mapping 1 */
   ...
   rvgc_pipe->display_mapping = index;
   ```

   또 atomic update/flush도 이 값을 그대로 쓴다.

   ```c
   display_idx = rvgc_pipe->display_mapping;
   ret = rvgc_taurus_layer_set_layer(rcrvgc, display_idx, hw_plane, ...);
   ret = rvgc_taurus_display_flush(rcrvgc, rvgc_pipe->display_mapping, ...);
   ```

   즉 Android `local:1` 이 실제 backend `display 1` 으로 간다는 보장이 자동으로 생기지 않는다. 부팅 인자, pipe enable 순서, backend numbering이 조금만 틀어져도 다른 panel로 간다.

5. **VBLANK도 display별로 따로 올라오므로 잘못된 panel로 간 commit은 성공처럼 보일 수 있다**

   protocol은 display별 이벤트를 분리한다.

   ```c
   #define RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY0 ...
   #define RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY1 ...
   ```

   따라서 `DISPLAY_FLUSH` COMPLETE와 VBLANK가 모두 정상이어도, 그 signal이 의도한 logical display에 대응하는지 확인하지 않으면 triage가 어긋난다.

## 리스크

- Android `displayPort`/`local:N` 정책과 RVGC `display_mapping` 이 어긋나면 **정상 flush + wrong panel output** 을 black screen으로 오판할 수 있다.
- secondary home/user picker가 `local:1` 에만 뜨는 구성을 모른 채 driver display만 보면, userspace boot failure로 잘못 결론내리기 쉽다.
- `mode0~mode9` enable 순서와 backend display numbering이 고정돼 있지 않으면, 동일 이미지라도 부팅 옵션에 따라 panel mapping이 달라질 수 있다.
- ACK/COMPLETE/VBLANK가 정상이라는 이유만으로 panel routing까지 정상이라고 가정하면 디버깅이 오래 돈다.

## 다음 액션

다음 글에서는 Day65를 이어서, **display 0/1 misroute를 실제 bring-up에서 가장 빨리 가르는 관측점(launcher 위치, user picker, VBLANK source, flush 로그)** 을 체크리스트 형태로 정리하겠다.

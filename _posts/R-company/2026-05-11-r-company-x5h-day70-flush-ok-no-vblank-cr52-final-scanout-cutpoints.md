---
title: R사 X5H Day70 - flush 성공 후 no-VBLANK: CR52 enable 순서와 final scanout 절단면
author: JaeHa
date: 2026-05-11 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day70, display, flush, vblank, cr52, scanout, bringup]
---

Day69에서 `no-flush` 가지를 `reserve/set/flush` 전 단계로 잘랐다면, 그 반대편에서 가장 먼저 봐야 할 패턴은 **`RVGC_FLUSH` 는 성공하는데 driver panel 쪽 `VBLANK_DISPLAY0` 가 비는 경우** 다. 이때는 Android/HWC보다 아래, 즉 **CR52 display enable 순서·backend display route·final scanout 활성화** 쪽을 먼저 의심하는 편이 bring-up 시간을 훨씬 줄인다.

## 핵심 요약

- `RVGC_FLUSH result=OK` 와 `COMPLETE` 가 반복되는데 목표 panel의 `VBLANK_DISPLAYN` 이 비면, 상단 composition보다는 **final visible output path** 문제로 보는 편이 맞다.
- Day64 기준 `DISPLAY_FLUSH -> ACK -> COMPLETE` 는 command lane 성공을 뜻할 뿐, **실제 refresh 생존** 까지 보장하지는 않는다.
- Day65~68에서 이미 본 것처럼 `display_mapping` 과 살아 있는 `VBLANK_DISPLAYN` 이 어긋나면, panel dead처럼 보여도 실체는 **backend numbering/enable 순서 mismatch** 일 수 있다.
- 현장 1차 절단 순서는 **flush target 확인 → COMPLETE 확인 → target panel VBLANK 확인 → 다른 panel VBLANK 존재 여부 확인** 이 가장 빠르다.

## 코드 포인트

1. **flush COMPLETE는 scanout 생존 증거가 아니다**

   Day64에서 정리한 commit chain은 여기서 닫힌다.

   ```c
   ret = rvgc_taurus_display_flush(rcrvgc, display_idx, ...);
   ret = wait_for_completion_interruptible(&event->ack);
   ret = wait_for_completion_interruptible(&event->completed);
   ```

   이 조합이 모두 성공해도 보장되는 것은 **CR52/backend가 명령을 받아 작업 완료를 회신했다** 는 사실뿐이다. panel timing enable, route select, 실제 refresh 발생은 그 다음 층이다. 그래서 `flush ok + black screen` 이면 userspace를 더 파기보다 VBLANK와 route를 먼저 붙여 봐야 한다.

2. **최종 truth source는 여전히 display별 VBLANK다**

   protocol은 display event를 분리해 둔다.

   ```c
   RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY0
   RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY1
   ```

   bring-up에서 중요한 건 raw interrupt 개수보다 **flush 대상과 같은 번호의 VBLANK가 살아 있는지** 다. 예를 들어 `display_slot=0` 으로 flush가 성공했는데 500ms 창에서 `DISPLAY0` 만 0회라면, panel 전단 scanout이 실제로 안 돌고 있다고 보는 편이 맞다.

3. **다른 panel의 VBLANK가 살아 있으면 panel dead보다 misroute를 먼저 의심한다**

   Day65~67의 핵심은 backend가 pipe 생성 시 `display_mapping` 을 독자적으로 쓴다는 점이었다.

   ```c
   rvgc_pipe->display_mapping = index;
   ret = rvgc_taurus_display_flush(rcrvgc, rvgc_pipe->display_mapping, ...);
   ```

   따라서 아래 패턴은 물리 panel 고장으로 바로 가면 안 된다.

   ```text
   RVGC_FLUSH seq=104 display_slot=0 result=OK
   RVGC_COMPLETE seq=104 display_slot=0 result=OK
   RVGC_VBLANK_SUMMARY window=500ms display_slot=0 count=0
   RVGC_VBLANK_SUMMARY window=500ms display_slot=1 count=31
   ```

   이 경우는 `display_slot=0` 에 그린다고 믿었지만 실제 살아 있는 refresh는 1번이라는 뜻이므로, **CR52 route / panel numbering / enable order** 를 먼저 좁히는 편이 빠르다.

4. **`DISPLAY_GET_INFO` 와 실제 panel 기대치가 어긋나면 flush 이후에도 빈 화면이 날 수 있다**

   Day64에서 본 capability 응답은 여전히 중요하다.

   ```c
   uint32_t width;
   uint32_t height;
   uint32_t pitch;
   uint32_t layers;
   ```

   flush 전 단계가 모두 `OK` 여도 backend가 믿는 width/height/pitch와 panel bring-up 쪽 timing/format 기대가 어긋나면, final scanout은 정상 refresh로 못 이어질 수 있다. 즉 Day69처럼 `no-flush` 는 아니지만, 그렇다고 panel glass fault로 단정해도 안 된다.

5. **실전 로그는 enable 순서 판단이 되게 네 줄로 묶는 편이 좋다**

   ```text
   UI_ROUTE seq=104 ui_target=local:0
   RVGC_FLUSH seq=104 display_slot=0 result=OK
   RVGC_COMPLETE seq=104 display_slot=0 result=OK
   RVGC_VBLANK_SUMMARY window=500ms display_slot=0 count=0
   ```

   여기에 반대편 panel 관측을 하나 더 붙이면 좋다.

   ```text
   RVGC_VBLANK_SUMMARY window=500ms display_slot=1 count=31
   ```

   이 다섯 줄이면 적어도 `userspace no-flush` 와 `final scanout dead/misroute` 는 거의 바로 갈린다.

## 리스크

- `COMPLETE` 를 panel 표시 성공으로 오해하면, CR52/backend route 문제를 Android/HWC 버그로 오래 추적하게 된다.
- target panel VBLANK만 안 보고 전체 black screen만 보면, 실제로는 다른 panel로 정상 출력 중인 misroute를 놓칠 수 있다.
- `DISPLAY_GET_INFO` 와 실제 panel timing 기대치를 비교하지 않으면, flush 이후 blank를 panel HW fault로 과도하게 몰 수 있다.
- enable 순서 로그 없이 육안 관측만 하면 재현성 없는 "가끔 켜짐" 문제를 구조적으로 설명하기 어렵다.

## 다음 액션

다음 글에서는 Day70을 이어서, **flush ok / no-vblank 가지에서 CR52 route·panel enable·timing을 10분 안에 자를 수 있는 현장 체크리스트** 로 더 압축해 보겠다.

---
title: R사 X5H Day64 - RVGC plane commit 경로와 black-screen signature 분리
author: JaeHa
date: 2026-05-05 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day64, rvgc, disfwk_fe, rpmsg, black-screen, bringup]
---

Day63에서 DomD/CR52/RPMsg hand-off 절단면을 봤다면, 다음 bring-up 핵심은 **"commit이 어디까지 갔는지"** 를 분리하는 것이다. X5H display 경로에서는 GSX/HWC 쪽에서 frame 생산이 정상이어도, `disfwk_fe` 의 RVGC commit chain이 ACK/COMPLETE/VBLANK 어느 단계에서 끊기느냐에 따라 black screen의 원인이 완전히 달라진다.

## 핵심 요약

- `disfwk_fe-taurus.c` 기준 RVGC commit은 **`LAYER_SET_*`/`LAYER_SET_LAYER` → `DISPLAY_FLUSH` → RPMsg ACK → RPMsg COMPLETE** 순서로 닫힌다.
- `rvgc_taurus_send_command()` 는 `wait_for_completion_interruptible(&event->ack)` 와 `...(&event->completed)` 를 둘 다 기다리므로, **timeout 위치만 구분해도** "채널 죽음 / backend reject / flush 후 panel 미표시" 를 빠르게 나눌 수 있다.
- `DISPLAY_GET_INFO` 의 `width/height/pitch/layers` 는 panel policy보다 먼저 확인해야 할 **backend capability truth source** 다.
- black screen triage는 `display init/get_info 성공 여부 → layer reserve/set addr 성공 여부 → flush ack/complete 여부 → vblank/event 유무` 순서가 가장 빠르다.

## 코드 포인트

1. **RVGC commit은 단일 ioctl 한 번이 아니라 상태 누적 후 flush다**  
   `r_taurus_rvgc_protocol.h` 에는 plane commit 재료가 따로 정의돼 있다.

   ```c
   #define RVGC_PROTOCOL_IOC_LAYER_SET_ADDR
   #define RVGC_PROTOCOL_IOC_LAYER_SET_POS
   #define RVGC_PROTOCOL_IOC_LAYER_SET_SIZE
   #define RVGC_PROTOCOL_IOC_LAYER_SET_FMT
   #define RVGC_PROTOCOL_IOC_LAYER_SET_LAYER
   #define RVGC_PROTOCOL_IOC_DISPLAY_FLUSH
   ```

   즉 black screen이 나와도 바로 `DISPLAY_FLUSH` 만 의심하면 안 된다. 실제로는 **주소/포맷/크기 중 하나가 먼저 틀려서 backend가 빈 layer state를 flush** 했을 수 있다.

2. **driver는 ACK와 COMPLETE를 분리해서 기다린다**  
   `disfwk_fe-taurus.c` 의 핵심 절단면은 여기다.

   ```c
   ret = rpmsg_send(rpdev->ept, cmd_msg, sizeof(struct st_taurus_rvgc_cmd_msg));
   ...
   ret = wait_for_completion_interruptible(&event->ack);
   ...
   ret = wait_for_completion_interruptible(&event->completed);
   ```

   해석은 단순하다.
   - `rpmsg_send` 실패: endpoint/vring/remoteproc attach 전 단계 문제
   - ACK 대기에서 멈춤: CR52 backend가 메시지를 못 받거나 protocol dispatch 전에서 끊김
   - COMPLETE 대기에서 멈춤: backend는 받았지만 실제 display 작업/flush completion이 안 끝남

   이 셋은 모두 현상상 black screen일 수 있지만, 고쳐야 할 레이어는 서로 다르다.

3. **backend가 NACK/에러를 주면 display policy 문제보다 ABI/parameter 문제를 먼저 봐야 한다**  
   각 wrapper는 완료 후 공통으로 아래 판정을 한다.

   ```c
   if ((res_msg->hdr.Result != R_TAURUS_RES_COMPLETE) ||
       (res_msg->u_params.ioc_display_flush.res != 0)) {
       ret = -EIO;
   }
   ```

   같은 패턴이 `layer_reserve`, `layer_set_addr`, `layer_set_fmt`, `display_init` 에도 반복된다. 그래서 `-EIO` 가 보이면 먼저 **CR52 disfwk가 명령을 거절한 이유**를 봐야지, Android 상단 합성기부터 뒤지면 시간만 길어진다.

4. **`DISPLAY_GET_INFO` 는 panel truth source다**  
   protocol 헤더는 backend가 실제로 믿는 display shape를 그대로 돌려준다.

   ```c
   typedef struct st_taurus_rvgc_ioc_display_get_info_out {
       uint64_t cookie;
       uint64_t res;
       uint32_t width;
       uint32_t height;
       uint32_t pitch;
       uint32_t layers;
   } ...
   ```

   Day63의 `display-map/layer-map` 이 맞아 보여도, 여기서 `layers` 수나 `pitch` 기대치가 다르면 **plane reserve는 성공하지만 flush 결과가 빈 화면**으로 갈 수 있다. bring-up 초기에는 XML/HWC policy보다 이 값을 먼저 기준으로 잡는 편이 낫다.

5. **VBLANK event가 오지 않으면 "그리지 못함" 과 "그렸지만 scanout 안 됨" 을 분리할 수 있다**  
   protocol은 display별 VBLANK event를 따로 둔다.

   ```c
   #define RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY0 ...
   #define RVGC_PROTOCOL_EVENT_VBLANK_DISPLAY1 ...
   ```

   flush COMPLETE 까지 왔는데 VBLANK/event가 안 따라오면, RPMsg/command lane보다는 **panel timing, display route, final scanout side** 를 의심하는 편이 맞다.

## 리스크

- `LAYER_SET_ADDR` 만 성공하고 `SET_FMT/SET_SIZE` 가 backend 기대치와 어긋나면 **DMA는 되는데 화면은 비는** 패턴이 나온다.
- ACK 부재와 COMPLETE 부재를 구분하지 않으면 RPMsg chain 문제를 panel 문제로, 혹은 그 반대로 오판하기 쉽다.
- `DISPLAY_GET_INFO` 확인 없이 Android display XML/GSX plane routing만 만지면 **실제 backend capability mismatch** 를 오래 놓칠 수 있다.
- flush 성공 후에도 VBLANK/event가 없으면 CR52 firmware/display route 쪽 문제인데, 이를 userspace composition 문제로 몰아가면 복구 시간이 길어진다.

## 다음 액션

다음 글에서는 Day64를 이어서, **Android multi-display XML/HWC routing과 RVGC display index mismatch가 만드는 "정상 commit + 다른 화면 출력" 패턴**을 정리하겠다.

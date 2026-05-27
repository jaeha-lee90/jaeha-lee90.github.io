---
title: R사 X5H Day87 - sync group·BLASTBufferQueue·unsignaled latch 허용 정책 체크리스트
author: JaeHa
date: 2026-05-28 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day87, display, surfaceflinger, blastbufferqueue, sync, latch, bringup]
---

Day86에서 `configuration relaunch`, `buffer size 재협상`, `first draw resume` 까지 정상이라면, 다음 절단면은 **앱이 새 frame을 만들었는데도 sync group 또는 latch 정책 때문에 실제 표시가 지연되는 경우** 다. X5H 멀티디스플레이 bring-up 후반에는 BLASTBufferQueue, WM sync group, `latch_unsignaled` 정책이 한꺼번에 얽히면서 `buffer queued → transaction ready → latch defer → visible frame` 사이에 짧지만 치명적인 blank gap이 생길 수 있다.

## 핵심 요약

- Day86 이후에도 앱 전환 직후만 blank가 남으면 1차 절단 순서는 **sync group 참가 상태 → BLASTBufferQueue buffer queue → SurfaceFlinger latch defer 이유 → unsignaled 허용 여부** 다.
- `first draw` 로그만으로는 충분하지 않다. buffer가 queue 됐어도 sync group 완료 조건이나 fence signal 대기 때문에 `present` 가 계속 밀릴 수 있다.
- 최소 증적은 `SYNC_GROUP_STATE`, `BLAST_QUEUE`, `LATCH_DECISION`, `FIRST_PRESENT` 네 줄이다.
- 이 구간은 panel/HWC보다는 `WindowManager ↔ BLASTBufferQueue ↔ SurfaceFlinger` 조합의 frame release 정책 문제일 가능성이 높다.

## 코드 포인트

1. **window가 어떤 sync group에 묶였는지 먼저 고정한다**

   ```text
   SYNC_GROUP_STATE display=2 group=Transition#184 member=NavRoot ready=0 reason=waiting_children pending=2
   SYNC_GROUP_STATE display=2 group=Transition#184 member=StatusBar ready=1 pending=1
   ```

   app 메인 window가 준비돼도 같은 transition 그룹의 다른 member가 덜 끝났으면 WM이 transaction apply를 늦출 수 있다. 따라서 blank의 시작점이 app draw 지연인지 group completion 대기인지 먼저 갈라야 한다.

2. **BLASTBufferQueue가 새 buffer를 실제로 받았는지 본다**

   ```text
   BLAST_QUEUE layer=NavRoot buffer_id=0x441 size=1280x480 queued=1 acquire_fence=unsignaled pending_buffers=1
   BLAST_TX layer=NavRoot tx_id=7821 callback_pending=1 sync_group=Transition#184
   ```

   `ViewRootImpl` 첫 draw 후에도 BLAST 쪽에 pending buffer만 쌓이고 callback completion이 안 오면 사용자 입장에서는 여전히 검은 화면이다. queue 여부와 transaction completion을 분리해서 봐야 한다.

3. **SurfaceFlinger가 왜 latch를 미루는지 이유 코드를 잡는다**

   ```text
   LATCH_DECISION display=2 layer=NavRoot frame=6192 latched=0 reason=sync_group_not_ready latch_unsignaled=0
   LATCH_DECISION display=2 layer=NavRoot frame=6193 latched=0 reason=acquire_fence_pending wait_ms=16
   ```

   `sync_group_not_ready`, `acquire_fence_pending`, `desired_present_miss` 는 대응 축이 다르다. reason을 못 고정하면 WM transition 문제를 GPU/render fence 문제로 잘못 몰게 된다.

4. **unsignaled latch 허용 정책이 제품 정책과 맞는지 확인한다**

   ```text
   LATCH_POLICY display=2 layer=NavRoot allow_unsignaled=0 class=app_main debug_override=0
   LATCH_POLICY display=2 layer=ClusterOverlay allow_unsignaled=1 class=critical_overlay
   ```

   cluster/center 정책이 다르면 어떤 layer는 unsignaled latch가 허용되고, 어떤 layer는 반드시 fence signal을 기다린다. bring-up 단계에서 이 정책이 과도하게 보수적이면 첫 전환 frame이 계속 늦어질 수 있다.

5. **첫 present 시점을 최종 판정선으로 묶는다**

   ```text
   FIRST_PRESENT display=2 layer=NavRoot frame=6194 present_fence=signaled present_delay_ms=49
   VISIBLE_SWAP display=2 prev_top=BootGap next_top=NavRoot visible=1
   ```

   queue, draw, latch가 모두 찍혀도 실제 present fence가 늦으면 사용자는 blank를 본다. 최종 완료 판정은 `FIRST_PRESENT` 기준으로 잡아야 WM/SF 로그가 실제 체감과 맞는다.

## 리스크

- sync group 참가 window가 많을수록 unrelated overlay 하나 때문에 메인 app 첫 표시가 같이 지연될 수 있다.
- unsignaled latch 정책을 무작정 완화하면 tearing이나 stale frame 노출로 다른 품질 문제가 생길 수 있다.
- BLAST callback 누락/지연은 간헐 재현이 많아 cold boot와 task switch 양쪽에서 같은 계측점을 유지해야 한다.
- cluster/center display별 transition 정책이 다르면 한쪽만 blank가 재현돼 ownership 분쟁이 길어질 수 있다.

## 다음 액션

다음 글에서는 Day87을 이어서, **sync group과 latch 정책은 정상인데 특정 전환에서만 blank가 남을 때 transaction merge order·leash hierarchy·visibility bit propagation 절단면** 을 정리하겠다.

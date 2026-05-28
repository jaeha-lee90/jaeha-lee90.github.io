---
title: R사 X5H Day88 - transaction merge order·leash hierarchy·visibility bit propagation 체크리스트
author: JaeHa
date: 2026-05-29 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day88, display, windowmanager, surfaceflinger, transaction, leash, visibility, bringup]
---

Day87에서 `sync group`, `BLASTBufferQueue`, `latch_unsignaled` 정책까지 정상이면, 이제 남는 blank는 **transaction 자체는 흘렀지만 merge 순서나 leash hierarchy, visibility bit 전파가 꼬여 최종 show/hide 상태가 뒤집히는 경우** 로 좁혀진다. 특히 전환 애니메이션, task reparent, display-area 이동이 겹치면 app surface는 살아 있는데 leash 쪽 `hidden` 이 늦게 풀려 실제 화면은 계속 비어 보일 수 있다.

## 핵심 요약

- Day87 이후에도 특정 transition에서만 blank가 남으면 1차 절단 순서는 **transaction merge order → leash parent/child 연결 → visibility bit propagation → final show/hide apply** 다.
- `buffer queued`, `latch ok`, `present ok` 만으로는 충분하지 않다. 상위 leash가 `hidden` 이면 하위 app surface가 정상이어도 검게 보인다.
- 최소 증적은 `TX_APPLY_ORDER`, `LEASH_TREE`, `VISIBILITY_PROP`, `FINAL_SHOW_STATE` 네 줄이다.
- 이 구간은 panel/HWC 문제가 아니라 `WindowContainerTransaction ↔ TransitionController ↔ SurfaceControl` 계층 정합 문제일 가능성이 높다.

## 코드 포인트

1. **merge 순서부터 고정한다**

   ```text
   TX_APPLY_ORDER display=2 transition=195 tx=8911 merge_seq=14 source=app_open ops=show,alpha,parent
   TX_APPLY_ORDER display=2 transition=195 tx=8912 merge_seq=15 source=task_reparent ops=hide,parent_crop
   ```

   같은 전환 안에서 `show` transaction 뒤에 오래된 `hide` transaction 이 merge되면 frame은 준비됐어도 마지막 적용 상태가 `hidden` 으로 끝난다. blank가 시작된 시점의 최종 merge winner를 먼저 잡아야 한다.

2. **leash hierarchy가 기대한 parent 체인으로 붙는지 본다**

   ```text
   LEASH_TREE display=2 layer=NavRoot leash=0xb400 parent=Task#72 leash_parent=TransitLeash#195 root=DisplayArea#2
   LEASH_TREE display=2 layer=NavRoot surface=0xa812 reparented=1 old_parent=Task#71 new_parent=Task#72
   ```

   task 이동이나 split/cluster 전환 때 parent leash가 잘못 남으면 `show` 는 app layer에 적용됐는데 실제 visible tree는 다른 branch에 매달려 있을 수 있다. `surface` 와 `leash` parent를 분리해서 봐야 한다.

3. **visibility bit가 어느 단계에서 꺼졌는지 추적한다**

   ```text
   VISIBILITY_PROP display=2 wc=Task#72 requested=1 attached=1 drawn=1 leash_hidden=1 inherited_from=TransitLeash#195
   VISIBILITY_PROP display=2 wc=NavRoot requested=1 committed=1 final_visible=0 reason=ancestor_hidden
   ```

   app window 자체 상태는 `visible` 이어도 ancestor leash 한 단계가 숨겨져 있으면 최종 출력은 0이 된다. `requestedVisible` 과 `finalVisible` 을 한 줄로 묶어야 원인을 빨리 자를 수 있다.

4. **show/hide·alpha·layer stack 최종값을 한 프레임 기준으로 묶는다**

   ```text
   FINAL_SHOW_STATE display=2 layer=NavRoot frame=6208 show=1 alpha=1.0 hidden=0 layer=241 z=top
   FINAL_SHOW_STATE display=2 layer=TransitLeash#195 frame=6208 show=0 alpha=1.0 hidden=1 child_visible=1
   ```

   하위 layer는 정상인데 상위 transition leash가 hide 상태로 남는 패턴이 가장 흔한 함정이다. 최종 판정은 앱 layer 하나가 아니라 visible tree 루트까지 같이 봐야 한다.

5. **전환 종료 시 cleanup transaction이 실제로 수행됐는지 확인한다**

   ```text
   TRANSITION_FINISH display=2 transition=195 finish_tx=8920 cleanup=show_target,remove_leash applied=1
   TRANSITION_FINISH display=2 transition=195 orphan_leash=0 pending_hide=0
   ```

   animation finish callback이 늦거나 cleanup transaction이 유실되면 한 번 생긴 hidden leash가 다음 전환까지 남는다. 그래서 blank가 `특정 앱 진입 직후만` 반복되는 패턴으로 보인다.

## 리스크

- transition merge order를 잘못 해석하면 실제 stale hide transaction을 HWC/panel 문제로 오진해 디버깅 시간이 길어진다.
- leash cleanup을 무리하게 강제하면 애니메이션 중간 프레임 손실이나 flicker가 생길 수 있다.
- multi-display 환경에서는 display-area 이동과 task reparent가 동시에 발생해 center/cluster 중 한쪽만 ancestor-hidden 상태가 남을 수 있다.
- 로그 포인트가 layer 단위에만 있으면 ancestor leash hidden 문제가 보이지 않아 재현성이 낮은 blank로 남는다.

## 다음 액션

다음 글에서는 Day88을 이어서, **merge/leash/visibility는 정상인데도 특정 장면에서만 검게 남을 때 wallpaper·dim layer·color layer·trusted overlay가 topmost를 덮는 절단면** 을 정리하겠다.

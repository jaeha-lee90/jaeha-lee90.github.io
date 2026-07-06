---
title: R사 X5H Day108 - bootanimation 제거·launcher window reveal·color-fade overlay teardown final UI reveal cutline
author: JaeHa
date: 2026-07-07 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day108, display, bootanimation, launcher, colorfade, overlay, reveal, bringup]
---

Day107에서 `BACKLIGHT_UNBLANK_TRACE`, `LOCAL_DIMMING_RELEASE`, `DISPLAY_POWER_HANDOFF` 까지 모두 정상인데도 사용자가 첫 홈 화면 진입 순간 black/flicker를 본다면, 이제 남는 절단면은 **실제 픽셀 출력 위에 마지막으로 덮이는 UI/시스템 레이어** 다. 즉 `bootanimation 제거`, `launcher window reveal`, `color-fade overlay teardown` 셋 중 하나가 늦으면 panel은 이미 켜졌어도 사용자는 여전히 검은 전환면을 보게 된다.

## 핵심 요약

- Day108의 절단 순서는 **bootanimation 제거 확인 → launcher reveal 확인 → color-fade teardown 확인 → final UI reveal verdict 고정** 이다.
- Day107까지 정상인데 black/flicker가 남으면 원인은 panel power보다 뒤쪽인 **window/overlay teardown 경계** 일 가능성이 높다.
- 최소 증적은 `BOOTANIM_REMOVE_TRACE`, `LAUNCHER_REVEAL_TRACE`, `COLOR_FADE_TEARDOWN`, `UI_REVEAL_VERDICT` 네 줄이면 충분하다.
- 이 cutline이 나오면 display driver보다 **SystemUI/WindowManager/SurfaceFlinger transition owner** 가 먼저 받아야 한다.

## 코드 포인트

1. **bootanimation surface가 첫 launcher frame 전에 detach 되었는지 먼저 고정한다**

   ```text
   BOOTANIM_REMOVE_TRACE display=2 frame=6412 bootanim_layer=removed leash=gone txn_seq=9120 before_launcher=1
   BOOTANIM_REMOVE_TRACE display=2 frame=6412 bootanim_layer=alive leash=visible txn_seq=9120 before_launcher=0
   ```

   `before_launcher=0` 이면 launcher가 첫 frame을 제출해도 bootanimation black layer가 한 beat 더 남아 화면을 가릴 수 있다.

2. **launcher task/window가 실제로 reveal 가능한 상태인지 확인한다**

   ```text
   LAUNCHER_REVEAL_TRACE display=2 frame=6412 task=launcher surface=ready alpha=1 visible=1 starting_window=gone
   LAUNCHER_REVEAL_TRACE display=2 frame=6412 task=launcher surface=ready alpha=0 visible=0 starting_window=alive
   ```

   `surface=ready` 여도 `alpha=0` 이거나 `starting_window=alive` 면 사용자 기준 첫 홈 화면은 아직 노출되지 않은 상태다.

3. **color-fade/dim overlay teardown이 transaction 적용과 같은 프레임에 끝났는지 본다**

   ```text
   COLOR_FADE_TEARDOWN display=2 frame=6412 fade_layer=destroyed dim_layer=gone txn_applied=1
   COLOR_FADE_TEARDOWN display=2 frame=6412 fade_layer=visible dim_layer=alive txn_applied=0
   ```

   `fade_layer=visible` 또는 `dim_layer=alive` 면 panel 출력이 정상이어도 실제 시야는 black/fade overlay가 계속 점유한다.

4. **세 신호를 같은 transaction/frame 축으로 묶어 최종 reveal 여부를 남긴다**

   ```text
   UI_REVEAL_TRACE display=2 frame=6412 txn_seq=9120 bootanim=removed launcher=visible colorfade=gone visible=1
   UI_REVEAL_TRACE display=2 frame=6412 txn_seq=9120 bootanim=alive launcher=hidden colorfade=visible visible=0
   ```

   frame id만 맞고 transaction sequence가 다르면 실제 가림 레이어와 다음 프레임 복구를 혼동하기 쉽다.

5. **최종 verdict를 전환 레이어 owner 기준으로 한 줄 고정한다**

   ```text
   UI_REVEAL_VERDICT display=2 frame=6412 panel=ok visible=late cause=bootanim_teardown_late owner=bootanim_exit
   UI_REVEAL_VERDICT display=2 frame=6412 panel=ok visible=late cause=colorfade_overlay_residual owner=wm_transition
   ```

   verdict가 있어야 Day107의 광출력 문제와 Day108의 상위 레이어 teardown 문제를 명확히 분리할 수 있다.

## 리스크

- bootanimation 프로세스 종료만 보고 layer detach 완료로 오판하면 residual leash/overlay를 놓칠 수 있다.
- launcher first draw 로그만 믿으면 starting window 또는 splash overlay 잔류를 구분하지 못한다.
- color-fade teardown은 WindowManager transaction과 SurfaceFlinger latch 사이 one-beat 차이로 보이는 경우가 있어 공통 `txn_seq` 없이는 해석이 흔들린다.
- 전환 레이어 owner가 불명확하면 display 팀과 framework 팀 사이에서 재현 책임이 계속 왕복될 수 있다.

## 다음 액션

다음 글에서는 Day108 다음 절단면으로, **전환 레이어까지 모두 내려갔는데도 첫 진입에서 순간 검은 면이 남을 때 wallpaper attach, task snapshot 잔류, trusted overlay occlusion 중 무엇이 마지막 가림원인지** 를 자르는 체크포인트를 정리하겠다.

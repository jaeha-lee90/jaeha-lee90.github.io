---
title: R사 X5H Day109 - wallpaper attach·task snapshot 잔류·trusted overlay occlusion final screen-cover cutline
author: JaeHa
date: 2026-07-08 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day109, display, wallpaper, snapshot, overlay, occlusion, bringup]
---

Day108에서 `BOOTANIM_REMOVE_TRACE`, `LAUNCHER_REVEAL_TRACE`, `COLOR_FADE_TEARDOWN` 이 모두 정상인데도 첫 홈 진입 순간 black/flicker가 남는다면, 이제 남는 절단면은 **최종 화면을 가리는 비앱 레이어** 다. 특히 `wallpaper attach 지연`, `task snapshot 잔류`, `trusted overlay occlusion` 셋은 panel/launcher가 정상이어도 사용자가 마지막에 검은 면을 보는 대표 패턴이다.

## 핵심 요약

- Day109의 절단 순서는 **wallpaper attach 확인 → task snapshot detach 확인 → trusted overlay occlusion 확인 → final cover owner verdict 고정** 이다.
- Day108까지 정상이면 원인은 display pipe가 아니라 **WindowManager/SurfaceFlinger 상단 cover layer** 일 가능성이 높다.
- 최소 증적은 `WALLPAPER_ATTACH_TRACE`, `TASK_SNAPSHOT_TRACE`, `TRUSTED_OVERLAY_TRACE`, `FINAL_OCCLUSION_VERDICT` 네 줄이면 충분하다.
- 이 cutline에서 `occluded=1` 이 남으면 driver 재디버깅보다 **framework transition owner** 추적이 우선이다.

## 코드 포인트

1. **wallpaper가 launcher reveal 직전 frame에 실제 attach 되었는지 먼저 고정한다**

   ```text
   WALLPAPER_ATTACH_TRACE display=2 frame=6413 wallpaper_token=attached visible=1 alpha=1 before_launcher=1
   WALLPAPER_ATTACH_TRACE display=2 frame=6413 wallpaper_token=missing visible=0 alpha=0 before_launcher=0
   ```

   `before_launcher=0` 이면 launcher 배경이 아직 비어 있고, 그 틈을 snapshot 또는 black color layer가 덮을 수 있다.

2. **task snapshot starting surface가 실제 detach 되었는지 확인한다**

   ```text
   TASK_SNAPSHOT_TRACE display=2 frame=6413 task=launcher snapshot=gone leash=destroyed real_buffer=latched
   TASK_SNAPSHOT_TRACE display=2 frame=6413 task=launcher snapshot=alive leash=visible real_buffer=latched
   ```

   `real_buffer=latched` 여도 `snapshot=alive` 면 사용자는 실제 UI 대신 검은 snapshot 또는 오래된 starting surface를 본다.

3. **trusted overlay/system overlay가 상단 occlusion을 계속 점유하는지 본다**

   ```text
   TRUSTED_OVERLAY_TRACE display=2 frame=6413 overlay=none top_uid=launcher occluded=0
   TRUSTED_OVERLAY_TRACE display=2 frame=6413 overlay=sysui_scrim top_uid=systemui occluded=1
   ```

   `occluded=1` 이면 첫 프레임이 존재해도 보이지 않는다. 이 경우 cutline은 panel이 아니라 overlay owner다.

4. **세 신호를 같은 frame/transaction 축으로 묶어 최종 cover state를 남긴다**

   ```text
   FINAL_OCCLUSION_TRACE display=2 frame=6413 txn_seq=9124 wallpaper=ready snapshot=gone overlay=none visible=1
   FINAL_OCCLUSION_TRACE display=2 frame=6413 txn_seq=9124 wallpaper=missing snapshot=alive overlay=sysui_scrim visible=0
   ```

   `txn_seq` 를 맞춰야 wallpaper attach와 overlay teardown의 one-beat 차이를 오판하지 않는다.

5. **최종 verdict를 가림 owner 기준으로 한 줄 고정한다**

   ```text
   FINAL_OCCLUSION_VERDICT display=2 frame=6413 cause=task_snapshot_residual owner=wm_starting_surface visible=late
   FINAL_OCCLUSION_VERDICT display=2 frame=6413 cause=trusted_overlay_occlusion owner=systemui_transition visible=late
   ```

   verdict가 있어야 Day108의 reveal 문제와 Day109의 최종 cover 문제를 명확히 분리할 수 있다.

## 리스크

- wallpaper attach 여부를 앱 first draw와 혼동하면 배경 미부착 문제를 launcher 성능 문제로 오판할 수 있다.
- snapshot detach는 task surface ready보다 늦게 끝날 수 있어 `real_buffer=latched` 만 보고 정상 판정하면 놓치기 쉽다.
- trusted overlay는 SysUI scrim, 권한 overlay, transition leash 등 owner가 다양해 단일 레이어명만 보면 원인 분리가 안 된다.
- frame 기준만 보고 transaction sequence를 안 맞추면 실제 black frame이 아니라 직전 teardown 잔상으로 잘못 해석할 수 있다.

## 다음 액션

다음 글에서는 Day109 다음 절단면으로, **cover layer까지 모두 내려갔는데도 첫 화면이 순간 어둡거나 비정상 색으로 보일 때 screenshot path, color-transform, client-target 재사용 중 무엇이 최종 시각 오류를 만드는지** 를 자르는 체크포인트를 정리하겠다.

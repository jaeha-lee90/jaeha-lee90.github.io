---
title: R사 X5H Day91 - splashscreen·starting window·real content attach black window 절단면
author: JaeHa
date: 2026-06-01 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day91, display, splashscreen, startingwindow, windowmanager, surfaceflinger, blackwindow, bringup]
---

Day90에서 `screenshot/client composition/GPU target` 까지 확인했는데도 **GPU 결과는 정상인데 앱 창 내부가 계속 검게 남는 경우**, 이제 봐야 할 절단면은 `SplashScreen → Starting Window → 실제 app content attach` 전환 구간이다. 이 구간은 첫 draw 완료 로그만으로는 충분하지 않고, **누가 현재 top window인지**, **실제 buffer producer가 언제 바뀌는지**, **old surface detach와 new surface latch 사이에 black gap이 생기는지**를 같이 봐야 한다.

## 핵심 요약

- Day91의 1차 절단 순서는 **top window 상태 → starting window 제거 시점 → real content 첫 buffer latch → old/new surface detach-attach gap** 이다.
- GPU target이 정상인데 화면이 검으면 panel 문제가 아니라 `WindowManager/SurfaceControl 전환` 문제일 확률이 높다.
- `firstDraw` 보다 중요한 건 **first non-black buffer latched** 시점과 **starting window removed** 시점의 상대순서다.
- 최소 증적은 `TOP_WINDOW_STATE`, `STARTING_WINDOW_SWAP`, `REAL_CONTENT_LATCH`, `SURFACE_GAP` 네 줄이면 된다.

## 코드 포인트

1. **현재 사용자에게 보이는 top window가 누구인지 먼저 고정한다**

   ```text
   TOP_WINDOW_STATE display=2 task=104 top=Splash Screen com.r.cluster visible=1 alpha=1.0
   TOP_WINDOW_STATE display=2 task=104 top=ActivityRecord com.r.cluster/.MainActivity requested_visible=1
   ```

   app process가 살아 있어도 top surface가 아직 splash 또는 starting window면 black 원인은 app render 이전의 전환 정책일 수 있다. `top`, `requested_visible`, `alpha`를 같이 봐야 한다.

2. **starting window 제거와 실제 content 등장 순서를 frame 단위로 묶는다**

   ```text
   STARTING_WINDOW_SWAP display=2 frame=6288 starting=shown real_content=not_latched remove_reason=app_drawn
   STARTING_WINDOW_SWAP display=2 frame=6291 starting=removed real_content=latched gap_frames=3
   ```

   starting window가 먼저 사라지고 real content latch가 늦으면 그 빈 프레임 구간이 사용자에게 black window로 보인다. `remove_reason` 과 `gap_frames` 가 핵심이다.

3. **real content의 첫 buffer가 실제로 non-black인지 분리한다**

   ```text
   REAL_CONTENT_LATCH display=2 layer=MainActivity#0 frame=6291 buffer_id=0x91af first_latch=1 nonblack_tiles=0
   REAL_CONTENT_LATCH display=2 layer=MainActivity#0 frame=6293 buffer_id=0x91b4 first_nonblack=1 dataspace=srgb
   ```

   첫 latch가 있다고 끝이 아니다. 첫 buffer 자체가 black이면 attach 순서가 아니라 app 초기 렌더 내용 문제다. `first_latch` 와 `first_nonblack` 을 분리해 찍어야 한다.

4. **surface destroy/recreate 또는 leash 재부모화로 인한 빈 구간을 추적한다**

   ```text
   SURFACE_GAP display=2 old_sc=0x44a1 new_sc=0x44d9 destroy_frame=6289 create_frame=6290 latch_frame=6293
   SURFACE_GAP display=2 reparent=TaskDisplayArea leash_visible=1 child_visible=0 gap_ms=48
   ```

   전환 중 `SurfaceControl` 이 새로 생기거나 leash 아래 자식 가시성이 늦게 전파되면 GPU는 정상이어도 화면은 비어 보인다. Day85/88에서 본 race가 여기서 다시 재발할 수 있다.

5. **SplashScreen exit animation이나 starting window policy가 black clear를 삽입하는지 확인한다**

   ```text
   STARTING_POLICY display=2 splash_exit_anim=fade theme_bg=#000000 duration_ms=250
   STARTING_POLICY display=2 transfer_surface=0 defer_remove_for_rotation=0
   ```

   exit animation 배경색이 검정이고 실제 content latch가 늦으면 사용자는 이를 black screen으로 체감한다. 이 경우 driver가 아니라 테마/정책/타이밍 문제다.

## 리스크

- `app_drawn` 이벤트만 믿으면 starting window 제거 이후 몇 프레임의 black gap을 놓친다.
- 첫 latch와 첫 non-black buffer를 구분하지 않으면 WM 전환 이슈와 app 초기 렌더 이슈를 섞어 오판한다.
- multi-display에서 task가 다른 display area로 재배치되면 top window 추적이 꼬여 잘못된 display의 정상 로그를 보고 안심할 수 있다.
- splash exit animation 배경색/시간값을 보지 않으면 재현 빈도가 낮은 black flash를 panel 문제로 잘못 올릴 수 있다.

## 다음 액션

다음 글에서는 Day91 이후 분기인 **real content buffer까지 정상인데도 특정 앱에서만 첫 프레임이 검게 남는 경우, EGL surface 생성·buffer queue dequeue·첫 texture upload 지연 절단면** 을 정리하겠다.

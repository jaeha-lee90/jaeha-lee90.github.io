---
title: R사 X5H Day76 - alpha/z-order/plane enable 정상 후 남는 CSC/RGB-YUV/range swap/panel blanking 체크리스트
author: JaeHa
date: 2026-05-17 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day76, display, csc, rgb, yuv, range, panel-blanking, black-screen, bringup]
---

Day75에서 `plane enable`, `alpha`, `z-order`, `cover layer` 까지 정상인데 panel이 여전히 검다면, 이제 남는 축은 **pixel 값은 나오지만 panel이 기대하는 색공간/범위/출력 상태와 어긋나는 경우** 다. 이 단계에서는 보통 `RGB↔YUV mismatch`, `limited/full range swap`, `CSC coefficient 오적용`, `panel-side blanking/mute` 네 가지가 black screen 또는 사실상 black처럼 보이는 화면을 만든다.

## 핵심 요약

- `fetch OK + blend OK + visible layer OK` 다음에도 검다면, 이제는 **pixel interpretation** 과 **panel output mute 상태** 를 먼저 봐야 한다.
- bring-up 초반 절단 순서는 **input format vs CSC path → range(full/limited) → live CSC coeff/load → panel blank/mute/gate** 가 가장 빠르다.
- 특히 `RGB buffer` 를 `YUV input` 으로 해석하거나, `limited range` 를 잘못 적용하면 화면이 완전 black 또는 거의 black으로 보일 수 있다.
- 최소 로그는 `IN_FMT`, `CSC_MODE`, `RANGE`, `PANEL_BLANK` 네 줄이면 1차 절단이 된다.

## 코드 포인트

1. **source가 보인다고 panel이 같은 색공간으로 받는 것은 아니다**

   아래처럼 source/plane state가 정상이어도 아직 충분하지 않다.

   ```text
   PLANE_STATE plane=0 enable=1 alpha=255 z=12 format=ARGB8888
   TOP_LAYER display_slot=0 layer_id=cluster_home opaque=1
   ```

   여기에 실제 출력 해석 경로를 붙여야 한다.

   ```text
   OUT_PIPE display_slot=0 in_fmt=ARGB8888 csc_mode=YUV2RGB range=limited
   PANEL_IF panel=cluster bus_fmt=RGB888 blank=0 mute=0
   ```

   `ARGB8888` 입력인데 `YUV2RGB` 가 걸려 있으면 결과는 정상 색이 아니라 거의 무의미한 값이 된다.

2. **RGB/YUV mismatch는 black screen처럼 보이되 timing fault와 패턴이 다르다**

   ```text
   OUT_PIPE display_slot=0 in_fmt=ARGB8888 csc_mode=YUV2RGB coeff_set=bt601
   SCREEN_OBS panel=cluster very_dark flicker=0 vblank_alive=1
   ```

   이런 경우는 panel dead보다 **잘못된 CSC 경로** 가 더 유력하다. VBLANK가 살아 있고 top layer도 정상인데 화면이 매우 어둡거나 색이 무너져 있으면, timing보다 format 해석을 먼저 자르는 편이 빠르다.

3. **range swap은 '검은 화면 비슷함' 으로 오래 숨는다**

   ```text
   RANGE_CFG plane=0 input=full output=limited expected=full
   LUMA_SAMPLE min=0 max=18 avg=4
   ```

   full-range UI를 limited-range로 잘못 누르면 저휘도 구간이 바닥에 깔리면서 사용자가 거의 black으로 느낄 수 있다. 특히 dark theme, boot logo fade, camera preview shadow 영역에서는 완전 무출력처럼 오해하기 쉽다.

4. **panel-side blanking/mute는 compositor 정상 후단에서 덮어쓴다**

   ```c
   set_plane_state(...);
   apply_csc(...);
   panel_unblank(panel);
   ```

   여기서 `panel_unblank()` 가 빠지거나 race로 다시 blank가 걸리면, 앞단 합성이 다 정상이어도 최종 출력은 없다.

   ```text
   PANEL_BLANK panel=cluster state=1 reason=post_modeset_default
   POST_COMMIT panel=cluster blank=1 backlight=1 video_gate=0
   ```

   이 패턴이면 graphics stack보다 **panel-side blank gate** 가 root cause다.

5. **최소 ledger는 네 줄로 묶어 두는 게 좋다**

   ```text
   OUT_PIPE display_slot=0 in_fmt=ARGB8888 out_fmt=RGB888 csc_mode=bypass range=full
   CSC_LIVE display_slot=0 coeff=identity result=OK
   PANEL_BLANK panel=cluster state=0 mute=0 video_gate=1
   LUMA_PROBE panel=cluster min=12 max=231 avg=118
   ```

   - `csc_mode=bypass` 여야 하는데 변환이 걸려 있지 않은지
   - `range=full/limited` 가 panel 기대치와 맞는지
   - `blank/mute/video_gate` 가 실제 live 상태에서 열려 있는지
   - 최종 휘도 샘플이 0 근처로 눌려 있지 않은지

   이 네 줄이면 Day75 이후 남는 black screen을 훨씬 빨리 줄일 수 있다.

## 리스크

- CSC 기본 경로를 명시하지 않으면 RGB UI가 YUV 변환을 타면서 black 또는 극저휘도처럼 보일 수 있다.
- limited/full range 불일치는 완전 무출력보다 더 애매한 현상이라 panel 불량이나 content 문제로 오판하기 쉽다.
- commit 전 CSC 설정만 찍고 live coeff/panel blank 상태를 안 찍으면 post-commit 덮어쓰기 race를 놓친다.
- backlight ON 만 보고 panel blank가 풀렸다고 가정하면 후단 mute/video gate 문제를 오래 못 잡는다.

## 다음 액션

다음 글에서는 Day76을 이어서, **CSC/range/panel blanking까지 정상인데도 black screen일 때 backlight-PWM/TE sync/lane status를 묶어 panel electrical/live-link 축을 빠르게 자르는 체크리스트** 로 더 좁혀 보겠다.

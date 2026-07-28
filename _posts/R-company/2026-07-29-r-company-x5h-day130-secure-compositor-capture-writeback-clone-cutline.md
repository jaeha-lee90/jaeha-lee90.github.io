---
title: R사 X5H Day130 - secure compositor fallback·capture block·writeback/clone path cutline
author: JaeHa
date: 2026-07-29 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day130, display, protected, capture, writeback, clone, bringup]
---

Day129에서 `TEE session`, `HDCP`, `display policy` 까지 정상인데도 사용자는 여전히 **secure video가 black frame처럼 보인다** 면, 이제 절단면은 `secure layer의 compositor fallback`, `screen capture 차단`, `writeback evidence 부재`, `clone/mirror path 단절` 이다. 즉 보호 상태기는 ready지만, **사용자가 보는 경로와 우리가 관측하는 경로가 서로 다른데 그 차이를 black frame으로 오판하는지** 를 잘라야 한다.

## 핵심 요약

- Day130의 핵심 순서는 **secure layer 합성 경로 확인 → capture/writeback 차단 여부 분리 → physical display vs clone/mirror path 분리 → 최종 관측 owner 고정** 이다.
- secure playback에서는 `FLAG_SECURE/client composition fallback`, `protected layer screenshot block`, `writeback disabled on secure plane`, `secondary display clone route 미연결` 중 하나만 있어도 사용자는 black으로 느끼지만 root cause는 panel 출력 자체가 아닐 수 있다.
- `SECURE_COMPOSE_TRACE`, `CAPTURE_POLICY_TRACE`, `WRITEBACK_PATH_TRACE`, `CLONE_ROUTE_TRACE`, `FINAL_SECURE_VISIBILITY_VERDICT` 를 같은 `frame`, `buffer_id`, `display_id`, `session_id` 로 묶어야 한다.
- 이 보드의 `config_supportsMultiDisplay=true`, `config_displayUniqueIdArray={local:0, local:1}`, `config_secondaryHomePackage=com.android.car.carlauncher`, 그리고 `disfwk_fe` 의 `nr_rvgc_pipes/display_mapping` 구조는 **표시 경로와 복제 경로를 분리해서 봐야 한다는 근거** 다.

## 코드 포인트

1. **secure layer가 HWC direct scanout인지 client composition fallback인지 먼저 고정한다**

   ```text
   SECURE_COMPOSE_TRACE frame=9901 layer=Video#12 protected=1 hwc_strategy=device client_comp=0 overlay_plane=vp2
   SECURE_COMPOSE_TRACE frame=9907 layer=Video#12 protected=1 hwc_strategy=client target_protected=0 result=force_black
   ```

   secure layer가 client target으로 떨어졌는데 client target이 protected contract를 못 가지면, panel 이전에 compositor가 검은 target을 만들 수 있다. 이 경우 root cause owner는 panel도 HDCP도 아니라 composition policy다.

2. **screen capture/screenshot black이 실제 panel black과 같은 사건인지 분리한다**

   ```text
   CAPTURE_POLICY_TRACE frame=9901 display=local:0 protected_layer=1 screenshot=blocked screenrecord=blocked writeback=blocked
   CAPTURE_POLICY_TRACE frame=9901 user_symptom=black_screenshot panel_visibility=unknown
   ```

   secure content에서는 screenshot/screenrecord가 black으로 나오는 것이 정상 정책일 수 있다. 따라서 `adb shell screencap` 결과만으로 panel black을 결론내리면 오진이다.

3. **writeback path가 secure plane에서 비활성인지 별도 기록한다**

   ```text
   WRITEBACK_PATH_TRACE frame=9901 display=0 secure_plane=1 wb_supported=0 wb_reason=protected_content
   WRITEBACK_PATH_TRACE frame=9908 display=0 secure_plane=1 crc_supported=1 crc_state=changed
   ```

   secure plane은 writeback/capture가 정책상 막힐 수 있다. 이때는 writeback black을 실패 증거로 쓰면 안 되고, CRC/TE/VBLANK 같은 비가시 telemetry로 우회해야 한다.

4. **clone/mirror path와 실제 primary panel path를 분리한다**

   ```text
   CLONE_ROUTE_TRACE frame=9910 src=local:0 sink=local:1 mirror_enabled=0 private_display=0 route_state=not_cloned
   CLONE_ROUTE_TRACE frame=9910 src=local:0 sink=HDMI-A-1 protected_content=1 policy=clone_blocked
   ```

   `android_device/overlay/.../config.xml` 에서 multi-display가 활성화되어 있고 mirror 정책은 별도 토글이다. 또한 `disfwk_fe-kms.c` 는 `nr_rvgc_pipes` 와 `display_mapping` 으로 파이프를 따로 잡기 때문에, primary panel 정상 + secondary/clone black 조합이 충분히 가능하다.

5. **최종 black-frame owner를 '보이는 경로' 기준으로 귀속한다**

   ```text
   FINAL_SECURE_VISIBILITY_VERDICT frame=9910 symptom=black_screenshot owner=capture_policy cause=protected_capture_block
   FINAL_SECURE_VISIBILITY_VERDICT frame=9914 symptom=secondary_black owner=clone_route cause=protected_content_not_mirrorable
   FINAL_SECURE_VISIBILITY_VERDICT frame=9921 symptom=all_black owner=composition_policy cause=secure_layer_client_fallback
   ```

   이렇게 남겨야 Day129의 protection state 문제와 Day130의 observation path 문제를 분리할 수 있다.

## 리스크

- secure 콘텐츠는 screenshot/writeback 자체가 막혀 있어 정상 출력인데도 black evidence만 남기 쉽다.
- multi-display automotive 구성에서는 primary, cluster, passenger, HDMI가 서로 다른 policy를 가져 사용자 제보만으로는 어떤 panel이 black인지 모호하다.
- HWC vendor 구현이 `client fallback` 과 `policy block` 을 같은 generic present success로 숨기면 SurfaceFlinger 로그만으로는 owner 분리가 늦어진다.
- clone/mirror 정책은 설정값, sink capability, secure content level이 동시에 얽혀 재현 조건이 쉽게 흔들린다.

## 다음 액션

다음 글에서는 Day130 다음 절단면으로, **secure visibility 경계까지 분리한 뒤에도 남는 경우 decoder dequeue 지연·acquire fence 정체·protected first-frame starvation이 black start로 보이는 경로** 를 정리하겠다.

---
title: R사 X5H Day83 - HWC assignment 정상 후 남는 GPU client composition·overlay capability·fence starvation 체크리스트
author: JaeHa
date: 2026-05-24 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day83, display, hwc, gpu, overlay, fence, bringup]
---

Day82에서 `logical display ↔ physical display mapping`, `layer assignment`, `virtual display/mirror policy` 까지 정상이면, 이제는 **layer가 target display에 도착한 뒤 실제로 scanout 가능한 조합인지** 를 잘라야 한다. X5H bring-up 막바지에는 `HWC validate/present` 는 통과했는데 `GPU client composition fallback`, `overlay capability mismatch`, `acquire/present fence starvation` 때문에 target panel만 blank로 남는 경우가 자주 남는다.

## 핵심 요약

- Day82를 통과했는데도 target panel만 blank면 1차 절단 순서는 **GPU client target 생성 여부 → overlay capability mismatch → fence 진행 정체 → retire/present latency** 다.
- `validateDisplay()` 성공만으로는 충분하지 않다. 실제 visible frame은 `client target buffer`, `plane capability`, `fence signal` 3개가 모두 맞아야 올라간다.
- `DEVICE` 조합이 무너져 `CLIENT` fallback으로 내려가면 GPU output은 존재해도 target display용 client target이 비거나 stale buffer가 재사용될 수 있다.
- 최소 증적은 `COMPOSITION_FALLBACK`, `OVERLAY_REJECT`, `FENCE_STATE`, `PRESENT_LATENCY` 네 줄이면 충분하다.

## 코드 포인트

1. **GPU client target이 실제로 생성·갱신되는지 먼저 본다**

   ```text
   COMPOSITION_FALLBACK display=2 frame=4421 device_layers=1 client_layers=3 client_target_ready=0 reason=unsupported_yuv_rotation
   CLIENT_TARGET display=2 buffer_id=0x0 width=1920 height=720 dataspace=UNKNOWN stale=1
   ```

   HWC가 일부 layer를 `CLIENT` 로 돌렸는데 SurfaceFlinger가 target display용 `client target` 을 못 만들면 present는 돌아도 화면은 비어 있다. 특히 회전된 YUV, mixed dataspace, dim+secure overlay 조합에서 자주 터진다.

2. **overlay capability mismatch는 'validate 성공 + visible 0장' 패턴을 만든다**

   ```text
   OVERLAY_REJECT display=2 layer=ClusterApp plane=vid1 format=RGBA1010102 modifier=afbc reason=plane_format_unsupported
   OVERLAY_REJECT display=2 layer=NavBar plane=gfx0 transform=ROT_270 reason=rotation_limit
   ```

   target display plane이 요구 포맷, modifier, scaling, rotation 범위를 못 받으면 HWC는 fallback을 시도한다. 그런데 fallback 대상 GPU path가 준비되지 않았거나 bandwidth budget이 넘치면 최종 visible layer가 0이 될 수 있다.

3. **acquire/release/present fence 중 어디가 멈췄는지 분리해야 한다**

   ```text
   FENCE_STATE display=2 frame=4421 acquire_all=signaled release_prev=signaled present=timeout retire=none wait_ms=100
   BUFFER_QUEUE display=2 producer=app consumer=sf queued=3 acquired=1 blocked_on=present_fence
   ```

   `acquire fence` 가 안 열리면 새 frame이 못 들어오고, `present fence` 가 안 닫히면 다음 buffer recycle이 막힌다. 이 둘을 분리하지 않으면 GPU hang, DPU stall, app producer starvation을 한 덩어리로 오판하게 된다.

4. **bandwidth/plane budget 초과는 fallback loop를 만든다**

   ```text
   HWC_BUDGET display=2 planes_used=3 planes_max=2 bw_mbps=14800 bw_limit=12000 fallback=client
   SF_REVALIDATE display=2 frame=4421 reason=client_target_recompose_loop count=4
   ```

   multi-display SKU에서는 다른 display가 plane/bandwidth를 먼저 먹어 target panel이 반복적으로 `DEVICE → CLIENT → revalidate` 루프에 빠질 수 있다. 이 경우 단발성 reject보다 `same frame 반복 validate` 가 더 좋은 신호다.

5. **최종 판정은 present latency와 visible seq 증가를 같이 본다**

   ```text
   PRESENT_LATENCY display=2 frame=4421 validate_ms=0.7 present_ms=18.9 retire_ms=none visible_seq=1572->1572
   SF_FRAME_RESULT display=2 submitted=4 gpu_composed=1 device_composed=0 final_visible=0
   ```

   present 호출이 끝나도 `visible_seq` 가 증가하지 않으면 사용자가 본 frame은 갱신되지 않은 것이다. `retire_ms` 부재와 `visible_seq` 정체를 함께 봐야 진짜 blank를 빠르게 확정할 수 있다.

## 리스크

- `validate/present success` 만 믿으면 실제로는 stale client target을 반복 표시하는데도 panel이나 app 문제로 잘못 몰 수 있다.
- capability mismatch는 SKU별 plane 수, AFBC/linear 제약, rotation 지원 차이 때문에 개발 보드와 양산 보드 증상이 다르게 보이기 쉽다.
- fence starvation은 GPU, HWC, kernel sync timeline, app producer 어디서든 시작될 수 있어 로그 상관관계를 미리 고정하지 않으면 재현 때마다 결론이 흔들린다.
- bandwidth fallback loop는 thermal/perf mode, dual-display 활성화, camera preview 동시 구동 여부에 따라 간헐적으로만 드러나 장기 잠복 결함이 되기 쉽다.

## 다음 액션

다음 글에서는 Day83을 이어서, **GPU fallback과 fence까지 정상인데 첫 사용자 frame만 늦거나 사라질 때 boot animation 전환·launcher attach·first app transaction 타이밍을 자르는 체크리스트** 를 정리하겠다.

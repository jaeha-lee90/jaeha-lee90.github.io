---
title: R사 X5H Day73 - VBLANK는 살아 있는데 black screen일 때 plane content/address/stride 1차 절단 체크리스트
author: JaeHa
date: 2026-05-14 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day73, display, vblank, plane, stride, address, black-screen, bringup]
---

Day72에서 `DISPLAY_GET_INFO` 와 panel mode/format mismatch가 왜 black screen으로 이어지는지 봤다면, 다음으로 빨리 잘라야 할 축은 **`VBLANK는 정상인데 화면만 검다`** 는 경우다. 이 상태는 timing generator나 panel enable보다, 대개 **plane source address / stride / content validity** 쪽에서 끊긴다.

## 핵심 요약

- `VBLANK alive` 는 panel refresh loop가 돈다는 뜻이므로, `flush ok + black screen` 이면 **timing보다 plane fetch 입력값** 을 먼저 봐야 한다.
- bring-up 초반 1차 절단은 `addr valid?`, `stride valid?`, `content non-zero?` 세 줄이면 충분하다.
- framebuffer 주소가 IOMMU 매핑 밖이거나 stale physical address를 가리키면, commit은 성공해도 실제 panel에는 검은 프레임만 반복될 수 있다.
- `pitch == expected_pitch` 여도 buffer 자체가 zero-filled 이거나 wrong layer를 보고 있으면 결과는 똑같이 black screen이다.

## 코드 포인트

1. **VBLANK가 있으면 panel/timing보다 source fetch를 먼저 의심한다**

   아래 패턴이면 refresh는 이미 돌고 있다.

   ```text
   RVGC_FLUSH seq=144 result=OK
   RVGC_COMPLETE seq=144 result=OK
   RVGC_VBLANK_SUMMARY window=500ms display_slot=0 count=30
   SCREEN_STATE panel=cluster black
   ```

   이 경우 `enable 안 됨` 보다, plane이 읽는 주소나 stride가 틀렸을 확률이 더 높다.

2. **첫 절단은 address/stride/content 3종 세트로 남긴다**

   bring-up 로그는 아래 정도면 충분하다.

   ```text
   PLANE_SRC display_slot=0 plane=0 iova=0x7c800000 size=0x007e9000 pitch=7680 format=ARGB8888
   PLANE_EXPECT display_slot=0 width=1920 height=1080 expected_pitch=7680 min_size=0x007e9000
   PLANE_CONTENT display_slot=0 plane=0 first64_nonzero=0 checksum32=0x91ab23de
   ```

   여기서 바로 볼 것은 세 가지다.

   - `iova` 가 현재 frame에 대해 유효한가
   - `pitch` 가 format/Bpp 기준 기대값과 맞는가
   - 실제 메모리 내용이 0으로만 채워져 있지 않은가

3. **stale address는 commit success 뒤에 숨어 버린다**

   ```c
   plane->addr = buf->iova;
   plane->stride = buf->stride;
   submit_plane(plane);
   ```

   이 코드 경로가 에러 없이 지나가도, `buf->iova` 가 이전 frame의 해제된 mapping이거나 DomD/CR52가 접근 못 하는 aperture면 visible output은 검게 남는다. 즉 `submit ok` 는 **address reachability 보증** 이 아니다.

4. **stride가 맞아도 content가 0이면 화면은 검다**

   Day72에서는 format/pitch mismatch를 봤지만, 이번 단계에서는 정합해 보여도 buffer producer가 실제 픽셀을 안 썼을 수 있다.

   ```text
   GPU_RENDER_DONE layer=cluster out_iova=0x7c800000
   PLANE_CONTENT first64_nonzero=0 checksum32=0x00000000
   ```

   이런 로그면 display path보다 producer path를 먼저 봐야 한다. camera-to-composer 또는 GPU clear-only frame일 수도 있다.

5. **wrong plane / wrong layer route도 같은 증상으로 보인다**

   ```text
   SF_LAYER name=ClusterHome target_display=1 z=12
   RVGC_PLANE_BIND display_slot=0 plane=0 layer_id=rearcam
   ```

   panel은 살아 있고 주소도 유효하지만, 기대한 layer가 아니라 다른 plane content를 보고 있으면 사용자는 그냥 black screen으로 인식한다. 그래서 address 검증과 함께 **현재 plane에 어떤 layer가 bind됐는지** 도 반드시 같이 찍어야 한다.

## 리스크

- `VBLANK alive` 를 보고도 timing만 계속 파면 실제 원인인 stale address/IOMMU aperture 문제를 놓친다.
- stride만 맞는다고 안심하면 zero-filled buffer 또는 wrong-layer bind를 늦게 발견한다.
- producer 완료 로그만 믿고 content sample/checksum을 생략하면 “렌더링 완료인데 왜 검지?” 상태가 길어진다.
- plane source 주소 로그 없이 virtualization 경계(DomA/DomD/CR52)를 넘나들면 ownership 책임이 흐려진다.

## 다음 액션

다음 글에서는 Day73을 이어서, **plane source address가 맞는데도 보이지 않을 때 IOMMU/passthrough/cache sync 축을 10분 안에 자르는 체크리스트** 로 더 좁혀 보겠다.

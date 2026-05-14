---
title: R사 X5H Day74 - plane source는 맞는데 안 보일 때 IOMMU/passthrough/cache sync 10분 체크리스트
author: JaeHa
date: 2026-05-15 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day74, display, iommu, passthrough, cache-sync, dma-buf, black-screen, bringup]
---

Day73에서 `VBLANK alive + black screen` 을 address/stride/content로 1차 절단했다면, 그다음 가장 자주 남는 가지는 **주소값은 멀쩡해 보이는데 실제 fetch가 메모리를 제대로 못 읽는 경우** 다. X5H 같은 가상화 환경에서는 이 단계가 보통 `IOMMU mapping`, `passthrough aperture`, `cache sync` 셋 중 하나로 수렴한다.

## 핵심 요약

- `plane addr/stride/content` 가 모두 정상처럼 보여도, **display master가 그 주소를 실제로 접근 가능하다는 보장** 은 아니다.
- bring-up 초반 10분 절단 순서는 **IOVA aperture 유효성 → domain ownership/passthrough → cache clean/invalidate 여부** 가 가장 빠르다.
- `GPU/producer checksum OK` 와 `panel black` 가 동시에 보이면, producer 버그보다 **translation 또는 cache coherence** 를 먼저 의심하는 편이 맞다.
- virtualization 경계에서는 `submit success` 보다 **누가 그 buffer를 map했고 어느 master가 읽는지** 를 한 줄 ledger로 남기는 게 훨씬 중요하다.

## 코드 포인트

1. **주소값이 있다고 곧바로 fetch 가능하다는 뜻은 아니다**

   아래 로그만으로는 아직 충분하지 않다.

   ```text
   PLANE_SRC display_slot=0 plane=0 iova=0x7c800000 pitch=7680 format=ARGB8888
   PLANE_CONTENT display_slot=0 plane=0 checksum32=0x91ab23de
   ```

   여기에 반드시 붙어야 하는 건 display master 관점의 접근 가능성이다.

   ```text
   IOMMU_MAP master=RVGC domain=disp0 iova=0x7c800000 size=0x007e9000 result=OK
   ```

   이 한 줄이 없으면 `checksum OK` 는 CPU나 producer가 본 내용일 뿐, RVGC/CR52 fetch 경로가 같은 메모리를 읽는다는 증거가 아니다.

2. **passthrough ownership mismatch는 black screen으로 조용히 끝날 수 있다**

   Day62에서 정리한 ownership ledger를 display plane에도 그대로 적용해야 한다.

   ```text
   BUF_OWNER buf=cluster_main owner=DomD exporter=AndroidVM importer=RVGC aperture=disp0
   PASSTHROUGH_CHECK master=RVGC iova=0x7c800000 result=NG reason=outside_window
   ```

   이 패턴이면 주소 숫자는 맞아 보여도 실제 display master의 allowed aperture 밖이다. 현상은 단순 black screen이지만 실체는 **virtualization ownership 계약 위반** 이다.

3. **cache sync 누락은 '가끔 보임/가끔 검음' 패턴으로 나타난다**

   ```c
   dma_buf_begin_cpu_access(dmabuf, DMA_TO_DEVICE);
   update_frame(buf);
   dma_buf_end_cpu_access(dmabuf, DMA_TO_DEVICE);
   ```

   또는 device-side fence/cache maintenance가 빠지면, producer가 갱신한 frame을 display가 이전 cache line으로 읽을 수 있다. 이 경우 checksum을 CPU에서만 보면 정상인데 panel은 검거나 한 프레임 늦게 보일 수 있다.

4. **10분 안에 자르려면 checksum 위치를 둘로 나눠야 한다**

   ```text
   PRODUCER_CSUM buf=cluster_main checksum32=0x91ab23de
   DISPLAY_TAP_CSUM master=RVGC plane=0 checksum32=0x00000000
   ```

   두 checksum이 다르면, format보다 먼저 **cache flush / mapping alias / stale translation** 을 봐야 한다. 반대로 둘이 같다면 Day73의 wrong-layer bind 또는 alpha/blend 문제 쪽으로 더 빨리 넘어갈 수 있다.

5. **최소 ledger는 map-owner-sync 세 줄이면 충분하다**

   ```text
   IOMMU_MAP master=RVGC domain=disp0 iova=0x7c800000 size=0x007e9000 result=OK
   BUF_OWNER buf=cluster_main exporter=AndroidVM importer=DomD consumer=RVGC
   CACHE_SYNC buf=cluster_main dir=to-device fence=184 result=OK
   ```

   이 세 줄이 같은 `seq/frame_id` 로 묶이면, bring-up 현장에서 `주소는 맞는데 왜 안 보이냐` 를 훨씬 빨리 자를 수 있다.

## 리스크

- CPU 기준 checksum만 믿으면 IOMMU translation 실패를 producer 버그로 오판하기 쉽다.
- passthrough aperture 로그가 없으면 DomA/DomD/CR52 중 누가 ownership을 깨는지 책임이 흐려진다.
- cache sync 누락은 완전 재현 실패보다 간헐 현상으로 보이기 쉬워, race나 panel 불량으로 잘못 분류될 수 있다.
- `map OK` 와 `unmap timing` 을 같이 안 남기면 stale mapping 재사용 문제를 놓치기 쉽다.

## 다음 액션

다음 글에서는 Day74를 이어서, **IOMMU/cache sync까지 정상인데도 black screen일 때 alpha/blend/z-order/plane enable 축을 빠르게 자르는 체크리스트** 로 더 좁혀 보겠다.

---
title: R사 X5H Day122 - compression metadata·AFBC/UBWC header fetch·plane rotation/scaler line-buffer cutline
author: JaeHa
date: 2026-07-21 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day122, display, afbc, ubwc, scaler, rotation, linebuffer, bringup]
---

Day121에서 `FENCE_ORDER_TRACE`, `PREFETCH_MARGIN_TRACE`, `FETCH_WATERMARK_TRACE` 까지 정상인데도 특정 압축 surface, 90/270도 회전, scaling-heavy overlay에서만 **block 깨짐 / diagonal corruption / one-frame black tile** 이 남는다면 다음 절단면은 `compression metadata`, `AFBC/UBWC header fetch`, `plane rotation/scaler line-buffer` 다. 즉 fetch 시작 타이밍은 맞지만 **display가 해석해야 할 메타데이터나 중간 라인 버퍼 경로가 같은 frame 안에서 안정적으로 채워지는지** 봐야 한다.

## 핵심 요약

- Day122의 절단 순서는 **압축 포맷/modifier 고정 → header/payload fetch 동기 확인 → rotation/scaler line-buffer 수위 확인 → 최종 metadata-path verdict 확정** 이다.
- bandwidth, cache, fence가 정상이어도 압축 header 주소나 line-buffer budget이 틀리면 panel에는 메모리 손상처럼 보이는 artifact가 나온다.
- `COMP_META_TRACE`, `HEADER_FETCH_TRACE`, `LINEBUF_TRACE`, `FINAL_META_VERDICT` 를 같은 frame/plane으로 묶어야 수정 owner를 자를 수 있다.
- 최소 증적은 `modifier`, `header_ready`, `payload_ready`, `linebuf_fill_lines`, `scale_ratio` 다섯 축이면 충분하다.

## 코드 포인트

1. **plane이 기대한 압축 modifier와 실제 buffer modifier가 일치하는지 먼저 고정한다**

   ```text
   COMP_META_TRACE frame=7304 plane=1 format=NV12 modifier=AFBC_16X16_HEADER_SPLIT expected_modifier=AFBC_16X16_HEADER_SPLIT meta_stride=512 payload_stride=4096
   COMP_META_TRACE frame=7305 plane=1 format=NV12 modifier=LINEAR expected_modifier=AFBC_16X16_HEADER_SPLIT meta_stride=0 payload_stride=4096
   ```

   `modifier != expected_modifier` 면 payload 내용이 정상이어도 display는 **잘못된 해석 경로** 로 들어가 block artifact를 만든다.

2. **AFBC/UBWC header fetch가 payload fetch보다 늦지 않는지 분리한다**

   ```text
   HEADER_FETCH_TRACE frame=7305 plane=1 header_ready=0 payload_ready=1 header_latency_us=41 payload_latency_us=18 header_iova=0x7c400000
   HEADER_FETCH_TRACE frame=7306 plane=1 header_ready=1 payload_ready=1 header_latency_us=12 payload_latency_us=17 header_iova=0x7c408000
   ```

   `payload_ready=1` 인데 `header_ready=0` 이면 underflow처럼 보여도 실상은 **compression metadata ingress 실패** 다.

3. **rotation/scaler line-buffer가 현재 비율과 pixel format을 감당하는지 본다**

   ```text
   LINEBUF_TRACE frame=7305 plane=1 rotation=90 scale_ratio=1.75 linebuf_fill_lines=6 required_lines=10 taps_h=8 taps_v=6
   LINEBUF_TRACE frame=7306 plane=1 rotation=0 scale_ratio=1.00 linebuf_fill_lines=12 required_lines=6 taps_h=4 taps_v=4
   ```

   특히 `rotation=90|270` 이고 `linebuf_fill_lines < required_lines` 면 fetch 자체보다 **중간 재배열/스케일 버퍼 부족** 이 더 강한 원인이다.

4. **topology change 직후 metadata base나 scaler program 재적용 누락을 자른다**

   ```text
   META_REPROGRAM_TRACE frame=7305 event=overlay_promote header_base_changed=1 scaler_coeff_reload=0 rotation_ctx_reload=0
   ```

   overlay 승격, format 전환, virtual display detach 직후 `scaler_coeff_reload=0` 이면 steady-state는 멀쩡해도 첫 1~2 frame만 깨질 수 있다.

5. **최종 verdict를 metadata-path owner 기준으로 고정한다**

   ```text
   FINAL_META_VERDICT display=2 frame=7305 cause=afbc_header_late owner=display_dma symptom=black_tile
   FINAL_META_VERDICT display=2 frame=7305 cause=linebuf_underprovision owner=scaler_program symptom=diagonal_corruption
   ```

   이 verdict가 있어야 Day121의 fetch-timing 이슈와 Day122의 **metadata/line-buffer 이슈** 를 다른 수정 경로로 분리할 수 있다.

## 리스크

- 일부 드라이버는 AFBC/UBWC header와 payload를 같은 trace로만 노출해 header 지연을 직접 못 볼 수 있다.
- line-buffer 부족은 실제 buffer 문제보다 색공간 변환, tap 수 증가, 10bit path 진입 때문에 갑자기 드러날 수 있다.
- 압축을 임시로 끄면 증상은 사라져도 대역폭/전력/thermal 조건이 달라져 root cause를 가릴 수 있다.
- rotation/scaler coefficient reload는 frame boundary race와 겹쳐 재현성이 낮을 수 있으니 한 번의 clean run만으로 정상 판정하면 위험하다.

## 다음 액션

다음 글에서는 Day122 다음 절단면으로, **metadata와 line-buffer까지 정상인데도 남는 경우 writeback CRC·post-blend capture·panel ingress 비교로 display pipe 후단 vs panel 내부 경계를 어떻게 자를지** 정리하겠다.

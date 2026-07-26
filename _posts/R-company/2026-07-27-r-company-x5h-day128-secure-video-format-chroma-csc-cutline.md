---
title: R사 X5H Day128 - secure video format·chroma siting·CSC 경계 black/green frame cutline
author: JaeHa
date: 2026-07-27 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day128, display, protected, yuv, chroma, csc, bringup]
---

Day127에서 `crop`, `viewport`, `secure scaler capability` 까지 맞는데도 **secure video만 black 또는 green tint로 깨진다** 면 이제 절단면은 `YUV format 해석`, `chroma siting`, `CSC/range programming`, `10bit path 축소` 다. 즉 geometry는 맞지만, **secure plane fetch 이후 pixel unpack/CSC 단계가 protected format 계약을 끝까지 지키는지** 를 잘라야 한다.

## 핵심 요약

- Day128의 핵심 순서는 **decoder output format 고정 → gralloc/HWC format mapping 대조 → secure plane CSC capability 확인 → black/green signature와 최종 verdict 결합** 이다.
- secure path에서는 `NV12`, `NV21`, `P010`, `Y410`, `BT.601/709/2020`, `limited/full range`, `cosited/interstitial chroma` 중 하나라도 잘못 연결되면 증상은 black frame, green frame, purple tint, crushed luma로 나타난다.
- `SECURE_FMT_TRACE`, `CHROMA_SITE_TRACE`, `CSC_MATRIX_TRACE`, `PIXEL_UNPACK_TRACE`, `FINAL_SECURE_COLOR_VERDICT` 를 같은 `buffer_id`, `plane`, `frame` 으로 묶어야 한다.
- 최소 증적은 `fourcc/hal_format`, `bitdepth`, `range`, `chroma_siting`, `csc_result` 다섯 축이면 된다.

## 코드 포인트

1. **decoder가 생산한 실제 YUV 계약을 먼저 고정한다**

   ```text
   SECURE_FMT_TRACE buffer_id=0xa301 protected=1 codec_fmt=P010 visible=3840x2160 bitdepth=10 range=limited primaries=bt2020
   SECURE_FMT_TRACE buffer_id=0xa302 protected=1 codec_fmt=NV12 visible=1920x1080 bitdepth=8 range=limited primaries=bt709
   ```

   secure video는 userspace dump가 막히는 경우가 많아서, 디코더가 무엇을 만들었는지 초기에 고정하지 않으면 뒤 단계가 모두 추정이 된다.

2. **gralloc/HWC format mapping이 secure path에서 축소되지 않는지 본다**

   ```text
   PIXEL_UNPACK_TRACE buffer_id=0xa301 hal_format=YCBCR_P010 plane_fmt=NV12_8B owner=hwc_import result=bitdepth_downgrade
   ```

   import는 성공해도 secure plane이 10bit를 못 받아 8bit NV12로 강등되면 luma/chroma 해석이 틀어져 black 또는 green output으로 끝날 수 있다.

3. **chroma siting과 UV order를 별도 축으로 분리한다**

   ```text
   CHROMA_SITE_TRACE frame=9440 plane=vp2 format=NV12 chroma_order=UV siting=cosited expected=interstitial mismatch=1
   ```

   NV12/NV21 swap, cosited/interstitial mismatch는 crash 없이도 피부색 왜곡이나 녹색 프레임으로 드러난다. secure 경로에서는 캡처가 제한되어 더 놓치기 쉽다.

4. **secure plane CSC/range capability를 일반 plane과 분리 기록한다**

   ```text
   CSC_MATRIX_TRACE plane=vp2 protected=1 matrix=bt2020_to_rgb range=limited secure_csc_ok=0 fallback=bt709_limited
   ```

   일반 plane은 되는 CSC 조합이 secure plane에서는 제한될 수 있다. 특히 `P010 + BT.2020 + limited range` 조합은 capability table이 좁으면 validate 후에도 최종색이 깨진다.

5. **최종 증상을 pixel-contract owner 기준으로 귀속한다**

   ```text
   FINAL_SECURE_COLOR_VERDICT frame=9440 display=2 cause=p010_not_supported_on_secure_plane owner=plane_pixel_unpack symptom=green_frame
   FINAL_SECURE_COLOR_VERDICT frame=9445 display=2 cause=nv12_nv21_uv_swap owner=buffer_format_mapping symptom=purple_green_tint
   ```

   이렇게 정리해야 Day127의 geometry 이슈와 Day128의 color contract 이슈를 분리해 수정 포인트를 정확히 잡을 수 있다.

## 리스크

- secure buffer는 CPU readback과 screenshot이 제한되어 black frame과 wrong-color frame을 증적으로 남기기 어렵다.
- vendor codec이 color aspects를 vendor-private metadata로만 전달하면 SurfaceFlinger/HWC 로그만으로 `range`, `primaries`, `transfer` 를 복원하기 어렵다.
- 일부 backend는 unsupported CSC를 reject하지 않고 nearest matrix로 대체해 간헐 color shift처럼 보여 triage를 흐린다.
- 10bit→8bit 강등, UV swap, chroma siting mismatch는 특정 콘텐츠와 밝기 구간에서만 눈에 띄어 재현성이 낮을 수 있다.

## 다음 액션

다음 글에서는 Day128 다음 절단면으로, **secure YUV format/CSC까지 맞는데도 남는 경우 decryption/TEE session·content protection state·HDCP/secure display policy가 final black frame으로 이어지는 경로** 를 정리하겠다.

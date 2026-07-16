---
title: R사 X5H Day118 - panel self-refresh exit retention·DSC line-buffer underflow·ESD recovery latent artifact cutline
author: JaeHa
date: 2026-07-17 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day118, display, psr, dsc, esd-recovery, underflow, bringup]
---

Day117에서 `SSC_TRACE`, `EMI_MARGIN_TRACE`, `LANE_DESKEW_TRACE` 까지 정상인데도 장시간 soak 이후 또는 panel idle 복귀 직후에만 **잔상성 line tear, 한 줄 깨짐, 수 초에 한 번 나오는 latent artifact** 가 남는다면, 이제 절단면은 `panel self-refresh exit retention`, `DSC line-buffer underflow`, `ESD recovery replay` 다. 즉 PHY margin은 충분하지만 **panel 내부 상태 복귀나 압축 파이프 재동기화가 늦는지** 확인해야 한다.

## 핵심 요약

- Day118의 절단 순서는 **PSR exit retention 확인 → DSC line-buffer underflow 분리 → ESD recovery replay 여부 확인 → latent artifact verdict 확정** 이다.
- 증상이 고휘도 edge 전반이 아니라 soak 후 첫 interaction, panel idle 복귀, rare one-line corruption에 몰리면 EMI보다 **panel internal state machine** 을 먼저 보는 편이 맞다.
- `psr_exit_us`, `retention_valid`, `dsc_underflow_cnt`, `slice_crc_mismatch`, `esd_recover_seq` 를 같은 frame 축으로 남겨야 owner를 자를 수 있다.
- 최소 증적은 `PSR_EXIT_TRACE`, `DSC_LINEBUF_TRACE`, `ESD_RECOVERY_TRACE`, `FINAL_LATENT_ARTIFACT_VERDICT` 네 줄이면 충분하다.

## 코드 포인트

1. **panel self-refresh exit 직후 retention buffer가 실제로 비워졌는지 먼저 본다**

   ```text
   PSR_EXIT_TRACE display=2 frame=6711 exit_reason=input_wakeup psr_exit_us=824 retention_valid=1 first_new_frame=0
   PSR_EXIT_TRACE display=2 frame=6712 exit_reason=input_wakeup psr_exit_us=824 retention_valid=0 first_new_frame=1
   ```

   `retention_valid=1` 인 채 첫 새 frame이 들어오면 black screen 대신 **이전 line이 한 beat 섞인 artifact** 로 보일 수 있다.

2. **DSC slice 복원 직후 line-buffer underflow가 튀는지 자른다**

   ```text
   DSC_LINEBUF_TRACE display=2 frame=6712 slice=1 rc_model_reset=1 linebuf_level=3 underflow_cnt=1
   DSC_LINEBUF_TRACE display=2 frame=6713 slice=1 rc_model_reset=0 linebuf_level=11 underflow_cnt=0
   ```

   `underflow_cnt=1` 이 soak 복귀 직후만 찍히면 source content보다 **DSC decode restart budget 부족** 쪽 owner가 더 강하다.

3. **slice CRC mismatch가 panel ingress line corruption과 같은 frame에 겹치는지 본다**

   ```text
   DSC_SLICE_CRC_TRACE display=2 frame=6712 slice0=ok slice1=mismatch slice2=ok slice3=ok panel_line=438
   ```

   특정 slice만 mismatch면 full-frame 문제보다 **부분 line decode 재동기화 실패** 로 해석하는 게 맞다.

4. **ESD recovery가 조용히 replay되며 이전 command state를 재주입하는지 확인한다**

   ```text
   ESD_RECOVERY_TRACE display=2 frame=6710 detected=1 reset_only=0 cmd_replay=1 replay_set=display_on,pps,wrmemstart
   ESD_RECOVERY_TRACE display=2 frame=6711 detected=0 reset_only=0 cmd_replay=0 replay_done=1
   ```

   사용자는 화면이 살아 있다고 느끼지만 내부적으로 `cmd_replay=1` 이 한 번 끼면 **희귀 line 깨짐이나 stale panel state** 가 남을 수 있다.

5. **최종 verdict를 panel internal restart owner 기준으로 고정한다**

   ```text
   FINAL_LATENT_ARTIFACT_VERDICT display=2 frame=6712 cause=psr_retention_not_flushed owner=panel_psr symptom=stale_line_mix
   FINAL_LATENT_ARTIFACT_VERDICT display=2 frame=6712 cause=dsc_linebuf_underflow owner=panel_dsc symptom=single_line_corruption
   ```

   이 verdict가 있어야 Day117의 **signal integrity 이슈** 와 Day118의 **panel internal resume/decode 이슈** 를 다른 수정 경로로 분리할 수 있다.

## 리스크

- panel vendor가 PSR/ESD 내부 상태를 외부로 거의 노출하지 않으면 로그만으로는 retention 잔류와 command replay를 완전히 분리하기 어렵다.
- DSC underflow counter는 panel SKU나 bridge 유무에 따라 의미가 달라, 동일 threshold를 모든 모델에 재사용하면 오판할 수 있다.
- soak 재현은 온도, idle residency, AOD/blank 정책에 민감해서 짧은 bench test로는 놓치기 쉽다.
- ESD recovery replay를 무조건 꺼버리면 artifact는 줄어도 실제 field 복구력이 떨어질 수 있다.

## 다음 액션

다음 글에서는 Day118 다음 절단면으로, **panel internal resume까지 정상인데도 장시간 사용 후 누적된 frame drop/jank가 남을 때 memory bandwidth QoS·interconnect DVFS·display underflow telemetry 경로를 어떻게 자를지** 정리하겠다.

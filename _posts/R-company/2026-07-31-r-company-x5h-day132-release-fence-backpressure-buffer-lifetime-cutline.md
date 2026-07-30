---
title: R사 X5H Day132 - release fence backpressure·decoder surface recycle·protected buffer lifetime mismatch cutline
author: JaeHa
date: 2026-07-31 01:00:00 +0900
categories: [R사, X5H, Xen, BSP, Bring-up]
tags: [R사, X5H, Day132, display, protected, release-fence, decoder, buffer-lifetime, bringup]
---

Day131에서 `decoder output`, `BufferQueue depth`, `acquire fence` 까지 정상인데도 **재생 시작 직후 한두 번 black flash가 끼면**, 이제 절단면은 `release fence backpressure`, `decoder output surface recycle 지연`, `protected buffer lifetime mismatch` 다. 즉 첫 프레임은 보였지만, **이전 보호 버퍼가 제때 반환되지 않아 다음 프레임이 끊기며 black으로 번지는지** 를 잘라야 한다.

## 핵심 요약

- Day132의 핵심 순서는 **present 후 release fence 반환 시점 고정 → decoder surface slot recycle 지연 확인 → protected buffer retain/release owner 분리 → intermittent black flash verdict 고정** 이다.
- secure playback에서는 `release fence late`, `Surface dequeue blocked by outstanding protected slot`, `HWC retire ack 지연`, `buffer lifetime contract mismatch` 중 하나만 있어도 첫 visible frame 뒤에 짧은 black flash가 생길 수 있다.
- `RELEASE_FENCE_TRACE`, `DECODER_RECYCLE_TRACE`, `PROTECTED_LIFETIME_TRACE`, `BLACK_FLASH_TRACE`, `FINAL_PROTECTED_FLASH_VERDICT` 를 같은 `session_id`, `frame`, `buffer_id`, `slot` 으로 묶어야 한다.
- 이 단계의 목적은 **first-frame starvation 이후의 steady-state 진입 실패** 를 가르는 것이다. 첫 프레임은 보였으므로 Day131과 owner가 다르다.

## 코드 포인트

1. **present된 protected buffer의 release fence 반환이 늦는지 먼저 고정한다**

   ```text
   RELEASE_FENCE_TRACE frame=1210 layer=Video#12 buffer_id=0x91 slot=3 present=1 release_fd=245 signaled=0 wait_ms=33.4
   RELEASE_FENCE_TRACE frame=1211 layer=Video#12 buffer_id=0x91 slot=3 present=1 release_fd=245 signaled=1 wait_ms=49.8
   ```

   첫 프레임은 나갔는데 release fence가 늦게 열리면 decoder는 같은 protected slot을 재사용하지 못하고 다음 출력이 막힌다.

2. **decoder output surface가 recycle 대기 때문에 dequeue block 되는지 본다**

   ```text
   DECODER_RECYCLE_TRACE session=42 output_slot=3 dequeue_in=blocked outstanding=4 recyclable=0 blocked_by=release_fence_245
   DECODER_RECYCLE_TRACE session=42 output_slot=3 dequeue_in=ok outstanding=3 recyclable=1 buffer_id=0x97
   ```

   secure decoder는 사용 가능한 output slot 수가 타이트해서 한 슬롯만 늦어도 다음 프레임 공급이 끊기기 쉽다.

3. **protected buffer lifetime owner가 codec인지 SF/HWC인지 분리한다**

   ```text
   PROTECTED_LIFETIME_TRACE buffer_id=0x91 producer=codec consumer=sf imported=1 latched=1 retire_pending=1 release_owner=hwc
   PROTECTED_LIFETIME_TRACE buffer_id=0x91 producer=codec consumer=sf imported=1 latched=1 retire_pending=0 release_owner=cleared
   ```

   buffer가 이미 decode 완료인데 retire pending 때문에 살아 있으면 codec stall처럼 보여도 실제 owner는 display retire path다.

4. **black flash가 fence backpressure와 같은 프레임군에서 발생하는지 묶는다**

   ```text
   BLACK_FLASH_TRACE session=42 visible_frame=1210 next_frame=1211 flash_ms=33 symptom=intermittent_black correlates=release_fence_backpressure
   BLACK_FLASH_TRACE session=42 visible_frame=1211 next_frame=1212 flash_ms=0 symptom=cleared correlates=none
   ```

   간헐 black은 `frame drop` 이 아니라 `next protected frame 공급 중단` 으로 보는 편이 정확하다.

5. **최종 owner를 steady-state 진입 실패 기준으로 귀속한다**

   ```text
   FINAL_PROTECTED_FLASH_VERDICT session=42 symptom=post_start_black_flash owner=release_fence cause=late_release_ack
   FINAL_PROTECTED_FLASH_VERDICT session=43 symptom=post_start_black_flash owner=decoder_surface_pool cause=protected_slot_recycle_block
   FINAL_PROTECTED_FLASH_VERDICT session=44 symptom=post_start_black_flash owner=buffer_lifetime_contract cause=retire_release_mismatch
   ```

   이렇게 남겨야 Day131의 first-frame starvation과 Day132의 post-start backpressure를 분리할 수 있다.

## 리스크

- protected playback은 output slot 수가 작아 release fence 한 번만 늦어도 체감 glitch가 크게 나타난다.
- vendor HWC가 retire/release telemetry를 충분히 안 주면 codec stall과 display backpressure가 같은 현상처럼 보인다.
- secure path는 메모리 덤프가 막혀 buffer lifetime을 fence/slot bookkeeping으로만 추적해야 하므로 오판 여지가 있다.
- 앱이 adaptive playback이나 해상도 전환을 함께 수행하면 recycle 지연과 stream reconfigure가 섞여 보일 수 있다.

## 다음 액션

다음 글에서는 Day132 다음 절단면으로, **secure playback steady-state에서 반복 black glitch로 번지는 decoder reconfigure·resolution switch·protected pipeline renegotiation 경로** 를 정리하겠다.

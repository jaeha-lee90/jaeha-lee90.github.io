---
title: R사 X5H Day35 - Zygote/SystemServer 부팅 완료 게이트와 framework ready 차단면
author: JaeHa
date: 2026-04-07 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day35, zygote, system_server, framework, bootanim, bringup]
---

Day35는 Day34의 HAL 등록 게이트 다음 단계로, **vendor HAL이 등록된 뒤 framework가 Zygote → SystemServer → bootanim 종료까지 수렴하는 구간**을 잠근다. X5H처럼 Xen·vendor HAL·Android framework 의존성이 얽힌 구조에서는 HAL 프로세스가 살아 있어도 framework ready가 늦거나 멈추면 실제 제품은 부팅 실패와 동일하게 보인다.

## 핵심 요약

- bring-up의 실질적인 완료 기준은 HAL 기동이 아니라 **Zygote 기동, SystemServer 주요 서비스 publish, `sys.boot_completed=1` 수렴**까지다.
- vendor 영역 문제도 최종적으로는 `system_server` 대기, `bootanimation` 장기 유지, home 미표시로 나타나므로 framework 관측 키를 따로 고정해야 한다.
- Day33(init/property), Day34(VINTF/HAL) 검증 뒤에는 **framework ready deadline**을 독립 게이트로 두는 것이 재현성과 triage 속도 모두에 유리하다.

## 코드 포인트

1. **부팅 체인 완료 조건을 property 기반으로 명시**
   - 최소 관측 포인트:
     - `init.svc.zygote` 또는 `init.svc.zygote64`
     - `init.svc.system_server`(직접 노출되지 않으면 log/event 기반 대체)
     - `service.bootanim.exit=1`
     - `sys.boot_completed=1`
   - 권장 규칙:
     - HAL ready는 선행조건
     - `sys.boot_completed=1`은 최종 완료조건
     - 중간 단계 timeout은 별도 원인코드로 분리

2. **SystemServer publish 지연을 서비스 단위로 분해**
   - 핵심 체크 축:
     - `SurfaceFlinger` 연결 가능 여부
     - `package`, `activity`, `display`, `audio` 계열 서비스 publish latency
     - 첫 foreground activity 기동 시간
   - 실무적으로는 `logcat -b system -b events`에서 아래 이벤트를 추적하면 triage가 빨라진다.
     - `Zygote started`
     - `SystemServer: Start*Service`
     - `BootReceiver` / `BOOT_COMPLETED`
     - `wm_boot_animation_done`

3. **framework ready 계측 훅 표준화**
   - 권장 로그 키:
     - `BOOT_PHASE=framework_ready`
     - `STAGE=<zygote|system_server|bootanim_exit|boot_completed>`
     - `STATE=<start|ready|timeout|crashloop>`
     - `LATENCY_MS=<n>`
     - `BLOCKER=<hal_wait|binder_timeout|service_crash|surfaceflinger_wait>`
   - 이렇게 고정해두면 vendor/HAL 문제와 framework 내부 문제를 같은 축에서 비교할 수 있다.

4. **게이트 정책을 boot success에서 user-visible ready로 상향**
   - cold boot 기준 예시:
     - Zygote start timeout 0건
     - SystemServer crash/restart 0건
     - bootanimation 95p 종료시간 상한 고정
     - `sys.boot_completed=1` 미수렴 0건
   - 즉, serial console 기준 “끝까지 부팅됨”이 아니라 **사용자가 조작 가능한 상태 도달**을 게이트로 써야 한다.

## 리스크

- HAL은 정상 등록됐지만 framework가 해당 HAL 응답을 기다리며 `system_server`에서 장시간 block될 수 있다.
- bootanimation 종료만 보고 성공 처리하면 launcher 미기동, binder timeout, service crashloop를 놓칠 수 있다.
- `sys.boot_completed`만 단일 지표로 쓰면 어디서 막혔는지 알 수 없어 재현 실험 비용이 커진다.

## 다음 액션

- `HAL ready → zygote → system_server → bootanim exit → boot_completed` 5단계 timeline 수집 스크립트 초안 작성.
- cold boot 20회 기준 framework ready p95/p99를 측정해 Day30 SOP의 최종 게이트에 연결.
- `system_server` 지연 상위 5개 서비스(display/package/activity/audio/input) 로그 포인트를 표준화해 blocker taxonomy를 고정.

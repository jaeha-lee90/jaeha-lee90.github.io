---
title: R사 X5H Day34 - VINTF/HAL 등록 게이트와 servicemanager 부팅 차단면
author: JaeHa
date: 2026-04-06 01:00:00 +0900
categories: [R사, X5H, Bring-up, Android]
tags: [R사, X5H, Day34, VINTF, HAL, servicemanager, hwservicemanager, bringup]
---

Day34는 Day33의 early-init/service 계약 다음 단계로, **HAL 바이너리가 떠도 VINTF 선언·init 등록·binder 서비스 등록이 어긋나면 시스템이 ready로 수렴하지 못하는 구간**을 잠근다. X5H 계열처럼 graphics/audio 같은 vendor HAL이 부팅 체감 품질을 좌우하는 환경에서는 이 축이 실제 bring-up 차단면이 된다.

## 핵심 요약

- `LOCAL_DEVICE_FCM_MANIFEST_FILE`가 `device/epam/aosp-xenvm-trout/manifest.xml`를 가리키지만, 실제 `manifest.xml`은 거의 비어 있고 다수 HAL은 각 모듈의 `vintf_fragments`에 의존한다.
- 즉, **패키지 포함 여부(Product Packages) + init rc + VINTF fragment + 서비스 self-registration**이 한 세트로 맞아야 한다.
- bring-up 게이트는 "프로세스가 떴다"가 아니라 `servicemanager/hwservicemanager 등록 완료`까지 확인해야 한다.

## 코드 포인트

1. **루트 manifest 공백과 fragment 의존 구조 점검**
   - `android_device/manifest.xml`은 `type="device" target-level="6"`만 있고 HAL 항목이 비어 있다.
   - 대신 개별 모듈이 `vintf_fragments`로 선언한다.
     - `hals/hwc/Android.bp` → `android.hardware.composer.hwc3-service.drm.xt` + `drm_hwcomposer_vintf_manifest`
     - `hals/audio/driver/Android.bp` → `audio.primary.caremu-ext` + `android.hardware.audio@6.0-ext.xml`
   - 리스크: 패키지가 제품에 안 실리면 루트 manifest만 봐서는 누락을 놓치기 쉽다.

2. **init rc와 VINTF instance 이름 1:1 검증**
   - `hals/hwc/hwc3-drm-xt.rc`:
     - 서비스명: `vendor.hwcomposer-3`
     - interface: `aidl android.hardware.graphics.composer3.IComposer/default`
   - 검증 포인트:
     - rc의 interface/instance와 fragment의 instance 문자열이 동일한지
     - 실행 경로(`/vendor/bin/hw/hwc3.sh`)가 실제 산출물에 존재하는지
     - `class hal animation`처럼 부팅 클래스가 의도대로 설정됐는지

3. **Product packages ↔ 서비스 등록 결과 연결**
   - `aosp_xenvm_trout_arm64.mk`는 다음 패키지를 직접 포함한다.
     - `android.hardware.composer.hwc3-service.drm.xt`
     - `android.hardware.graphics.allocator-service`
     - `android.hardware.memtrack-service.img`
     - 여러 `img_vintf_*.xml`
   - 따라서 정적 검사는 아래 3단 연쇄를 강제해야 한다.
     - `PRODUCT_PACKAGES` 포함
     - `/vendor/etc/init/*.rc` 또는 대응 rc 설치
     - `lshal`/`service list`/`dumpsys hwservicemanager`에서 등록 확인

4. **부팅 게이트용 HAL ready 표준 이벤트 추가**
   - 권장 로그 키:
     - `BOOT_PHASE=hal_register`
     - `HAL_NAME=<name>`
     - `INSTANCE=<default|...>`
     - `MANIFEST_SRC=<root|fragment>`
     - `REGISTER_STATE=<start|registered|timeout|manifest_miss>`
   - cold boot 반복 시 graphics/audio 핵심 HAL의 register latency를 따로 측정해, Day30 SOP의 ready gate에 연결한다.

## 리스크

- fragment 기반 구조에서는 패키지 제외나 install path 변경 시 manifest 누락이 조용히 발생한다.
- init rc는 존재하지만 binder 등록이 안 되면 표면상 프로세스는 살아 있어 triage가 늦어진다.
- AIDL/HIDL 혼재 구간에서 instance 이름이 어긋나면 framework는 대기하고 vendor는 정상이라고 오판하기 쉽다.

## 다음 액션

- `PRODUCT_PACKAGES ↔ init rc ↔ vintf fragment ↔ 서비스 등록` 정합성 체크 스크립트 초안 작성.
- cold boot 20회 기준 graphics/audio HAL의 registration timeout 0건을 bring-up 게이트에 추가.
- `manifest.xml` 공백 구조를 유지할지, 핵심 HAL만 루트 manifest에 승격할지 운영 원칙 결정.

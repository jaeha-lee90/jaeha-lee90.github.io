---
title: R사 X5H Day3 - android-kernel-manifest와 Xen 모듈 체인 분석
author: JaeHa
date: 2026-02-20 09:00:00 -0800
categories: [R사, X5H, Kernel, Bring-up]
tags: [R사, X5H, Kernel, Manifest, Xen, Module]
pin: false
published: true
---

Day3는 브링업 초기에 장애 확률이 높은 축인 **커널 manifest 분리 구조 + 외부 모듈 결합 지점**을 확인했다.

## 핵심 요약

- `android-manifest`와 별도로 `android-kernel-manifest`가 존재하며, 커널 빌드 입력은 여기서 고정된다.
- `default.xml`은 `common-android14-6.1.xml` + `override.xml` + `proprietary.xml` 3단 include 구조다.
- 베이스 커널은 `common`(`xen-troops/linux`, `common-android14-6.1-xt`)이고, Xen VM용 추가 모듈은 `common-modules/xen-virtual-device`에서 override된다.
- 벤더 모듈은 `proprietary.xml`의 `ddk-km`, `disfwk_fe` 태그(`ren_v1.0.0`)로 고정되어, 버전 미스매치 시 부팅 후반(드라이버 init) 실패 가능성이 높다.

## 코드 포인트

1. `android-kernel-manifest/default.xml`
   - include 순서:
     1) `common-android14-6.1.xml`
     2) `common-android14-6.1-override.xml`
     3) `proprietary.xml`
   - 실무적으로는 **base → board override → vendor closed** 우선순위로 읽는 게 맞다.

2. `common-android14-6.1.xml`
   - `build/kernel`, `kernel/configs`, `kernel/tests`, `prebuilts/clang`, `prebuilts/gcc`까지 커널 빌드 도구체인이 함께 잠긴다.
   - 즉, 커널 문제를 볼 때 소스만이 아니라 **toolchain revision drift**도 함께 점검해야 한다.

3. `common-android14-6.1-override.xml`
   - `xen-virtual-device`를 별도 project로 덮어쓴다.
   - Xen bring-up 이슈(virtio 초기화, xenbus attach)는 이 override 레이어를 먼저 보는 게 빠르다.

4. `proprietary.xml`
   - `modules/imagination/ddk` ← `ddk-km`
   - `modules/renesas/disfwk_fe` ← `disfwk_fe`
   - 오픈소스 커널이 정상이어도, 여기 태그 조합이 틀리면 display/GPU 쪽에서 late-fail이 난다.

## 리스크

- manifest가 분리되어 있어(android vs kernel) 한쪽만 sync/태그 변경 시 재현 불가능 빌드가 쉽게 발생한다.
- Xen override 리비전과 vendor 모듈 태그 간 검증 규칙이 없으면, 부팅 성공 후 기능 실패 형태로 늦게 터진다.
- `EXT_MODULES` 기반 모듈 빌드는 CI에서 명시하지 않으면 로컬에서는 되고 배포 빌드에서는 깨지는 불일치가 생긴다.

## 다음 액션

- Day4에서 `kernel manifest + android manifest` 간 공통/분리 책임을 1페이지 매트릭스로 정리한다.
- 동시에 `ddk-km`, `disfwk_fe` 모듈 로딩 경로(ko 배치, init 시점, 의존 심볼) 체크리스트를 작성한다.
- 최소 재현 명령셋(`repo init/sync`, `BUILD_CONFIG`, `EXT_MODULES`)을 팀 표준 문서로 고정한다.

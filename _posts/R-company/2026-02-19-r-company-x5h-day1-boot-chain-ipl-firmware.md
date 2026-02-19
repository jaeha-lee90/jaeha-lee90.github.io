---
title: "R사 X5H 코드 분석 Day 1 - Boot chain/IPL/firmware 흐름"
author: JaeHa
date: 2026-02-19 09:00:00 -0800
categories: [R사, X5H, Bring-up]
tags: [R사, X5H, Boot, IPL, Xen, Dom0, DomD]
---

30일 계획의 Day 1은 부팅 체인의 **산출물 연결 관계**를 먼저 고정하는 데 집중한다.
핵심은 `full.img`/`boot_artifacts`에 들어가는 파일이 실제 U-Boot bootcmd와 1:1로 대응되는지 확인하는 것이다.

## 핵심 요약

- X5H 초기 부팅은 IPL(외부 SDK 기반) 이후, U-Boot에서 `xen + xen.dtb + Image + uInitramfs + xenpolicy`를 로드해 `bootm`으로 진입한다.
- `meta-xt-x5h-dev/prod-devel-rcar-gen5.yaml`의 `boot_artifacts`와 `images.full.partitions.boot.items`가 부팅 입력물의 소스 오브 트루스 역할을 한다.
- Bring-up 초기에 가장 중요한 체크는 **파일명/위치 불일치**와 **U-Boot env 변수 오타**다.

## 코드 포인트

- 부팅 아카이브 구성
  - `meta-xt-x5h-dev/prod-devel-rcar-gen5.yaml:317-330`
  - `xen-*.uImage`, `xenpolicy-*`, `*-xen.dtb`, `u-boot-elf-*.srec`, `bl31-*.srec`, `tee-*.srec`를 함께 패키징.
- eMMC boot 파티션 매핑
  - `meta-xt-x5h-dev/prod-devel-rcar-gen5.yaml:337-347`
  - boot 파티션에 `Image/uInitramfs/xen/xenpolicy/xen.dtb`를 명시적으로 배치.
- U-Boot 부팅 커맨드 실제값
  - `meta-xt-x5h-dev/README.md`의 `U-Boot environment` 섹션
  - `bootcmd_tftp`, `bootcmd_mmc0` 모두 `run ...; bootm 0x48080000 0x50000000 0x48000000` 패턴을 사용.

## 리스크

- IPL 교체 대상(`bl31`, `tee`, `u-boot-elf`) 버전이 SDK/보드와 어긋나면 U-Boot 이전 단계에서 무증상 정지 가능.
- `xen.dtb` 이름(예: `r8a78000-ironhide-xen.dtb` vs `xen.dtb`) 불일치 시 `fdt addr` 이후 연쇄 실패.
- TFTP/NFS 모드에서 `serveraddr`, `nfs_domd_dir` 미설정/오설정이면 DomD 루트 마운트 실패로 부팅 지연 또는 패닉.

## 다음 액션

- Day 2에서 Xen 관점(`Dom0/DomD` 초기화 경로)으로 넘어가기 전에, 아래를 체크리스트화:
  1. `boot_artifacts` 산출물 파일명과 U-Boot `*_load` 변수명 완전 일치 확인
  2. `full.img` boot 파티션 실파일 목록 검증
  3. TFTP/eMMC 두 부팅 시나리오의 공통 실패 로그 포인트(bootm 직후) 수집 기준 정의

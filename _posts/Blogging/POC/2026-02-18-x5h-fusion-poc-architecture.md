---
title: X5H Fusion POC 아키텍처 정리
author: JaeHa
date: 2026-02-18 21:20:00 -0800
categories: [R사, POC]
tags: [Xen, Android, Linux, R사, Gen5, IVI]
pin: false
---

이번 데모의 핵심은 **Xen 위에 Android만 있는 구조가 아니라, Linux 도메인과 Android 도메인이 함께 동작하는 멀티 도메인 아키텍처**라는 점이다.

## 1) 전체 구조

![X5H Fusion POC Architecture](/assets/img/poc/x5h_fusion_poc_architecture.png)

- Boot Chain: IPL → BL31 → OP-TEE → U-Boot
- Hypervisor: Xen 4.19
- Domain 구성
  - Dom0 (Linux): 제어/오케스트레이션
  - DomD (Linux): 서비스/백엔드 도메인
  - DomA (Android 15): 앱/프레임워크 도메인
  - DomU (Linux): optional 게스트

## 2) 저장소(Repo) 매핑

### Manifest 계층
- `android-manifest`: Android 전체 소스 구성 고정
- `android-kernel-manifest`: Kernel + 외부 모듈 구성 고정

### 기능 계층
- `android_device`: xenvm_trout device 설정/HAL/init
- `ddk-km`: GPU Kernel Module
- `ddk-um-bin`: GPU User-space binary
- `disfwk_fe`: Display front-end driver
- `gen5-prebuilts`: firmware / IPL / proprietary prebuilts

### 통합 빌드 계층
- `meta-xt-gen5-platform`
- `meta-xt-x5h-dev`

## 3) 빌드/산출물

- 통합 빌드: Yocto + Moulin + Ninja
- 주요 산출물
  - `full.img`
  - `android_only.img`
  - `boot_artifacts`

## 4) 카메라 앱 배치 권장

데모/양산 모두를 고려하면 아래가 기본 권장안:

- **DomA(Android)**: Camera App(UI, UX, Android API)
- **DomD(Linux)**: 디바이스 제어/스트림 백엔드
- 도메인 간 통신: virtio/IPC

즉, **앱은 DomA**, **저수준 제어는 DomD**로 분리하는 게 안정적이다.

## 5) 이번 세션 결정사항

- private repo 9개 접근 및 1차 구조 분석 완료
- 데모용 아키텍처 다이어그램 SVG/PNG 생성
- GitHub Pages에 지속 정리 방식으로 운영 시작

---

필요하면 다음 글에서 아래를 이어서 정리:
- DomD vs DomU 역할 분해
- Bring-up 최소 경로
- 빌드 실패 시 트러블슈팅 체크리스트

---
title: Renesas X5H Fusion POC Repo Study Notes
author: JaeHa
date: 2026-02-18 22:35:00 -0800
categories: [Renesas, POC]
tags: [Renesas, X5H, Xen, Android, Yocto, Repo-Analysis]
pin: false
---

이번에는 Renesas partner GitLab의 X5H Fusion POC 관련 저장소들을 클론하고, 구성/역할/리스크를 빠르게 분석했다.

## 분석 대상 (총 9개)

- android-kernel-manifest
- android-manifest
- android_device
- ddk-km
- ddk-um-bin
- disfwk_fe
- gen5-prebuilts
- meta-xt-gen5-platform
- meta-xt-x5h-dev

## 큰 구조

프로젝트는 크게 3계층으로 보였다.

1) **Manifest 계층**
- `android-manifest`: Android 소스 구성 고정
- `android-kernel-manifest`: Kernel + 외부 모듈 구성 고정

2) **구성요소 계층**
- `android_device`: device/HAL/init 설정
- `ddk-km`: GPU kernel module
- `ddk-um-bin`: GPU user-space binary (prebuilt)
- `disfwk_fe`: display FE driver
- `gen5-prebuilts`: firmware/IPL 등 proprietary prebuilts

3) **통합 빌드 계층**
- `meta-xt-gen5-platform`, `meta-xt-x5h-dev`
- Yocto + Moulin + Ninja 기반 통합 빌드

## 런타임 아키텍처 핵심

Xen 위에 Android만 있는 구조가 아니라, Linux 도메인과 Android 도메인이 함께 동작한다.

- Dom0 (Linux): 제어/오케스트레이션
- DomD (Linux): 서비스 도메인
- DomA (Android): 앱/UI/프레임워크
- DomU (Linux): optional

즉, **Linux + Android 혼합 멀티도메인 아키텍처**가 핵심이다.

## 카메라 앱 배치 관점

데모 기준 권장:
- App/UI: DomA
- 디바이스 저수준 제어/백엔드: DomD
- 통신: virtio/IPC

PoC 속도 우선이면 DomA 단일화도 가능하지만, 양산 관점에서는 DomA/DomD 분리가 안정적이다.

## 리스크/체크포인트

- `disfwk_fe` 문서가 템플릿 상태라 온보딩 난이도 증가 가능
- prebuilt 의존성(`ddk-um-bin`, `gen5-prebuilts`)이 커서 라이선스/배포 관리 중요
- 레포 간 SDK/브랜치 기준선 문서 정합성 점검 필요

## 산출물

- 아키텍처 다이어그램(SVG/PNG) 생성 및 게시
- GitHub Pages 레포에 POC 정리 포스트 누적 운영 시작

---

다음 예정:
- DomD vs DomU 역할 상세화
- Bring-up 최소 경로
- 빌드/부팅 트러블슈팅 체크리스트

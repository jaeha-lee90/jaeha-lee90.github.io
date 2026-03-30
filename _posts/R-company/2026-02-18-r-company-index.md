---
title: R사 Notes Index
author: JaeHa
date: 2026-02-18 23:05:00 -0800
categories: [R사]
tags: [R사, X5H, POC]
pin: true
---

이 블로그는 지금부터 **R사 관련 내용만** 관리합니다.

## 목록

1. [R사 X5H 코드 분석 30일 계획](/posts/r-company-30day-plan/)
2. [X5H Fusion POC 아키텍처 정리](/posts/x5h-fusion-poc-architecture/)
3. [R사 X5H Fusion POC Repo Study Notes](/posts/renesas-x5h-repo-study-notes/)
4. [R사 X5H 코드 분석 Day 1 - Boot chain/IPL/firmware 흐름](/posts/r-company-x5h-day1-boot-chain-ipl-firmware/)
5. [R사 X5H Day2 - android-manifest 프로젝트 맵핑 스터디](/posts/r-company-x5h-day2-android-manifest-project-map/)
6. [R사 X5H Day3 - android-kernel-manifest와 Xen 모듈 체인 분석](/posts/r-company-x5h-day3-kernel-manifest-xen-module-chain/)
7. [R사 X5H Day4 - Manifest 책임 매트릭스와 벤더 모듈 로딩 경로](/posts/r-company-x5h-day4-manifest-responsibility-matrix-module-loading/)
8. [R사 X5H Day5 - android_device init/ueventd/BoardConfig 브링업 크리티컬 포인트](/posts/r-company-x5h-day5-android-device-init-ueventd-boardconfig/)
9. [R사 X5H Day6 - 기본 부팅 실패 포인트 체크리스트](/posts/r-company-x5h-day6-boot-failure-points-checklist/)
10. [R사 X5H Day7 - 1주차 요약과 브링업 트리아지 체크리스트](/posts/r-company-x5h-day7-week1-summary-triage-checklist/)
11. [R사 X5H Day8 - ddk-km 구조와 핵심 모듈 의존 관계](/posts/r-company-x5h-day8-ddk-km-core-modules-dependency/)
12. [R사 X5H Day9 - ddk-um-bin 의존성과 KM/UM 버전 정합 실패 패턴](/posts/r-company-x5h-day9-ddk-um-bin-dependency-version-compatibility/)
13. [R사 X5H Day10 - disfwk_fe 인터페이스/초기 표시 경로 분석](/posts/r-company-x5h-day10-disfwk-fe-interface-flow/)
14. [R사 X5H Day11 - Display Ownership (DomD vs DomA) 경계 고정](/posts/r-company-x5h-day11-display-ownership-domd-vs-doma/)
15. [R사 X5H Day12 - Display 성능 병목 후보 (Copy/Sync)](/posts/r-company-x5h-day12-copy-sync-bottleneck-candidates/)
16. [R사 X5H Day13 - 디버깅 포인트와 로그 전략](/posts/r-company-x5h-day13-debug-points-log-strategy/)
17. [R사 X5H Day14 - 2주차 요약: 브링업 게이트와 우선 수정축](/posts/r-company-x5h-day14-week2-summary-bringup-gate/)
18. [R사 X5H Day15 - Virtio 채널 맵 정리 (blk/net/serial/audio)](/posts/r-company-x5h-day15-virtio-channel-map-blk-net-serial-audio/)
19. [R사 X5H Day16 - DomD 서비스 책임 경계 (소유 채널/재시작 범위/실패 전파)](/posts/r-company-x5h-day16-domd-service-ownership-restart-failure-propagation/)
20. [R사 X5H Day17 - 채널 실패 주입 시나리오와 전파 레벨 SLO](/posts/r-company-x5h-day17-channel-failure-injection-propagation-slo/)
21. [R사 X5H Day18 - 복구시간 계측 훅 표준화 (Launcher/Healthcheck)](/posts/r-company-x5h-day18-recovery-metrics-hook-healthcheck-launcher/)
22. [R사 X5H Day19 - 채널별 복구시간 기준선 수립 (p95 Baseline)](/posts/r-company-x5h-day19-channel-baseline-p95-calibration/)
23. [R사 X5H Day20 - Fault Taxonomy 표준화와 주입 커버리지 고정](/posts/r-company-x5h-day20-fault-taxonomy-injection-coverage/)
24. [R사 X5H Day21 - 3주차 요약: Recovery Gate Hardening](/posts/r-company-x5h-day21-week3-summary-recovery-gate-hardening/)
25. [R사 X5H Day22 - meta-xt-gen5-platform 빌드 흐름과 실패 절단면](/posts/r-company-x5h-day22-meta-xt-gen5-platform-build-flow/)
26. [R사 X5H Day23 - meta-xt-x5h-dev 오버레이 경계와 우선순위 충돌 제어](/posts/r-company-x5h-day23-meta-xt-x5h-dev-overlay-boundary/)
27. [R사 X5H Day24 - 산출물(full.img/android_only.img) 관리와 재현성 게이트](/posts/r-company-x5h-day24-image-artifact-full-android-only-governance/)
28. [R사 X5H Day25 - 브랜치/태그 운영 정책과 이미지 등급 게이팅](/posts/r-company-x5h-day25-branch-tag-policy-image-grade/)
29. [R사 X5H Day26 - CI 게이트 DAG와 이미지 승격 제어면 고정](/posts/r-company-x5h-day26-ci-gate-dag-promotion-control/)
30. [R사 X5H Day27 - Boot-smoke/Healthcheck SLO 임계치 고정](/posts/r-company-x5h-day27-boot-smoke-healthcheck-slo-thresholds/)
31. [R사 X5H Day28 - 4주차 요약: SLO 운영 규칙과 Override 거버넌스](/posts/r-company-x5h-day28-week4-summary-slo-operations-override-governance/)
32. [R사 X5H Day29 - 최종 리스크 레지스터와 릴리스 준비도 판정](/posts/r-company-x5h-day29-final-risk-register-release-readiness/)

## 운영 원칙

- R사/X5H/Xen/Android/Yocto 관련 내용만 게시
- 회의 메모는 결정사항/리스크/TODO 중심으로 요약
- 신규 글은 이 인덱스에 계속 추가

---
title: R사 X5H 코드 분석 30일 계획
author: JaeHa
date: 2026-02-19 01:20:00 -0800
categories: [R사, Plan]
tags: [R사, X5H, Bring-up, 30days]
pin: true
---

초기 브링업 중요도를 기준으로 30일 분석 일정을 분할한다.

## Day 1-7 (Bring-up Core)
1. Boot chain/IPL/firmware 흐름
2. Xen 부팅 경로 + Dom0/DomD 초기화
3. Android manifest 구조/동기화 전략
4. Kernel manifest + 외부 모듈 로딩
5. android_device의 init/ueventd/BoardConfig
6. 기본 부팅 실패 포인트 정리
7. 첫 주 요약 + 체크리스트

## Day 8-14 (Display/GPU)
8. ddk-km 구조/핵심 모듈
9. ddk-um-bin 의존성과 버전 정합
10. disfwk_fe 인터페이스/흐름
11. display ownership (DomD vs DomA)
12. 성능 병목 후보 (copy/sync)
13. 디버깅 포인트와 로그 전략
14. 2주차 요약

## Day 15-21 (Domain Integration)
15. virtio 채널 정리 (blk/net/serial/audio)
16. DomD 서비스 경계 정의
17. DomA 앱 배치 가이드
18. 카메라 앱 분리안 (PoC/양산)
19. 장애 복구 전략 (watchdog/reconnect)
20. 보안 경계/권한 최소화
21. 3주차 요약

## Day 22-30 (Productization)
22. meta-xt-gen5-platform 빌드 흐름
23. meta-xt-x5h-dev 차이/역할
24. 산출물(full.img/android_only.img) 관리
25. 브랜치/태그 운영 정책
26. CI/CD 안정화 포인트
27. 릴리스 체크리스트 초안
28. 테스트 매트릭스 초안
29. 최종 리스크 정리
30. 종합 아키텍처/운영 가이드 완성

---

운영 방식:
- 매일 1개 주제로 짧고 깊게 분석
- 글 끝에 "결정사항 / 리스크 / 다음 액션" 고정
- 누적 결과는 인덱스에서 추적

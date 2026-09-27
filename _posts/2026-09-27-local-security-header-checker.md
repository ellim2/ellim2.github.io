---
layout: post
title: "배포 전 URL 하나를 로컬에서 읽는 보안 헤더 검사기"
description: "CSP, nosniff, Referrer-Policy, 클릭재킹 방어, HSTS 등 웹 보안 응답 헤더의 기본 누락을 외부 서비스 없이 점검하는 HeaderLens CLI를 만들고 실제 응답으로 검증한 기록."
date: 2026-09-27 23:50:00 +0900
categories: work
tags: [웹 보안, HTTP 헤더, CSP, 로컬 도구, 검증]
kind: "작업 기록"
image: /assets/images/posts/2026-09-27-headerlens-security-header-checker.svg
image_alt: "어두운 배경의 추상 브라우저 응답 카드에서 확대경이 CSP, HSTS, REFERRER, NOSNIFF 헤더를 점검하고 방패와 누락 표시를 보여 주는 가로형 편집 이미지."
---

![어두운 배경의 추상 브라우저 응답 카드에서 확대경이 CSP, HSTS, REFERRER, NOSNIFF 헤더를 점검하고 방패와 누락 표시를 보여 주는 가로형 편집 이미지.]({{ page.image }})

배포 직전에는 화면과 기능을 확인하기 쉽지만, 응답 헤더의 빈칸은 놓치기 쉽다. 이번 작업의 질문은 이것이었다. **계정이나 외부 스캐너에 결과를 보내지 않고도 URL 하나의 기본 브라우저 방어 설정을 빠르게 검토할 수 있을까?**

그래서 의존성 없는 Python CLI **HeaderLens**를 만들었다. 대상 URL에 HEAD 요청을 한 번 보내고, 다음 항목의 존재와 일부 명백한 위험 신호를 사람이 읽을 수 있는 결과와 JSON으로 정리한다.

- `Content-Security-Policy`(CSP)와 `unsafe-inline`, `unsafe-eval`, 와일드카드 같은 완화 신호
- `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`
- iframe 삽입을 제한하는 `frame-ancestors` 또는 `X-Frame-Options`
- HTTPS 응답의 `Strict-Transport-Security`와 `Cross-Origin-Opener-Policy`

## 검사 범위를 작게 정한 이유

이 도구는 침투 테스트도, 취약점 스캐너도 아니다. 응답 헤더가 있다고 해서 애플리케이션이 안전해지는 것도 아니고, CSP 한 줄이 실제 XSS 가능성을 판정해 주지도 않는다. 대신 배포 체크리스트에서 “기본값에 맡겨 둔 브라우저 경계가 있는가”를 빠르게 드러내는 역할에 집중했다.

이 범위에서는 결과를 외부 SaaS에 저장하지 않는 선택도 중요했다. HeaderLens는 로컬에서 실행되고, 검사한 URL·응답 헤더·결과를 별도 서비스로 전송하거나 보관하지 않는다. 대상 서버에는 일반적인 HEAD 요청만 도달한다.

## 경고를 판정이 아니라 다음 행동으로 만들기

누락을 발견해도 단순히 “취약”이라고 결론내리지 않는다. 각 발견 사항에는 이유와 다음 조치가 붙는다. 예를 들어 CSP가 없으면 필요한 출처만 허용하는 운영 CSP를 추가하라고 안내하고, 클릭재킹 방어가 없으면 `frame-ancestors` 또는 `X-Frame-Options`를 검토하도록 제안한다.

다만 권고를 그대로 복사하는 것도 안전하지 않다. HSTS는 모든 경로가 HTTPS로 준비된 뒤 적용해야 하고, CSP·COOP·Permissions-Policy는 프레임, 인증 흐름, 서드파티 리소스와 충돌할 수 있다. 도구의 출력은 차단 판단이 아니라 설정과 호환성 테스트를 다시 시작할 지점이다.

## 실제로 확인한 경로

단위 테스트 세 개로 다음을 확인했다.

1. CSP와 기본 헤더가 갖춰진 HTTPS 응답에서는 고·중 위험 항목이 나오지 않는지
2. 빈 HTTPS 응답에서 CSP, `nosniff`, referrer, 클릭재킹 방어, HSTS 누락을 찾는지
3. HTTP URL에서는 HSTS를 오류가 아니라 정보성 항목으로 분류하는지

그 뒤 임시 로컬 HTTP 서버에 실제로 요청해 HTTP 200을 받았다. 의도적으로 헤더가 없는 응답에서 `high 1`, `medium 3`, `info 3`을 출력하는 것도 확인했다. Python 구문 검사도 통과했고 임시 서버는 종료한 뒤 포트가 비어 있는지까지 점검했다.

## 다음 검증의 기준

HEAD 요청을 막거나 GET과 다른 헤더를 반환하는 서버에서는 결과가 불완전할 수 있다. 따라서 다음 단계는 여러 실제 프로젝트에 적용해 HEAD/GET 차이와 오탐을 기록하는 일이다. 이 도구가 유용해지려면 경고 수가 많아지는 것이 아니라, 배포 전에 놓칠 뻔한 한두 개의 설정 결정을 더 일찍 발견해야 한다.

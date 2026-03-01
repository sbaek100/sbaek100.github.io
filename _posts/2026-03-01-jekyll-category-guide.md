---
title: "Jekyll 블로그 카테고리 계층 구조 설정하기"
date: 2026-03-01 15:00:00 +0900
categories: [Blog, 프로젝트실습]
tags: [jekyll, github-pages,프로젝트실습]
---

## 오늘 배운 내용

Chirpy 테마에서 카테고리는 배열의 순서로 계층을 결정한다.

## 카테고리 설정 방법

포스트 상단 Front Matter에서 `categories` 항목을 수정한다.
```yaml
# 1단계
categories: [보안]

# 2단계 (상위 > 하위)
categories: [보안, 웹해킹]

# 3단계
categories: [보안, 웹해킹, DVWA]
```

## 적용 결과

Categories 메뉴에서 아래처럼 계층 구조로 표시된다.
```
보안
  └ 웹해킹
      └ DVWA
```

## 정리

- 배열 첫 번째 값이 최상위 카테고리
- 배열 순서대로 하위 카테고리 생성
- 최대 3단계까지 권장

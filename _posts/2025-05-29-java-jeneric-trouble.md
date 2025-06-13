---
title: "[Java] Java 제네릭과 정적 메소드의 타입 추론 문제 해결하기"
excerpt: "빌더 패턴 적용중, 정적 메소드 뒤에 제네릭을 넘겨주지 못하는 문제를 알아보자"

categories: # 카테고리 설정
  - Java
tags: # 포스트 태그
  - [Java, 제네릭, Builder, Spring, DeepDive]

permalink: /java/generic # 포스트 URL

toc: true # 우측에 본문 목차 네비게이션 생성
toc_sticky: true # 본문 목차 네비게이션 고정 여부

date: 2025-05-29 # 작성 날짜
last_modified_at: 2025-05-29 # 최종 수정 날짜
comments: true
---

---

## 들어가며

Spring에서 REST API를 설계하다 보면, 모든 응답을 일관적으로 응답하고 싶어진다. 
이를 위해 ApiResponse<T>라는 공통 응답클래스를 만들고 싶었고 모든 응답이 Response<ApiResponse<T>>형식을 사용해야한다.

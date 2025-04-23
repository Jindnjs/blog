---
title: "Java 이중 반복문은 좋은구조일까? — while(true) 속 메뉴 로직 깔끔하게 짜기"
excerpt: "이중 반복문 구조에서 흔히 겪는 문제와 그 해결법을 여러 가지 방식으로 비교해보자"

categories: # 카테고리 설정
  - Project
tags: # 포스트 태그
  - [Java, Project]

permalink: /java/03/ # 포스트 URL

toc: true # 우측에 본문 목차 네비게이션 생성
toc_sticky: true # 본문 목차 네비게이션 고정 여부

date: 2025-04-23 # 작성 날짜
last_modified_at: 2025-04-23 # 최종 수정 날짜
---
---

## 들어가며

계산기 프로젝트를 작성하면서 생긴 문제를 정리해보았다.  

---

## 왜 이중 반복문을 사용할까?

프로그램을 작성하다보면, 특히 콘솔기반의 자판기, 은행등의 토이프로젝트는 사용자에게 메뉴를 보여주고 계속 입력을 받기위한 인터페이스를 구현하다보면 `while(true)`와 같은 무한 반복문을 사용한다.  

코드
```java
while (true) {
    System.out.println("1. 계산하기");
    System.out.println("2. 종료");
    int choice = scanner.nextInt();

    switch (choice) {
        case 1:
            // 계산기 로직
            break;
        case 2:
            System.out.println("종료합니다.");
            break;
    }
}
```
하지만 계산하기 같은 동작 안에서도 다시 반복 입력을 받아야 할 경우, 자연스럽게 이중 반복문이 생긴다.

---

## 흔히 겪는 문제: 이중 반복문에서 빠져나오기

예를 들어 계산기 메뉴 안에서 또 반복적으로 연산을 입력받고 싶다면 이렇게 된다

```java
while (true) {
    System.out.println("1. 계산기");
    System.out.println("2. 종료");
    int menu = scanner.nextInt();

    switch (menu) {
        case 1:
            while (true) {
                System.out.println("덧셈할 두 수 입력:");
                int a = scanner.nextInt();
                int b = scanner.nextInt();
                System.out.println("결과: " + (a + b));
                
                System.out.println("계속할까요? (y/n)");
                if (scanner.next().equals("n")) break;
            }
            break;
        case 2:
            break; // 이건 내부 switch만 탈출
    }
}
```
문제는 사용자가 계산기를 끝내고 프로그램도 종료하고 싶을 때 밖의 while문을 벗어날 방법이 마땅치 않다는 것이다. break는 현재 루프만 빠져나오기 때문이다.

---

## 해결 방안 1: 플래그 변수 사용

```java
boolean run = true;
while (run) {
    System.out.println("1. 계산기");
    System.out.println("2. 종료");
    int menu = scanner.nextInt();

    switch (menu) {
        case 1:
            boolean calc = true;
            while (calc) {
                // 계산 로직
                if (scanner.next().equals("n")) {
                    calc = false;
                }
            }
            break;
        case 2:
            run = false;
            break;
    }
}
```

✅ 장점  
> - 직관적이고 가독성 좋음
> - break 대신 조건으로 빠져나감

❌ 단점
> - 플래그 변수 많아지면 코드가 지저분해짐

---

## 해결 방안 2: 메서드 분리로 구조 개선

```java
public static void runCalculator() {
    Scanner scanner = new Scanner(System.in);
    while (true) {
        // 계산기 로직
        if (scanner.next().equals("n")) break;
    }
}
```
while (true) {
    System.out.println("1. 계산기");
    System.out.println("2. 종료");
    int menu = scanner.nextInt();

    switch (menu) {
        case 1:
            runCalculator();
            break;
        case 2:
            return; // 메인 메서드 종료
    }
}

✅ 장점
	•	구조가 명확해지고 테스트도 쉬움
	•	루프 탈출보다 더 명확하게 동작 제어 가능

❌ 단점
	•	단순한 프로젝트에선 과도할 수 있음


---

## 해결 방안 3: 라벨을 활용한 break

```java
outer:
while (true) {
    System.out.println("1. 계산기");
    System.out.println("2. 종료");
    int menu = scanner.nextInt();

    switch (menu) {
        case 1:
            while (true) {
                System.out.println("계산기 작동 중...");
                if (scanner.next().equals("exitAll")) {
                    break outer; // 바깥 루프까지 탈출
                } else {
                    break; // 내부만 탈출
                }
            }
            break;
        case 2:
            break outer;
    }
}
```

✅ 장점
	•	복잡한 탈출 조건일 때 유용
	•	코드 줄 수는 가장 적음

❌ 단점
	•	라벨 사용은 권장되지 않음 (가독성 저하, 유지보수 어려움)


---

## 각 방법의 장단점 비교

---

## 마무리 및 추천 방식


---

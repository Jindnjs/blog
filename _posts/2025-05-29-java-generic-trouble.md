---
title: "[Java] Java 제네릭과 정적 메소드의 타입 추론 문제 해결하기"
excerpt: "Java 에서 빌더 패턴 적용시, static 메서드와 제네릭 타입 추론 문제로 인해 발생하는 문제를 알아보자"

categories: # 카테고리 설정
  - Java
tags: # 포스트 태그
  - [Java, 제네릭, 빌더패턴, Spring, DeepDive]

permalink: /java/generic # 포스트 URL

toc: true # 우측에 본문 목차 네비게이션 생성
toc_sticky: true # 본문 목차 네비게이션 고정 여부

date: 2025-05-29 # 작성 날짜
last_modified_at: 2025-05-29 # 최종 수정 날짜
comments: true
---

---

## 들어가며

Spring으로 API를 설계하다 보면, 일관된 응답 형식이 필요할 때가 많다.  
이를 `ApiResponse<T>`라는 공통 응답 클래스를 만들어, 모든 응답을 `ResponseEntity<ApiResponse<T>>` 형태로 반환하고자 했다.

이 과정에서, 각 컨트롤러에서 매번 `ResponseEntity`를 리턴하고, 내부의 제네릭 타입을 맞추기 위해 `ApiResponse` 객체를 반복적으로 생성해야 반복 작업이 생겨, 중복을 줄이고 코드를 더 간결하게 만들 필요성이 느껴졌다.

이를 위해 `ApiResponse` 클래스에 빌더 패턴을 적용하려 했는데, 이 과정에서 Java의 제네릭과 static 메서드의 특성 때문에 타입 추론 문제가 발생했다.  
이 글에서는 그 과정에서 겪은 내용과 해결법을 정리하고자 한다.

---

## 빌더 패턴으로 API 응답을 설계하고 싶었던 이유

Spring에서 API 응답을 반환할 때, 가장 기본적으로 사용하는 방법은 ResponseEntity를 직접 활용하는 것이다. 예를 들어, 아래와 같이 작성할 수 있다.

```java
// <h4>기존 ResponseEntity 사용 예시</h4>
return ResponseEntity
        .status(HttpStatus.OK)
        .body(dto);
```

하지만 단순히 데이터만 반환하는 것이 아니라, 성공 여부, 코드, 메시지 등 공통 필드를 포함한 일관된 응답 구조가 필요할 경우도 존재한다. 그래서 아래처럼 ApiResponse라는 래퍼 클래스를 만들어 사용하게 된다.

<h4>ApiResponse 래퍼를 활용한 공통 응답 예시</h4>

```java
return ResponseEntity
        .status(HttpStatus.OK)
        .body(new ApiResponse<>(true, "요청 성공", dto));
```

이렇게 일관된 포맷으로 공통 응답을 처리할 수 있지만, 컨트롤러마다 ApiResponse 객체를 직접 생성하여 DTO를 감싸고, `status`·`message`·`data` 같은 공통 필드를 세팅해야 하므로 컨트롤러마다 보일러플레이트 코드가 쌓인다.

그래서 ResponseEntity의 직관적인 체이닝 문법은 그대로 두면서, 공통 응답 포맷을 강제할 수 있는 빌더 패턴을 적용하고 싶었다. 즉, 아래와 같이 한 줄로 간결하게 작성하는 것이 목표였다.

```java
return ApiResponse.status(BaseCode.SUCCESS).body(dto);

public enum BaseCode {
    SUCCESS(HttpStatus.OK, "요청 성공"),
    INVALID_REQUEST(HttpStatus.BAD_REQUEST, "잘못된 요청");
    private final HttpStatus httpStatus;
    private final String message;
    // ...생성자 및 getter
}
```

아래에서는 ApiPresponse 클래스에 빌더 패턴을 적용할때 발생한 문제에 대하여 원인과 해결방법을 정리해보겠다.

---

## 문제상황

아래와 같이 ApiResponse 클래스에 빌더 패턴을 적용하여 체이닝 방식으로 사용할 수 있도록 설계했다.  
`status()` 메서드로 BaseCode를 받아 ApiResponse 객체를 생성하고, `body()` 메서드로 실제 데이터를 설정한 후 최종적으로 ResponseEntity를 반환하는 구조이다.

<h4>ApiResponse 클래스</h4>

```java
public class ApiResponse<T> {

    private int statusCode;
    private String message;
    private T body;

    // ... 생성자

    //Builder
    public static <T> Builder<T> status(BaseCode baseCode) {
        return new Builder<>(baseCode);
    }

    public static class Builder<T> {
        private final ApiResponse<T> response;

        private Builder(BaseCode baseCode) {
            response = new ApiResponse<>();
            response.statusCode = baseCode.getHttpStatus().value();
            response.message = baseCode.getMessage();
        }
        
        public ResponseEntity<ApiResponse<T>> body(T body){
            return ResponseEntity
                    .status(response.statusCode)
                    .body(new ApiResponse<>(response.statusCode,response.message,body));
        }
        // ... 생략
    }
}
```

이 ApiResponse 클래스는 빌더 패턴을 적용하여 체이닝 방식으로 사용할 수 있도록 설계했다. `status()` 메서드로 BaseCode를 받아 ApiResponse 객체를 생성하고, `body()` 메서드로 실제 데이터를 설정한 후 최종적으로 ResponseEntity를 반환하는 구조이다.

하지만 이 클래스를 실제로 사용할 때 문제가 발생했다. 아래와 같이 사용하려고 하면,

<h4>원하는 사용 형태</h4>

```java
return ApiResponse.status(BaseCode.SUCCESS).body(dto);
```

다음과 같은 오류가 발생하였다.

<figure>
	<a href="/assets/images/java-generic-trouble/image.png"><img src="/assets/images/java-generic-trouble/image.png"></a>
</figure>

```
필요한 타입: ResponseEntity<ApiResponse<String>>
제공된 타입: ResponseEntity<ApiResponse<Object>>
```

오류를 해결하기 위해서는 아래와 같은 형식으로 사용해야하는 불편함이 발생하였다.

<h4>제네릭 타입을 명시해야 하는 형태</h4>

```java
return ApiResponse.<Dto>status(BaseCode.SUCCESS).body(dto);
```

이렇게 매번 `.<Void>`나 `.<UserDto>` 같은 제네릭 타입을 명시해야 하는 것은 실용적이지 않고, 코드도 지저분해진다. 특히 `Void` 타입의 경우 더욱 불편하다.

문제의 원인을 자세히 알아보자.

---

## 정적 메서드의 타입 추론 문제

이 문제의 원인은 Java에서 static 메서드가 클래스 레벨에서 실행되기 때문에, 인스턴스의 제네릭 타입 정보를 알 수 없다는 점이다. 즉, 아래와 같은 메서드 시그니처를 사용하면,

<h4>문제가 되는 static 제네릭 메서드</h4>

```java
public static <T> ApiResponse<T> status(BaseCode baseCode)
```

static 메서드 호출 시점에 타입 파라미터 T를 명시적으로 지정하지 않으면, 이후 체이닝에서 타입 추론이 제대로 동작하지 않는다.

이 문제가 발생하는 이유를 자세히 단계별로 살펴보자.

---

### 1단계: static 메서드 호출 시점

```java
ApiResponse.status(BaseCode.SUCCESS)
```

위와 같이 static 메서드가 호출되는 시점에서 컴파일러는 `status` 메서드의 제네릭 타입 T를 추론해야 한다. 

왜냐하면 인스턴스 메서드의 경우, 아래와 같이 초기화 시점에 타입 T에대한 추론이 가능하다.

```java
// 인스턴스 메서드 - 타입 정보가 있음
ApiResponse<String> response = new ApiResponse<>();
response.someMethod(); // T가 String으로 이미 결정됨
```

하지만 static 메서드는 클래스 레벨에서 실행되므로, 인스턴스의 타입 정보가 없다.

```java
// static 메서드 - 타입 정보가 없음
ApiResponse.status(BaseCode.SUCCESS); // T가 무엇인지 알 수 없음
```

즉 static 메서드가 인스턴스로 부터 타입 정보를 참조 할 수 없기 때문에,  컴파일러는 T를 `Object`로 추론하는 것이 가장 안전한 선택이다.

이로 인해 컴파일러는 T를 가장 상위 타입인 `Object`로 추론한다.

---

### 2단계: Builder 생성

```java
public static <T> Builder<T> status(BaseCode baseCode) {
    return new Builder<>(baseCode);  // T가 Object로 추론됨
}
```

정적 팩토리 메소드에 의해, Builder의 타입 T가 Object로 결정되었으므로
`Builder<Object>`가 생성된다.

---

### 3단계: body 메서드 호출

```java
public ResponseEntity<ApiResponse<T>> body(T body){...}

.body(dto) 
```

`body(T body)` 메서드 사용 시점에 T는 이미 `Object`로 결정되어 있다. 따라서 `body(Object body)`가 되어, Dto를 전달할때 타입 불일치가 발생한다.

---

### 4단계: 최종 반환 타입

Builder가 `Object` 타입으로 생성되었으므로, 최종적으로 `ResponseEntity<ApiResponse<Object>>`가 반환된다. 하지만 원하는 리턴타입은 `ResponseEntity<ApiResponse<Dto>>`이다.

이것이 바로 "필요한 타입: ResponseEntity<ApiResponse<Dto>>, 제공된 타입: ResponseEntity<ApiResponse<Object>>" 에러가 발생하는 이유이다.

결국 아래처럼 타입 파라미터를 명시적으로 지정해야만 한다.

```java
return ApiResponse.<Dto>status(BaseCode.SUCCESS).body(dto);
```

---

## 시도했던 두 가지 해결책

이 문제를 해결하기 위해 두 가지 방법을 시도했다.

### 1. public static Builder<?> status(BaseCode code)</h4>

<div class="notice" markdown="1">

* Builder의 타입 파라미터를 와일드카드(?)로 선언
* 장점: 호출 시 타입 명시가 필요 없음
* 단점: 이후 body(T body)에서 타입 안정성이 떨어지고, 경고가 발생할 수 있음

</div>

### 2. body 메서드에 제네릭 추가

<div class="notice" markdown="1">

* Builder는 <?>로 생성하되, body 메서드에 <U>를 선언하여 타입을 유연하게 처리
* 장점: 다양한 타입의 body를 받을 수 있음
* 단점: 여전히 타입 안정성이 완벽하지 않고, IDE에서 경고가 발생할 수 있음

</div>

---

## 5. 최종 선택한 빌더 패턴 구현

여러 시행착오 끝에, 아래와 같은 형태로 빌더 패턴을 구현했다.

<h4>최종 선택한 ApiResponse 빌더 패턴</h4>

```java
public static Builder<?> status(BaseCode baseCode) {
    return new Builder<>(baseCode);
}

public static class Builder<T> {
    private BaseCode baseCode;

    private Builder(BaseCode baseCode) {
        this.baseCode = baseCode;
    }

    public ResponseEntity<ApiResponse<Void>> body(){
        return ResponseEntity
                .status(baseCode.getHttpStatus())
                .body(new ApiResponse<>(success, baseCode.getMessage(), null));
    }

    public <T> ResponseEntity<ApiResponse<T>> body(T body){
        return ResponseEntity
                .status(baseCode.getHttpStatus())
                .body(new ApiResponse<>(success, baseCode.getMessage(), body));
    }
}
```

이 방식의 장점과 단점은 다음과 같다.

<div class="notice" markdown="1">

<h4>장점</h4>

* 호출 시 타입 파라미터를 명시하지 않아도 됨
* 다양한 타입의 body를 유연하게 처리 가능
* 코드가 간결하고, 실사용에 불편함이 없음

</div>

<div class="notice" markdown="1">

<h4>단점</h4>

* Builder<?>로 생성하므로, 타입 안정성이 100% 보장되지는 않음
* IDE에서 경고가 발생할 수 있으나, 실사용에는 큰 문제 없음

</div>

---

## 6. 결론 및 실무에서의 시사점

이번 경험을 통해, Java의 제네릭과 static 메서드가 결합될 때 타입 추론에 한계가 있다는 점을 알게 되었다. 빌더 패턴을 적용할 때는, "가독성"과 "타입 안정성" 사이에서 현실적인 타협이 필요하다. 결국, 실용성과 코드의 일관성을 우선시해 최종 구현을 선택했다.

"완벽한 타입 안정성"보다는, 실제 사용에서의 편의성과 유지보수성을 고려하는 것이 더 중요할 때가 많다. 앞으로도 이런 고민들 사이에서, 더 나은 설계와 구현을 고민해야겠다는 생각이 든다.

---
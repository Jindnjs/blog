---
title: "[Jwt] JWT에 userId를 넣어도 안전할까? - 보안과 성능 사이의 균형점"
excerpt: "사용자 식별자인 userId를 JWT에 직접 포함시켜도 되는지에 대한 의문이 든다. 
과연 JWT의 특성을 고려했을 때 userId를 포함하는것이 적절할까?"

categories: # 카테고리 설정
  - Jwt
tags: # 포스트 태그
  - [Java, Jwt, Security, DeepDive]

permalink: /security/jwt # 포스트 URL

toc: true # 우측에 본문 목차 네비게이션 생성
toc_sticky: true # 본문 목차 네비게이션 고정 여부

date: 2025-06-10 # 작성 날짜
last_modified_at: 2025-06-10 # 최종 수정 날짜
comments: true
---

---

## 들어가며

개발을 하다 보면 JWT 토큰에 어떤 정보를 담을지 고민하게 된다. 
특히 사용자 식별자인 userId를 JWT에 직접 포함시켜도 되는지에 대한 의문이 든다.   
Long 타입의 순차적 ID는 변조 위험이 있어 UUID를 사용하기도 하지만, UUID는 데이터베이스 성능에 영향을 줄 수 있다.   
이러한 상황에서, 과연 JWT에는 userId와 UUID중 어떤것을 포함시켜야할까?
이 글에서는 ID의 전략과 JWT의 관한 내용을 정리하고, JWT에 userId를 어떻게 포함해야하는지를 알아본다.

---

## ID의 변조 위험성

일반적으로 API 개발에서 Long 타입의 순차적 userId를 URL 경로에 노출하는 것은 보안상 위험하다고 여겨진다.

<h4> 위험한 예시 - 순차적 ID 노출 </h4>

```java
GET /api/users/12345/profile
GET /api/users/12346/profile  
```

위와같은 예시에서, 사용자가 임의로 ID를 입력할때, 다른 사용자의 정보 접근이 가능하다.  
이런 이유로 많은 개발자들이 URL 경로에 노출하는 식별자로 아래와 같이 UUID를 사용한다.

<h4> UUID 사용 예시 </h4>

```java
GET /api/users/550e8400-e29b-41d4-a716-446655440000/profile
```

---

## Long 타입 ID와 UUID의 차이

이러한 UUID를 사용할때, Long 타입 ID와 비교하여 어떤 차이가 있는지 알아보자.

---

### Long 타입 ID

Long타입 ID는 데이터 베이스의 기본키로 활용되는 식별자로, 흔히 `autoIncrement`속성으로 많이 사용한다. 
정수형이라 빠른 인덱싱 효율이 장점이며, Spring JPA Entity의 기본키 전략으로도 많이 사용한다.  
하지만, URL상에 노출시 REST API의 계층 구조 특징상 순차적인 값의 특징으로 예측 가능하여 보안 위험이 있다는 단점이 존재한다.

---

### UUID 타입 ID

UUID는 이러한 Long 타입 ID의 문제점을 보완하기 위한 방법중 하나로, 랜덤한 값을 사용하여 예측 불가능한 점을 바탕으로
보안상 문제를 해결하는 방법이다. 하지만 랜덤한 값의 특성으로 인해, 추가 연산 비용이 들고 기본키로 사용시 인덱스 성능 저하의 문제가 있다.

---

### Long 타입 ID과 UUID의 특정 정리

#### Long 타입 ID의 특징


<div class="notice" markdown="1">
<h4> 장점 </h4>

- 데이터베이스 성능 최적화 (인덱스 효율성)
- 메모리 사용량 적음
- 정렬과 비교 연산 빠름

</div>

<div class="notice" markdown="1">
<h4> 단점 </h4>

- 순차적이라 예측 가능
- URL에 노출시 보안 위험

</div>

#### UUID의 특징

<div class="notice" markdown="1">
<h4> 장점 </h4>

- 예측 불가능성
- 분산 시스템에서 충돌 없음

</div>

<div class="notice" markdown="1">
<h4> 단점 </h4>

- 36바이트 크기 (Long의 4.5배)
- 인덱스 성능 저하 (랜덤한 값으로 인한 페이지 분할)
- 문자열 비교 연산 비용

</div>

---

## JWT란?

이번에는 JWT를 살펴보자. JWT는 정보를 안전하게 전송하기 위한 표준중 하나로, 
토큰 자체에 정보를 담고 있어서 별도의 저장소 없이도 사용자 인증과 정보 교환이 가능하다.

---

### JWT의 구조


<h4> JWT의 구조 </h4>

```bash
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0NSIsImlhdCI6MTYzMjQ2MDgwMH0.signature
[Header]            [Payload]                                    [Signature]
```

JWT는 점(.)으로 구분된 세 부분으로 구성된다. 각 부분에는 아래와 같은 정보를 저장한다.  

<h4> Header: 토큰 타입과 암호화 알고리즘 정보 </h4>

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

<h4> Payload: 실제 전송할 데이터 (Claims) </h4>

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}
```

<h4> Signature: Header와 Payload를 조합해서 만든 서명 </h4>

JWT에서 중요한 점은 Payload 부분이 Base64로 인코딩되어 있을 뿐, 암호화되지 않는다는 것이다.
즉, 클라이언트에서 JWT를 디코딩하면 내부 정보를 볼 수 있다는 것이다.  
그렇다면 클라이언트에서 JWT 내용을 쉽게 볼 수 있고, 심지어 수정도 가능하다면 보안에 문제가 없을까라는 의문이 든다.  
아래에서 JWT의 보안 방법에 대하여 살펴보자.

---

### JWT의 변조 방지


JWT는 변조 방지 기능을 통해, 보안성을 보장한다. 
JWT는  **Signature(서명)** 을 통해서 토큰의 안전성을 확보한다.

<h4> 서명 생성 과정 </h4>

```java
// 1. Header와 Payload를 Base64로 인코딩
String encodedHeader = Base64.encode(header);
String encodedPayload = Base64.encode(payload);

// 2. 둘을 합쳐서 서명 생성
String signature = HMAC_SHA256(encodedHeader + "." + encodedPayload, SECRET_KEY);

// 3. 최종 JWT 생성
String jwt = encodedHeader + "." + encodedPayload + "." + signature;
```

핵심은 **서버만 알고 있는 SECRET_KEY**를 사용한다는 점이다.

<h4> 변조 시도와 차단 </h4>

클라이언트가 userId를 변조하려고 시도하는 상황을 살펴보자.

```javascript
// 악의적인 사용자가 시도할 수 있는 변조
const originalToken = "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0NSJ9.original_signature";

// 1. Payload 부분을 추출하여 수정
const fakePayload = btoa(JSON.stringify({
    "sub": "11",  // userId를 1에서 11로 변조
    "iat": 1516239022
}));

// 2. 변조된 토큰 생성 시도
const fakeToken = "eyJhbGciOiJIUzI1NiJ9." + fakePayload + ".original_signature";
```

만약 클라이언트에서 payload부분을 디코딩하며 변경후, 다시 인코딩하여 저장했다고 가정해보자.
하지만 서버에서 이 토큰을 검증하게된다면 아래와 같은 과정으로 서버에서는 토큰 변조를 확인 가능하다.

```java
public Long getUserIdFromToken(String token) {
    try {
        Claims claims = Jwts.parser()
            .setSigningKey(SECRET_KEY)
            .parseClaimsJws(token)  // 여기서 서명 검증 수행
            .getBody();
            
        return Long.parseLong(claims.getSubject());
    } catch (SignatureException e) {
        // 서명이 일치하지 않음
        throw new UnauthorizedException("토큰이 변조되었습니다");
    } catch (JwtException e) {
        throw new UnauthorizedException("유효하지 않은 토큰입니다");
    }
}
```

위의 코드에서 클라이언트가 Payload를 변조 하였지만, SECRET_KEY 없이는 올바른 서명을 만들 수 없기 때문에,
변조된 토큰은 변조된 토큰의 Payload에 맞는 서명을 생성할수 없고, 이로 인해, 서버가 서명을 검증할때 변조 여부를 확인할 수 있다.

즉, JWT는 **내용은 공개되지만 변조는 불가능한** 구조로 구성되어있다.

---

## 그렇다면 어떤 userId를 JWT에 넣어야 할까?

앞선 내용에서 ID의 두가지 방식과 JWT의 동작방식을 살펴보았다. 이 두가지의 내용을 조합하여 오늘의 주제인 JWT토큰에 
Long타입의 userId를 포함하여도 문제가 없을지를 살펴보자.   

JWT의 방식에서 Long타입의 userId를 포함하여도 변조 가능성은 없지만, 
토큰의 내용을 누구나 볼수 있다는 점에서 데이터베이스의 기본키를 클라이언트에 노출하는것 역시 피하고 싶은 생각이 든다.

JWT에 UUID타입의 userId를 사용한다면, 클라이언트에 노출되어도 내부 시스템의 구조를 유추하기 어려운 장점은 있지만
토큰의 payload 크기를 증가시키고, 인덱싱 및 검색 성능이 떨어질 수 있는 단점이 있다.

하지만 Long 타입의 ID가 정말로 보안에 있어 위험할지는 좀 더 생각해볼 필요가 있다.   
userId가 노출된다는 사실 자체가 불편하게 느껴질 수 있지만 실제로 이 값이 노출되었을 때
공격자가 어떤 행위를 할 수 있는지, 즉 시스템에 어떤 영향을 줄 수 있는지를 파악해 보자

---

## JWT의 userId는 정말 위험할까?

JWT에 포함된 Long 타입의 userId가 외부에 노출되는 상황을 보안 측면에서 자세히 생각해보자.
예를 들어, 다음과 같은 토큰이 있다고 가정해보자.

```json
{
  "sub": "12345",
  "name": "Alice",
  "role": "USER"
}
```

이 경우, 사용자가 userId의 정보를 토큰 디코딩을 통해 확인 가능하다.  
하지만 적절한 인증/인가 과정을 통해 JWT에 userId를 포함시켜도 안전하다 할 수 있다.

JWT에 userId를 포함시키는 것이 안전한 이유는 **토큰 기반 인증의 흐름** 때문이다.

```java
@RestController
public class UserController {
    
    @GetMapping("/api/users/{requestedUserId}/profile")
    public UserProfile getUserProfile(
            @PathVariable Long requestedUserId,
            @AuthenticationPrincipal UserDetails userDetails) {
        
        Long authenticatedUserId = Long.parseLong(userDetails.getUsername());
        
        // 인증된 사용자가 본인의 정보만 요청할 수 있도록 검증
        if (!authenticatedUserId.equals(requestedUserId)) {
            throw new ForbiddenException("본인의 정보만 조회할 수 있습니다");
        }
        
        return userService.getUserProfile(authenticatedUserId);
    }
}
```

위 코드에서, path에 들어온 요청된 리소스의 userId와, 토큰의 userId가 다르다면, 리소스에 접근이 불가하다.  
즉, 토큰의 userId만 검사하는것이 아닌, 토큰의 userId와 리소스의 userId를 비교하는 방식으로 안전하게
인증/인가를 처리한다.

토큰의 userId는 변조가 불가능하고, 임의로 다른 사용자의 userId를 path에 지정하여 리소스를 요청하여도 
본인의 리소스에만 접근이 가능한 것이다.

정리하자면, 아래와 같은 인증/인가 절차로 보안성을 확보한다.

<div class="notice" markdown="1">

1. **인증 우선**: JWT 토큰이 유효한지 먼저 검증
2. **권한 확인**: 토큰의 userId와 요청하는 리소스의 소유자가 일치하는지 확인
3. **최소 권한**: 사용자는 본인의 리소스에만 접근 가능

</div>


## 성능과 보안을 고려한 방법

JWT에는 Long타입의 userID, api의 path에는 UUID를 사용한다면, 외부에 기본키를 노출하는것에 대한 우려를 해소 가능하다.
UUID를 사용하지만, UUID를 api용으로, Long Id를 내부용으로 사용한다.

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;  // 내부 식별자 Long 사용
    
    @Column(unique = true)
    private String publicId = UUID.randomUUID().toString();  // 외부 공개용
}
```

**JWT 구성:**
```java
public String generateToken(User user) {
    return Jwts.builder()
        .setSubject(user.getId().toString())  // 내부 Long ID 사용
        .claim("publicId", user.getPublicId())  // 필요시 public ID 추가
        .setIssuedAt(new Date())
        .setExpiration(new Date(System.currentTimeMillis() + EXPIRATION_TIME))
        .signWith(SignatureAlgorithm.HS256, SECRET_KEY)
        .compact();
}
```

JWT토큰에는 userID를 Long타입으로 사용하고, 필요시에는 UUID도 포함시킨다

<h4> API 설계 </h4>

```java
@GetMapping("/api/users/me/profile")  // 본인 정보는 'me' 엔드포인트
public UserProfile getMyProfile(@AuthenticationPrincipal UserDetails userDetails) {
    Long userId = Long.parseLong(userDetails.getUsername());
    return userService.getUserProfile(userId);
}

@GetMapping("/api/users/{publicId}/profile")  // 타인 정보는 public ID 사용
public UserProfile getUserProfile(@PathVariable String publicId) {
    // 공개 프로필 조회 로직
}
```

publicId에는 UUID가 들어가므로, url을 통한 경로 예측을 막을수 있으며, 이를 통해 보안성을 높인다.
본인 정보 요청에는 토큰의 userId를 통해 본인에게만 자신의 리소스를 응답하도록 하여 안전한 접근제어가 가능하다.

## 방법 비교

| 구분 | Long ID | UUID  | 복합          |
|------|---------|-------|-------------|
| 저장 크기 | 8바이트    | 36바이트 | 8바이트 (주요 키) |
| 인덱스 성능 | 높음      | 보통    | 보통          |
| 보안성 | 안전      | 높음    | 보통          |
| 메모리 사용 | 적음      | 많음    | 적음          |

## 결론

JWT에 Long 타입 userId를 포함시키는 것은 **충분히 안전하다**고 볼 수 있다.

처음에는 Long 타입 userId는 위험하다는 고정관념에 사로잡혀 있었지만, 
JWT의 변조 방지를 알아보고, 여기에 더해 
적절한 인증/인가 로직을 결합하면 보안상 문제가 없다는 것을 깨달았다. 
핵심은 **토큰 기반 인증의 특성을 제대로 활용하는 것**이다.

비록 Long 타입 ID는 예측 가능성이라는 단점이 있지만, 토큰이 없으면 접근이 불가능하고, 
토큰 내 userId와 리소스의 userID 검증 로직이 제대로 구성되어 있다면 실질적인 보안 
문제는 발생하지 않는다는것을 알 수 있다.

UUID는 강력한 보안성을 제공하지만, 모든 상황에서 필수는 아니다. 
특히 데이터베이스 성능이 중요한 서비스에서는 UUID는 인덱싱 성능 저하나 개발 복잡도를 발생 시킬 수 있다.
Long ID + JWT 조합이 더 효율적일 수 있다. 
정말 필요한 경우에만 부분적으로 사용한다면 좋을것이다.

중요한 점은 상황에 맞는 적절한 보안 수준을 찾는 것이다. 
보안은 성능과의 관계가 밀접하다. 보안을 최대한으로 챙기면, 성능은 타협을 보아야할것이다.

물론 Long타입의 userId가 100% 안전하다고 장담할 수 없다.   
하지만 JWT의 무결성, 그리고 userId와 리소스의 userId를 비교하는 인가 로직이 잘 구현되어 있다면,
그 자체만으로도 충분히 높은 수준의 보안성을 확보할 수 있다.

결국, 보안 설계에서 중요한 것은 완벽함이 아니라, 위험을 충분히 관리하면서도 효율적인 구조를 유지하는 것이다.

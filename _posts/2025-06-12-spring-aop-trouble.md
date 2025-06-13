---
title: "[Spring] RequestBody가 AOP에서 읽히지 않는 진짜 이유 - 파라미터 바인딩 시점의 비밀
excerpt: "AOP를 통해 로깅을 하는 도중, request의 body가 읽히지 않는 문제를 발견했다. 
  이에 대한 trouble shooting 과정을 정리해보자."

categories: # 카테고리 설정
  - Spring
tags: # 포스트 태그
  - [Java, Spring, AOP, trouble Shooting, DeepDive]

permalink: /spring/aoptrouble # 포스트 URL

toc: true # 우측에 본문 목차 네비게이션 생성
toc_sticky: true # 본문 목차 네비게이션 고정 여부

date: 2025-06-12 # 작성 날짜
last_modified_at: 2025-06-12 # 최종 수정 날짜
comments: true
---

---

## 들어가며

Spring에서 AOP를 통해 log를 남기는중 컨트롤러에서는 멀쩡히 읽히던 RequestBody가 
AOP에서는 왜 읽히지 않는문제가 발생하였다. 이에 대한 문제를 알아보던중, 
단순히 "InputStream은 한 번만 읽을 수 있다"는 설명만으로는 이해하기 어려운 부분이 발생하였다. 
AOP가 컨트롤러보다 먼저 실행되는데 왜 컨트롤러에서는 값이 정상적으로 파싱되지만, AOP에서는 안되는 걸까? 
이 글에서는 Spring의 요청 처리 과정을 단계별로 분석하며, ArgumentResolver와 AOP의 실행 순서를 명확히 이해해보고자 한다.

---

## 문제 상황과 첫 번째 의문

AOP에서 HTTP 요청의 body를 읽으려고 시도했을 때 AOP내부에서는 body를 읽을 수 없는 문제가 발생하였다.

<h4>컨트롤러 코드</h4>

```java
@RestController
public class UserController {
    
    @PostMapping("/api/users/role")
    public ResponseEntity<String> changeUserRole(@RequestBody UserRoleChangeRequest request) {
        log.info("컨트롤러에서 받은 데이터: {}", request.getUserId());
        return ResponseEntity.ok("Role changed successfully");
    }
}
```

<h4> AOP 코드 (문제 발생) </h4>

```java
@Aspect
@Component
@Slf4j
public class RequestLoggingAspect {
    
    @Around("@annotation(org.springframework.web.bind.annotation.PostMapping)")
    public Object logRequest(ProceedingJoinPoint joinPoint) throws Throwable {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
        // body를 읽으려고 시도
        String body = StreamUtils.copyToString(request.getInputStream(), StandardCharsets.UTF_8);
        log.info("AOP에서 읽은 body: {}", body);
        
        return joinPoint.proceed();
    }
}
```

<h4> 결과 </h4>

```bash
AOP에서 읽은 body: null
컨트롤러에서 받은 데이터: user123
```

AOP에서 HttpServeletRequest의 body값을 getInputStream()으로 읽고, log로 찍어본 결과 null이 나온다.
컨트롤러에서는 @RequestBody에 의해 정상적으로 값이 파싱되어 log에 찍힌다.
왜 컨트롤러에서는 데이터가 제대로 들어왔는데 AOP에서는 값을 읽을수 없는지 의문이 들어 찾아보았다.

---

## HttpServletRequest의 Request Body는 한번만 읽을 수 있다.

검색을 통해 찾아본 결과, HttpServletRequest의 Body는 InputStream을 통해 읽을 경우, 한번만 읽을 수 있다는 점을 발견했다. 
실제 많은 블로그를 통해 `httpservletrequest body`를 검색해보면 한번만 읽을 수 있다는 내용을 많이 찾을 수 있다.

이는, `ServletRequest`의 `getInputStream()`메소드의 문서에서도 확인 가능한 내용이다.
HTTP 요청의 body는 `ServletInputStream`을 통해 제공되며, 이는 한 번만 읽을 수 있다는 특성이 있다.

하지만 이것만으로는 위의 의문이 해결되지 않는다. 통상적으로는 Controller앞에 AOP를 지정하면, 컨크롤러 실행전에 AOP가 실행될것인데
AOP에서 body를 읽을 수 없다는것은, 이미 앞에서 다른 어떠한 곳에서 inputStream을 소비하였다는것인데 AOP이후에 실행되는 컨트롤러에서는 어떻게 @RequestBody
를 통해 값을 읽을까?  
AOP에서 먼저 읽었다면 컨트롤러에서 읽을 수 없어야 하는데, 실제로는 반대로 컨트롤러에서는 값을 읽고있다.

---

## Spring 요청 처리 과정

위의 궁금증을 해결하기 위해, HttpMessege가 Spring에서 처리되는 정확한 순서를 찾아보았다.  
Spring의 요청 처리 과정을 자세히 살펴보면 다음과 같은 순서로 진행된다.

<h4>Spring 요청 처리 과정</h4>

```java
// 1. DispatcherServlet이 요청 수신
public class DispatcherServlet extends FrameworkServlet {
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) {
        // 2. HandlerMapping을 통해 컨트롤러 찾기
        HandlerExecutionChain mappedHandler = getHandler(processedRequest);
        
        // 3. HandlerAdapter 조회
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
        
        // 4. 여기서 ArgumentResolver들이 파라미터 바인딩
        ModelAndView mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
    }
}
```

먼저 Http Request가 DispatcherServlet을 통해 들어오면, DispatcherServlet내부에서는 
HandlerMapping이 실행될 컨트롤러를 탐색한다. 이후, HandlerAdapter가 컨트롤러 호출을 위한 어댑터 역할을 수행하고
ArgumentResolver가 파라미터 바인딩을 수행한다.  

중요한점은, 이 과정들이 수행되는 동안 AOP는 실행되지 않는다는것이다.

---

## ArgumentResolver와 AOP 실행 순서

실제 AOP가 실행되는 순서를 더 자세히 알아보자.

<h4>RequestMappingHandlerAdapter의 파라미터 처리</h4>

```java
public class RequestMappingHandlerAdapter {
    protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
            HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        
        // 1. ArgumentResolver가 파라미터 해석
        Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
        
        // 2. 그 다음에 실제 메서드 호출 (여기서 AOP가 적용됨)
        Object returnValue = doInvoke(args);
        
        return getModelAndView(mavContainer, modelFactory, webRequest);
    }
}
```

HandlerAdapter가 찾아낸 컨트롤러가 어떤 방식으로 후출되어야하는지를 결정하고 실행할때, 내부에서는 파라미터 바인딩이 수행된다.
자세한 과정은 RequestMappingHandlerAdapter를 통해 확인 가능했다.   
내부에서 ArgumentResolver를 통해 컨트롤러의 메소드 파라미터를 바인딩하는 과정이 이루어진다.
실제 컨트롤러의 @PathVariable, @RequestParam, @ModelAttribute, @RequestBody을 처리하는 과정이다.
이후에, doInvoke를 통해 실제 메소드를 수행한다. **실제 AOP가 실행되는 시점은 이 시점이다.**

생각해보면, AOP의 joinPoint를 통해 컨트롤러의 파라미터에 접근 가능한점을 생각해본다면, 
AOP이전에 컨트롤러의 파라미터 바인딩이 먼저 수행되는점을 알 수 있다.

전체 과정을 요약해보자

<div class="notice" markdown="1">
<h4>실행 순서 정리</h4>

1. **DispatcherServlet** 요청 수신
2. **HandlerMapping**이 컨트롤러 메서드 결정
3. **HandlerAdapter**가 컨트롤러 호출방법 결정
4. **ArgumentResolver**가 파라미터 바인딩 (@RequestBody 처리)
5. **AOP Proxy** 컨트롤러 호출시 동작 (Around advice 시작)
6. **컨트롤러 메서드** 실제 실행
7. **AOP Proxy** 결과 반환 (Around advice 종료)

</div>

---

## 실제 로그로 확인하는 실행 순서

실행 순서를 명확히 확인하기 위해, ArgumentResolver를 직접 작성해보고
순서마다 로그를 찍어 순서를 확인해보자.

<h4>커스텀 ArgumentResolver</h4>

```java
@Component
@Slf4j
public class CustomArgumentResolver implements HandlerMethodArgumentResolver {
    
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CustomRequestBody.class) && 
               parameter.getParameterType() == UserRoleChangeRequest.class;
    }
    
    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        
        log.info("=== ArgumentResolver 실행 시작 ===");
        
        HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
        String body = StreamUtils.copyToString(request.getInputStream(), StandardCharsets.UTF_8);
        
        log.info("ArgumentResolver에서 읽은 body: {}", body);
        
        ObjectMapper mapper = new ObjectMapper();
        UserRoleChangeRequest result = mapper.readValue(body, UserRoleChangeRequest.class);
        
        log.info("=== ArgumentResolver 실행 완료 ===");
        return result;
    }
}
```

<h4>AOP</h4>

```java
@Aspect
@Component
@Slf4j
public class CustomAop {

    @Around("@annotation(org.springframework.web.bind.annotation.PostMapping)")
    public Object logRequest(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("=== AOP 시작 ===");

        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();

        log.info("AOP에서 InputStream 읽기...");
        String body = StreamUtils.copyToString(request.getInputStream(), StandardCharsets.UTF_8);
        log.info("AOP에서 읽은 body: '{}'", body);

        log.info("=== 컨트롤러 메서드 호출 ===");
        Object result = joinPoint.proceed();
        log.info("=== AOP 종료 ===");

        return result;
    }
}
```

<h4>컨트롤러</h4>

```java
@RestController
@Slf4j
public class UserController {
    
    @PostMapping("/api/users/role")
    public ResponseEntity<String> changeUserRole(@CustomRequestBody UserRoleChangeRequest request) {
        log.info("=== 컨트롤러 메서드 실행 ===");
        log.info("컨트롤러에서 받은 데이터: userId={}", request.getUserId());
        log.info("=== 컨트롤러 메서드 완료 ===");
        return ResponseEntity.ok("Role changed successfully");
    }
}
```

<h4>실행 결과 로그</h4>

```
=== ArgumentResolver 실행 시작 ===
ArgumentResolver에서 읽은 body: {"userId":"user123","newRole":"ADMIN"}
=== ArgumentResolver 실행 완료 ===

=== AOP 시작 ===
AOP에서 InputStream 읽기...
AOP에서 읽은 body: ''

=== 컨트롤러 메서드 호출 ===
=== 컨트롤러 메서드 실행 ===
컨트롤러에서 받은 데이터: userId=user123
=== 컨트롤러 메서드 완료 ===

=== AOP 종료 ===
```

이 로그를 통해 ArgumentResolver, AOP, 컨트롤러간의 실행순서를 정확히 확인 가능하다.

<div class="notice" markdown="1">

1. **ArgumentResolver가 먼저 실행**되어 InputStream을 소비한다
2. **AOP는 그 이후 실행**되어 이미 소비된 스트림을 읽으려 시도한다
3. 그렇게 때문에 AOP에서는 getInputStream()을 통해 값을 읽을 수 없다
4. **컨트롤러는 이미 파싱된 객체**를 파라미터로 받음

</div>

---

## 해결 방법과 대안

앞서 AOP에서 HttpServletRequest의 body값을 읽을 수 없음과 동시에 Controller에서는 값이 읽히는 이유를 자세히 알아보았다.
그렇다면 AOP내부에서 HttpServletRequest의 body값을 읽을 수 있도록 코드 수정을 해보자.  
HttpServletRequest를 ContentCachingRequestWrapper를 사용하여 캐싱하면 body값을 여러번 읽을 수 있다.

먼저 필터를 통해서 Filter를 통해서 서버로 들어오는 ServletRequest를 ContentCachingRequestWrapper를 사용하여 캐싱한다.

<h4>RequestCachingFilter</h4>

```java
@Component
@Slf4j
public class RequestCachingFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        // ContentCachingRequestWrapper로 래핑
        ContentCachingRequestWrapper wrappedRequest = new ContentCachingRequestWrapper(httpRequest);
        
        chain.doFilter(wrappedRequest, response);
    }
}
```

그 이후 AOP에서는 RequestContextHolder를 통해 가져온 HttpServletRequest의 body값을 읽을 수 있다.

<h4>AOP에서 사용</h4>

```java
@Around("@annotation(org.springframework.web.bind.annotation.PostMapping)")
public Object logRequest(ProceedingJoinPoint joinPoint) throws Throwable {
    HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
    
    if (request instanceof ContentCachingRequestWrapper) {
        ContentCachingRequestWrapper wrapper = (ContentCachingRequestWrapper) request;
        
        // proceed() 호출 후에 캐시된 내용을 읽을 수 있음
        Object result = joinPoint.proceed();
        
        byte[] content = wrapper.getContentAsByteArray();
        String body = new String(content, StandardCharsets.UTF_8);
        log.info("캐시된 body: {}", body);
        
        return result;
    }
    
    return joinPoint.proceed();
}
```

---

## 결론

이번 트러블슈팅을 통해 Spring의 요청 처리 과정에 대한 이해가 한층 깊어졌다. 

처음 검색 결과에서는 ServletRequest를 inputStream을 통해 읽으면 한번만 읽을 수 있고,
여러번 읽고 싶다면, ContentCachingRequestWrapper를 사용해서 캐싱처리하여 여러번 읽을 수 있다는 점을 알게 되었다.

하지만, 이 내용만으로는 AOP가 먼저 수행됨에도 불구하고, 
AOP내부에서 HttpServletRequest의 body값을 읽을 수 없는 이유를 이해하기 어려웠다.

그래서 Spring 요청 처리 과정을 더 자세히 알아보아, 왜 이런 문제가 발생하는지를 알아보았다. 

핵심 내용은 AOP는 컨트롤러 메서드 실행 시점을 감싸지만, 파라미터 바인딩은 그보다 먼저 일어난다.
즉, 메소드의 실행부분이랑 파라미터 부분은 실행시점이 다르다는것이다. 
이로인해, 파라미터 바인딩 시점에 InputStream이 이미 소비되므로, AOP에서는 값을 읽을 수 없다는것이다.

이번기회에 Spring의 내부 동작 원리를 더 깊이 이해하게 되었고, 
단순한 API 사용법을 넘어서 프레임워크의 실행 순서를 파악하는 것이 얼마나 중요한지 다시 한번 느꼈다. 
앞으로는 트러블슈팅 시 현상만 보고 판단하지 말고, 내부 동작 과정을 차근차근 분석하는 습관을 길러야겠다.

---
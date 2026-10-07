# Spring 예외 처리 — @ControllerAdvice, ErrorCode, Validation 예외 흐름

> **한 줄 요약**: Spring 예외 처리는 컨트롤러에서 터진 예외를 `@ControllerAdvice`가 모아 HTTP 응답으로 바꾸는 흐름이다. 핵심은 도메인 예외, 검증 예외, 예상 못 한 예외를 구분해 응답 형식을 일관되게 만드는 것.

관련 노트: [@Valid · @Validated](./validation.md) · [Mockito 서비스 테스트](../test/mockito-service-test.md) · [도메인 검증 위치](../design/domain-validation.md)

---

## 1. 기본 흐름

```text
Controller
  -> Service
  -> Domain
  -> 예외 발생
  -> @ControllerAdvice
  -> ErrorResponse JSON
```

서비스나 도메인에서 예외를 던지고, 웹 계층에서 HTTP 응답으로 변환한다. 도메인 계층이 HTTP 상태 코드를 직접 알 필요는 없다.

---

## 2. 도메인 예외 패턴

```java
public enum ErrorCode {
    USER_NOT_FOUND(404, "USER_NOT_FOUND", "사용자를 찾을 수 없습니다."),
    INVALID_AMOUNT(400, "INVALID_AMOUNT", "금액이 올바르지 않습니다.");

    private final int status;
    private final String code;
    private final String message;
}
```

```java
public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }
}
```

---

## 3. @ControllerAdvice

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException e) {
        ErrorCode code = e.getErrorCode();
        return ResponseEntity
                .status(code.getStatus())
                .body(ErrorResponse.from(code));
    }

    @ExceptionHandler(Exception.class)   // 최후의 안전망 — 위에서 안 잡힌 것만
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception e) {
        log.error("unhandled exception", e);              // 원인은 로그로 남기고
        return ResponseEntity.status(500)
                .body(ErrorResponse.of("INTERNAL_ERROR")); // 상세 메시지는 노출 안 함
    }
}
```

컨트롤러마다 try-catch를 두지 않고 공통 처리한다. **한 advice 클래스 안에서는** 가장 구체적인 예외 타입의 핸들러가 우선 매칭되므로, `Exception.class` fallback은 위의 `BusinessException` 핸들러를 가리지 않는다. 단 fallback에서 **원인을 잃지 않게 반드시 로그를 남긴다**(§7 함정) — 안 그러면 500만 보이고 stack trace가 사라진다.

⚠️ **`Exception.class` fallback은 스프링 MVC 자체 예외까지 500으로 바꾼다.** 405(`HttpRequestMethodNotSupportedException`), 415, 400(`MissingServletRequestParameterException`·`HandlerMethodValidationException`), 6.1+의 정적 리소스 404(`NoResourceFoundException`)도 `Exception`의 하위라, 스프링 기본 처리(`DefaultHandlerExceptionResolver`)보다 먼저 이 핸들러에 잡힌다. 표준 해법은 advice가 **`ResponseEntityExceptionHandler`를 상속**하는 것 — 내장 웹 예외를 올바른 상태 코드의 RFC 9457 `ProblemDetail`로 바꿔 주고, 필요한 것만 오버라이드한다(Boot는 `spring.mvc.problemdetails.enabled=true`로 이걸 자동 등록).

⚠️ **advice가 여러 개면 "구체성"보다 "순서"가 먼저다.** 스프링은 `@Order` 순으로 advice를 훑어 **매칭되는 핸들러가 하나라도 있는 첫 advice**의 것을 쓴다. 우선순위가 높은 advice에 `Exception.class` 핸들러가 있으면, 뒤 advice의 더 구체적인 핸들러는 호출되지 않는다. fallback은 한 advice에 모으거나 가장 낮은 순서의 advice에 둔다.

---

## 4. Validation 예외

`@RequestBody @Valid`에서 실패하면 보통 `MethodArgumentNotValidException`이 발생한다.

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e) {
    List<String> messages = e.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .toList();

    return ResponseEntity.badRequest()
            .body(ErrorResponse.validation(messages));
}
```

> ⚠️ **검증 위치에 따라 터지는 예외가 다르다.** 위 `MethodArgumentNotValidException`은 `@RequestBody @Valid`(본문 객체) 케이스다. `@RequestParam`/`@PathVariable`에 직접 건 제약(`@Min` 등)이 깨지면 **다른 예외**가 뜬다.
> - Spring 6.1+ 기본(컨트롤러 클래스에 `@Validated` 없음): `HandlerMethodValidationException` — 스프링이 기본으로 **400**을 내지만, 위 핸들러 형식과 다른 기본 오류 본문이 나간다. 같은 메서드의 `@Valid @RequestBody` 오류도 이 예외로 바뀔 수 있다.
> - 컨트롤러 클래스에 `@Validated`(AOP 경로, 6.0 이하는 이것뿐): `ConstraintViolationException` — 핸들러를 따로 등록하지 않으면 **500으로 샌다.**
>
> 어떤 어노테이션이 무엇을 트리거하나 → [@Valid · @Validated §4](./validation.md)

---

## 5. 무엇을 어디서 검증하나

| 위치 | 담당 |
|---|---|
| DTO Bean Validation | null, 형식, 길이 같은 입력 모양 |
| 도메인 생성자/메서드 | 불변식, 값 객체 규칙 |
| 도메인 서비스 | 여러 엔티티나 저장소 조회가 필요한 규칙 |
| DB 제약 | 유니크, FK, 최종 무결성 |

배치 기준("그 검증에 필요한 정보가 어디 있나")은 [도메인 검증 위치](../design/domain-validation.md).

---

## 6. 판단 기준

- 사용자가 고칠 수 있는 입력 문제는 400 계열로 명확히 응답한다.
- 리소스가 없으면 404, 권한 문제는 403, 인증 문제는 401로 분리한다.
- 서버 버그나 예상 못 한 예외는 상세 메시지를 그대로 노출하지 않는다.
- 예외 메시지는 사람이 읽는 설명, `code`는 클라이언트가 분기할 안정적인 값으로 둔다.

---

## 7. 함정

- 도메인 예외가 `ResponseEntity`나 HTTP 상태를 직접 알면 계층이 섞인다.
- 모든 예외를 `Exception.class` 하나로 잡으면 문제 원인을 잃는다.
- validation 에러 응답 형식이 도메인 예외 응답 형식과 너무 다르면 프론트에서 다루기 어렵다.
- `@ControllerAdvice` 테스트는 `@WebMvcTest`로도 충분한 경우가 많다.
- **`@ExceptionHandler`가 잡은 예외는 기본으로 로그가 안 남는다.** "예외가 나면 콘솔에 스택 트레이스가 찍힌다"는 건 **아무도 처리하지 않은 예외**일 때의 이야기다. 처리되지 않은 예외는 `DispatcherServlet` 밖까지 올라가고, Tomcat이 `Servlet.service() for servlet [dispatcherServlet] … threw exception`을 ERROR로 남긴다. 반면 `@ExceptionHandler`가 응답으로 바꾸면 "해결된 예외"가 되어, Spring은 `Resolved [예외]`를 **DEBUG**로만 남긴다(`AbstractHandlerExceptionResolver`, `setWarnLogCategory`로 WARN 승격 가능). 그래서 공통 처리를 붙이는 순간 로그가 조용해진다. 다른 프로젝트에서 에러 로그가 보였다면 대개 그 핸들러 안에 `log.warn/error`가 있었던 것이다. 로컬에서 잠깐 보려면 `--logging.level.org.springframework.web=DEBUG`. 비즈니스 예외(400·404·409)는 안 남겨도 되지만, `Exception.class` fallback에는 반드시 직접 남긴다(바로 위 항목).
- **필터에서 난 예외는 `@ControllerAdvice`가 못 잡는다.** `@ControllerAdvice`는 `DispatcherServlet` **안에서** 컨트롤러가 던진 예외를 처리한다. 필터는 `DispatcherServlet`보다 **바깥**에서 돌기 때문에, JWT 검증 필터 같은 곳에서 던진 예외는 거기까지 가지 못하고 컨테이너 기본 에러 응답이 나간다. 해법은 두 가지: (1) 필터 체인 맨 앞에 **예외를 잡는 필터**를 두거나, (2) 필터 안에서 `HandlerExceptionResolver`를 직접 불러 `@ControllerAdvice`와 같은 형식으로 응답을 쓴다.

### 앞의 필터가 뒤의 필터 예외를 잡는 원리 — 필터 체인은 중첩된 메서드 호출이다

`filterChain.doFilter(request, response)`는 "다음 필터를 실행하라"는 **메서드 호출**이다. 앞 필터는 이 줄에서 **멈춰서** 뒤의 필터·컨트롤러가 전부 끝나기를 기다린다. 그래서 뒤에서 던진 예외는 호출 스택을 거슬러 올라오다가, 아직 실행 중인 앞 필터의 `try`에 걸린다.

```text
ExceptionFilter.doFilterInternal        try { ← 여기서 doFilter 반환을 기다리는 중
  └ filterChain.doFilter()
      └ SignatureFilter.doFilterInternal
          └ filterChain.doFilter()
              └ JwtFilter.doFilterInternal
                  └ throw new AuthException()   ← 던짐
예외 이동: JwtFilter(catch 없음) → SignatureFilter(catch 없음) → ExceptionFilter의 catch에서 잡힘
```

일반 자바 코드와 똑같다.

```java
void a() { try { b(); } catch (RuntimeException e) { /* c의 예외가 여기서 잡힘 */ } }
void b() { c(); }
void c() { throw new RuntimeException(); }
```

- `doFilter` **앞**의 코드는 요청이 들어가는 길, **뒤**의 코드는 응답이 나오는 길에서 실행된다. 요청 로그는 앞에, 처리 시간 측정은 앞뒤에 둔다.
- 그래서 예외 처리 필터는 **반드시 맨 앞**이어야 한다. 자기보다 앞에 있는 필터의 예외는 이미 그 바깥에서 터져서 못 잡는다.
- 컨트롤러 예외는 보통 `@ControllerAdvice`가 `DispatcherServlet` 안에서 응답으로 바꿔 버리므로 예외 상태로 필터까지 올라오지 않는다. 필터의 `catch (Exception e)`에 걸리는 건 처리되지 않은 예외뿐이다.
- **필터가 요청을 막는 방법은 두 가지다.** `doFilter`를 부르기 **전에** 예외를 던지거나, 응답을 직접 쓰고 `doFilter`를 부르지 않은 채 `return`한다. 어느 쪽이든 뒤의 필터와 컨트롤러는 아예 실행되지 않는다.
- **`doFilter`에 넘긴 객체가 뒤쪽 전체가 보는 요청이다.** 요청 바디는 스트림이라 한 번 읽으면 끝인데, 서명 검증처럼 필터가 바디를 먼저 읽어야 하면 바디를 저장해 두는 래퍼로 감싸서 `doFilter(wrapper, response)`로 넘긴다. 그래야 컨트롤러가 바디를 다시 읽을 수 있다. 필터가 검증한 결과(예: 확인된 클라이언트 ID)는 `request.setAttribute`로 실어 보내고, 컨트롤러 쪽 인자 리졸버가 꺼내 쓴다.

---

## 8. 참고

- [Spring MVC — Controller Advice](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-advice.html) (전역 `@ExceptionHandler`는 로컬 다음에 적용)
- [Spring MVC — Error Responses (ProblemDetail·`ResponseEntityExceptionHandler`)](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html) (Boot `spring.mvc.problemdetails.enabled`, Boot 핸들러의 order 0)
- [Spring MVC — Validation](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-validation.html) (`MethodArgumentNotValidException`과 `HandlerMethodValidationException`을 둘 다 처리하라)
- 소스 — `ExceptionHandlerExceptionResolver.getExceptionHandlerMethod`(advice를 순서대로 훑어 첫 매칭 반환) · `HandlerMethodValidationException`(입력 검증 400, 반환값 검증 500)
- 관련 노트: [@Valid · @Validated](./validation.md) · [도메인 검증 위치](../design/domain-validation.md)

---

**학습 날짜**: 2026-06-08 (보강 2026-06-26 · 2026-10-02)
**계기**: `@ControllerAdvice`·ErrorCode 기반 공통 예외 응답 구조를 정리하다, 검증 예외와 필터 예외가 같은 흐름을 타지 않는 이유를 파고듦

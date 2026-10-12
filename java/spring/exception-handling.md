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

### 1-1. 실제로 예외를 응답으로 바꾸는 주체 — `HandlerExceptionResolver`

위 그림의 "@ControllerAdvice로 간다"는 사이에 `DispatcherServlet`과 **해결사(resolver)**가 끼어 있다.

```java
public interface HandlerExceptionResolver {
    // 이 예외를 응답으로 바꿀 수 있으면 응답을 쓰고 ModelAndView를, 못 하면 null을 돌려준다
    ModelAndView resolveException(HttpServletRequest req, HttpServletResponse res, Object handler, Exception ex);
}
```

컨트롤러가 예외를 던지면 `DispatcherServlet`이 해결사들에게 차례로 "이거 처리할 수 있어?"를 묻고, 처음으로 `null`이 아닌 답을 준 해결사의 응답이 나간다. Spring MVC는 해결사 셋을 하나로 묶은 **`HandlerExceptionResolverComposite`**를 `handlerExceptionResolver`라는 이름의 빈으로 등록한다(`WebMvcConfigurationSupport.handlerExceptionResolver`).

| 순서 | 해결사 | 맡는 것 |
| --- | --- | --- |
| 1 | `ExceptionHandlerExceptionResolver` | `@ExceptionHandler` 메서드 찾기(컨트롤러 안 → `@ControllerAdvice`). 우리 advice는 여기서 불린다 |
| 2 | `ResponseStatusExceptionResolver` | `@ResponseStatus`가 붙은 예외, `ResponseStatusException` |
| 3 | `DefaultHandlerExceptionResolver` | 아무도 안 잡은 스프링 MVC 내장 예외를 맞는 상태 코드로(`sendError`) |

**필터에서 이 빈을 주입받을 때 `@Qualifier("handlerExceptionResolver")`가 필요한 이유:** 스프링 부트의 `DefaultErrorAttributes`도 `HandlerExceptionResolver`를 구현한다(예외를 `/error` 페이지용으로 기록만 하고 `null`을 반환). 같은 타입 빈이 둘이라 타입만으로 주입하면 "후보가 여럿"이라 실패하고, 이름으로 MVC 묶음을 집어야 한다. 필터에서 `resolver.resolveException(...)`을 직접 부르는 건 **컨트롤러 예외 때 `DispatcherServlet`이 하는 일을 손으로 대신 하는 것**이다(→ §7 필터 예외).

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

### 2-1. 하위 예외 클래스를 따로 만들 때 vs `ErrorCode` 하나로 충분할 때

`BusinessException` + `ErrorCode`가 있으면 오류의 종류(코드·상태·메시지·재시도 여부)는 enum 상수가 이미 말해 준다. 하위 클래스(`InvalidLunarDateException extends ChartException extends BusinessException`)는 **enum이 못 하는 일이 있을 때만** 만든다.

| 하위 클래스가 필요한 경우 | 예 |
|---|---|
| 오류에 **추가 데이터**가 붙는다 | `AiUnavailableException(reason, retryAt)`, `InvalidAiOutputException(violations)` |
| 누군가 **타입으로 잡아서** 다르게 처리한다 | 일시 장애만 골라 재시도. 단, `errorCode().retryable()` 같은 플래그로 갈 수 있으면 타입은 필요 없다 |
| 라이브러리 경계에서 실패 계약을 타입으로 공개한다 | `IOException` 계열 |

- 인자 없는 생성자에 `ErrorCode` 하나를 박아 둔 하위 클래스는 **enum 상수의 별칭**일 뿐이다. 오류 하나가 생길 때마다 클래스와 enum 상수가 같이 늘어난다.
- 별칭 클래스는 공용 가드와도 잘 맞지 않는다. `requireNonNull(value, ErrorCode)` 같은 가드는 상위 `BusinessException`만 던질 수 있어서, 타입을 지키려면 `Supplier` 오버로드를 따로 두거나 한 줄짜리 검사 함수를 모듈마다 둬야 한다.
- 실제로 흔히 보는 모양: 한 모듈(계산 어댑터 등)을 먼저 만들 때 예외를 타입으로 정의했다. 나중에 `ErrorCode` 체계를 들이면서 그 모듈만 `extends BusinessException`으로 이어 붙여 두 방식이 섞인다. 섞여 있으면 git 기록으로 이런 사정을 먼저 확인한다.
- **입력 오류가 전부 하나의 코드(`invalid_input`)로 모일 때 잃는 것과 대안.**
  - 흔한 모양이다. 필수값 누락, 형식 오류, 없는 날짜, 시각 모순이 모두 400 `invalid_input`과 같은 문구로 나간다.
  - 예외를 하위 클래스로 나눠 두었더라도 같은 `ErrorCode`를 쓰면 응답은 원래 같았다. 하위 클래스를 없애도 응답에서 잃는 정보는 없다.
  - 잃는 것은 둘이다.
    - 사용자: 어느 칸이 틀렸는지 모른다. 보통은 클라이언트가 칸별 검사를 먼저 하고, 서버의 400은 마지막 방어선이라 MVP에서는 견딜 만하다.
    - 개발자: 어느 규칙에 걸렸는지 모른다. 처리된 업무 예외는 보통 로그를 남기지 않기 때문이다.
  - 대안 1. 칸별 코드나 필드 목록을 응답에 넣는다(`invalid_birth_date`, `{field, reason}`). 화면이 칸을 짚어 줘야 할 때 고른다. 응답 계약이 바뀌고 코드가 는다. 입력값은 응답에 되돌려 넣지 않는다.
  - 대안 2. 응답은 그대로 두고, 처리기에서 DEBUG 수준으로 던진 위치(스택의 첫 프레임: 클래스·메서드·줄)만 남긴다.
    - 검사 함수에 규칙 이름을 붙여 두었다면(`BirthDate.validateSolarDate`) 메서드 이름이 곧 규칙 이름이 되어, 던지는 곳을 하나도 고치지 않아도 된다.
    - 공용 가드(`requireNonNull`)는 첫 프레임이 가드 자신이므로 한 칸 위 호출자 프레임을 본다.
    - 값은 남지 않는다.
- **오류 응답의 `retryable`·`retryAfterSeconds`는 서버가 알아서 다시 실행한다는 뜻이 아니다.** 클라이언트에게 주는 안내다.
  - `retryable`(ErrorCode의 성질): 같은 요청을 다시 보내 볼 의미가 있는가. 입력 오류는 `false`(고치지 않으면 또 실패), 외부 서비스 일시 장애나 요청 제한은 `true`.
  - `retryAfterSeconds`(그때그때 계산한 값): 언제 다시 보내면 되는가. HTTP 표준 `Retry-After` 헤더로도 함께 보낸다. 브라우저가 이 헤더를 보고 자동으로 다시 보내지는 않는다. 다시 보낼지는 클라이언트 코드가 정한다.
  - 서버 내부의 자동 재시도(외부 API 일시 실패 시 한 번 더 호출하기 등)는 어댑터 안에서 따로 처리하는 일이고, 이 두 값과는 상관없다.
  - 사용자가 직접 누르는 "재시도" API를 만든다면, 서버는 `retryable`을 허용 조건으로 쓸 수 있다.
- 타입으로 단언하는 테스트(`isInstanceOf(XxxException.class)`)는 `errorCode()` 비교로 바꿔도 같은 것을 검증한다. 다만 오류 코드가 같고 출처만 다른 경우(같은 `INVALID_INPUT`을 다른 모듈이 던질 때)는 구분하지 못한다.

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

**상속하면 어떻게 동작하나.** 부모 클래스에 `@ExceptionHandler({내장 예외 20여 개})`가 붙은 `handleException` 하나가 있고, 예외 종류별로 `protected` 메서드(`handleHttpMessageNotReadable`·`handleMethodArgumentNotValid`·`handleHttpMediaTypeNotSupported`·`handleHttpRequestMethodNotSupported`·`handleTypeMismatch`·`handleNoResourceFoundException` 등)로 나눠 보낸다. 이 메서드들이 **오버라이드 지점**이다. 이미 알맞은 상태 코드(`status`)와 헤더(`headers`, 예: 415의 `Accept`)를 인자로 받으므로, 본문만 바꿔서 `handleExceptionInternal(ex, 내_본문, headers, status, request)`로 넘기면 상태·헤더는 스프링 것을 그대로 쓴다. 즉 **상태 코드를 정하는 건 부모의 분기**이고, `handleExceptionInternal`은 이미 정해진 상태·헤더에 본문을 담아 `ResponseEntity`로 묶는 공통 출구다.

⚠️ **상속한 클래스에서 부모가 이미 처리하는 예외에 `@ExceptionHandler`를 또 달면 기동이 실패한다.** 예: 상속하면서 `@ExceptionHandler(HttpMessageNotReadableException.class)`를 추가 → 같은 예외에 핸들러가 둘이 되어 `IllegalStateException: Ambiguous @ExceptionHandler method mapped for [...]`. 상속했다면 그 예외는 **`handleXxx` 오버라이드로만** 바꾼다. (상속하지 않는 advice는 반대로 `@ExceptionHandler`로 직접 단다 — 같은 목적, 다른 연결 방식.)

```java
@Override
protected ResponseEntity<Object> handleHttpMessageNotReadable(
        HttpMessageNotReadableException ex, HttpHeaders headers, HttpStatusCode status, WebRequest request) {
    return handleExceptionInternal(ex, ErrorResponse.of(INVALID_INPUT), headers, status, request);  // 400은 그대로, 본문만 우리 형식
}
```

⚠️ **오버라이드하지 않은 예외는 `ProblemDetail` 형식으로 나간다.** 우리 형식(`{"error":{...}}`)으로 몇 개만 오버라이드하면, 나머지(405·406·타입 변환 실패·없는 경로 404 등)는 `application/problem+json`의 `{"type","title","status","detail","instance"}`로 나가 **응답 형식이 두 가지로 섞인다.** 프론트가 한 형식만 처리한다면 (a) 클라이언트가 실제로 받을 수 있는 예외는 전부 오버라이드하거나, (b) `handleExceptionInternal` 하나를 오버라이드해 `body`가 `ProblemDetail`이면 우리 형식으로 바꾸는 방법이 있다. (b)는 한 곳에서 끝나지만, 상태 코드에서 에러 코드를 매핑하는 규칙이 필요하다.

⚠️ **advice가 여러 개면 "구체성"보다 "순서"가 먼저다.** 스프링은 `@Order` 순으로 advice를 훑어 **매칭되는 핸들러가 하나라도 있는 첫 advice**의 것을 쓴다. 우선순위가 높은 advice에 `Exception.class` 핸들러가 있으면, 뒤 advice의 더 구체적인 핸들러는 호출되지 않는다. fallback은 한 advice에 모으거나 가장 낮은 순서의 advice에 둔다.

### 3-1. 상속 vs 직접 나열 — "HTTP 상태 코드를 쓰는 API인가"로 고른다

`ResponseEntityExceptionHandler`를 상속하지 않고, MVC 내장 예외를 `@ExceptionHandler`로 **하나씩 직접 나열**하는 advice도 실무에 흔하다.

| | 상속 + 필요한 것만 오버라이드 | 직접 나열 |
| --- | --- | --- |
| 상태 코드·헤더 | 스프링이 맞게 채움(405 `Allow`, 415 `Accept` 등) | 핸들러마다 직접 정함. 반환 타입이 DTO뿐이고 `@ResponseStatus`가 없으면 **전부 200**이 된다 |
| 응답 형식 | 오버라이드 안 한 예외는 `ProblemDetail`로 섞임 | 나열한 것은 전부 내 형식 |
| 빠뜨린 예외 | 부모가 받아 줌 | `Exception` fallback으로 떨어짐 → 서버 오류처럼 기록·알림될 수 있음. 그래서 MVC 예외 다수의 부모인 `ServletException` 핸들러를 중간 안전망으로 두기도 한다 |
| 어울리는 API 규약 | **HTTP 상태 코드로 결과를 구분**(400/409/429 + `Retry-After`) | **"항상 200 + 본문의 결과 코드"** 규약. 이 규약에선 상속의 장점(상태·헤더)이 쓸모없어 나열이 자연스럽다 |

둘 다 틀린 게 아니다. 결정 질문은 "클라이언트가 HTTP 상태 코드를 보고 분기하나, 본문 코드를 보고 분기하나"다. 상태 코드를 쓰는 API에서 직접 나열을 택하면 `ResponseEntity`로 상태를 직접 지정해야 하고, 스프링 버전이 올라가며 새로 생긴 내장 예외(예: 6.1의 `NoResourceFoundException`)를 따라 추가해야 한다.

**어느 쪽이든 가져갈 만한 것 — 수준별 로그.**
- 의도된 거부(검증 실패·권한 없음·비즈니스 규칙): **WARN + 짧은 위치 정보**. 전체 스택은 노이즈라 `getStackTrace()`에서 내 패키지 프레임만 골라 `A.method:12 > B.method:30`처럼 한 줄로 남기는 방식이 쓸 만하다(프록시 `$$`·필터 `doFilter` 프레임은 제외).
- 비즈니스 예외라도 **원인 예외(`getCause()`)를 감싸고 있으면 ERROR** — 무언가 실제로 깨진 것이다.
- 예상 못 한 예외: **ERROR + 전체 스택**(`log.error("…", ex)`처럼 마지막 인자로 예외). 운영 환경에서만 메신저 알림을 붙이기도 한다.
- 클라이언트가 연결을 끊은 경우(`ClientAbortException`): DEBUG. 서버 문제가 아니다.
- 로그에 남기는 `getMessage()`·요청 URI에 개인정보가 섞일 수 있는 서비스라면, 메시지 대신 예외 클래스 이름과 위치만 남기는 규칙을 따로 둔다. 예외 메시지에는 입력값이 자주 섞인다(날짜 생성 실패의 날짜, JSON 파싱 오류의 원문 조각, DB 제약 위반의 키 값). 구현은 **메시지 없는 사본**을 만들어 로그에 넘기는 방식이 단순하다:
  ```java
  private static class RedactedException extends RuntimeException {
      private final String originalClassName;
      RedactedException(Throwable original, int depth) {
          super(null, depth < 10 && original.getCause() != null            // 원인 체인도 사본으로, 순환 대비 깊이 제한
                  ? new RedactedException(original.getCause(), depth + 1) : null,
                false, true);                                              // suppressed 끔(그쪽 메시지도 새지 않게)
          originalClassName = original.getClass().getName();
          setStackTrace(original.getStackTrace());                         // 위치는 원본 그대로
      }
      @Override public String toString() { return originalClassName; }   // 첫 줄에 원래 종류만
  }
  // log.error("Unexpected exception", new RedactedException(ex, 0));
  ```
  ⚠️ 첫 줄이 `toString()`대로 찍히는 건 **로깅 구현에 달렸다.** Logback 1.5는 `toString()`이 기본 형식(`클래스: 메시지`)과 다르면 그걸 그대로 쓴다(`ThrowableProxy`의 overridingMessage — 스프링 부트 기본 콘솔 패턴 `%wEx`도 이 경로). 다른 구현에선 사본 클래스 이름이 찍힐 수 있다 — 메시지가 새지는 않지만 원래 종류가 안 보이니, 출력 캡처 테스트(`OutputCaptureExtension`)로 "원래 클래스 이름은 있고 메시지는 없다"를 고정해 둔다.

  - `super(message, cause, enableSuppression, writableStackTrace)`는 `protected`라 하위 클래스에서만 부를 수 있는 생성자다. **`writableStackTrace`는 반드시 `true`** — `false`면 `setStackTrace()`가 아무 효과도 없어 원본 위치를 못 옮긴다(`true`라서 생성 시점 스택이 한 번 채워지지만 곧바로 덮어쓴다).
  - **Lombok 생성자 어노테이션으로는 못 줄인다.** `@AllArgsConstructor`·`@RequiredArgsConstructor`는 "인자를 필드에 대입"만 만들고 부모는 인자 없는 `super()`로만 부른다 — 부모 생성자에 인자를 넘기거나, 인자로 필드 값을 계산하거나(`getClass().getName()`), 재귀·`setStackTrace` 같은 로직을 넣을 수 없다. 예외 전용 `@StandardException`도 표준 생성자 4개(없음·메시지·원인·메시지+원인)만 만들고, `@ToString`은 `RedactedException(originalClassName=…)` 형식이라 "클래스 이름만"을 못 낸다. 생성자에 로직이 있으면 손으로 쓰는 게 맞다.
  - 코드 대신 **설정으로** 하는 길도 있다: Logback의 예외 변환기(`%ex`)를 메시지를 빼는 커스텀 `ThrowableProxyConverter`로 바꾸면, 우리 핸들러뿐 아니라 Tomcat·라이브러리가 남기는 모든 예외 로그에서 메시지가 빠진다. 범위가 넓은 대신 설정 파일(`logback-spring.xml`)과 변환기 클래스가 생기고, 메시지가 무해한 로그까지 전부 가린다.

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

  (2)의 전형적인 모양:
  ```java
  // 주입: @Qualifier("handlerExceptionResolver") HandlerExceptionResolver resolver
  //  → MVC가 쓰는 '해결사 묶음' 빈. 안에 @ExceptionHandler를 찾는 해결사가 들어 있다.
  private void reject(HttpServletRequest req, HttpServletResponse res, ErrorCode code) {
      BusinessException ex = new BusinessException(code);
      if (resolver.resolveException(req, res, null, ex) == null) {  // null = 아무도 처리 못 함
          throw ex;                                                  // 조용히 삼키지 않고 원래대로 올림
      }
  }
  // 호출부: reject(...); return;   ← chain.doFilter를 부르지 않아야 요청이 여기서 멈춘다
  ```
  - **`handler` 자리가 `null`인 이유:** 필터 단계라 아직 어느 컨트롤러로 갈지 정해지지 않았다. 그래서 컨트롤러 클래스 안의 `@ExceptionHandler`는 못 찾고, **전역 `@ControllerAdvice`의 핸들러만** 찾힌다.
  - **반환값:** 처리했으면 (빈) `ModelAndView`, 못 했으면 `null`. `null`일 때 예외를 다시 던지거나 `sendError`로 최소한의 응답을 보내야 응답이 빈 200으로 나가는 사고를 막는다.
  - **Spring Security에서도 같은 패턴을 쓴다.** 보안 필터(방화벽 거절 `RequestRejectedHandler`, 인증 실패 `AuthenticationEntryPoint`, 권한 거부 `AccessDeniedHandler`)도 전부 `DispatcherServlet` 앞에서 돈다. 이 핸들러들에 resolver를 주입해 같은 방식으로 부르면, 보안 거절도 컨트롤러 에러와 같은 JSON 형식이 된다.
  - 이렇게 처리된 거절도 DEBUG 로그에는 `Using @ExceptionHandler …` / `Resolved [...]`로 찍히지만, `DispatcherServlet`의 `POST "/경로"` 줄은 없다 — 컨트롤러 앞에서 막혔다는 표시.

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

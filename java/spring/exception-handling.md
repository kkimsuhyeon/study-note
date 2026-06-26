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

컨트롤러마다 try-catch를 두지 않고 공통 처리한다. 가장 **구체적인 예외 타입의 핸들러가 우선 매칭**되므로, `Exception.class` fallback은 위의 `BusinessException` 핸들러를 가리지 않는다. 단 fallback에서 **원인을 잃지 않게 반드시 로그를 남긴다**(§7 함정) — 안 그러면 500만 보이고 stack trace가 사라진다.

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

> ⚠️ **검증 위치에 따라 터지는 예외가 다르다.** 위 `MethodArgumentNotValidException`은 `@RequestBody @Valid`(본문 객체) 케이스다. **`@RequestParam`/`@PathVariable`에 `@Validated`로 건 제약**(`@Min` 등)이 깨지면 `ConstraintViolationException`(스프링 6.1+는 `HandlerMethodValidationException`)으로 **다른 예외**가 뜬다 → 핸들러를 따로 등록하지 않으면 400이 아니라 500으로 샌다. (어떤 어노테이션이 무엇을 트리거하나 → [@Valid · @Validated](./validation.md))

---

## 5. 무엇을 어디서 검증하나

| 위치 | 담당 |
|---|---|
| DTO Bean Validation | null, 형식, 길이 같은 입력 모양 |
| 도메인 생성자/메서드 | 불변식, 값 객체 규칙 |
| 도메인 서비스 | 여러 엔티티나 저장소 조회가 필요한 규칙 |
| DB 제약 | 유니크, FK, 최종 무결성 |

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

---

## 8. 참고

- Spring Framework Exception Handling
- Bean Validation

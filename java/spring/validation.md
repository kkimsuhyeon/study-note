# @Valid · @Validated — Bean Validation 동작과 위치

> **한 줄 요약**: `@Valid`(표준)는 "여기서 검증을 시작/전파해라"는 **트리거**, `@Validated`(Spring)는 그 위에 **그룹 검증 + 메서드 파라미터 검증**을 얹은 것. 제일 헷갈리는 함정은 **중첩 객체/리스트는 "부모 *필드*"에 `@Valid`를 붙여야 안쪽 제약이 검증된다**는 것 — 클래스 선언부에 붙이면 아무 일도 안 한다.

관련 노트: [@Transactional](./transactional.md) · [Jackson 어노테이션](../jackson/annotations.md)

---

## 1. 둘의 차이

| | `@Valid` | `@Validated` |
|---|---|---|
| 패키지 | `jakarta.validation` (**표준**) | `org.springframework...` (**Spring**) |
| 역할 | 검증 **트리거 / cascade(전파)** | `@Valid` + **그룹(groups)** + **메서드 파라미터 검증** |
| 주 위치 | 파라미터, **필드**, 메서드 | 주로 **클래스**(Controller/Service) |

- **그룹 검증**이 필요하면 `@Validated(OnCreate.class)` — `@Valid`는 그룹 지정 불가.
- `@RequestBody` 검증은 **둘 중 아무거나** 트리거로 동작(클래스에 `@Validated` 없어도 `@Valid`만으로 됨).

---

## 2. "어디에 붙이나" — 역할이 3가지로 갈린다

```
① 트리거    : 컨트롤러 파라미터  @Valid @RequestBody Dto dto   ← "이 바디를 검증 시작"
② 규칙 정의 : DTO 필드          @NotBlank @Size(...) ...      ← 실제 제약
③ 전파(cascade): 중첩 필드       @Valid private List<Child> ... ← 안쪽으로 검증 타고 들어감
```

- ①이 없으면 ②(필드 제약)가 아예 안 돈다.
- ②가 중첩 객체 안에 있으면, ③(부모 필드의 `@Valid`)이 있어야 검증된다.

---

## 3. ⭐ 핵심 함정 — cascade는 "필드"에, "클래스"에 붙이면 무효

중첩 객체·컬렉션 안의 제약을 검증하려면 **부모 필드에 `@Valid`**가 있어야 한다. **클래스 선언부에 붙인 `@Valid`는 표준상 cascade 트리거가 아니라 아무 동작도 안 한다.**

```java
// ❌ 잘못 — 안쪽 @NotBlank/@Size가 전부 무시됨
private List<WkTmList> wkTimeList;        // 필드에 @Valid 없음

@Valid                                    // 클래스에 붙여도 효과 없음
public static class WkTmList {
    @NotBlank @Size(min = 4, max = 4)
    private String breakStHhmm;           // 검증 안 됨
}
```

```java
// ✅ 올바름 — @Valid를 "필드"로 옮긴다
@Valid
private List<WkTmList> wkTimeList;        // 리스트 각 요소로 검증 전파

public static class WkTmList {            // 클래스의 @Valid는 제거
    @NotBlank @Size(min = 4, max = 4)
    private String breakStHhmm;           // 이제 검증됨
}
```

- 단일 중첩 객체도 동일: `@Valid private WorkPatternDtlRequest workPatternDtl;`
- 컬렉션의 각 요소도 전파됨(`@Valid List<Child>` = 모든 Child 검증).
- **중첩 필드마다 각각** `@Valid`가 필요하다. 한 군데 붙였다고 다른 필드까지 되는 게 아니다.

> 즉 "검증이 안 도는데?"의 1순위 원인 = **중첩 필드에 `@Valid` 누락** 또는 **클래스에 잘못 붙임.**

---

## 4. 동작 원리 — 두 경로, 두 예외

| 경로 | 트리거 | 처리 주체 | 실패 예외 |
|------|--------|-----------|-----------|
| **@RequestBody 검증** | 파라미터 `@Valid`/`@Validated` | Spring MVC (ArgumentResolver) | `MethodArgumentNotValidException` |
| **메서드 파라미터 검증** (`@RequestParam @Min` 등) | **클래스에 `@Validated`** | AOP (`MethodValidationPostProcessor`) | `ConstraintViolationException` |

```java
// (A) 바디 검증 — 클래스 @Validated 불필요
@PostMapping
public X save(@Valid @RequestBody Dto dto) { ... }   // 실패 → MethodArgumentNotValidException

// (B) 단일 파라미터 검증 — 클래스 @Validated 필수
@Validated                                            // ← 이게 있어야 아래 @Min이 동작
@RestController
public class C {
    @GetMapping
    public X list(@RequestParam @Min(1) int page) { ... }  // 실패 → ConstraintViolationException
}
```

- cascade는 Validator가 객체를 훑다 **`@Valid` 붙은 필드**를 만나면 그 안으로 재귀하는 방식. `@Valid` 없는 필드는 "그냥 값"으로 보고 안 들어간다.
- **예외 타입이 다르므로** `@RestControllerAdvice`에서 둘 다 핸들링해야 일관된 에러 응답이 나온다.

---

## 5. 자주 쓰는 표준 제약 + 커스텀

| 제약 | 의미 |
|------|------|
| `@NotNull` / `@NotBlank` / `@NotEmpty` | null 금지 / 공백 문자열 금지 / 빈 컬렉션·문자열 금지 |
| `@Size(min, max)` | 문자열·컬렉션 길이 |
| `@Min` / `@Max` / `@Positive` | 숫자 범위 |
| `@Pattern(regexp=...)` | 정규식 |
| `@Email` | 이메일 형식 |

- **커스텀 제약**(예: `@YN`): `@Constraint(validatedBy = ...)` + `ConstraintValidator` 구현으로 직접 만든 어노테이션. 표준 제약처럼 필드에 붙여 쓴다.
- ⚠️ `@NotNull` vs `@NotBlank` vs `@NotEmpty` 혼동 주의: 문자열 필수는 보통 `@NotBlank`(공백만 있어도 실패).

---

## 6. 💡 판단 기준

| 상황 | 선택 |
|------|------|
| 요청 바디(@RequestBody) 검증 | 파라미터에 `@Valid` |
| 바디 안의 중첩 객체/리스트도 검증 | **그 필드마다** `@Valid` |
| 쿼리/경로 단일 파라미터(@RequestParam 등) 검증 | **클래스에 `@Validated`** |
| 생성/수정 등 상황별 다른 규칙 | `@Validated(groups)` |

> 한 줄: **`@Valid`는 "검증을 켜고 안으로 전파"하는 스위치 — 켜야 할 지점(파라미터·중첩 필드)마다 붙인다.** 클래스에 붙이는 건 `@Validated`(메서드 검증/그룹)일 때만 의미 있다.

---

## 7. 참고
- [Jakarta Bean Validation 스펙](https://beanvalidation.org/)
- [Spring - Validation](https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html)
- 관련 노트: [@Transactional](./transactional.md)

---

**학습 날짜**: 2026-06-05
**계기**: 컨트롤러의 `@Valid @RequestBody`, 클래스의 `@Valid`, 중첩 리스트 요소 클래스의 `@Valid`가 각각 어떻게 동작하는지 헷갈려서 정리. 실제 코드에서 **중첩 리스트 필드에 `@Valid`가 빠져 안쪽 제약이 안 돌던 버그**(클래스에 잘못 붙임)를 발견한 게 계기.

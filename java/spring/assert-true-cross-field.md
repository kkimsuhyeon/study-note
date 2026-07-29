# @AssertTrue — boolean getter로 하는 필드 조합(cross-field) 검증

> **한 줄 요약**: `@AssertTrue`는 "boolean이 true인지" 검사하는 단순 제약이지만, 진짜 쓰임새는 **boolean getter 메서드에 붙여 여러 필드를 조합한 규칙("A가 X면 B 필수")을 DTO 안에 선언**하는 것. 핵심 함정 = **메서드 이름이 `is`/`get`으로 시작하지 않으면 조용히 무시**되고, **다른 필드가 null이어도 실행되므로 getter 안에 null 가드가 필수**.

관련 노트: [@Valid · @Validated](./validation.md) — 검증 트리거/cascade/제약 카탈로그(@NotBlank 비교, 타입별 제약 표)는 그쪽 · [도메인 검증 위치](../design/domain-validation.md)

---

## 1. 언제 쓰나

- 단일 필드 제약(`@NotNull`, `@Size`…)으로 표현이 안 되는 **필드 간 조건부 규칙**:
  - "평가가 '나쁨'이면 피드백 필수"
  - "시작일 ≤ 종료일"
  - "타입이 '기타'면 기타사유 필수"
- 커스텀 `ConstraintValidator`를 만들 정도는 아닌, **그 DTO 하나에서만 쓰는 국소 규칙**일 때.
- 반대 개념 `@AssertFalse`(false여야 통과)도 있지만 실무에선 거의 `@AssertTrue`만 쓴다.

---

## 2. 사용 예시

```java
@Data
public class ChangeAnswerRatingRequest {
    @NotNull
    private Rating rating;                 // 평가없음 / 좋음 / 나쁨

    @Valid                                 // 있으면 내부까지 검증 (cascade)
    private FeedbackRequest feedback;      // "나쁨"일 때만 필수

    @AssertTrue(message = "나쁨일 경우 feedback은 필수")
    public boolean isFeedbackPresentWhenDislike() {
        if (!Rating.DISLIKE.equals(rating)) {
            return true;                   // 규칙 대상 아님 → 통과
        }
        return feedback != null;
    }
}
```

- 동작 원리: Bean Validation은 `isXxx()`/`getXxx()` 형태 메서드를 **JavaBean 프로퍼티로 취급**해서, `@Valid` 트리거 시 이 메서드를 실행하고 반환이 `false`면 위반으로 처리한다. 필드 제약과 완전히 같은 흐름 — 커스텀 Validator 클래스가 필요 없다.
- 날짜 범위 검증 패턴:

```java
@AssertTrue(message = "시작일은 종료일보다 늦을 수 없습니다")
public boolean isPeriodValid() {
    if (startDate == null || endDate == null) {
        return true;                       // null 가드 — 필수 여부는 각 필드 @NotNull이 담당
    }
    return !startDate.isAfter(endDate);
}
```

> null이면 `true`로 통과시키는 게 관례다. "값이 없다"는 위반은 `@NotNull`이 각자 잡게 하고, 이 메서드는 **조합 규칙 하나만** 책임진다 — 안 그러면 에러 메시지가 이중으로 쌓인다.

---

## 3. cross-field 검증 방법 3가지 비교

| 방법 | 코드량 | 재사용 | 에러가 붙는 위치 |
|---|---|---|---|
| **`@AssertTrue` getter** | 최소 (메서드 1개) | ❌ 그 DTO 안에서만 | getter 프로퍼티명 (`isPeriodValid` → `periodValid`) |
| **클래스 레벨 커스텀 제약** (`@Constraint` + `ConstraintValidator<A, Dto>`) | 많음 (어노테이션+Validator 클래스) | ✅ 여러 DTO에 재사용 | 기본은 클래스(루트). `ConstraintValidatorContext`로 특정 필드에 지정 가능 |
| **서비스/엔티티에서 검증** | — | 도메인 규칙으로 공유 | 예외 처리 별도 (`@ControllerAdvice`) |

- 커스텀 어노테이션 만드는 법 자체는 [커스텀 어노테이션](../annotation/custom-annotation.md) 참고.
- "요청 형식 검증 vs 도메인 불변식" 구분은 [도메인 검증 위치](../design/domain-validation.md) 참고 — `@AssertTrue`는 어디까지나 **요청 형식** 층이다.

---

## 4. ⚠️ 함정 / 메커니즘

1. **메서드 이름이 `is`/`get`으로 시작해야 한다.** JavaBean getter 규약(파라미터 없음 + `is`는 boolean 반환)을 따라야 프로퍼티로 인식된다. `checkValid()` 같은 이름이면 **컴파일 에러도, 런타임 에러도 없이 그냥 검증이 안 돈다** — 제일 위험한 함정.
2. **null은 통과.** `Boolean` 필드에 붙였는데 값이 null이면 유효로 간주(대부분의 제약과 동일한 "null은 각자 @NotNull로" 원칙). 필수면 `@NotNull` 병행.
3. **다른 제약과 실행 순서 보장이 없고, 다른 필드가 위반이어도 실행된다.** `@NotNull` 실패한 필드를 getter에서 그대로 참조하면 NPE → 검증 전체가 `ValidationException`으로 터진다(400이 아니라 500). **getter 안 null 가드는 선택이 아니라 필수.** 순서를 강제하고 싶으면 `@GroupSequence`가 있지만 대부분 null 가드로 충분.
4. **에러 응답의 필드명 = 프로퍼티명.** `isValid()`면 `valid`라는 필드로 위반이 보고된다. 프론트가 "무슨 필드가 틀렸다는 거야?"가 되지 않게 **메서드명을 규칙이 드러나게** 짓는다 (`isFeedbackPresentWhenDislike` → `feedbackPresentWhenDislike`).
5. **Jackson 직렬화에 노출된다.** Jackson도 getter를 프로퍼티로 보므로, 이 DTO를 응답으로도 쓰면 `"feedbackPresentWhenDislike": true`가 JSON에 튀어나온다. 요청 전용 DTO면 무관하지만, 겸용이면 `@JsonIgnore`를 붙인다.
6. **부수효과 금지.** 검증 중 몇 번 호출될지 보장이 없다(그룹, 재검증 등). 순수 판정 로직만.

---

## 5. 💡 판단 기준

실제 케이스: "평가가 '나쁨'이면 피드백 필수" 규칙을 어디 두나 고민 → 이 요청 DTO 하나에서만 쓰는 **형식 규칙**이라 `@AssertTrue` getter가 최소 비용으로 정답이었다. 커스텀 제약을 만들었다면 어노테이션+Validator 2개 파일이 규칙 하나 때문에 생겼을 것.

> 한 줄: **한 DTO 안의 조건부 형식 규칙 = `@AssertTrue` getter. 같은 규칙이 DTO 여러 개에 반복되면 클래스 레벨 커스텀 제약으로, "저장된 상태의 정합성"(도메인 불변식)이면 엔티티/도메인으로 올린다.**

---

## 6. 참고

- [Jakarta Bean Validation 스펙 — built-in constraints](https://beanvalidation.org/2.0/spec/#builtinconstraints) (`@AssertTrue`: "null elements are considered valid")
- [Hibernate Validator Reference — declaring constraints (property-level)](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#validator-usingvalidator-annotate)
- 관련 노트: [@Valid · @Validated](./validation.md) · [도메인 검증 위치](../design/domain-validation.md) · [커스텀 어노테이션](../annotation/custom-annotation.md)

---

**학습 날짜**: 2026-07-29
**계기**: 회사 프로젝트 요청 DTO에서 `@AssertTrue(message = "나쁨일 경우 feedback은 필수")`가 붙은 `isValid()` 메서드를 보고 "필드도 아닌 메서드에 붙는 게 어떻게 동작하지?"가 궁금해서 정리. 같은 프로젝트의 엑셀 업로드 행 검증 DTO는 이 패턴을 수십 개 나열해 행 단위 검증기를 만들고 있었다.

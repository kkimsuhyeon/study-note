# 파라미터화 테스트 — `@ParameterizedTest` (JUnit 5)

> **한 줄 요약**: **같은 로직, 다른 입력**을 한 메서드로 여러 번 돌리는 기능. `@Test` 대신 `@ParameterizedTest`를 쓰고, 값 공급원(`@ValueSource`/`@EnumSource`/`@CsvSource`/`@MethodSource`)을 붙여 입력을 받는다. 핵심 판단: **검증 *내용*이 같고 입력만 다를 때만** 쓴다 — 케이스마다 기대가 다르면 그냥 별도 `@Test`.

관련 노트: [JUnit 라이프사이클·구조](./junit-lifecycle.md) · [AssertJ](./assertj.md) · [테스트 픽스처](./test-fixtures.md)

---

## 1. 기본 형태 — `@Test`를 교체하고 파라미터로 받는다

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

@ParameterizedTest                       // @Test 대신 (둘 다 붙이면 ❌)
@ValueSource(ints = {0, -1, -100})
void deduct_fail_whenNotPositive(int amount) {   // ← 공급된 값이 인자로 들어옴
    assertThatThrownBy(() -> user.deductBalance(BigDecimal.valueOf(amount)))
            .isInstanceOf(BusinessException.class);
}
// → 0, -1, -100 으로 3번 실행. 리포트에도 3건으로 표시.
```

- 의존성: `junit-jupiter-params` (Spring Boot의 `spring-boot-starter-test`에 보통 포함).
- 메서드가 **파라미터를 받는다**는 게 `@Test`와의 큰 차이.

> 💡 리포트 가독성: `@ParameterizedTest(name = "{0} → 예외")`로 케이스 이름을 커스터마이즈한다. `{0}`,`{1}`…은 인자, `{index}`는 회차 → 실패한 케이스를 한눈에 찾는다.

---

## 2. 값 공급원(Source) — 자주 쓰는 4개

| 어노테이션 | 공급 대상 | 예 |
|---|---|---|
| `@ValueSource` | 단일 값 목록(int/long/String/…) | `@ValueSource(strings = {"", " "})` |
| `@EnumSource` | enum 상수 | enum 전수/일부 |
| `@CsvSource` | **여러 인자 조합**(행 = 한 케이스) | `"1000, 500, 500"` |
| `@MethodSource` | 메서드가 만든 **복잡한 객체/스트림** | VO, 엔티티 등 |
| `@ArgumentsSource` | **`ArgumentsProvider` 구현 클래스** (여러 테스트서 재사용·확장) | `@ArgumentsSource(MyProvider.class)` |

### `@EnumSource` — enum 케이스 돌리기 (이 프로젝트 pay 예시)
```java
@ParameterizedTest
@EnumSource(value = PaymentStatus.class, names = {"SUCCESS", "FAIL", "CANCEL"})
void pay_fail_whenNotPending(PaymentStatus status) {
    Payment payment = PaymentFixture.aPayment().status(status).build();
    assertThatThrownBy(payment::pay)
            .isInstanceOf(BusinessException.class)
            .hasMessage(PaymentErrorCode.DISABLE_PAY.getMessage());
}
```
> 💡 **`mode = EXCLUDE`로 "하나만 빼고 전부"가 더 강하다** — 미래에 enum 값이 늘어도 자동 포함:
> ```java
> @EnumSource(value = PaymentStatus.class, names = "PENDING", mode = EnumSource.Mode.EXCLUDE)
> ```
> "PENDING이 아닌 모든 상태에선 결제 불가"를 의도 그대로 표현 + `REFUNDED` 추가돼도 테스트가 따라옴.

### `@CsvSource` — 입력→기대값 쌍
```java
@ParameterizedTest
@CsvSource({
    "1000, 500, 500",     // balance, 차감액, 기대결과
    "1000, 1000, 0"
})
void deduct(long balance, long amount, long expected) {
    // balance로 셋업 → deduct(amount) → expected 검증
    assertThat(result).isEqualByComparingTo(BigDecimal.valueOf(expected));
}
```
- 빈 문자열/널 표현: `@CsvSource({"'', ...", "...,"})` 또는 `nullValues`/`emptyValue` 옵션.

### `@MethodSource` — 객체·복잡 케이스
```java
@ParameterizedTest
@MethodSource("notPayableStatuses")
void pay_fail(PaymentStatus status) { ... }

static Stream<PaymentStatus> notPayableStatuses() {     // static, 메서드 이름을 문자열로 참조
    return Stream.of(PaymentStatus.SUCCESS, PaymentStatus.CANCEL);
}
// 여러 인자면 Stream<Arguments>: Arguments.of(a, b, expected)
```
- 기본은 **static** 메서드여야 하지만, `@TestInstance(Lifecycle.PER_CLASS)`면 non-static도 공급원으로 쓸 수 있다(→ [JUnit 라이프사이클](./junit-lifecycle.md)). 다른 클래스의 메서드는 `@MethodSource("com.example.Providers#notPayable")`처럼 FQN으로 참조.

### `@ArgumentsSource` — 재사용 가능한 공급원 클래스 (가장 일반적 확장점)

`@MethodSource`가 "테스트 클래스 안 *메서드*"로 인자를 만든다면, `@ArgumentsSource`는 **별도 클래스(`ArgumentsProvider` 구현체)**로 만든다. 여러 테스트 클래스에서 같은 인자 세트를 **재사용**하거나 생성 로직이 복잡할 때.

```java
@ParameterizedTest
@ArgumentsSource(NotPayableStatusProvider.class)
void pay_fail(PaymentStatus status) {
    assertThatThrownBy(() -> PaymentFixture.aPayment().status(status).build().pay())
            .isInstanceOf(BusinessException.class);
}

// 별도 클래스 — 다른 테스트에서도 재사용 가능
static class NotPayableStatusProvider implements ArgumentsProvider {
    @Override
    public Stream<? extends Arguments> provideArguments(ExtensionContext context) {
        return Stream.of(
                Arguments.of(PaymentStatus.SUCCESS),
                Arguments.of(PaymentStatus.CANCEL)
        );
    }
}
```

> 💡 **사실 `@ValueSource`/`@EnumSource`/`@CsvSource`/`@MethodSource`는 전부 내부적으로 `@ArgumentsSource` 위에 만들어진 특수화 버전이다.** (각각 전용 `ArgumentsProvider`를 메타 어노테이션으로 달고 있음) → `@ArgumentsSource`는 그 **토대이자 "기성 소스로 안 될 때 직접 만드는" 최종 확장점.**

| | `@MethodSource` | `@ArgumentsSource` |
|---|---|---|
| 인자 출처 | 테스트(또는 외부) **메서드** | **별도 클래스**(`ArgumentsProvider`) |
| 재사용 | 같은 클래스 위주 | **여러 테스트 클래스서 공유** 쉬움 |
| 적합 | 그 테스트 한정 케이스 | 공용·복잡·설정 필요한 케이스 |

> 대부분은 `@MethodSource`로 충분. `@ArgumentsSource`는 **같은 인자 세트를 여러 테스트가 공유**하거나, provider에 설정을 넘겨 동작을 바꾸고 싶을 때(커스텀 어노테이션 + `AnnotationConsumer`) 꺼낸다.

### `@ValueSource` 보충 — 단일 인자 전용 + `null`/`empty`는 별도 소스

가장 단순한 소스지만 제약이 둘 있다.

- **인자 1개만** 공급 (여러 컬럼 조합 ❌ → `@CsvSource`/`@MethodSource`).
- 지원 타입은 **리터럴**뿐: `short`·`byte`·`int`·`long`·`float`·`double`·`char`·`boolean`·`String`·`Class<?>`.
- ⚠️ **`null`은 못 넣는다**(리터럴이라). 널·빈값 케이스는 전용 소스를 **함께** 붙인다:

```java
@ParameterizedTest
@NullAndEmptySource                    // null + "" 두 케이스
@ValueSource(strings = {" ", "  "})    // + 공백들
void name_blank_fail(String name) {    // null, "", " ", "  " → 총 4번
    assertThatThrownBy(() -> user.changeName(name)).isInstanceOf(BusinessException.class);
}
```

| 소스 | 공급 |
|---|---|
| `@NullSource` | `null` 하나 |
| `@EmptySource` | 빈 값(`""`, 빈 `List`/배열 등) |
| `@NullAndEmptySource` | 위 둘 합침 |

> 보통 `@NullAndEmptySource` + `@ValueSource(strings = {" "})`로 "널·빈값·공백"을 한 번에 검증. (여러 소스를 한 메서드에 겹쳐 붙이면 케이스가 합산된다.)

---

## 3. ⚠️ 함정

- **`@Test`와 같이 쓰지 마라** — `@ParameterizedTest`로 *교체*. 같이 붙이면 중복 실행/오작동.
- **공급원 없으면 에러** — `@ParameterizedTest`만 있고 `@…Source`가 없으면 실행 불가.
- **케이스마다 검증이 다르면 파라미터화 금지** — 입력에 따라 "성공 vs 예외" 또는 기대 메시지가 갈리면, 메서드 안에 `if`가 생겨 더 나빠진다. 그땐 **별도 `@Test`** 가 맞다.
- **`@CsvSource` 타입 변환** — 문자열을 선언 타입으로 자동 변환. enum도 이름으로 변환됨(`"SUCCESS"` → `PaymentStatus.SUCCESS`).

---

## 4. 💡 판단 기준

| 상황 | 선택 |
|---|---|
| 같은 로직 + 입력만 다름(경계값, enum 전수) | **`@ParameterizedTest`** |
| 입력마다 기대(성공/예외/메시지)가 다름 | 별도 `@Test` |
| enum "하나 빼고 전부" | `@EnumSource(mode = EXCLUDE)` |
| 입력→기대 쌍 | `@CsvSource` |
| 복잡 객체/조합 | `@MethodSource` |

> 한 줄: **"로직은 하나, 입력만 N개"면 파라미터화로 묶어 중복을 없앤다.** enum 전수는 `@EnumSource(EXCLUDE)`가 미래에 강하고, 경계값은 `@ValueSource`, 입력-기대 쌍은 `@CsvSource`. 단 **검증 내용이 갈리면 쪼개라**(억지 파라미터화는 if 분기를 부른다).

---

## 5. 참고
- [JUnit 5 - Parameterized Tests](https://docs.junit.org/current/user-guide/#writing-tests-parameterized-tests)
- 관련 노트: [JUnit 라이프사이클·구조](./junit-lifecycle.md) · [AssertJ](./assertj.md)

---

**학습 날짜**: 2026-06-19
**계기**: Payment `pay()`의 "PENDING이 아니면 실패"를 CANCEL 하나로만 테스트하다가 — 비-PENDING 상태 전수를 한 메서드로 돌리려고 `@ParameterizedTest`+`@EnumSource`를 처음 접함. 값 공급원 4종(Value/Enum/Csv/Method)과 "검증이 같을 때만 파라미터화" 기준 정리.

**추가(2026-06-25)**: §2에 `@ArgumentsSource`(별도 `ArgumentsProvider` 클래스로 재사용·확장 / 다른 소스들의 토대)와 `@ValueSource` 보충(단일 인자 전용·지원 타입·`null` 못 넣음 → `@NullSource`/`@EmptySource`/`@NullAndEmptySource`) 추가. 문서에 없던 소스 발견해서 보강.

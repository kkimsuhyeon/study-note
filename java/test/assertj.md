# AssertJ — `assertThat(...)` 단언 종합

> **한 줄 요약**: `assertThat(실제값).단언(...)` 형태로 읽기 쉬운 테스트 검증을 작성하는 라이브러리. Spring Boot의 `spring-boot-starter-test`에 기본 포함. 핵심 함정은 **BigDecimal은 `isEqualTo`가 아니라 `isEqualByComparingTo`**, 예외는 `assertThatThrownBy`.

관련 노트: [테스트 작성 가이드](./test-writing-guide.md) · [BigDecimal](../bigdecimal/bigdecimal.md)

```java
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
```

> ⚠️ **import 함정**: `assertThat`은 JUnit·Hamcrest에도 있다. 자동완성에 AssertJ 단언(`isEqualByComparingTo` 등)이 안 뜨면 **`org.assertj.core.api.Assertions`가 아닌 걸 import**한 것. 그것부터 확인.

---

## 1. 기본 형태

```java
assertThat(actual).isEqualTo(expected);   // actual을 assertThat으로 감싸고 → 단언 메서드 체이닝
```

`assertThat(x)`는 x의 타입에 맞는 **Assert 객체**(`AbstractBigDecimalAssert`, `AbstractStringAssert` 등)를 반환하고, 그 객체에 단언 메서드들이 붙어 있다. **단언 메서드는 값 자체가 아니라 이 Assert 객체에 있는 것.**

---

## 2. ⭐ BigDecimal — `isEqualByComparingTo` (가장 중요한 함정)

`BigDecimal.equals()`는 scale(소수 자릿수)까지 비교해서 `"500"`과 `"500.0"`을 다르다고 본다. `isEqualTo`는 내부적으로 `equals`를 쓰므로 **scale이 다르면 실패**한다.

```java
// ❌ scale까지 비교 → "500"과 "500.0"이면 실패
assertThat(user.getBalance()).isEqualTo(BigDecimal.valueOf(500));

// ✅ 내부적으로 compareTo 사용 → 값만 같으면 통과 (scale 무시)
assertThat(user.getBalance()).isEqualByComparingTo(BigDecimal.valueOf(500));
assertThat(user.getBalance()).isEqualByComparingTo("500");  // 문자열도 받음
```

| | `isEqualTo` | `isEqualByComparingTo` |
|---|---|---|
| 내부 비교 | `expected.equals(actual)` | `expected.compareTo(actual) == 0` |
| `"500"` vs `"500.0"` | **실패** (scale 0 ≠ 1) | **통과** (값만 같으면 OK) |

> BigDecimal 단언은 거의 항상 `isEqualByComparingTo`. (이유는 [BigDecimal §5](../bigdecimal/bigdecimal.md)) — `AbstractBigDecimalAssert`에 정의된 오래된 메서드라 버전 걱정 없음.

---

## 3. 예외 검증 — `assertThatThrownBy`

테스트에서 가장 자주 빠뜨리는 부분. "예외가 나는지"를 단언으로 검증한다.

```java
assertThatThrownBy(() -> user.deductBalance(BigDecimal.valueOf(-100)))
    .isInstanceOf(IllegalArgumentException.class)      // 예외 타입
    .hasMessageContaining("음수");                      // 메시지(선택)
```

```java
// 예외가 "안" 나는지도 검증 가능
assertThatCode(() -> user.deductBalance(BigDecimal.TEN))
    .doesNotThrowAnyException();
```

> JUnit의 `assertThrows`도 가능하지만, AssertJ는 메시지·원인까지 체이닝으로 검증해 더 풍부하다.

### 메시지 단언 — `hasMessage` vs `hasMessageContaining`

| 단언 | 의미 |
|------|------|
| `hasMessage("...")` | 메시지 **완전 일치** |
| `hasMessageContaining("...")` | 메시지 **부분 포함** |
| `hasMessageStartingWith` / `hasMessageEndingWith` | 접두/접미 일치 |
| `hasMessageMatching("정규식")` | 정규식 매칭 |

### 도메인 예외(ErrorCode) 검증 — 실무 패턴

에러 코드 enum을 쓰는 프로젝트는, 메시지를 하드코딩하지 말고 **enum이 들고 있는 메시지/코드와 비교**한다.

```java
// (A) 메시지로 비교 — enum의 getMessage()와 완전 일치
assertThatThrownBy(() -> user.deductBalance(BigDecimal.valueOf(1001)))
    .isInstanceOf(BusinessException.class)
    .hasMessage(UserErrorCode.NOT_ENOUGH_POINT.getMessage());

// (B) errorCode 필드로 비교 — 더 견고 (extracting으로 필드 추출)
assertThatThrownBy(() -> user.deductBalance(BigDecimal.valueOf(1001)))
    .isInstanceOf(BusinessException.class)
    .extracting("errorCode")
    .isEqualTo(UserErrorCode.NOT_ENOUGH_POINT);
```

> ⚠️ **메시지 비교(A)는 메시지 문구가 바뀌면 깨진다.** 문구는 자주 바뀌므로, 가능하면 **errorCode 필드 비교(B)** 가 더 안정적이다. 단 (B)는 예외에 `errorCode` 같은 식별 필드가 있어야 함. (이 구분은 server-java `TEST_GUIDE.md` §7.4 권장 사항과 동일)

---

## 4. 자주 쓰는 단언

```java
// 동등/null/불리언
assertThat(x).isEqualTo(y);
assertThat(x).isNotNull();           assertThat(x).isNull();
assertThat(flag).isTrue();           assertThat(flag).isFalse();

// 숫자 비교
assertThat(n).isPositive();          assertThat(n).isZero();
assertThat(n).isGreaterThan(10).isLessThanOrEqualTo(100);
assertThat(d).isCloseTo(3.14, within(0.01));   // 부동소수점 근사

// 문자열
assertThat(s).contains("부분").startsWith("Hello").hasSize(5);

// 컬렉션 (강력함)
assertThat(list).hasSize(3)
    .contains("a", "b")
    .containsExactly("a", "b", "c")          // 순서까지 정확히
    .extracting("name").containsExactly("kim", "lee");  // 필드 뽑아서 비교

// Optional (자세히는 §4-1)
assertThat(opt).isPresent().get().isEqualTo(value);
assertThat(opt).isEmpty();
```

---

## 4-1. Optional 검증 — "존재 확인 + 값으로 체이닝"

AssertJ의 `.get()`은 **자바 `Optional.get()`이 아니라 Assert 메서드**다. **존재를 자동 검증**한 뒤 값으로 내려가, 이어서 체이닝할 수 있다. (비어 있으면 `NoSuchElementException`이 아니라 명확한 단언 실패 메시지)

```java
Optional<User> found = repo.findByEmail("a@test.com");

// 값으로 내려가 이어서 검증 ("존재 가정하고 get해서 체인")
assertThat(found).get()                      // isPresent 자동 검증 + User로 내려감
    .extracting(User::getEmail)
    .isEqualTo("a@test.com");
```

| 패턴 | 용도 |
|------|------|
| `assertThat(opt).contains(v)` / `.hasValue(v)` | present + 값이 v와 같음 (한 방) |
| `assertThat(opt).get().satisfies(o -> {...})` | 값으로 내려가 **여러 필드** 검증 |
| `assertThat(opt).hasValueSatisfying(o -> {...})` | 위와 동의, 더 명시적 |
| `assertThat(opt).map(User::getEmail).hasValue("..")` | 변환 후 비교 |
| `assertThat(opt).get(as(STRING)).startsWith("..")` | **타입 지정** get → 그 타입 전용 단언 |
| `assertThat(opt).isEmpty()` / `.isNotPresent()` | 비어있음 |

```java
// 여러 필드 — get().satisfies 또는 hasValueSatisfying
assertThat(found).get().satisfies(u -> {
    assertThat(u.getEmail()).isEqualTo("a@test.com");
    assertThat(u.getBalance()).isEqualByComparingTo("0");
});

// 타입 지정 get (String 전용 단언 쓰기)
import static org.assertj.core.api.InstanceOfAssertFactories.STRING;
import static org.assertj.core.api.Assertions.as;
assertThat(optName).get(as(STRING)).startsWith("kim");
```

> 💡 단순 값 일치 → `contains`/`hasValue` / 값으로 내려가 여러 검증 → `.get().satisfies(...)` 또는 `hasValueSatisfying`. `isPresent()` 없이 `.get()`만 써도 존재 검증이 포함된다.

---

## 5. 단언 묶기 / 설명

```java
import static org.assertj.core.api.Assertions.assertThat;

// 여러 단언을 한 번에 (하나 실패해도 나머지 다 검사 → 실패 원인 한눈에)
org.assertj.core.api.Assertions.assertThatObject(user)
    .satisfies(u -> {
        assertThat(u.getId()).isEqualTo("u1");
        assertThat(u.getBalance()).isEqualByComparingTo("500");
    });

assertThat(value).as("잔액은 충전 후 500이어야 함").isEqualByComparingTo("500");  // 실패 메시지에 설명 표시
```

---

## 6. 참고
- [AssertJ 공식 - Core 단언 가이드](https://assertj.github.io/doc/)
- 관련 노트: [테스트 작성 가이드](./test-writing-guide.md) · [BigDecimal](../bigdecimal/bigdecimal.md) · [@Lock 실무 패턴(동시성 테스트)](../jpa/lock-practical.md)

---

**학습 날짜**: 2026-05-28 (2026-06-03 hasMessage vs hasMessageContaining·ErrorCode 예외 검증 패턴 보강 — server-java UserTest.java 기반)
**계기**: `isEqualByComparingTo`가 BigDecimal 메서드가 아니라 AssertJ 단언이라는 걸 정리하며, 자주 쓰는 AssertJ 단언을 함께 모음. 이후 실제 테스트 코드(UserTest)에서 쓴 `hasMessage`·ErrorCode 검증을 보강

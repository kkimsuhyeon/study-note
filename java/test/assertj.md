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

### ⚠️ 별도 `isPresent()` 줄은 중복

```java
// ❌ 중복 — .get()이 이미 존재를 검사 + assertThat(found) 두 번
assertThat(found).isPresent();
assertThat(found).get().satisfies(u -> { ... });

// ✅ 한 줄로 — .get()이 존재까지 검증
assertThat(found).get().satisfies(u -> { ... });

// ✅ "존재 후 내용" 의도를 남기려면 한 체인으로
assertThat(found).isPresent().get().satisfies(u -> { ... });
```

> 비어 있으면 `.get()`이 `Expecting Optional to contain a value`로 실패하므로, 앞에 `isPresent()`를 따로 둬도 커버리지는 같다. 같은 대상을 두 문장으로 나누지 말 것.

---

## 4-2. 체이닝 — 같은 대상 vs 대상 이동(navigation)

체이닝은 두 종류다. 이 구분이 핵심.

| 종류 | 예 | 동작 |
|------|-----|------|
| **같은 대상 단언** | `isEqualTo` `hasSize` `isNotBlank` `contains` | 같은 assert 반환 → 계속 단언 |
| **대상 이동(navigation)** | `extracting` `get` `first` `filteredOn` `singleElement` `cause` `asInstanceOf` `usingRecursiveComparison` | **검증 대상을 갈아탐** → 이후 새 대상에 맞는 단언 |

```java
// extracting — 필드 뽑아 그 값으로 이동 (단일 / tuple)
assertThat(user).extracting(User::getEmail).isEqualTo("a@test.com");
assertThat(user).extracting(User::getEmail, User::getRole)
    .containsExactly("a@test.com", UserRole.USER);

// returns — 메서드 반환값 인라인 검증 (여러 개 체인)
assertThat(user)
    .returns("a@test.com", User::getEmail)
    .returns(UserRole.USER, User::getRole);

// 컬렉션 navigation
assertThat(users).hasSize(2)
    .extracting(User::getEmail)                        // 각 요소 필드 → 리스트
    .containsExactlyInAnyOrder("a@..", "b@..");
assertThat(users).filteredOn(User::isAdmin)           // 필터 후
    .singleElement()                                   // 정확히 1개 + 그 요소로 이동
    .extracting(User::getEmail).isEqualTo("admin@..");
assertThat(users).first().returns("a@..", User::getEmail);  // first/last/element(i)

// 예외 navigation
assertThatThrownBy(() -> service.x())
    .isInstanceOf(BusinessException.class)
    .hasMessageContaining("부족")
    .extracting("errorCode").isEqualTo(UserErrorCode.NOT_ENOUGH_POINT);
assertThatThrownBy(...).rootCause().hasMessage("..");  // 원인 예외로 이동

// usingRecursiveComparison — DTO 필드 단위 통째 비교 (equals() 불필요)
assertThat(actual)
    .usingRecursiveComparison()
    .ignoringFields("id", "createdAt")
    .isEqualTo(expected);

// asInstanceOf / asString — 타입 좁혀 그 타입 전용 단언
assertThat(value).asString().startsWith("ka");
```

> 💡 실무 빈출 3개: **`extracting`**(필드/리스트 뽑아 비교), **`usingRecursiveComparison`**(DTO 비교 — `equals` 없이 필드 단위, id 등 제외 가능), **`singleElement`/`filteredOn`**("조건 맞는 1개"). navigation 뒤엔 **대상이 바뀌었다**는 걸 의식하면 체인이 안 꼬인다.

---

## 4-3. 컬렉션 매칭 — `contains*` 계열 + 순서/빈값

리스트(필터·목록 조회 결과 등) 검증의 핵심. `Page.getContent()`는 `List`라 그대로 이 단언들이 붙는다. ([Page/Pageable](../jpa/spring-data-query.md))

| 단언 | 의미 |
|---|---|
| `contains("a", "b")` | **포함** (순서·여분 무관) |
| `containsExactly("a", "b", "c")` | **순서까지** 정확히 그것들만 |
| `containsExactlyInAnyOrder("c", "a", "b")` | 그것들만 (**순서 무관**) |
| `containsOnly("a", "b")` | 이 집합만 (중복·순서 무관) |
| `doesNotContain("z")` | 미포함 |
| `isEmpty()` / `isNotEmpty()` | **0건** / 1건 이상 |

```java
assertThat(result.getContent())
    .extracting(User::getEmail)
    .containsExactlyInAnyOrder("a@test.com", "b@test.com");

assertThat(result.getContent()).isEmpty();   // 필터가 아무것도 못 잡음 = 0건
```

> ⚠️ **순서 함정**: 정렬(`Sort`)을 안 준 쿼리 결과나 `Set`은 **순서가 보장 안 된다.** `containsExactly`(순서까지 검사)는 환경 따라 깨질 수 있으니 결과 2건↑이면 **`containsExactlyInAnyOrder`** 가 안전. 순서를 검증하려면 `PageRequest.of(0, n, Sort.by("email"))`로 정렬을 명시한 뒤 `containsExactly`.

> ⚠️ **`isEmpty()` 두 종류 구분**: 컬렉션 `isEmpty()` = "원소 0개", `Optional.isEmpty()` = "값 없음"(§4-1). 같은 이름이지만 대상 타입에 따라 다른 단언.

> 💡 **중복 단언 쌓지 말 것**: `containsExactly(...)`는 **크기+내용+순서**를 한 번에 못박는다 → 뒤에 `hasSize`·`isNotEmpty`를 또 붙이면 군더더기. `containsExactly("a@")` 하나면 "정확히 1건, a@"가 끝. 원소 1개면 `containsExactly` = `containsExactlyInAnyOrder`. **그 테스트 의도를 가장 잘 담는 강한 단언 하나**만 남긴다(약한 단언 여러 개 쌓기 ✗).

---

## 4-4. 객체 동등성 — `isSameAs` vs `isEqualTo` vs 필드 단언 (equals 함정)

객체 자체를 비교할 때, **그 객체에 `equals`가 있느냐**로 쓸 수 있는 단언이 갈린다.

| 단언 | 비교 방식 | 언제 통과 |
|---|---|---|
| `isSameAs(x)` | **`==` (참조)** | 메모리상 *똑같은 인스턴스*일 때만 |
| `isEqualTo(x)` | **`equals()`** | equals가 값 비교하면 값 같을 때 / **equals 없으면 `==`로 떨어짐** |
| 필드 단언 `getX()...` | 값 직접 | equals 없어도 OK |
| `usingRecursiveComparison()` | 필드 단위 자동 | equals 없이 전체 필드 비교(§4-2) |

```java
// UserCriteria: @Getter @Builder 만 (equals 재정의 없음)
assertThat(captor.getValue()).isSameAs(UserCriteria.empty());  // ❌ empty()는 매번 new → 다른 인스턴스
assertThat(captor.getValue()).isEqualTo(UserCriteria.empty()); // ❌ equals 없어 ==로 떨어짐 → 역시 실패
assertThat(captor.getValue().getEmail()).isNull();             // ✅ 값을 직접 본다
assertThat(captor.getValue().getRole()).isNull();
```

- ⚠️ **`isSameAs`는 팩토리/빌더로 매번 새로 만드는 객체엔 거의 항상 실패** — `empty()`, `of()`, `builder().build()`는 호출마다 다른 인스턴스. 같은 객체 레퍼런스를 직접 넘긴 경우(예: pass-through로 "내가 준 그 객체가 그대로 나왔나")에만 `isSameAs`가 의미 있다.
- ⚠️ **`equals` 없는 DTO는 `isEqualTo`도 참조 비교**(Object 기본 equals = `==`) → 값이 같아도 실패. (Mockito 인자 매칭이 안 되던 것과 같은 뿌리 — stub의 `when(repo.f(criteria))`도 equals로 매칭하니 새 객체면 안 맞음 → `any()`+captor를 쓰는 이유.)

> 💡 **값 비교하려면: equals 있으면 `isEqualTo`, 없으면 필드 단언(또는 `usingRecursiveComparison`).** `isSameAs`는 "같은 값"이 아니라 "같은 객체"를 묻는 것 — pass-through 검증 외엔 거의 안 씀. **테스트 편하자고 프로덕션에 `equals`를 넣지 말 것** — equals는 "값 동등성이 도메인에 필요할 때"(VO 등)만 추가하고, 단지 비교하려는 거면 필드 단언이 맞다.

---

## 5. 단언 묶기 / 설명

```java
import static org.assertj.core.api.Assertions.assertThat;

// 여러 필드를 한 람다로 묶어 검증 (satisfies = 첫 실패에서 멈춤 — soft 아님)
org.assertj.core.api.Assertions.assertThatObject(user)
    .satisfies(u -> {
        assertThat(u.getId()).isEqualTo("u1");
        assertThat(u.getBalance()).isEqualByComparingTo("500");
    });

assertThat(value).as("잔액은 충전 후 500이어야 함").isEqualByComparingTo("500");  // 실패 메시지에 설명 표시
```

> ⚠️ **`satisfies`는 soft assertion이 아니다 — 첫 단언 실패에서 멈춘다.** "하나 실패해도 나머지까지 다 검사해 한 번에 모아 보고"는 **`assertSoftly`(SoftAssertions)**의 동작이다:
> ```java
> import static org.assertj.core.api.SoftAssertions.assertSoftly;
> assertSoftly(softly -> {
>     softly.assertThat(u.getId()).isEqualTo("u1");
>     softly.assertThat(u.getBalance()).isEqualByComparingTo("500");   // 앞이 실패해도 이것도 검사
> });   // 블록 끝에서 실패들을 한꺼번에 리포트
> ```
> `satisfies` = "여러 필드를 한 묶음으로 표현", `assertSoftly` = "실패를 모아 한 번에 보기". 용도가 다르다.

---

## 6. 참고
- [AssertJ 공식 - Core 단언 가이드](https://assertj.github.io/doc/)
- 관련 노트: [테스트 작성 가이드](./test-writing-guide.md) · [BigDecimal](../bigdecimal/bigdecimal.md) · [@Lock 실무 패턴(동시성 테스트)](../jpa/lock-practical.md)

---

**학습 날짜**: 2026-05-28 (2026-06-03 hasMessage vs hasMessageContaining·ErrorCode 예외 검증 패턴 보강 — server-java UserTest.java 기반)
**계기**: `isEqualByComparingTo`가 BigDecimal 메서드가 아니라 AssertJ 단언이라는 걸 정리하며, 자주 쓰는 AssertJ 단언을 함께 모음. 이후 실제 테스트 코드(UserTest)에서 쓴 `hasMessage`·ErrorCode 검증을 보강

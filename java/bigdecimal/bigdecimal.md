# BigDecimal — 돈·정밀 계산을 위한 십진 타입

> **한 줄 요약**: `double`/`float`는 2진 부동소수점이라 `0.1 + 0.2 != 0.3`처럼 오차가 난다. 돈·정밀 계산은 **`BigDecimal`**(임의 정밀도 십진)으로 한다. 단 **불변·`equals`가 scale까지 비교·나눗셈은 반올림 필수** 등 함정이 많다.

관련 노트: [AssertJ 단언 (isEqualByComparingTo)](../test/assertj.md) · [Read-Modify-Write (잔액 예시)](../jpa/read-modify-write.md)

---

## 1. 왜 쓰나 — double의 부동소수점 오차

```java
System.out.println(0.1 + 0.2);          // 0.30000000000000004 ❗
System.out.println(0.1 + 0.2 == 0.3);   // false ❗
```

`double`은 2진법으로 `0.1`을 정확히 표현 못 한다(무한소수). 그래서 **돈·수량·이자처럼 정확해야 하는 값에 `double`을 쓰면 안 된다.** `BigDecimal`은 10진수를 정확히 담는다.

> 규칙: **돈은 `BigDecimal`(또는 정수 단위 — 원 단위 `long`).** 절대 `double`/`float`로 금액 계산 금지.

---

## 2. 생성 — `new BigDecimal(double)`이 함정

```java
new BigDecimal("0.1");          // ✅ 정확히 0.1   (문자열 권장)
BigDecimal.valueOf(0.1);        // ✅ 0.1          (내부적으로 Double.toString)
new BigDecimal(0.1);            // ❌ 0.1000000000000000055511...  (double 오차를 그대로 흡수!)
```

| 생성 방법 | 결과 | 비고 |
|-----------|------|------|
| `new BigDecimal("0.1")` | 정확 | **문자열로 만드는 게 가장 안전** |
| `BigDecimal.valueOf(0.1)` | 정확 | double을 받지만 내부에서 안전 변환. 리터럴엔 편함 |
| `new BigDecimal(0.1)` | **오차** | double의 2진 오차를 그대로 가져옴 — **쓰지 말 것** |
| `BigDecimal.valueOf(100L)` | 정확 | long/int는 안전 |

> 상수: `BigDecimal.ZERO`, `BigDecimal.ONE`, `BigDecimal.TEN`.

---

## 3. ⚠️ 불변(immutable) — 연산 결과를 받아야 한다

`BigDecimal`은 **불변**이다. `add()` 등은 자기를 바꾸지 않고 **새 객체를 반환**한다. String과 똑같은 함정.

```java
BigDecimal a = new BigDecimal("1000");
a.add(new BigDecimal("500"));     // ❌ 반환값을 안 받음 → a는 그대로 1000
System.out.println(a);            // 1000

a = a.add(new BigDecimal("500")); // ✅ 결과를 다시 받아야 함
System.out.println(a);            // 1500
```

| 연산 | 메서드 |
|------|--------|
| 덧셈 | `a.add(b)` |
| 뺄셈 | `a.subtract(b)` |
| 곱셈 | `a.multiply(b)` |
| 나눗셈 | `a.divide(b, scale, RoundingMode)` (→ §4) |
| 절댓값/부호반전 | `a.abs()` / `a.negate()` |

---

## 4. 나눗셈 — 반올림 안 주면 예외

나눗셈은 **딱 떨어지지 않으면 `ArithmeticException`**(무한소수)을 던진다. **scale + RoundingMode를 반드시** 줘야 한다.

```java
new BigDecimal("10").divide(new BigDecimal("3"));                       // 💥 ArithmeticException
new BigDecimal("10").divide(new BigDecimal("3"), 2, RoundingMode.HALF_UP); // ✅ 3.33
```

### RoundingMode 주요 값
| 모드 | 동작 | 예 (`2.5`) |
|------|------|-----------|
| `HALF_UP` | 반올림 (0.5 올림) — 가장 흔함 | 3 |
| `HALF_EVEN` | 은행가 반올림 (0.5는 짝수로) — 통계/회계 | 2 |
| `DOWN` | 버림 (0 방향) | 2 |
| `UP` | 올림 (0에서 먼 방향) | 3 |
| `FLOOR` / `CEILING` | 음의/양의 무한대 방향 | - |

> 돈 계산은 보통 `HALF_UP`. 회계 표준이 필요하면 `HALF_EVEN`.

---

## 5. ⚠️ 비교는 `compareTo`, `equals` 아님 (scale 함정)

`BigDecimal.equals()`는 **값 + scale(소수 자릿수)** 을 모두 본다. 그래서 `"500"`과 `"500.0"`이 **다르다고** 나온다.

```java
new BigDecimal("500").equals(new BigDecimal("500.0"));     // false ❗ (scale 0 vs 1)
new BigDecimal("500").compareTo(new BigDecimal("500.0"));  // 0      ✅ (값만 비교)
```

→ **"값이 같은가"는 항상 `compareTo`로 판단.**

### compareTo 반환값 (부호만 의미 있음)
```java
a.compareTo(b)
//  음수(-1) → a < b
//   0       → a == b (값 같음)
//  양수(1)  → a > b
```

```java
amount.compareTo(BigDecimal.ZERO) <  0   // amount < 0  (음수)
amount.compareTo(BigDecimal.ZERO) == 0   // amount == 0
amount.compareTo(BigDecimal.ZERO) <= 0   // amount <= 0 (0도 막음)
amount.compareTo(BigDecimal.ZERO) >  0   // amount > 0
```

### 0 판별도 compareTo (또는 signum)
```java
amount.equals(BigDecimal.ZERO)        // ❌ "0.00"은 false (scale 다름)
amount.compareTo(BigDecimal.ZERO) == 0 // ✅
amount.signum() == 0                   // ✅ signum: 부호만 (-1/0/1) 반환, 0 판별에 깔끔
```

> 검증 코드에서 "0 이하 막기"는 `amount.compareTo(ZERO) <= 0`, "음수만 막고 0 허용"은 `< 0`. **두 메서드(충전/차감)의 0 정책을 일관되게** 가져갈 것.

---

## 6. scale / stripTrailingZeros

- **scale** = 소수점 이하 자릿수. `new BigDecimal("500.00").scale()` → `2`.
- `setScale(2, RoundingMode.HALF_UP)` — 자릿수 고정(반올림). 돈을 "원 단위/소수 2자리"로 맞출 때.
- `stripTrailingZeros()` — 뒤 0 제거(`500.00` → `5E+2`. ⚠️ 정수부 0도 지수로 빠지니, 표시엔 반드시 `toPlainString()`: `5E+2` → `"500"`).

```java
new BigDecimal("500.0").setScale(0, RoundingMode.HALF_UP);  // 500
new BigDecimal("1.50").stripTrailingZeros().toPlainString(); // "1.5"
```

---

## 7. 자주 하는 실수 정리

| 실수 | 올바름 |
|------|--------|
| `double`로 돈 계산 | `BigDecimal` 또는 정수 단위 |
| `new BigDecimal(0.1)` | `new BigDecimal("0.1")` / `valueOf` |
| `a.add(b)` 결과 안 받음 | `a = a.add(b)` (불변) |
| `divide` 반올림 미지정 | `divide(b, scale, RoundingMode)` |
| `equals`로 값 비교 | `compareTo(...) == 0` |
| `equals(ZERO)`로 0 판별 | `compareTo(ZERO) == 0` / `signum() == 0` |

---

## 8. 참고
- [Java 공식 - BigDecimal](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/math/BigDecimal.html)
- [Baeldung - BigDecimal and BigInteger](https://www.baeldung.com/java-bigdecimal-biginteger)
- 관련 노트: [AssertJ 단언](../test/assertj.md) · [Read-Modify-Write](../jpa/read-modify-write.md)

---

**학습 날짜**: 2026-05-28
**계기**: 테스트에서 `isEqualByComparingTo`와 `compareTo(ZERO)`를 쓰다가 BigDecimal의 equals/compareTo/scale 차이와 돈 계산 주의점을 정리

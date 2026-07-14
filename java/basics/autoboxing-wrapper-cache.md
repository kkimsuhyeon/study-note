# 오토박싱 & 래퍼 캐시 — `Integer`끼리 `==` 하면 안 되는 이유

> **한 줄 요약**: `Integer`/`Long`은 **객체**라서 `==`는 값이 아니라 **참조**를 비교한다. 그런데 JVM이 **-128~127을 캐싱**해서 작은 값에선 우연히 `true`가 나온다 → **작은 데이터로 짠 테스트는 통과하고, 값이 커지는 순간 조용히 틀린다.** 래퍼끼리 비교는 `equals()` 또는 원시 타입으로 언박싱.

## 언제 쓰나 (= 언제 터지나)

- `Map<String, Integer>`, `List<Long>` 등 **컬렉션에 숫자를 담을 때** (제네릭은 객체만 받으므로 래퍼가 강제됨)
- **JPA 엔티티 ID** (`Long id`) 를 비교할 때 ← 실무에서 제일 자주 터지는 자리
- 카운팅/집계 결과(`merge`로 센 값)를 서로 비교할 때

## 메커니즘 — 왜 127까지만 되나

오토박싱 `Integer a = 5;` 는 컴파일러가 `Integer.valueOf(5)` 로 바꾼다. 그리고 `valueOf`는 **캐시를 먼저 뒤진다**:

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)      // 기본 -128 ~ 127
        return IntegerCache.cache[i + (-IntegerCache.low)];   // 캐시된 "같은 객체" 반환
    return new Integer(i);                                     // 범위 밖 → 매번 "새 객체"
}
```

JLS 5.1.7이 **-128~127 범위는 박싱 결과가 항상 동일 객체(`r1 == r2`)임을 보장**한다. 그래서:

```java
Integer a = 127, b = 127;
System.out.println(a == b);   // true   ← 캐시된 같은 객체

Integer c = 128, d = 128;
System.out.println(c == d);   // false  ← 각자 new Integer(128)  ⚠️
```

**같은 코드가 값 크기에 따라 다르게 동작한다.** 이게 이 버그의 악질적인 점.

### 래퍼별 캐시 범위

| 타입 | 캐시 범위 | 조정 가능? |
|---|---|---|
| `Integer` | -128 ~ 127 | **가능** (`-XX:AutoBoxCacheMax=N`) |
| `Long`, `Short`, `Byte` | -128 ~ 127 (고정) | 불가 |
| `Character` | 0 ~ 127 (고정) | 불가 |
| `Boolean` | true/false 둘 다 캐시 | — |
| `Float`, `Double` | **캐시 없음** | — |

> `Integer`의 상한은 **JVM 옵션(`-XX:AutoBoxCacheMax`)으로 바뀔 수 있다.** 즉 "127까지는 `==`가 되니까 괜찮다"는 가정은 **런타임 옵션에 의존하는 코드**라는 뜻 — 애초에 기대면 안 되는 이유.
> 참고로 `Long`은 실제로 캐시되지만 **JLS가 명시적으로 보장하진 않는다** (JDK-7190924). 즉 `Long`의 `==`는 스펙상 근거조차 없다.

## 해결 — 두 가지

```java
// ① equals() — 래퍼끼리 값 비교
if (!count1.equals(count2)) { ... }

// ② 원시 타입으로 받기 (권장) — ==가 값 비교로 동작
int c1 = map.getOrDefault(key1, 0);
int c2 = map.getOrDefault(key2, 0);
if (c1 != c2) { ... }
```

## ⚠️ 함정

- **`Integer` vs `int` 비교는 안전하다.** 한쪽이 원시 타입이면 **자동 언박싱되어 값 비교**가 된다.
  ```java
  Integer boxed = map.get(k);
  if (boxed == 0)         // ✅ 값 비교 (int 0과 비교 → 언박싱됨)
  if (boxed == other)     // ⚠️ 둘 다 Integer면 참조 비교
  ```
  → **이 미묘함이 오히려 위험하다.** "어제는 `==`가 잘 됐는데?" 하는 착각을 만든다. 한쪽이 원시냐 아니냐를 매번 따지느니, **꺼내는 즉시 `int`로 받는 습관**이 낫다.

- **언박싱은 NPE를 만든다.** `int n = map.get("없는키");` → 내부적으로 `null.intValue()` → 💥 NPE. `.intValue()`로 **명시해도 똑같이 터진다** (명시성은 안전장치가 아니다). → `getOrDefault(k, 0)`로 애초에 null을 안 만드는 게 해법.

- **캐시는 오토박싱/`valueOf`에서만 동작한다.** `new Integer(5)`는 범위 안이어도 항상 새 객체. (Java 9+ deprecated, 16+ 제거 대상이라 쓸 일은 없음)

- **`Long`이 실무에서 더 위험하다.** JPA 엔티티 ID가 보통 `Long`이라 **테스트 데이터(ID 1~10)에선 캐시 덕에 통과하다가, 운영에서 ID가 커지면 조용히 실패**한다.
  ```java
  if (order.getUserId() == user.getId())          // ⚠️ Long끼리 == → 참조 비교
  if (order.getUserId().equals(user.getId()))     // ✅
  ```

## 💡 판단 기준

- **기본은 원시 타입(`int`/`long`), 래퍼는 "어쩔 수 없을 때만".** 어쩔 수 없는 건 딱 둘 — **① 제네릭 안**(`Map<String, Integer>` — 컴파일러가 강제), **② `null`이 "값 없음"이라는 의미를 가질 때**(nullable 컬럼 매핑 등). 그 외 지역변수·계산·리턴은 전부 원시 타입.
- **컬렉션에서 꺼내는 즉시 원시로 받는다**: `int c = map.getOrDefault(k, 0);` ← 이 한 줄이 `==` 버그와 NPE를 **동시에** 막는다. *맵 안은 `Integer`, 꺼낸 뒤론 `int`* — 이 경계를 습관으로.
- **래퍼끼리 `==`/`!=`가 보이면 무조건 버그로 의심한다.** "지금 테스트는 통과하는데?"는 근거가 못 된다 — 127 이하라서 우연히 맞은 것일 수 있다.

### 구체 케이스 (막힌 지점)
코테 "완주하지 못한 선수"(프로그래머스)에서 참가자/완주자를 각각 `HashMap<String, Integer>`로 카운팅한 뒤, 개수가 다른 사람을 찾으려고 이렇게 썼다:

```java
Integer test1 = map1.getOrDefault(name, 0);
Integer test2 = map2.getOrDefault(name, 0);
if (test1 != test2) return name;   // ⚠️ Integer끼리 != → 참조 비교
```

- 동명이인 **100명** 케이스 → **통과** (값 100 ≤ 127 → 캐시된 같은 객체 → `!=`가 false)
- 동명이인 **200명** 케이스 → **오답** (값 200 > 127 → 서로 다른 객체 → 값이 같아도 `!=`가 true → **완주한 사람을 답이라고 반환**)

정확성 테스트는 다 통과하는데 **효율성 테스트만 실패**해서 "시간 초과인가?" 했지만, 사실은 **입력이 커서 카운트가 127을 넘은 것** — 성능 문제가 아니라 **오답**이었다.

→ **교훈: "큰 입력에서만 실패"가 항상 성능 문제인 건 아니다.** 값의 크기가 동작을 바꾸는 코드(래퍼 캐시)를 먼저 의심할 것.

## 참고

- [JLS 5.1.7 — Boxing Conversion (-128~127 동일 객체 보장)](https://docs.oracle.com/javase/specs/jls/se17/html/jls-5.html#jls-5.1.7)
- [Oracle Javadoc — Integer.valueOf](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Integer.html#valueOf(int))
- [JDK-7190924 — JLS는 Long 캐싱을 명시하지 않는다](https://bugs.openjdk.org/browse/JDK-7190924)
- 관련 노트: [Map 주요 메서드](../collections/map-methods.md) (카운팅 = merge), [BigDecimal](../bigdecimal/bigdecimal.md) (equals vs compareTo — "객체 비교는 ==가 아니다"의 다른 사례)

---
학습 날짜: 2026-07-14
계기: 코테(프로그래머스 "완주하지 못한 선수") 첫 문제에서 `Integer != Integer`로 효율성 테스트만 실패 → 시간 초과인 줄 알았으나 실제론 Integer 캐시(127) 경계 오답. 실무 JPA `Long id` 비교와 동일 메커니즘이라 확장 정리.

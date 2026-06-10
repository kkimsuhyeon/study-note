# 리스트 생성 — singletonList · List.of · Arrays.asList (불변 × 원소 개수)

> **한 줄 요약**: `Collections.singletonList(x)`의 "singleton"은 **불변이 아니라 "원소 1개"**라는 뜻이다 — 불변은 크기가 1로 고정되면서 *딸려오는* 성질. 작은 리스트를 만드는 방법들(`singletonList`/`List.of`/`Arrays.asList`/`new ArrayList`)은 **가변성 · null 허용 · 크기 고정**이 제각각이라, "어느 성질이 필요한가"로 고른다. 가장 큰 함정: 불변 리스트에 `add()` 하면 **컴파일 에러가 아니라 런타임 `UnsupportedOperationException`**.

---

## 1. singletonList — "원소 1개" 전용 리스트

```java
List<String> list = Collections.singletonList("A");

list.size();        // 1
list.get(0);        // "A"
list.add("B");      // ❌ UnsupportedOperationException
list.set(0, "B");   // ❌ UnsupportedOperationException
```

- **핵심 의미 = 원소 정확히 1개.** 크기가 1로 고정 → 그 결과로 불변이 된다.
- **경량**: `ArrayList`처럼 내부 배열·용량 관리를 하지 않고 **값 하나만 필드로 들고 있는 전용 구현** → 생성/메모리 비용이 더 작다.
- 이름에 "1개"라는 **의도가 드러난다** — 읽는 사람이 "이 리스트는 항상 1개구나"를 바로 안다.

### singleton 가족 (전부 `java.util.Collections`)

| 메서드 | 의미 |
|---|---|
| `Collections.singletonList(x)` | 원소 1개 불변 **List** |
| `Collections.singleton(x)` | 원소 1개 불변 **Set** |
| `Collections.singletonMap(k, v)` | 엔트리 1개 불변 **Map** |
| `Collections.emptyList()` / `emptySet()` / `emptyMap()` | 원소 0개 불변 (공유 인스턴스라 할당 0) |

---

## 2. ⭐ "원소 개수"와 "가변성"은 독립적인 두 축

"singletonList = 불변 리스트"로 읽으면 축이 섞인 것. 둘은 별개 축이고 singletonList는 그중 **"1개 × 불변" 칸**일 뿐이다:

| | 0개 | 1개 | n개 |
|---|---|---|---|
| **불변** | `emptyList()` | `singletonList(x)` | `List.of(a, b, c)` |
| **가변** | `new ArrayList<>()` | `new ArrayList<>(List.of(x))` | `new ArrayList<>(src)` |

---

## 3. 생성 방법 비교 (전체 지도)

| 방법 | 버전 | 가변? | null 원소 | 비고 |
|---|---|---|---|---|
| `Collections.singletonList(x)` | 1.3 | ❌ 불변 | ✅ 허용 | 1개 전용, 경량 |
| `List.of(...)` | **9+** | ❌ 불변 | ❌ **NPE** | 0~n개. 9+면 현대적 기본값 |
| `Arrays.asList(...)` | 1.2 | ⚠️ **고정 크기** (`set` ⭕ / `add` ❌) | ✅ | 원본 **배열의 뷰** |
| `new ArrayList<>(...)` | — | ✅ 가변 | ✅ | 진짜 수정할 때만 |
| `Collections.unmodifiableList(src)` | 1.2 | ❌ (뷰) | — | **복사 아님** — 원본 바뀌면 같이 바뀜 |
| `List.copyOf(src)` | 10+ | ❌ 불변 | ❌ NPE | 진짜 스냅샷 복사 |
| `stream.collect(Collectors.toList())` | 8+ | ⚠️ **보장 없음** | ✅ | 명세상 가변성 약속 없음(현재 구현은 ArrayList) |
| `stream.toList()` | 16+ | ❌ 불변 | ✅ | 16+면 스트림 종결은 이걸로 |

---

## 4. 실전 패턴 — 어디서 쓰게 되나

### (a) 단건 → List 어댑트: "주는 쪽은 1개, 받는 쪽은 List"
```java
// factory는 단건을 주는데 merger는 List를 받을 때
WorkAttendanceTrace trace = AttendanceTraceFactory.fromWorkSystem(exp, ...);
ExpectedWorkTime r = AttendanceTraceMerger.merge(Collections.singletonList(trace));
```

### (b) Optional → List 변환
```java
// 값 있으면 [그 값], 없으면 [] — Optional을 0~1개짜리 리스트로 펼친다
List<WorkAttendanceTrace> traces = factory.fromWorkSystem(...)   // Optional<T>
        .map(Collections::singletonList)
        .orElse(Collections.emptyList());
```

### (c) 테스트 입력 — 1개짜리 입력을 간결·불변으로
```java
merger.merge(Collections.singletonList(holiday));   // "입력은 이 1개" 가 한눈에
```
불변이라 **테스트 대상이 입력 리스트를 몰래 수정하면 즉시 예외** → 입력 불변 보장이 공짜로 따라온다.

---

## 5. ⚠️ 함정

- **불변 위반은 런타임에 터진다.** 타입은 다 같은 `List<T>`라 시그니처만 보고는 가변/불변을 구분할 수 없다 → `add()` 하는 순간 `UnsupportedOperationException`. 받은 리스트를 수정해야 하면 `new ArrayList<>(받은것)`으로 복사부터.
- **`Arrays.asList`는 불변이 아니라 "고정 크기"** — `add`/`remove`는 ❌지만 **`set`은 된다.** 게다가 원본 배열의 뷰라서 배열을 바꾸면 리스트에도 비친다(반대도 마찬가지).
- **`List.of`는 null 불허** — 원소에 null 넣으면 NPE. (덜 알려진 함정: `List.of("a").contains(null)`도 NPE를 던진다.) null 원소가 정말 필요하면 `singletonList` — 단, 그 전에 "리스트에 null이 들어가는 설계"부터 의심.
- **`unmodifiableList`는 복사가 아니라 뷰** — 감싼 원본을 바꾸면 "불변" 리스트 내용도 바뀐다. 진짜 스냅샷이 필요하면 `List.copyOf`(10+).

---

## 6. 💡 판단 기준

| 상황 | 선택 |
|---|---|
| 원소 1개, 안 바꿈 (Java 8↓ 또는 null 필요) | `Collections.singletonList(x)` |
| 원소 0~n개, 안 바꿈 (Java 9+) | **`List.of(...)`** — 기본값 |
| 만들고 나서 add/remove 할 것 | `new ArrayList<>(...)` |
| 받은 리스트의 불변 스냅샷 | `List.copyOf(src)` (10+) |

> 💡 한 줄: **`singletonList`를 보면 "불변 리스트"가 아니라 "원소 1개짜리"로 읽어라** — 불변은 결과지 이름의 뜻이 아니다. 그리고 작은 리스트 생성법은 전부 **(원소 개수 × 가변성 × null 허용)** 세 성질의 조합 — 이름이 아니라 필요한 성질로 고른다.

---

## 7. 참고
- [Collections.singletonList (Java SE API)](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Collections.html#singletonList(T))
- [List.of (Java SE API)](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/List.html#of(E))

---

**학습 날짜**: 2026-06-10
**계기**: `AttendanceTraceFactory.fromWorkSystem`(단건/Optional 리턴) 결과를 `AttendanceTraceMerger.merge(List)`에 넘기는 테스트에서 `.map(Collections::singletonList).orElse(emptyList())`를 보고 "왜 이렇게 썼지?"에서 출발 — singletonList가 **불변이라는 뜻인 줄 알았는데, 핵심은 "원소 1개"고 불변은 딸려오는 성질**임을 잡음. 김에 List.of/Arrays.asList/copyOf까지 생성법 지도를 정리.

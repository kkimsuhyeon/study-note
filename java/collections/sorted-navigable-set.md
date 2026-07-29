# SortedSet · NavigableSet — 정렬 집합에서 min/max·근처 탐색

> **한 줄 요약**: `TreeSet`의 진짜 인터페이스는 `NavigableSet`이다 — 원소가 항상 정렬돼 있고, 양 끝(`first/last`)과 "근처"(`floor/ceiling/lower/higher`)를 O(log n)에 꺼낸다. `Collections.min/max`로 전체 순회할 거면 애초에 정렬 집합을 쓰고 양 끝을 바로 꺼내는 게 맞다. 핵심 함정: TreeSet의 "같음"은 **equals가 아니라 compare==0** — 비교 기준이 곧 중복 판정 기준이 된다.

## 계층 위치

```
Set
 └─ SortedSet        (1.2)  정렬 유지 + first/last + headSet/tailSet/subSet
     └─ NavigableSet (1.6)  + floor/ceiling/lower/higher + pollFirst/pollLast + descendingSet
         └─ TreeSet          레드-블랙 트리 구현체 — add/remove/contains O(log n)
         └─ ConcurrentSkipListSet   동시성 버전 (스킵 리스트)
```

- `NavigableSet`은 Java 6에서 `SortedSet`을 확장한 후속 — **새 코드에서 정렬 집합을 선언할 땐 사실상 이걸 쓴다** (SortedSet만으로 되는 경우에도, TreeSet이 어차피 NavigableSet).
- Map 쪽에 똑같은 짝이 있다: `SortedMap → NavigableMap → TreeMap` (`firstKey/floorEntry/ceilingKey…`). 개념이 1:1 대응이라 한쪽을 알면 다른 쪽도 아는 것.
- 전체 컬렉션 계층은 → [collection-hierarchy.md](./collection-hierarchy.md)

## 언제 쓰나

- 원소를 넣으면서 **정렬 상태가 계속 유지**돼야 할 때 (한 번만 정렬하면 되면 List + sort가 낫다)
- **min/max를 반복해서** 꺼낼 때 — 날짜 집합에서 조회 범위(시작~끝) 잡기 등
- "이 값 **이하 중 가장 큰 것**" 같은 근처 탐색 — 스케줄에서 기준일 직전 이벤트 찾기, 등급 커트라인 등
- 범위 뷰(`subSet`)로 구간만 다룰 때

## 사용 예시 (문법)

### 양 끝 — first / last / pollFirst / pollLast
```java
NavigableSet<LocalDate> dates = new TreeSet<>(Set.of(d0608, d0615, d0620));

dates.first();       // 6/8  — 최소. 비어 있으면 NoSuchElementException!
dates.last();        // 6/20 — 최대
dates.pollFirst();   // 최소를 "꺼내면서 제거". 비어 있으면 null (예외 아님 — first와 다름)
dates.pollLast();
```

### 근처 탐색 — floor / ceiling / lower / higher
```java
dates.floor(d0617);    // 6/15 — d0617 "이하" 중 가장 큰 것 (≤)
dates.ceiling(d0617);  // 6/20 — d0617 "이상" 중 가장 작은 것 (≥)
dates.lower(d0615);    // 6/8  — "미만" (<)   ← 자기 자신 제외
dates.higher(d0615);   // 6/20 — "초과" (>)   ← 자기 자신 제외
// 없으면 전부 null 반환
```
| 메서드 | 방향 | 자기 포함 |
|---|---|---|
| `floor(e)` | ↓ 아래로 | 포함 (≤) |
| `lower(e)` | ↓ 아래로 | 제외 (<) |
| `ceiling(e)` | ↑ 위로 | 포함 (≥) |
| `higher(e)` | ↑ 위로 | 제외 (>) |

### 범위 뷰 — headSet / tailSet / subSet
```java
dates.headSet(d0615);                 // 6/15 "미만" (SortedSet판 — to 제외 고정)
dates.tailSet(d0615);                 // 6/15 "이상" (from 포함 고정)
dates.subSet(d0608, true, d0615, true); // NavigableSet판 — 포함 여부를 boolean으로 지정
dates.descendingSet();                // 역순 "뷰"
```
- 전부 **복사본이 아니라 뷰** — 뷰에 add/remove하면 원본에도 반영된다. 뷰 범위 밖 원소를 add하면 `IllegalArgumentException`.

### 만들기 — 정렬 기준
```java
new TreeSet<>();                                  // 원소의 Comparable(자연 순서) 사용
new TreeSet<>(Comparator.comparing(Emp::getNo));  // 별도 기준
stream.collect(Collectors.toCollection(TreeSet::new));  // 스트림에서 바로
```
- Comparable도 아니고 Comparator도 안 줬으면 **add 시점에 `ClassCastException`** (컴파일 에러 아님).
- 자연 순서 TreeSet에 `null` add → NPE (비교를 못 하니까). null 허용하려면 `Comparator.nullsFirst(...)`.

## HashSet vs TreeSet 선택

| | HashSet | TreeSet |
|---|---|---|
| contains/add | O(1) | O(log n) |
| 순회 순서 | 무작위 | 정렬 순서 |
| min/max | `Collections.min/max` — O(n) 전체 순회 | `first()/last()` — O(log n) |
| 필요 조건 | equals/hashCode | Comparable 또는 Comparator |

- **정렬이 필요 없으면 HashSet.** "마지막에 한 번만 정렬된 결과"가 필요하면 HashSet(또는 스트림) + `sorted()`가 TreeSet 유지비용보다 싸다.
- `Collections.min/max`는 인자가 정렬 집합이어도 **모른 채 전체를 순회**한다 — TreeSet에 쓰고 있다면 `first()/last()`로 바꿀 신호.

## ⚠️ 함정 — "같음"이 equals가 아니다

TreeSet의 중복 판정·조회는 전부 **compare == 0** 기준이다. Set 인터페이스의 계약(equals 기반)과 어긋날 수 있다:

```java
// BigDecimal: equals는 scale까지 비교, compareTo는 값만 비교
Set<BigDecimal> hash = new HashSet<>(List.of(new BigDecimal("1.0"), new BigDecimal("1.00")));
Set<BigDecimal> tree = new TreeSet<>(List.of(new BigDecimal("1.0"), new BigDecimal("1.00")));
hash.size();  // 2 — equals가 다르니 둘 다 들어감
tree.size();  // 1 — compareTo==0 이라 "같은 원소"로 보고 하나 버림!
```

- 더 흔한 사고: `new TreeSet<>(Comparator.comparing(Emp::getDeptId))` — **부서가 같으면 사원이 통째로 dedup**된다. 한 필드 기준 Comparator를 준 순간 그 필드가 유일키가 되는 것. 정렬만 원했는데 원소가 사라졌다면 이걸 의심.
- 해법: 비교 기준에 tie-breaker를 붙인다 — `comparing(Emp::getDeptId).thenComparing(Emp::getEmpNo)`.

## 💡 판단 기준

- **반환 타입 선언이 곧 계약이다**: 그냥 `Set` 으로 선언하면 호출자는 정렬을 믿을 수 없다. "정렬돼 있고 양 끝을 꺼낼 수 있다"가 API의 의미라면 `NavigableSet`(또는 SortedSet)으로 선언해서 드러낸다. 구현체 `TreeSet` 반환 타입은 지양 — 뒤에서 `ConcurrentSkipListSet`으로 바꿀 수 있어야 하니까.
- 실제 케이스: 마감 계산 대상일 함수가 처음엔 `Set<LocalDate>` 반환 + 호출부 `Collections.min/max` 였다. "이 집합의 min~max로 DB 조회 범위를 잡는다"가 용도의 본질이었어서 `NavigableSet` 반환 + `first()/last()` 로 바꿈 — 타입이 "범위를 꺼내 쓰라고 만든 집합"임을 말해준다.
- **TreeSet에 커스텀 Comparator를 넣는 순간 "정렬 기준 = 중복 기준"이 된다**는 걸 자문할 것 — 정렬만 필요하고 원소는 다 보존해야 하면 List + sort가 안전하다.

## 참고

- [NavigableSet (Java SE 11 Javadoc)](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/NavigableSet.html)
- [TreeSet (Java SE 11 Javadoc)](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/TreeSet.html) — "not consistent with equals" 명시
- [SortedSet (Java SE 11 Javadoc)](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/SortedSet.html)

---
학습: 2026-07-29 — client_api 마감 조회/계산 분리 중, 대상일 집합의 반환 타입을 `NavigableSet`으로 잡으면서 (`targetDates().first()/last()` 로 조회 범위 산출).

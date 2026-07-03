# 컬렉션 계층 구조 — List/Set/Queue는 Collection, Map은 아니다

> **한 줄 요약**: `Iterable → Collection → List/Set/Queue`가 한 계통이고 **Map은 Collection이 아니라 별도 계통**이다. 그래서 Map엔 stream()이 없고(뷰로 우회), Set엔 Map처럼 특별한 메서드가 거의 없다(= Collection 공통 메서드가 곧 집합 연산).

## 계층 그림

```
Iterable<T>                     ← for-each 가능
 └─ Collection<T>               ← add/remove/contains/size/stream ...
     ├─ List<T>                 ← 순서 + 인덱스 접근 (get(i)) + 중복 허용
     │    └─ ArrayList, LinkedList
     ├─ Set<T>                  ← 중복 없음 (수학적 집합)
     │    ├─ HashSet            ← 순서 없음, O(1) contains
     │    ├─ LinkedHashSet      ← 삽입 순서 유지
     │    └─ TreeSet            ← 정렬 순서 유지 (SortedSet/NavigableSet)
     └─ Queue<T> / Deque<T>     ← 꺼내는 순서가 규약 (FIFO/LIFO)
          └─ ArrayDeque, PriorityQueue

Map<K, V>                       ← Collection 아님! 별도 최상위
 ├─ HashMap / LinkedHashMap / TreeMap / EnumMap
 └─ ConcurrentHashMap
```

## Map이 Collection이 아닌 이유와 그 결과

- Collection은 "원소 T의 모음", Map은 "K→V 쌍의 대응"이라 원소가 하나의 타입이 아님 → 계통 분리.
- 그래서 Map엔 `stream()`, `iterator()`가 직접 없다. **뷰(view) 컬렉션으로 우회**한다:

```java
map.keySet()     // Set<K>       — 키의 집합 뷰
map.values()     // Collection<V> — 값 뷰 (중복 가능하니 Set이 아님!)
map.entrySet()   // Set<Map.Entry<K,V>> — 쌍의 집합 뷰
map.entrySet().stream().filter(e -> ...).map(Map.Entry::getValue)...
```

- 이 뷰들은 **복사본이 아니라 원본과 연결**돼 있다 — 뷰에서 remove하면 원본 맵에서도 빠진다 (add는 불가).

## Set에 "특별한 함수"가 따로 있나?

거의 없다 — **Collection 공통 메서드가 곧 집합 연산**이다:

| 집합 연산 | 메서드 (Collection 공통) |
|---|---|
| 합집합 A∪B | `a.addAll(b)` |
| 교집합 A∩B | `a.retainAll(b)` |
| 차집합 A−B | `a.removeAll(b)` |
| 부분집합 A⊆B | `b.containsAll(a)` |
| 원소 판별 | `a.contains(x)` — HashSet이면 O(1) |

- 위 연산들은 **원본을 변경(mutate)**한다. 원본 보존하려면 `new HashSet<>(a)`로 복사 후 연산.
- Set 고유라 할 만한 건: `add()`의 **boolean 반환** — 이미 있으면 false. "처음 보는 원소인가" 판별을 add 한 번으로 처리하는 관용구:

```java
Set<String> seen = new HashSet<>();
if (!seen.add(key)) continue;   // 중복이면 스킵 — contains+add 2번 할 필요 없음
```

- 정렬 계열(TreeSet)은 추가 메서드가 있다: `first()/last()/headSet()/tailSet()/ceiling()/floor()`.

## ⚠️ 함정

- **Set의 중복 판별 = equals + hashCode.** 직접 만든 클래스를 HashSet/HashMap 키로 쓰려면 둘 다 재정의 필수 (Lombok `@EqualsAndHashCode` 등). 재정의 안 하면 동일 내용 객체가 "다른 원소"로 들어간다.
- **HashSet의 정체는 HashMap이다** — 내부적으로 값에 더미 객체를 넣은 `HashMap<E, Object>`. Set을 이해하려면 Map의 해싱을 이해하면 끝.
- `values()`는 Collection이지 Set이 아니다 (값은 중복 가능).
- `Set.of(...)`(Java 9+)는 **생성 시점에 중복이 있으면 IllegalArgumentException** — List.of와 다른 점. (불변 생성 자체는 [리스트 생성](./list-creation.md) 참고)
- Map/Set 순회 순서: HashMap/HashSet은 **순서 보장 없음**(리해싱 시 바뀔 수도) → 순서가 의미 있으면 LinkedHash*/Tree*를 명시적으로 선택. "테스트에서만 우연히 순서가 맞는" 코드가 전형적 지뢰.

## 구현체 선택 요약

| 필요한 것 | 선택 |
|---|---|
| 그냥 빠른 집합/맵 | HashSet / HashMap |
| 넣은 순서 유지 | LinkedHashSet / LinkedHashMap |
| 정렬 순서(범위 검색) | TreeSet / TreeMap |
| 키가 enum | **EnumSet / EnumMap** (배열 기반, 가장 빠르고 가벼움) |
| 멀티스레드 | ConcurrentHashMap (+ `ConcurrentHashMap.newKeySet()`) |

## 💡 판단 기준

- **"중복 없이 모으고 contains로 판별"이 목적이면 Set, "키로 값을 찾는"이 목적이면 Map, "순서/인덱스"가 목적이면 List** — 자료구조 선택은 메서드가 아니라 목적으로. 근태 마감 rewrite에서 날짜 모음은 전부 `Set<LocalDate>`(전사휴일/공휴일 — contains 판별용), 날짜→하루치 데이터는 `Map<LocalDate, DailyClosing>`(키 조회용), 정렬 순회가 필요한 대상일만 `TreeSet`(allDates)이었다.
- Map을 순회/스트림하고 싶으면 무조건 entrySet/keySet/values 뷰로 — "Map은 Collection이 아니다"만 기억하면 헤맬 일 없다.

## 참고

- [Oracle Javadoc — Collection (Java 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Collection.html)
- [Oracle Tutorial — Collections Framework Overview](https://docs.oracle.com/javase/tutorial/collections/intro/index.html)
- 관련 노트: [Map 주요 메서드](./map-methods.md), [리스트 생성](./list-creation.md)

---
학습 날짜: 2026-07-03
계기: computeIfAbsent 정리하다가 "Set은 또 다른 게 있나? 다 Collection인가?"로 확장

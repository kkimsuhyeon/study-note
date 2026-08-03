# Stream API 종합 — 파이프라인 구조와 연산 지도

> **한 줄 요약**: Stream은 "컬렉션 순회 for문을 **선언적 파이프라인**(소스 → 중간 연산 → 최종 연산)으로 바꾼 것". 중간 연산은 전부 **lazy** — 최종 연산이 원소를 소비할 때만 실행된다. 핵심 함정: 스트림은 1회용, `peek`은 디버깅용, `toMap`은 key 중복 시 예외.

## 큰 그림 — 파이프라인 3부분

```java
List<String> result =
    list.stream()                      // ① 소스: Stream 생성
        .filter(s -> s.length() > 3)   // ② 중간 연산: Stream 반환, 몇 개든 체인 (lazy)
        .map(String::toUpperCase)      // ②
        .toList();                     // ③ 최종 연산: 딱 1개, 여기서 비로소 실행
```

- **중간 연산**: `Stream`을 반환 → 계속 체인 가능. 최종 연산 전까지 **아무것도 실행 안 됨** (lazy — 상세 메커니즘은 [lambda-execution-timing](./lambda-execution-timing.md))
- **최종 연산**: 값/컬렉션/Optional을 반환하거나 소비. 파이프라인은 이때 실행되고 스트림은 소모됨
- 실행은 "filter 전부 → map 전부"가 아니라 **원소 하나가 파이프라인을 끝까지 통과**하는 방식(수직 실행). 그래서 `findFirst` 같은 단락 연산이 앞 원소 몇 개만 처리하고 멈출 수 있다

## 소스 만들기

| 방법 | 예시 |
|---|---|
| 컬렉션 | `list.stream()`, `set.stream()`, `map.entrySet().stream()` |
| 배열 | `Arrays.stream(arr)` |
| 직접 나열 | `Stream.of("a", "b")`, `Stream.empty()` |
| 숫자 범위 | `IntStream.range(0, 10)` (미포함), `rangeClosed(1, 10)` (포함) |
| 무한 | `Stream.iterate(1, n -> n * 2)`, `Stream.generate(supplier)` — **limit 필수** |
| 기타 | `Files.lines(path)`, `"abc".chars()`, `Optional.stream()` |

## 중간 연산 지도

| 분류 | 연산 | 역할 |
|---|---|---|
| 변환 | `map(f)` | 원소를 **다른 값으로** 1:1 변환 |
| 변환 | `flatMap(f)` | 원소마다 스트림을 만들고 **한 층으로 펴기** — `List<List<T>>` → 원소들, "방마다 질문 리스트" → 전체 질문 |
| 변환 | `mapToInt/mapToLong` / `boxed()` | 기본형 스트림으로/에서 전환 (박싱 비용 절약 ↔ 복귀) |
| 선별 | `filter(pred)` | 조건 통과만 |
| 선별 | `distinct()` | 중복 제거 (**equals/hashCode 기준**) |
| 선별 | `limit(n)` / `skip(n)` | 앞 n개 / 앞 n개 버림 (페이징 조합) |
| 선별 | `takeWhile` / `dropWhile` (Java 9) | 조건이 깨질 때**까지**/깨진 **뒤부터** — 정렬된 데이터 전제 |
| 정렬 | `sorted()` / `sorted(comparator)` | 전 원소를 버퍼링해서 정렬 (여기서만 lazy가 아님) |
| 관찰 | `peek(consumer)` | 원소를 **그대로 통과**시키며 부수 작업 — 디버깅용 (아래 ⚠️) |

`map` vs `peek`: `map`은 반환값으로 원소를 **교체**, `peek`은 반환값 없이 **그대로 통과**.

## 최종 연산 지도

| 분류 | 연산 | 반환 |
|---|---|---|
| 수집 | `collect(Collectors.…)` | 컬렉션/Map/문자열 (아래 Collectors) |
| 수집 | `toList()` (Java 16) | **불변** List — `collect(toList())`보다 짧고 요즘 기본 |
| 집계 | `count()` / `min` / `max` | long / Optional |
| 집계 | `reduce(identity, op)` | 원소들을 하나로 접기 — 합계·누적 등 |
| 집계 | `sum()` / `average()` | **IntStream 등 기본형 스트림 전용** |
| 매칭 | `anyMatch` / `allMatch` / `noneMatch` | boolean — **단락 연산** (결과 확정되면 즉시 중단) |
| 검색 | `findFirst()` / `findAny()` | `Optional<T>` — 단락. findAny는 병렬에서 "아무거나 빨리" |
| 소비 | `forEach(consumer)` | void — 소비만 (외부 컬렉션 채우기엔 쓰지 말 것, 아래 ⚠️) |
| 배열 | `toArray(T[]::new)` | 배열 |

## Collectors 지도

```java
import static java.util.stream.Collectors.*;
```

| Collector | 결과 | 메모 |
|---|---|---|
| `toList()` / `toSet()` | List / Set | |
| `toMap(keyF, valF)` | Map | ⚠️ key 중복 → `IllegalStateException`, value가 null → NPE |
| `toMap(keyF, valF, mergeF)` | Map | 중복 시 병합 규칙 지정 — `(a, b) -> b` = 나중 값 승리 |
| `groupingBy(classifier)` | `Map<K, List<T>>` | **그룹핑** — 부모ID별 자식 리스트 조립의 주역 |
| `groupingBy(f, downstream)` | `Map<K, ?>` | 그룹 안을 재집계 — `groupingBy(f, counting())` = 그룹별 개수 |
| `partitioningBy(pred)` | `Map<Boolean, List<T>>` | true/false 2분할 |
| `joining(", ", "[", "]")` | String | 문자열 이어붙이기 |
| `counting()` / `summingInt` / `averagingInt` | Long/Integer/Double | 주로 groupingBy의 downstream |
| `mapping(f, downstream)` | | 그룹 안에서 변환 후 수집 |
| `collectingAndThen(c, finisher)` | | 수집 결과를 한 번 더 가공 (불변 래핑 등) |
| `teeing(c1, c2, merger)` (Java 12) | | 한 번 순회로 두 Collector 동시 적용 후 병합 |

실무 조합 예 — 방별 질문 리스트 / 방별 질문 수:

```java
Map<Long, List<Question>> byRoom = questions.stream()
        .collect(groupingBy(Question::getChatRoomId));

Map<Long, Long> countByRoom = questions.stream()
        .collect(groupingBy(Question::getChatRoomId, counting()));
```

## ⚠️ 함정/메커니즘

- **스트림은 1회용**: 최종 연산 후 재사용하면 `IllegalStateException: stream has already been operated upon or closed`. 두 번 쓰려면 소스에서 다시 `.stream()`
- **lazy + 단락으로 중간 연산이 안 돌 수 있다**: `findFirst`/`anyMatch`/`limit`은 필요한 만큼만 소비. Java 9+의 `count()`는 크기를 아는 소스면 **파이프라인 실행을 통째로 생략**하기도 → peek/map 안의 부수 작업이 0번 실행될 수 있음
- **peek은 "체인 중간에 println 꽂는 자리"**: `filter(...).map(...)` 사이엔 문장을 넣을 수 없으니, 원소를 그대로 통과시키며 관찰하는 전용 연산을 둔 것 (Javadoc: "mainly to support debugging"). lazy라서 실제 소비된 원소만 찍힘 — 단락 여부를 눈으로 확인하는 용도로도 유용
  ```java
  .filter(s -> s.length() > 3)
  .peek(s -> System.out.println("filter 통과: " + s))
  .map(String::toUpperCase)
  ```
- **peek에 비즈니스 로직 넣지 않기**: 실무 코드에서 `peek(a -> question.addAnswer(a))`로 답변→질문 연결을 하면서 `toMap`으로 answerMap을 만드는 걸 봤다 — 동작은 하지만 (1) "Map 만드는 코드"인 줄 알고 읽다 숨은 mutation을 놓치고, (2) 나중에 최종 연산이 단락형으로 바뀌면 연결이 조용히 누락된다. 부수 작업이 목적이면 for문으로 분리 (순회 2번 = O(N)+O(N), 성능 무의미)
- **forEach로 외부 컬렉션 채우지 않기**: `forEach(list::add)`는 병렬 전환 시 race가 나고, 애초에 수집은 `collect`의 일 — "모으기"는 collect, "소비"만 forEach
- **무한 스트림 + limit 누락** = 무한 루프: `Stream.iterate(...)`, `generate(...)`엔 반드시 `limit`
- **parallelStream은 기본 금지에 가깝게**: 공유 가변 상태 mutation 금지, 전역 `ForkJoinPool.commonPool`을 공유하므로 블로킹 I/O 넣으면 앱 전체 병렬 작업이 굶는다. CPU 바운드 + 대량 데이터 + **측정으로 이득 증명**됐을 때만
- **박싱 비용**: 합계·통계는 `Stream<Integer>`의 `reduce`보다 `mapToInt(...).sum()` — 원소마다 박싱/언박싱이 사라짐
- **성능은 for문과 상수 차이**: 스트림이라서 빨라지지 않는다 (오히려 근소하게 느림) — 복잡도 관점은 [big-o-and-input-size](../../algorithm/big-o-and-input-size.md)

## 💡 판단 기준

- **"이 for문의 목적"을 말로 해보면 연산 이름이 된다**: 변환 → `map` / 걸러내기 → `filter` / 중첩 펴기 → `flatMap` / 그룹핑 → `groupingBy` / 하나로 접기 → `reduce` / 있는지만 → `anyMatch`. 목적이 이름과 일치할 때 스트림이 for문보다 읽기 좋다
- **부수 작업(객체 상태 변경, 외부 수집)이 목적이면 스트림 말고 for문** — peek 실무 사례에서 얻은 교훈: 스트림은 "입력 → 결과" 변환을 선언하는 도구지, 변환 도중 딴 일을 하는 도구가 아니다
- 파이프라인이 4~5단을 넘거나 람다가 중첩되면 for문 또는 중간 변수/메서드 추출이 더 읽기 쉽다 — 스트림은 목적이 아니라 수단
- 병렬은 기본 안 쓴다 — 쓰고 싶어지면 먼저 측정

## 참고

- Oracle Javadoc — `java.util.stream` 패키지 요약 (파이프라인·lazy·단락 공식 설명): https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/package-summary.html
- `Stream.peek` Javadoc ("exists mainly to support debugging"): https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Stream.html#peek(java.util.function.Consumer)
- 학습 날짜: 2026-08-03
- 계기: 실무 코드의 `peek(answer -> question.addAnswer(answer))`가 뭐 하는 연산인지에서 시작 → peek의 정체(디버깅용 관찰)·lazy 함정 → Stream 전체 연산 지도 정리로 확장

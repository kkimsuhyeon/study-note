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
| 변환 | `flatMap(f)` | 원소마다 스트림을 만들고 **한 층으로 펴기** — `List<List<T>>` → 원소들, "방마다 질문 리스트" → 전체 질문 (상세 ↓) |
| 변환 | `mapToInt/mapToLong` / `boxed()` | 기본형 스트림으로/에서 전환 (박싱 비용 절약 ↔ 복귀) |
| 선별 | `filter(pred)` | 조건 통과만 |
| 선별 | `distinct()` | 중복 제거 (**equals/hashCode 기준**) — ⚠️ "특정 필드 기준"은 못 한다 (↓ 함정) |
| 선별 | `limit(n)` / `skip(n)` | 앞 n개 / 앞 n개 버림 (페이징 조합) |
| 선별 | `takeWhile` / `dropWhile` (Java 9) | 조건이 깨질 때**까지**/깨진 **뒤부터** — 정렬된 데이터 전제 |
| 정렬 | `sorted()` / `sorted(comparator)` | 전 원소를 버퍼링해서 정렬 (여기서만 lazy가 아님) |
| 관찰 | `peek(consumer)` | 원소를 **그대로 통과**시키며 부수 작업 — 디버깅용 (아래 ⚠️) |

`map` vs `peek`: `map`은 반환값으로 원소를 **교체**, `peek`은 반환값 없이 **그대로 통과**.

## map vs flatMap — 1:1이냐 1:N이냐

`flatMap`의 람다는 **반드시 `Stream`을 반환**해야 한다. 시그니처가 그렇게 강제한다.

```java
<R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper)
//                                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ 반환 타입
```

**왜 하필 Stream인가**: `flatMap`이 하는 일이 "원소마다 여러 개로 펼친 뒤 그것들을 **이어붙이기**"라서다. 이어붙이려면 펼친 결과가 이어붙일 수 있는 형태여야 한다. `map`은 1:1이라 이어붙일 게 없지만, **1:N은 "몇 개가 나올지 모른다"**는 뜻이고 그걸 표현하는 타입이 `Stream`이다.

```
[A, B, C]
  ├─ A → Stream(a1, a2)   ┐
  ├─ B → Stream(b1)       ├─ 이어붙임
  └─ C → Stream(c1, c2)   ┘
[a1, a2, b1, c1, c2]
```

그래서 컬렉션 필드를 펼 때는 `.stream()`을 붙여야 한다.

```java
// 사용자 → 담당 목록(1:N) 을 전부 펼치기
usrEntities.stream()
        .flatMap(usr -> usr.getAssignments().stream())   // List 라서 .stream() 필요
        .map(Assignment::getTarget)
```

⚠️ **`map`으로 쓰면 한 겹이 남아 다음 연산이 막힌다.**
```java
.map(UsrEntity::getAssignments)     // Stream<List<Assignment>>  ← 중첩
.map(Assignment::getTarget)          // ❌ 컴파일 에러. 원소가 List지 Assignment가 아님
```

**컬렉션 평탄화 전용이 아니다** — Stream만 돌려주면 뭐든 된다.
```java
.flatMap(s -> Stream.of(s.split(",")))        // 배열 → 스트림
.flatMap(Optional::stream)                     // Java 9+. 비어 있으면 0개
.flatMap(u -> u.isActive() ? u.getItems().stream() : Stream.empty())  // 빈 스트림 = 그 원소 제거(filter 겸용)
```

> 💡 **같은 이름의 형제들** — 전부 "중첩을 한 겹 벗긴다"는 같은 개념이다. 하나를 이해하면 나머지도 같이 읽힌다.
>
> | | 벗기는 대상 |
> |---|---|
> | `Stream.flatMap` | `Stream<Stream<T>>` → `Stream<T>` |
> | `Optional.flatMap` | `Optional<Optional<T>>` → `Optional<T>` |
> | `CompletableFuture.thenCompose` | `Future<Future<T>>` → `Future<T>` ([도구 가이드 §2](../concurrency/concurrency-tool-guide.md)) |

> 원소당 소수만 방출하는 경우 Java 16+ `mapMulti(BiConsumer<T, Consumer<R>>)`가 대안 — 원소마다 `Stream` 객체를 만들지 않아 더 싸다. 다만 가독성은 `flatMap`이 낫다.

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

### ⚠️ "특정 필드 기준" 중복 제거 — `distinct()`로는 안 된다

`distinct()`는 **`equals`/`hashCode` 기준**이다. 엔티티에 `@EqualsAndHashCode`가 없으면 **참조(주소) 비교**라, "필드 X가 같으면 하나만"이 목적이면 못 쓴다.

```java
// ❌ Entity 에 equals 가 없으면 주소 비교 → 필드 기준이 아님
.distinct()

// ✅ ① Map 의 키로 접고 값만 꺼내기 (가장 일반적)
.collect(toMap(Item::getCode, item -> item, (prev, curr) -> prev))
.values().stream()

// ✅ ② 어차피 그룹핑도 한다면 downstream 을 toSet() 으로 — 중복 제거를 흡수시킨다
.collect(groupingBy(Item::getType,
                    mapping(this::toDto, toSet())))     // DTO 가 @Data 면 equals 있음

// △ ③ TreeSet + Comparator — 한 줄이지만 불필요한 정렬 비용 + "같음 = compare()==0"이라 의도가 흐려짐
.collect(toCollection(() -> new TreeSet<>(comparing(Item::getCode))))

// ❌ ④ Set<K> seen = new HashSet<>(); ... .filter(x -> seen.add(x.getCode()))
//    동작은 하지만 스트림에 외부 상태를 끌어들이는 부수효과 — 병렬에서 깨진다
```

> 💡 **①보다 ②를 먼저 검토한다.** 중복 제거 뒤에 어차피 `groupingBy`를 한다면, 중간 Map을 따로 만들 필요 없이 downstream `toSet()` 하나로 끝난다. 단 **변환 후 타입에 `equals`가 있어야** 한다 — Lombok `@Data`/`record`면 자동으로 있다.

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
- ⚠️ **lazy가 try-catch를 무력화한다**: 스트림을 그대로 **반환**하면 람다는 아직 안 돌았고, 호출한 쪽이 최종 연산을 부를 때 비로소 터진다 — 그땐 이미 `try` 블록 밖이다. 예외를 잡으려면 **`toList()` 등으로 블록 안에서 즉시 평가**해야 한다
  ```java
  // ❌ catch 가 절대 안 걸린다
  try { return futures.stream().map(CompletableFuture::join); }   // Stream 반환 = 미실행
  catch (CompletionException e) { ... }

  // ✅ 블록 안에서 소비
  try { return futures.stream().map(CompletableFuture::join).toList(); }
  catch (CompletionException e) { ... }
  ```
- ⚠️ **lazy가 병렬을 순차로 만든다**: "제출"과 "대기"를 한 체인에 이으면, 원소가 파이프라인을 **끝까지 통과한 뒤 다음 원소로** 가는 수직 실행(위 §큰 그림) 때문에 `CompletableFuture`를 써도 완전히 순차가 된다
  ```java
  // ❌ 제출 → 즉시 대기 → 다음 원소 제출... 순차
  list.stream()
      .map(x -> CompletableFuture.supplyAsync(() -> call(x), executor))
      .map(CompletableFuture::join)
      .toList();

  // ✅ toList() 로 끊어 "전부 제출" 후 "전부 대기"
  List<CompletableFuture<R>> futures = list.stream()
      .map(x -> CompletableFuture.supplyAsync(() -> call(x), executor))
      .toList();                                    // ← 여기서 전부 제출됨
  futures.stream().map(CompletableFuture::join).toList();
  ```
  코드만 보면 둘 다 병렬처럼 읽혀서 놓치기 쉽다. **비동기 제출을 스트림에 담았다면 대기 전에 반드시 한 번 끊는다.** (풀 설정은 [스레드 풀 내부](../concurrency/thread-pool.md))
- **무한 스트림 + limit 누락** = 무한 루프: `Stream.iterate(...)`, `generate(...)`엔 반드시 `limit`
- **parallelStream은 기본 금지에 가깝게**: 공유 가변 상태 mutation 금지, 전역 `ForkJoinPool.commonPool`을 공유하므로 블로킹 I/O 넣으면 앱 전체 병렬 작업이 굶는다. CPU 바운드 + 대량 데이터 + **측정으로 이득 증명**됐을 때만
- **박싱 비용**: 합계·통계는 `Stream<Integer>`의 `reduce`보다 `mapToInt(...).sum()` — 원소마다 박싱/언박싱이 사라짐
- **성능은 for문과 상수 차이**: 스트림이라서 빨라지지 않는다 (오히려 근소하게 느림) — 복잡도 관점은 [big-o-and-input-size](../../algorithm/big-o-and-input-size.md)

## 스트림 디버깅 (IntelliJ)

체인 중간에 문장을 못 넣으니 디버깅 방법도 스트림 전용 도구를 쓴다:

- **람다 중단점**: 람다가 있는 줄의 거터 클릭 → **"All / Line / Lambda" 선택 팝업** → Lambda를 고르면 그 람다가 실행될 때마다(= 원소마다) 멈추고 그 시점 원소를 Variables에서 확인. **Line은 파이프라인 "조립" 시점에 1번** 멈출 뿐 람다 안은 못 본다
  - 팁: **연산마다 줄을 쪼개** 두면 원하는 람다에 정확히 걸 수 있다 (한 줄 체인이면 선택이 헷갈림)
- **Stream Trace**: 스트림 줄에서 멈춘 상태로 디버그 창의 **"Trace Current Stream Chain"** → 연산별 입력→출력 원소 흐름을 시각화 (filter에서 뭐가 탈락했는지, map에서 뭐가 뭘로 변했는지 한 화면). 원소마다 멈출 필요 없이 전체 흐름 확인 — peek println보다 강력
- **조건부 중단점**: 원소가 많을 때 람다 중단점 우클릭 → Condition에 `a.getId().equals("Q-123")` → 그 원소만 멈춤
- **메서드 참조**(`String::toUpperCase`)는 람다 본문이 없어 그 줄에선 못 건다 → 참조된 메서드 **안에** 걸거나, 잠깐 람다로 풀어서
- **람다가 복잡하면 메서드 추출**(`filter(this::isValid)`)이 정답 — 중단점도 평범하게 걸리고 가독성도 좋아짐. 람다 2줄 넘으면 어차피 이쪽
- 빠르게 값만 보고 싶으면 `peek(System.out::println)` — 단, lazy라 소비된 원소만 찍힘 (위 ⚠️)

## 💡 판단 기준

- **"이 for문의 목적"을 말로 해보면 연산 이름이 된다**: 변환 → `map` / 걸러내기 → `filter` / 중첩 펴기 → `flatMap` / 그룹핑 → `groupingBy` / 하나로 접기 → `reduce` / 있는지만 → `anyMatch`. 목적이 이름과 일치할 때 스트림이 for문보다 읽기 좋다
- **부수 작업(객체 상태 변경, 외부 수집)이 목적이면 스트림 말고 for문** — peek 실무 사례에서 얻은 교훈: 스트림은 "입력 → 결과" 변환을 선언하는 도구지, 변환 도중 딴 일을 하는 도구가 아니다
- 파이프라인이 4~5단을 넘거나 람다가 중첩되면 for문 또는 중간 변수/메서드 추출이 더 읽기 쉽다 — 스트림은 목적이 아니라 수단
- **비동기 작업을 스트림으로 제출했다면 대기 전에 반드시 한 번 끊는다** — 8단 체인을 한 줄로 잇는 게 "깔끔한" 게 아니다. lazy 때문에 병렬이 순차가 되고 try-catch가 무력화되는 곳이 정확히 그 이음매다
- **중복 제거는 "그 뒤에 뭘 하는지" 보고 자리를 정한다** — 뒤에 `groupingBy`가 온다면 `toSet()` downstream으로 흡수시키는 게, 중간 Map을 따로 만들었다가 `values()`로 꺼내는 것보다 단계가 하나 적다
- 병렬은 기본 안 쓴다 — 쓰고 싶어지면 먼저 측정

## 참고

- Oracle Javadoc — `java.util.stream` 패키지 요약 (파이프라인·lazy·단락 공식 설명): https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/package-summary.html
- `Stream.peek` Javadoc ("exists mainly to support debugging"): https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Stream.html#peek(java.util.function.Consumer)
- `Stream.flatMap` Javadoc (매핑된 스트림은 내용 반영 후 닫힌다): https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Stream.html#flatMap(java.util.function.Function)
- `Stream.mapMulti` Javadoc (Java 16+, flatMap 대안): https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Stream.html#mapMulti(java.util.function.BiConsumer)
- 관련 노트: [람다 실행 타이밍](./lambda-execution-timing.md) · [스레드 풀 내부](../concurrency/thread-pool.md) · [동시성 도구 가이드](../concurrency/concurrency-tool-guide.md)
- 학습 날짜: 2026-08-03 (2026-08-12 보강)
- 계기: 실무 코드의 `peek(answer -> question.addAnswer(answer))`가 뭐 하는 연산인지에서 시작 → peek의 정체(디버깅용 관찰)·lazy 함정 → Stream 전체 연산 지도 정리로 확장
- 보강 계기(2026-08-12): 외부 API 반복 호출을 일괄+병렬로 바꾸며 — ① `flatMap`이 왜 `Stream`을 반환해야 하는지(1:N을 표현하는 타입) ② 특정 필드 기준 중복 제거는 `distinct()`로 안 되고 `groupingBy`+`toSet()`에 흡수시키는 게 낫다 ③ **lazy 때문에 try-catch가 무력화되고 `CompletableFuture`가 순차로 도는** 두 함정을 겪음. ③은 이 노트가 이미 적어둔 "수직 실행"이 실제로 물린 사례

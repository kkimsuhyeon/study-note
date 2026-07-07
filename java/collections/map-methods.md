# Map 주요 메서드 — getOrDefault / putIfAbsent / computeIfAbsent / merge

> **한 줄 요약**: Java 8부터 Map에 "조회+분기+저장"을 한 방에 하는 default 메서드들이 생겼다 — `get→if(null)→put` 3줄이 보이면 거의 항상 이 중 하나로 바뀐다. 핵심 함정: compute류 람다가 **null을 반환하면 저장 안 됨**, 람다 안에서 **같은 맵을 수정하면 안 됨**.

## 언제 쓰나

- 키가 없을 때 기본값/초기값을 넣고 싶다 (캐시, 그룹핑, 카운팅)
- `if (map.containsKey(k))` / `if (map.get(k) == null)` 분기가 반복될 때

## 사용 예시 (문법)

### 기본 동작 — put / get / remove (여기가 토대)
```java
map.put(key, value);   // 키 있으면 "덮어쓰기"(교체), 없으면 추가. 리턴 = "이전 값"(없었으면 null)
map.get(key);          // 값 or null. "없어서 null"인지 "null을 저장"인지 구별 안 됨 → containsKey
map.remove(key);       // 1인자: 키만 맞으면 "무조건" 제거. 리턴 = 지워진 값(없었으면 null)
```
- **`put`은 같은 키면 조용히 덮어쓴다** — "이미 있으면 에러" 같은 건 없다. 이전 값이 필요하면 리턴을 받는다.
- **`remove`는 2개다** ← 여기서 자주 헷갈린다:

| | 지우는 조건 | 리턴 |
|---|---|---|
| `remove(key)` | 키만 맞으면 **무조건** | 지워진 값 (V) |
| `remove(key, value)` | 키 + **값까지 일치**할 때만 | 지웠는지 여부 (boolean) |

```java
map.remove("A");             // A를 그냥 지움
map.remove("A", oldValue);   // A가 "지금 oldValue일 때만" 지움. 다른 값이면 그대로 둠 → false 리턴
```
→ 2인자 remove는 "내가 아는 그 값일 때만 지워"라는 **조건부 제거**. 왜 이런 게 필요한지는 아래 **'조건부 메서드(CAS)는 왜 있나'** 절 참고.

### 조회 계열
```java
map.getOrDefault(key, 0L);          // 없으면 기본값 반환 (맵에 저장은 안 함!)
map.containsKey(key);               // 존재 여부 — get()==null 판별과 다름 (null 값 허용 맵에서)
```

### 저장 계열
```java
map.putIfAbsent(key, value);        // 없을 때만 put. value를 "미리 만들어서" 전달 (즉시 평가)
map.computeIfAbsent(key, k -> expensiveLoad(k));   // 없을 때만 람다 실행해서 put (지연 평가)
map.computeIfPresent(key, (k, old) -> old + 1);    // 있을 때만 갱신. 람다가 null 반환 → 엔트리 제거
map.compute(key, (k, old) -> old == null ? 1 : old + 1);  // 있든 없든 실행
map.merge(key, 1, Integer::sum);    // 없으면 1 저장, 있으면 sum(old, 1) — 카운팅/누적의 정석
map.remove(key, value);             // 키+값이 둘 다 일치할 때만 제거 (조건부 제거)
map.replace(key, value);            // 키가 있을 때만 교체 (put과 달리 새 키를 만들지 않음)
```

### 순회 계열
```java
map.forEach((k, v) -> ...);
map.replaceAll((k, v) -> transform(v));   // 모든 값 일괄 변환
for (Map.Entry<K, V> e : map.entrySet()) { ... }   // 키+값 둘 다 필요하면 entrySet (keySet+get 반복보다 낫다)
```

## 실전 패턴

### ① 캐시 패턴 — computeIfAbsent
루프 안에서 같은 키의 비싼 로드(DB 조회 등)를 1회로 줄인다.
```java
// 한 달(31일) 루프에서 근로제(wkSysId)가 2종이면 buildPolicy(DB 조회)는 딱 2번만 실행
Map<String, WorkSystemPolicy> policyByWkSys = new HashMap<>();
for (LocalDate date : allDates) {
    WorkSystemPolicy policy = policyByWkSys.computeIfAbsent(
            wkSysId, id -> workSystemPolicyLoader.buildPolicy(orgId, id, maxWorkDate));
    ...
}
```
아래 3줄과 동일하다:
```java
WorkSystemPolicy policy = map.get(wkSysId);
if (policy == null) {
    policy = load(wkSysId);
    map.put(wkSysId, policy);
}
```

### ② 그룹핑 초기화 — computeIfAbsent + List
```java
map.computeIfAbsent(date, k -> new ArrayList<>()).add(segment);
// (stream이면 Collectors.groupingBy가 같은 역할)
```

### ③ 카운팅/합산 — merge
```java
counts.merge(word, 1, Integer::sum);               // 단어 세기
minutes.merge(date, dayMinutes, Long::sum);         // 일자별 분 합산
```

## putIfAbsent vs computeIfAbsent

| | putIfAbsent | computeIfAbsent |
|---|---|---|
| 값 생성 시점 | **호출 전에 이미 생성** (즉시 평가) | 키가 없을 때만 람다 실행 (지연 평가) |
| 반환값 | **기존 값** (없었으면 null) | **최종 값** (기존 값 or 새로 만든 값) |
| 값이 비싼 경우 | 낭비 (안 쓸 수도 있는데 만듦) | 적합 |

→ `new ArrayList<>()`처럼 싸면 아무거나, DB 조회처럼 비싸면 computeIfAbsent. 반환값을 이어서 쓸 거면 computeIfAbsent가 편하다 (putIfAbsent는 "없었으면 null"을 돌려줘서 한 번 더 분기 필요).

## 조건부 메서드(CAS)는 왜 있나 — remove(k,v) / replace(k, old, new)

`putIfAbsent`, `remove(key, value)`, `replace(key, old, new)`, `replace(key, value)`는 전부 **"현재 상태가 내 예상과 같을 때만 바꾼다"**는 한 부류다 — **CAS(compare-and-swap)** 형제들.

| 메서드 | 바꾸는 조건 |
|---|---|
| `putIfAbsent(k, v)` | 키가 **없을 때만** 넣음 |
| `remove(k, v)` | 값이 **v와 일치할 때만** 지움 |
| `replace(k, old, new)` | 값이 **old와 일치할 때만** new로 교체 |
| `replace(k, v)` | 키가 **있을 때만** 교체 (없으면 안 넣음 — put과 반대) |

**왜 그냥 `get→확인→put/remove` 3줄로 안 하고 이걸 쓰나?** — 그 사이에 **다른 스레드(또는 비동기 콜백)가 값을 바꿔치기**할 수 있어서. `ConcurrentHashMap`에서 이 메서드들은 "확인+변경"을 **원자적으로 한 번에** 처리해 중간 끼어듦을 막는다.

구체 케이스 (SSE emitter 교체): map에 `clientId → emitter`를 두고, 재구독 시 헌 emitter를 새 것으로 바꾸는 코드 —
```java
// 헌 emitter가 끝나면 자신을 map에서 지우는 콜백 (비동기 — 언제 돌지 보장 없음)
emitter.onCompletion(() -> map.remove(clientId, emitter));   // ★ 2인자
...
map.put(clientId, newEmitter);   // 새 걸로 교체
```
헌 emitter의 정리 콜백이 **put 이후에 뒤늦게 돌아도**, `remove(clientId, oldEmitter)`는 "지금 값이 oldEmitter일 때만" 지우므로 → 값이 이미 `newEmitter`면 **안 지움** → 방금 넣은 새 것이 보호된다. 만약 `remove(clientId)`(1인자)였다면 **뒤늦은 콜백이 새 emitter를 날리는 버그**가 됐을 것.
→ 판단: **"읽고-판단한 값이, 쓸 때도 그대로인가"를 보장해야 하면 조건부(CAS) 메서드.** (값 비교는 `equals` — 위 emitter처럼 equals 미정의면 `==` 객체 동일성 비교라 old/new가 정확히 갈린다.) 자세한 SSE 맥락은 → [SseEmitter 노트](../spring/sse-emitter.md).

## Collectors.toMap — 중복 키·null 함정

스트림을 Map으로 모을 때 쓰는 `Collectors.toMap`은 인자 개수에 따라 동작이 다르다:

```java
// 2인자 — 키 중복 시 IllegalStateException 던지고 죽는다!
stream.collect(Collectors.toMap(Record::getDate, Function.identity()));

// 3인자 — merge 함수가 충돌 시 승자를 결정
stream.collect(Collectors.toMap(Record::getDate, Function.identity(), (a, b) -> a));        // 먼저 것 승리
stream.collect(Collectors.toMap(Emp::getEmpNo, Function.identity(), (before, after) -> after)); // 나중 것 승리

// 4인자 — 맵 구현체까지 지정 (순서 유지 등)
stream.collect(Collectors.toMap(k, v, merge, LinkedHashMap::new));
```

- **2인자 버전은 "중복 키가 없다"는 단언이다.** 데이터가 유니크 보장(UNIQUE 제약 등)이면 2인자가 오히려 명확하고(중복=버그를 즉시 드러냄), 보장이 없으면 3인자로 방어해야 한다.
- **merge 함수의 "나중 것"은 스트림 순서에 달려 있다** — DB 조회 결과라면 ORDER BY가 없을 때 어느 row가 이길지 비결정적. "말일에 가까운 매핑 승리" 같은 규칙이 필요하면 정렬로 보장하고 나서 `(before, after) -> after`를 써야 한다.
- **value가 null이면 NPE** — toMap은 값 null을 허용하지 않는다 (HashMap 자체는 허용해도, toMap 구현이 merge 로직에서 null을 못 다룸). null 값 가능성이 있으면 `groupingBy`나 수동 for-put으로.

## Map.of / Map.ofEntries — 팩토리는 중복·null을 "에러"로 막는다

불변 맵을 리터럴처럼 만드는 팩토리:
```java
Map.of("a", 1, "b", 2);                  // 불변 맵 (최대 10쌍)
Map.ofEntries(Map.entry("a", 1), ...);   // 쌍이 많을 때
```
- **중복 키 → `IllegalArgumentException`** (생성 시점에 죽음): `Map.of("a", 1, "a", 2)` ❌. 일반 `put`이 조용히 덮어쓰는 것과 정반대 — 팩토리는 "중복 = 실수"로 보고 즉시 막는다.
- **null 키·값 → `NullPointerException`** (전부 금지).
- 반환은 **불변(immutable)** — `put`/`remove` 하면 `UnsupportedOperationException`. 수정할 거면 `new HashMap<>(Map.of(...))`로 감싼다.

→ **"중복 키"를 대하는 태도가 세 갈래**다: **`put` = 덮어씀(정상)** / **`Collectors.toMap` 2인자·`Map.of` = 예외 던짐("중복은 버그"라고 단언)** / **`toMap` 3인자 = merge 함수로 승자 결정**. 내 데이터가 유니크 보장이면 "예외 던짐" 쪽이 오히려 버그를 즉시 드러내서 낫다.

## ⚠️ 함정

- **compute류 람다가 null을 반환하면 맵에 저장되지 않는다** (computeIfPresent/compute/merge는 기존 엔트리 **제거**). computeIfAbsent에서 로드 실패로 null이 나오면 → 캐싱이 안 돼서 **다음 루프에서 또 실행**된다 (실패 캐싱 안 됨). 실패도 캐싱하려면 Optional이나 sentinel 값을 넣거나 try-catch로 따로 관리.
- **람다 안에서 같은 맵을 수정하면 안 된다** — Java 9+ HashMap은 `ConcurrentModificationException`을 던진다 (Java 8은 무한루프/데이터 깨짐 등 미정의 동작). 재귀 캐시(피보나치를 computeIfAbsent 재귀로) 같은 코드가 전형적인 지뢰.
- `getOrDefault`는 **저장하지 않는다** — 기본값을 맵에 남기고 싶으면 computeIfAbsent.
- **HashMap은 null 키 1개/null 값 허용, ConcurrentHashMap은 null 키·값 전부 금지.** null 값을 넣는 순간 `get()==null`이 "없음"인지 "null 저장"인지 구별 불가가 되므로, 애초에 null 값을 안 넣는 게 정신 건강에 좋다.
- ConcurrentHashMap의 computeIfAbsent는 **원자적**이지만, 실행 중 해당 빈(bin)이 잠기므로 람다에 오래 걸리는 작업(원격 호출)을 넣으면 병목이 된다.

## 💡 판단 기준

- **`get → if(null) → put` 3줄이 보이면 compute류 1줄로.** 근태 마감 rewrite에서 근로제 정책 캐시(`policyByWkSys`)를 이 패턴으로 정리 — "없으면 만들어 넣고 있으면 꺼내라"는 의도가 이름 그대로 드러난다.
- 값 생성이 **비싸면 computeIfAbsent**(지연), 싸면 아무거나. **누적/카운팅은 merge**, **있을 때만 갱신은 computeIfPresent**.
- 단, 람다에서 null이 나올 수 있는 로드는 "실패가 반복 실행된다"를 알고 쓰기 — 실패까지 캐싱해야 하면 수동 get/put + try-catch가 오히려 명확하다.

## 참고

- [Oracle Javadoc — Map (Java 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Map.html)
- [Javadoc computeIfAbsent — "recursive update 금지" 명시](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Map.html#computeIfAbsent(K,java.util.function.Function))
- 관련 노트: [컬렉션 계층 구조](./collection-hierarchy.md), [리스트 생성](./list-creation.md)

---
학습 날짜: 2026-07-03
계기: 근태 마감 rewrite 중 `policyByWkSys.computeIfAbsent(...)` 캐시 패턴을 보고 "이건 어떤 함수?"에서 출발
보강: 2026-07-08 — SSE emitter 교체 코드의 `remove(key, value)` 조건부 제거에서 막혀, 토대(기본 put/get/remove·remove 2종 대비)와 조건부(CAS) 형제·`Map.of` 중복 예외를 추가. (관련: [SseEmitter 노트](../spring/sse-emitter.md))

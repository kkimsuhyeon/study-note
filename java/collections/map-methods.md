# Map 주요 메서드 — getOrDefault / putIfAbsent / computeIfAbsent / merge

> **한 줄 요약**: Java 8부터 Map에 "조회+분기+저장"을 한 방에 하는 default 메서드들이 생겼다 — `get→if(null)→put` 3줄이 보이면 거의 항상 이 중 하나로 바뀐다. 핵심 함정: compute류 람다가 **null을 반환하면 저장 안 됨**, 람다 안에서 **같은 맵을 수정하면 안 됨**.

## 언제 쓰나

- 키가 없을 때 기본값/초기값을 넣고 싶다 (캐시, 그룹핑, 카운팅)
- `if (map.containsKey(k))` / `if (map.get(k) == null)` 분기가 반복될 때

## 사용 예시 (문법)

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

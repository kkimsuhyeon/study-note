# 동시성 컬렉션 — ConcurrentHashMap · CopyOnWriteArrayList · BlockingQueue

> **한 줄 요약**: 일반 컬렉션(`HashMap`·`ArrayList`)은 멀티스레드에서 **깨진다**(유실·이상 동작) — 스레드끼리 공유하는 컬렉션은 `java.util.concurrent`의 동시성 컬렉션을 쓴다. Map은 `ConcurrentHashMap`, 읽기 위주 List는 `CopyOnWriteArrayList`, 스레드 간 작업 전달은 `BlockingQueue`. ⚠️ 단 **개별 연산이 안전할 뿐, 연산 조합(check-then-act)은 여기서도 깨진다** — 그래서 `putIfAbsent`/`computeIfAbsent` 같은 원자적 복합 메서드가 따로 있다.

관련 노트: [JVM 동시성 도구](./jvm-concurrency-tools.md)(check-then-act·CAS) · [스레드 기초](./threads.md) · [스레드 풀 내부](./thread-pool.md)(BlockingQueue가 그 큐) · [락 개념](./locks.md)

---

## 1. 왜 필요한가 — HashMap은 멀티스레드에서 깨진다

`HashMap`·`ArrayList`는 **단일 스레드 전제**로 설계됐다. 여러 스레드가 동시에 `put()`하면:

- **갱신 유실**: 두 스레드가 같은 버킷에 동시에 넣으면 한쪽이 사라질 수 있다 — `count++`가 깨지던 것과 같은 read-modify-write 문제([jvm-tools §7-2](./jvm-concurrency-tools.md)).
- **내부 구조 파손**: resize(버킷 확장) 도중 다른 스레드가 끼어들면 링크가 꼬인다. Java 7에서는 이걸로 **무한 루프(CPU 100%)**가 나는 사례가 유명했고, Java 8에서 resize 방식이 바뀌어 그 고전 사례는 완화됐지만 **여전히 thread-safe하지 않다**(유실·깨진 상태는 그대로 가능).
- `size()`가 틀리거나, 넣은 값이 `get()`으로 안 보이는(가시성) 등 증상이 다양하고 **재현이 어렵다** — 동시성 버그답게.

> 📌 기준은 하나: **"이 컬렉션을 여러 스레드가 같이 만지는가?"** — 지역변수·단일 스레드면 일반 컬렉션, 필드로 공유되면 동시성 컬렉션. ([threads.md §1](./threads.md) — 힙의 필드가 공유되는 것)

### Collections.synchronizedMap은? (구식)
모든 메서드를 `synchronized`로 감싼 래퍼. 동작은 하지만 — ① 맵 전체가 자물쇠 하나라 **경합 시 성능 급락** ② 순회할 때 수동 동기화 필요 ③ check-then-act 조합은 **여전히 깨짐**. → `ConcurrentHashMap`이 모든 면에서 낫다. 레거시에서 보이면 교체 후보.

---

## 2. ConcurrentHashMap — 공유 Map의 기본값

```java
private final Map<Long, ReentrantLock> userLocks = new ConcurrentHashMap<>();

// ✅ 원자적 복합 연산 — "없으면 만들어 넣고, 있으면 그걸 반환"을 한 방에
ReentrantLock lock = userLocks.computeIfAbsent(userId, k -> new ReentrantLock());
```
([locks.md](./locks.md)의 userId별 락 예제가 정확히 이 패턴)

### 동작 원리 (Java 8+)
- 버킷(bin) 단위로 **CAS + 부분 synchronized** — 맵 전체를 잠그지 않아서 서로 다른 키를 만지는 스레드끼리는 거의 안 부딪친다. (읽기 `get()`은 락 없음)
- 순회(iterator)는 **weakly consistent** — 순회 중 남이 수정해도 `ConcurrentModificationException`이 안 나고, 수정 전/후 상태가 섞여 보일 수 있다(스냅샷 아님).

### ⚠️ 개별 연산만 원자적 — 조합은 깨진다 (check-then-act 재등장)
```java
// ❌ containsKey와 put 사이에 남이 끼어든다 — ConcurrentHashMap이어도!
if (!map.containsKey(key)) {
    map.put(key, createValue(key));   // 두 스레드가 둘 다 if 통과 → 중복 생성/덮어쓰기
}

// ✅ 원자적 복합 메서드로
map.putIfAbsent(key, value);                  // 없을 때만 넣기
map.computeIfAbsent(key, k -> create(k));     // 없으면 생성해서 넣기 (생성 비쌀 때)
map.compute(key, (k, v) -> v == null ? 1 : v + 1);  // 읽고-계산-쓰기를 원자적으로
map.merge(key, 1, Integer::sum);              // 카운팅 관용구 (있으면 합치기)
```

| 원자적 복합 메서드 | 뜻 |
|---|---|
| `putIfAbsent(k, v)` | 없을 때만 넣기 (이미 있으면 기존 값 반환) |
| `computeIfAbsent(k, fn)` | 없으면 fn으로 **생성**해서 넣기 — 값 생성이 비쌀 때 putIfAbsent보다 낫다(있으면 fn 실행 안 함) |
| `compute(k, fn)` / `computeIfPresent` | 현재 값 기반 갱신을 원자적으로 |
| `merge(k, v, fn)` | 없으면 v, 있으면 fn(기존, v) — 카운터/누적의 정석 |

### ⚠️ 그 외 함정
- **null 키/값 불허** (`HashMap`은 허용) — 동시성 환경에선 `get()==null`이 "키 없음"인지 "null 저장"인지 구분 불가라 설계에서 막았다.
- `computeIfAbsent`의 **fn 안에서 같은 맵을 또 수정하지 말 것** — 같은 버킷을 다시 잡으려다 교착/`IllegalStateException`.
- `size()`는 순간 스냅샷이 아니라 근사치에 가깝다 — 동시 수정 중엔 참고값으로만.

---

## 3. CopyOnWriteArrayList — 읽기 압도적, 쓰기 드묾

쓰기(`add`/`remove`) 때마다 **내부 배열을 통째로 복사**한다. 대신 읽기는 락도 복사도 없이 그냥 읽는다.

```java
private final List<EventListener> listeners = new CopyOnWriteArrayList<>();
// 등록/해제(쓰기)는 가끔, 이벤트 발행 때 순회(읽기)는 수만 번 — 딱 이 용도
```

- 순회는 **스냅샷** — 순회 시작 시점의 배열을 보므로, 도중에 남이 수정해도 안전(`ConcurrentModificationException` 없음). 대신 최신 변경이 안 보일 수 있다.
- ⚠️ **쓰기가 잦으면 최악** — 원소 1만 개 리스트에 add 한 번 = 1만 개 복사. 쓰기 많은 곳엔 절대 금지.
- 용도: 리스너/구독자 목록, 거의 안 바뀌는 설정 목록. 그 외 공유 List가 필요하면 보통 설계를 다시 본다(정말 List여야 하나?).

---

## 4. BlockingQueue — 스레드 간 작업 전달 (producer-consumer)

"넣는 쪽"과 "빼는 쪽" 스레드를 큐로 연결한다. 핵심은 **꽉 차면 put이, 비면 take가 알아서 대기(블로킹)** — 대기·타이밍 조율을 큐가 대신 해준다.

```java
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(100);  // 경계 있는 큐

// 생산자 스레드
queue.put(task);        // 꽉 차면 자리 날 때까지 대기

// 소비자 스레드
Task t = queue.take();  // 비어 있으면 들어올 때까지 대기
```

| 구현 | 특징 |
|---|---|
| `ArrayBlockingQueue` | **경계 있음**(생성 시 크기 고정). 배압(backpressure) 역할 |
| `LinkedBlockingQueue` | 기본 **무한** — [스레드 풀](./thread-pool.md)의 "큐가 무한이라 OOM" 그 큐 |
| `SynchronousQueue` | 크기 0 — 넣는 쪽과 빼는 쪽이 직접 만나야 전달 (`CachedThreadPool`이 사용) |

- `put`/`take`(무한 대기) 외에 `offer(v, timeout)`/`poll(timeout)`(시간제한) 버전 있음.
- **스레드 풀의 작업 큐가 바로 이것** — `ExecutorService`에 submit한 작업이 `BlockingQueue`에 쌓이고, 일꾼 스레드들이 `take()`로 꺼내 간다. 풀 = 일꾼들 + BlockingQueue 조합.

---

## 5. 💡 판단 기준

| 상황 | 선택 |
|---|---|
| 여러 스레드가 공유하는 Map | **`ConcurrentHashMap`** (공유 Map의 기본값) |
| "없으면 넣기/만들기" 등 확인+행동 | `putIfAbsent` / `computeIfAbsent` — **if+put 조합 금지** |
| 카운팅·누적 Map | `merge(k, 1, Integer::sum)` |
| 읽기 압도적·쓰기 드문 List (리스너 목록) | `CopyOnWriteArrayList` |
| 스레드 간 작업 전달, 속도 차 조절 | `BlockingQueue` (경계 있는 `ArrayBlockingQueue` 우선) |
| 컬렉션이 지역변수/단일 스레드 | 일반 `HashMap`/`ArrayList` — 동시성 컬렉션은 공짜가 아니다 |

> 한 줄: **공유되면 동시성 컬렉션, 공유 Map은 `ConcurrentHashMap`이 기본값.** 단 동시성 컬렉션도 **개별 연산만** 지켜준다 — "확인하고 행동"은 [check-then-act](./jvm-concurrency-tools.md)라 여기서도 깨지니, 반드시 전용 원자 메서드(`putIfAbsent`/`computeIfAbsent`/`merge`)로. (재고 -30 사고와 같은 구조가 Map에서도 반복된다)

---

## 6. 참고
- [Oracle - ConcurrentHashMap Javadoc](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
- [Oracle - BlockingQueue Javadoc](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/BlockingQueue.html)
- [Baeldung - Guide to ConcurrentHashMap](https://www.baeldung.com/java-concurrent-map)
- [Baeldung - CopyOnWriteArrayList](https://www.baeldung.com/java-copy-on-write-arraylist)

---

**학습 날짜**: 2026-08-12
**계기**: 동시성 7개 노트 1회독 완주 후 공백 점검 — locks.md 예제에서 `computeIfAbsent`를 이미 쓰고 있는데 동시성 컬렉션 정리가 없었다. check-then-act(재고 -30)가 Map에서도 같은 구조로 반복된다는 연결이 핵심.

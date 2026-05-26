# JVM 동시성 도구 — 락 & 동기화 장치 종합

> **한 줄 요약**: 같은 JVM(프로세스) 안의 스레드들을 제어하는 도구 모음. `synchronized`·`ReentrantLock` 등 **락**과, `Semaphore`·`CountDownLatch` 등 **동기화 장치(synchronizer)**로 나뉜다. (멀티 서버에선 무력 — 그땐 DB 락/분산락. [락 개념 종합](./locks.md) 참고)

관련 노트: [락 개념 종합](./locks.md) · [데드락](./deadlock.md)

대부분 `java.util.concurrent`(+ `.locks`) 패키지. `synchronized`만 언어 키워드.

---

## 0. 한눈에 분류

```
락(Lock) — "한 번에 하나/제한된 수만 자원 접근"
 ├─ synchronized        : 언어 키워드. 모니터 락. 자동 해제
 ├─ ReentrantLock       : 명시적 락. tryLock/타임아웃/공정성/Condition
 ├─ ReadWriteLock       : 읽기 공유 + 쓰기 배타
 └─ StampedLock         : ReadWriteLock 개선 + 낙관적 읽기

동기화 장치(Synchronizer) — "스레드들의 진행 시점을 맞춤"
 ├─ Semaphore           : 동시 접근 수(permit)를 N개로 제한
 ├─ CountDownLatch      : 카운트가 0 될 때까지 대기 (일회성)
 └─ CyclicBarrier       : N개가 다 모이면 함께 통과 (재사용)
```

---

## 0-1. 애초에 락은 왜 / 언제 쓰나

한 줄: **여러 스레드가 공유하는 "가변 데이터"를 동시에 고칠 때, 안 깨지게 막으려고.**

락이 없으면 **경쟁 상태(race condition)** → 갱신 분실(lost update):
```
잔액 1000, 두 스레드가 동시에 100 차감
A: read 1000      B: read 1000
A: write 900      B: write 900   ← 둘 다 깎았는데 900 (100 증발!)

락으로 read+계산+write를 한 덩어리로 묶으면:
A:[read→write]900   B: 대기 → [read 900→write 800] ✅
```
> 이건 [jpa/lock.md](../jpa/lock.md)의 갱신 분실과 **같은 문제**다. 거긴 DB 락, 여긴 JVM 락으로 막을 뿐(범위 축만 다름).

**기본 동작**: 한 번에 한 스레드만 통과시키고 나머지는 대기(상호 배제). 단 `ReadWriteLock`(읽기 여러 명)·`Semaphore`(N명)는 예외.

**사용처**: 공유 카운터/잔액/재고 변경, 비스레드세이프 컬렉션(`HashMap`) 보호, 캐시 갱신, 싱글톤 초기화 등 — **단일 서버** 한정.

**⚠️ 한계**: 단일 JVM에서만 유효. 서버 2대 이상이면 무력 → DB 락/분산락([locks.md](./locks.md)).

---

## 1. synchronized — 가장 기본 (모니터 락)

언어 키워드. 모든 자바 객체가 가진 **모니터(intrinsic lock)**를 사용한다.

```java
public synchronized void m() { ... }      // this 객체 락
public void m() {
    synchronized (lockObj) { ... }         // 특정 객체 락 (블록)
}
public static synchronized void s() { ... } // 클래스 객체(Class) 락
```

- **자동 획득/해제**: 블록을 벗어나면(정상 종료든 예외든) 자동 unlock → unlock 깜빡할 위험 없음.
- **재진입(reentrant)**: 같은 스레드가 이미 가진 락을 다시 획득 가능.
- **한계**: 비공정(순서 보장 X), **타임아웃·인터럽트·tryLock 불가**. 락을 못 잡으면 무한 대기.

> 가장 단순하고 안전(자동 해제)하지만, "기다리다 포기" 같은 세밀한 제어가 안 된다.

> ⚠️ **용어 함정: "동기화(synchronization)" ≠ "동기/비동기(sync/async)"**
> `synchronized`의 "동기화"는 **여러 스레드의 공유 자원 접근을 조율(상호 배제)**하는 것 — 락이 하는 일이다. 반면 **동기/비동기**는 "호출하고 결과를 기다리나(동기) / 안 기다리나(비동기)"라는 **제어 흐름**의 개념. 단어만 겹칠 뿐 다른 축이다.
> - 동기화 = "화장실을 한 명씩 쓰게 잠금" (누가 자원을 만지나)
> - 동기/비동기 = "그 앞에서 기다리나 / 번호표 받고 딴 일 하나" (결과를 기다리나)
>
> 마찬가지로 [locks.md](./locks.md)의 "JVM **내부** 락"에서 "내부"는 **락의 유효 범위(JVM 안)**를 뜻하지 동기/비동기와 무관하다.

---

## 2. ReentrantLock — 명시적 락 (synchronized의 강화판)

`java.util.concurrent.locks`. `lock()`/`unlock()`을 직접 호출 → **`try/finally` 필수**.

```java
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try { /* 임계 영역 */ }
finally { lock.unlock(); }   // ★ 반드시 finally에서 해제
```

synchronized가 못 하는 것들을 지원:

| 기능 | 설명 |
|------|------|
| `tryLock()` | 락을 **즉시 시도**, 못 잡으면 false (대기 안 함) |
| `tryLock(t, unit)` | **타임아웃** 동안만 대기 → 데드락/무한대기 회피 |
| `lockInterruptibly()` | 대기 중 **인터럽트** 가능 |
| 공정성 | `new ReentrantLock(true)` → 대기 순서(FIFO) 보장 (처리량은↓) |
| `Condition` | `newCondition()`으로 **여러 조건 큐** 분리 (wait/notify 세분화) |

```java
if (lock.tryLock(3, TimeUnit.SECONDS)) {
    try { ... } finally { lock.unlock(); }
} else {
    // 락 획득 실패 처리 (경합/데드락 회피)
}
```

> synchronized보다 유연하지만 **unlock 책임이 개발자**에게 있다(finally 누락 시 영구 락). 단순한 경우엔 synchronized가 더 안전.

---

## 3. ReadWriteLock — 읽기 공유 / 쓰기 배타

구현체: `ReentrantReadWriteLock`. 락을 **읽기 락 / 쓰기 락** 두 개로 분리.

```java
private final ReadWriteLock rw = new ReentrantReadWriteLock();
rw.readLock().lock();   // 여러 스레드 동시 보유 가능 (공유)
rw.writeLock().lock();  // 하나만 보유 (배타, 읽기·쓰기 모두 차단)
```

- **읽기끼리는 공존**, 쓰기는 독점 → **읽기 많고 쓰기 적은** 데이터에 처리량↑ (캐시, 설정값 등).
- 락 **다운그레이드**(쓰기락 보유 중 읽기락 획득 후 쓰기락 해제) 가능.
- 락 **업그레이드**(읽기락 → 쓰기락)는 **불가**(데드락 유발) — 둘 다 읽기락 쥐고 쓰기락 올리려 하면 [공유락 업그레이드 데드락](./deadlock.md). 이 한계를 푼 게 StampedLock.

---

## 4. StampedLock — ReadWriteLock 개선 + 낙관적 읽기 (Java 8+)

세 가지 모드. 핵심은 **낙관적 읽기(optimistic read)**.

```java
private final StampedLock sl = new StampedLock();

long stamp = sl.tryOptimisticRead();   // 락을 안 걸고 stamp만 받음
int x = this.value;                    // 일단 읽음
if (!sl.validate(stamp)) {             // 읽는 동안 쓰기가 있었나 검증
    stamp = sl.readLock();             // 충돌 → 정식 읽기락으로 재시도
    try { x = this.value; }
    finally { sl.unlockRead(stamp); }
}
```

| 모드 | 설명 |
|------|------|
| 쓰기락 `writeLock()` | 배타 |
| 읽기락 `readLock()` | 공유 (ReadWriteLock과 유사) |
| **낙관적 읽기** `tryOptimisticRead()` | **락 없이** 읽고 `validate()`로 사후 검증 → 충돌 드물면 매우 빠름 |

- 읽기가 압도적으로 많을 때 ReadWriteLock보다 빠름.
- **단점**: 재진입 X, `Condition` X, 사용법 까다로움(검증·재시도 직접). 잘못 쓰면 버그.
- 낙관적 읽기 = JVM 레벨의 "낙관적 락" 느낌 (DB `@Version`과 발상이 같음).

---

## 5. Semaphore — 동시 접근 수 제한

permit(허가증) **N개**를 두고, `acquire()`로 하나 얻고 `release()`로 반납.

```java
private final Semaphore sem = new Semaphore(3);  // 동시 3개까지
sem.acquire();           // permit 없으면 대기
try { /* 자원 사용 */ }
finally { sem.release(); }
```

- **자원 풀 제한**에 사용: 동시 DB 커넥션 N개, 동시 다운로드 N개, API 호출 동시성 제한 등.
- `N=1`이면 뮤텍스처럼 동작 — 단 **소유자 개념이 없어** 다른 스레드가 release 가능(뮤텍스와 다름).
- 공정성 옵션(`new Semaphore(n, true)`).

> 락은 "1명만", 세마포어는 "N명까지" → 세마포어는 뮤텍스의 일반화.

---

## 6. CountDownLatch — 카운트가 0 될 때까지 대기 (일회성)

카운트 N으로 시작 → `countDown()`으로 감소 → `await()`는 **0이 될 때까지 블록**.

```java
CountDownLatch latch = new CountDownLatch(3);

// 작업 스레드들
runAsync(() -> { work(); latch.countDown(); });  // ×3

latch.await();   // 3개가 다 countDown 할 때까지 대기
System.out.println("모든 작업 완료");
```

- **일회성**: 0이 되면 끝, 재사용 불가.
- 용례: **여러 작업이 다 끝나길 대기**, 또는 `N=1`로 "시작 신호" 동시 출발.

---

## 7. CyclicBarrier — N개가 다 모이면 함께 통과 (재사용)

N개 스레드가 각자 `await()`에 도달 → **마지막 하나가 도착하면 동시에 모두 통과**. 재사용 가능(cyclic).

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> merge());  // 다 모이면 merge 실행

// 각 스레드
computePart();
barrier.await();   // 3개가 다 도달할 때까지 대기 → 모두 통과
nextPhase();
```

- **재사용**: 통과 후 다시 N 모으면 또 동작 → 단계별(phase) 병렬 계산에 적합.
- `barrierAction`: 다 모인 순간 실행할 작업 지정 가능.

### CountDownLatch vs CyclicBarrier (자주 헷갈림)

| | CountDownLatch | CyclicBarrier |
|---|---|---|
| 방향 | 한쪽이 `countDown`, 다른 쪽이 `await` (역할 분리) | 참여 스레드들이 **서로** 기다림 |
| 재사용 | **불가** (일회성) | **가능** (cyclic) |
| 트리거 | 카운트 0 도달 | 참여자 전원 도착 |
| 용례 | "다 끝났나" 완료 대기 / 시작 신호 | "다 모였나" 단계 동기화 |

---

## 8. 선택 가이드

```
단순 상호배제, 자동 해제 원함        → synchronized
타임아웃/tryLock/인터럽트/공정성 필요 → ReentrantLock
읽기 많고 쓰기 적음                   → ReadWriteLock (더 빠르게: StampedLock 낙관적 읽기)
동시 접근 "수"를 제한 (자원 풀)        → Semaphore
여러 작업 완료를 한 번 대기            → CountDownLatch
여러 스레드를 단계마다 모음(반복)      → CyclicBarrier
```

| 도구 | 한 줄 |
|------|-------|
| synchronized | 기본 모니터 락, 자동 해제, 제어 옵션 없음 |
| ReentrantLock | 명시적 락, tryLock·타임아웃·공정성·Condition |
| ReadWriteLock | 읽기 공유 / 쓰기 배타 |
| StampedLock | RWLock + 락 없는 낙관적 읽기 (빠름, 까다로움) |
| Semaphore | permit N개로 동시 접근 제한 |
| CountDownLatch | 0까지 대기, 일회성 |
| CyclicBarrier | 전원 도착 시 통과, 재사용 |

---

## 9. 주의사항

- **이건 전부 단일 JVM 한정.** 서버 2대 이상이면 무력 → DB 락/분산락 필요([locks.md](./locks.md)).
- `ReentrantLock`/`ReadWriteLock`/`StampedLock`/`Semaphore`는 **finally에서 해제**(StampedLock은 stamp로). 누락 시 영구 락.
- 여러 락을 잡을 땐 **획득 순서 통일** → [데드락](./deadlock.md) 예방.
- `synchronized`/`ReentrantLock`은 재진입 가능하지만, `StampedLock`은 **재진입 불가**(같은 스레드가 또 잡으면 데드락).

---

## 10. 참고
- [Java 공식 - java.util.concurrent.locks 패키지](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/locks/package-summary.html)
- [Baeldung - Guide to java.util.concurrent.Locks](https://www.baeldung.com/java-concurrent-locks)
- [Baeldung - CountDownLatch vs CyclicBarrier](https://www.baeldung.com/java-cyclicbarrier-countdownlatch)
- 관련 노트: [락 개념 종합](./locks.md) · [데드락](./deadlock.md)

---

**학습 날짜**: 2026-05-26
**계기**: locks.md의 JVM 락 목록 표에 있던 도구들(synchronized/ReentrantLock/ReadWriteLock/StampedLock/Semaphore/CountDownLatch/CyclicBarrier)이 각각 정확히 뭔지·어떻게 쓰는지 따로 정리하고 싶어서 분리

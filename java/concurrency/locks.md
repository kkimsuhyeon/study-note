# 락(Lock) 개념 종합 정리 — 낙관적 / 비관적 / 분산 락

> **한 줄 요약**: 여러 주체(스레드/트랜잭션/서버)가 같은 자원에 동시에 접근할 때 데이터 정합성을 지키기 위한 동시성 제어 장치. "충돌을 어떻게 다루나"(전략 축)와 "락이 어디까지 유효한가"(범위 축)는 서로 다른 차원이다.

관련 노트: [JPA @Lock 어노테이션](../jpa/lock.md) · [영속성 컨텍스트 · flush · 더티 체킹](../jpa/persistence-context.md)

---

## 0. 큰 그림 — 락은 두 개의 축으로 나뉜다

락을 이해할 때 가장 헷갈리는 게 "낙관적/비관적"과 "분산락"을 같은 줄에 놓고 비교하는 것. **둘은 다른 축이다.**

```
축 1) 충돌을 다루는 "전략"      : 낙관적 락  ↔  비관적 락
축 2) 락이 유효한 "범위(scope)" : 단일 프로세스(JVM)  →  DB  →  분산(여러 서버)
```

이 둘은 직교(orthogonal)한다. 예를 들어:
- **비관적 + DB 범위** = `SELECT ... FOR UPDATE`
- **낙관적 + DB 범위** = `@Version`
- **비관적 + 분산 범위** = Redis 분산락(Redisson)
- **낙관적 + 분산 범위** = 버전 기반 분산 캐시 갱신

> 즉 "분산락 vs 낙관적/비관적 락"은 잘못된 비교다. 분산락은 *범위*의 문제고, 그 분산락을 구현할 때 내부적으로 낙관적이냐 비관적이냐가 또 갈린다.

---

## 1. 전략 축 — 낙관적 락 vs 비관적 락

### 낙관적 락 (Optimistic Lock)
> "충돌은 드물 것이다. 일단 진행하고, 충돌나면 그때 처리하자."

- **실제로 락을 걸지 않는다.** version(또는 timestamp) 컬럼으로 "내가 읽은 뒤 누가 바꿨는지" 감지.
- 커밋 시점에 version 비교 → 안 맞으면 **즉시 예외**(`OptimisticLockException`), 대기 없음.
- 해결: 애플리케이션에서 **재시도(retry)**.

```sql
UPDATE product SET stock = 9, version = 2
WHERE id = 1 AND version = 1;   -- version 안 맞으면 0건 → 충돌 감지
```

- 장점: 락 비용 없음 → 처리량 높음, 데드락 없음
- 단점: 충돌 잦으면 재시도 폭증 → 오히려 비효율
- 적합: 읽기 많고 쓰기 충돌이 드문 경우 (게시글 수정, 프로필 변경)

### 비관적 락 (Pessimistic Lock)
> "충돌은 자주 일어난다. 미리 잠그고 시작하자."

- **실제로 자원을 잠근다.** 다른 주체는 락이 풀릴 때까지 **대기(blocking)**.
- DB 레벨에서는 `SELECT ... FOR UPDATE`(배타) / `FOR SHARE`(공유).

```sql
SELECT * FROM product WHERE id = 1 FOR UPDATE;  -- 이 row를 잠금
-- 다른 트랜잭션은 커밋/롤백까지 대기
```

- 장점: 충돌 확실히 방지, 재시도 불필요
- 단점: 락 보유 → 성능 저하, **데드락** 가능, 타임아웃 관리 필요
- 적합: 충돌 잦고 정확성이 최우선 (재고 차감, 좌석 예매, 계좌 이체)

### 충돌 시 동작 비교 (핵심)

| | 충돌 시 | 해결 |
|---|---|---|
| 낙관적 | **즉시 예외**, 대기 X | 애플리케이션 재시도 |
| 비관적 | **대기(blocking)**, 타임아웃 시 예외 | DB가 순서대로 처리 |

---

## 2. 범위 축 — 락이 어디까지 유효한가

### (1) 프로세스(JVM) 내부 락 — 단일 서버에서만 유효
같은 JVM 안의 스레드끼리만 제어. **서버가 2대 이상이면 무용지물.**

| 도구 | 설명 |
|------|------|
| `synchronized` | 가장 기본. 메서드/블록 단위 모니터 락 |
| `ReentrantLock` | 명시적 락. tryLock, 타임아웃, 공정성(fair) 지원 |
| `ReadWriteLock` | 읽기는 공유, 쓰기는 배타로 분리 |
| `StampedLock` | ReadWriteLock 개선판. 낙관적 읽기 지원 |
| `Semaphore` | 동시 접근 가능 수(permit)를 N개로 제한 |
| `CountDownLatch` / `CyclicBarrier` | 락은 아니지만 스레드 동기화 도구 |

> 각 도구의 정확한 동작·API·차이는 [JVM 동시성 도구 종합](./jvm-concurrency-tools.md) 참고.

```java
// hhplus 프로젝트에서 쓰던 방식: userId별 ReentrantLock
private final Map<Long, ReentrantLock> userLocks = new ConcurrentHashMap<>();
ReentrantLock lock = userLocks.computeIfAbsent(userId, k -> new ReentrantLock());
lock.lock();
try { /* 포인트 충전 */ } finally { lock.unlock(); }
```

### (2) DB 락 — DB를 공유하는 모든 서버에서 유효
DB가 락을 관리하므로 서버가 여러 대여도 안전. (JPA `@Lock`이 여기에 해당)

| 종류 | 설명 |
|------|------|
| 행 락 (Row Lock) | 특정 row 잠금 (`FOR UPDATE`) |
| 테이블 락 (Table Lock) | 테이블 전체 잠금 (범위 넓어 위험) |
| 공유 락 / 배타 락 | S-Lock(읽기 공유) / X-Lock(독점) |
| 갭 락 (Gap Lock) | MySQL InnoDB, 인덱스 범위 사이 잠금 (팬텀 리드 방지) |
| 넥스트키 락 (Next-Key Lock) | 행 락 + 갭 락 (InnoDB 기본) |
| 네임드 락 (Named Lock) | MySQL `GET_LOCK('name')` — 임의 문자열에 거는 락. DB 기반 분산락으로도 활용 |

### (3) 분산 락 (Distributed Lock) — 여러 서버/프로세스 간 유효
**서버가 여러 대(스케일 아웃)일 때** 사용. JVM 락은 서버별로 따로 놀고, DB 락은 DB에 부하를 주므로, 별도의 공유 저장소로 락을 관리.

| 구현 | 설명 |
|------|------|
| **Redis** (Redisson, SETNX) | 가장 흔함. 빠르고 TTL로 자동 해제. Redlock 알고리즘 |
| **Zookeeper** | 임시 노드(ephemeral) 기반. 강한 일관성, 순서 보장 |
| **etcd** | 쿠버네티스 생태계. lease 기반 |
| **DB 기반** (네임드 락, 비관적 락) | 인프라 추가 없이 가능하나 DB 부하 |

```java
// Redisson 분산락 예시
RLock lock = redissonClient.getLock("point:lock:" + userId);
boolean acquired = lock.tryLock(5, 3, TimeUnit.SECONDS); // 대기5초, 점유3초
if (acquired) {
    try { /* 포인트 충전 */ } finally { lock.unlock(); }
}
```

---

## 3. 분산락이 왜 따로 필요한가 (스케일 아웃 시나리오)

```
[서버 1대일 때]
  서버 A: ReentrantLock으로 충분 ✅

[서버 2대로 늘리면 (로드밸런서)]
  서버 A: userId=1 락 잡음 (A의 JVM 안에서만 유효)
  서버 B: userId=1 락 잡음 (B의 JVM 안에서만 유효)
  → 둘 다 "내가 락 잡았다" 생각하고 동시에 차감 → 정합성 깨짐 ❌

[해결: 공유된 외부 저장소(Redis 등)로 락]
  서버 A: Redis에 "lock:1" 설정 성공 → 진행
  서버 B: Redis에 "lock:1" 이미 있음 → 대기 ✅
```

> 핵심: **JVM 락은 "프로세스 경계"를 못 넘는다.** 서버를 늘리는 순간 분산락(또는 DB 락)이 필요해진다.

> "JVM이 왜 여러 개가 되나" — Spring 인스턴스 = 1 JVM, 스케일 아웃(인스턴스 복제), 로드밸런서(분배) vs 오토스케일러(생성)는 [스케일 아웃 & 배포 모델](../../infra/scaling.md) 참고. (이 문서는 "락 범위"에만 집중)

### 분산락 사용 시 주의점
1. **TTL(만료 시간) 필수** — 락 잡은 서버가 죽으면 영원히 안 풀림 → 데드락. TTL로 자동 해제.
2. **TTL < 작업 시간이면 위험** — 작업이 TTL보다 길어지면 락이 먼저 풀려 다른 서버가 침입. (Redisson의 watchdog가 자동 연장으로 완화)
3. **락 해제는 본인만** — 내가 건 락을 남이 풀면 안 됨 (토큰/소유자 검증).
4. **네트워크 분할(split-brain)** — Redis 단일 노드 장애 대비 Redlock, 또는 강한 일관성이 필요하면 Zookeeper.

---

## 4. 그 외 락 관련 개념들

| 개념 | 설명 |
|------|------|
| **재진입 락 (Reentrant)** | 같은 스레드가 이미 가진 락을 다시 획득 가능 (`synchronized`, `ReentrantLock`) |
| **공정 락 (Fair Lock)** | 락 대기 순서(FIFO) 보장. 처리량은 떨어짐 |
| **스핀 락 (Spin Lock)** | 대기 대신 계속 확인(busy-wait). 짧은 대기에 유리, CPU 낭비 위험 |
| **뮤텍스 (Mutex)** | 상호 배제. 1개만 접근 (배타 락의 일반 용어) |
| **세마포어 (Semaphore)** | N개까지 동시 접근 허용 (뮤텍스의 일반화) |
| **데드락 (Deadlock)** | 서로의 락을 기다리며 영원히 멈춤. 락 획득 순서 통일로 예방 |
| **라이브락 / 기아(Starvation)** | 계속 양보하다 진행 못 함 / 특정 스레드가 락을 못 얻음 |
| **낙관적 읽기 (StampedLock)** | 락 없이 읽고, 그동안 변경됐는지 검증 (JVM 레벨의 낙관적 락 느낌) |

---

## 5. 선택 가이드 (의사결정)

```
충돌이 드문가?
├─ 예 → 낙관적 락 (@Version) — 처리량 우선
└─ 아니오(자주 충돌) → 비관적 락 또는 분산락

서버가 몇 대인가?
├─ 1대 → JVM 락(ReentrantLock)으로 충분
├─ 여러 대 + DB 공유 → DB 락(@Lock) 또는 분산락
└─ 여러 대 + 고성능 필요 → Redis 분산락(Redisson)

DB 부하가 부담되는가?
├─ 예 → 분산락(Redis)으로 DB 락 회피
└─ 아니오 → DB 락이 인프라 단순 (추가 컴포넌트 불필요)
```

| 상황 | 추천 |
|------|------|
| 단일 서버, 학습/프로토타입 | `ReentrantLock` (userId별) |
| 멀티 서버, 충돌 드뭄 | 낙관적 락 `@Version` + 재시도 |
| 멀티 서버, 충돌 잦음, DB 부하 OK | 비관적 락 `@Lock(PESSIMISTIC_WRITE)` |
| 멀티 서버, 고성능, DB 부하 회피 | Redis 분산락 (Redisson) |
| 강한 일관성/순서 보장 필요 | Zookeeper |

---

## 6. 정리 (한눈에)

- **낙관적/비관적은 "전략", 분산락은 "범위"** — 다른 축이라 같이 비교하면 안 됨.
- 분산락도 내부적으로는 낙관적/비관적으로 다시 나뉜다.
- 범위: **JVM 락(1대) → DB 락(DB 공유) → 분산락(N대)** 순으로 적용 범위가 넓어짐.
- 락은 종류가 많지만, 실무 선택은 **"충돌 빈도 × 서버 수 × DB 부하"** 세 가지로 결정.

---

## 7. 참고
- [Martin Kleppmann - How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [Redis 공식 - Distributed Locks (Redlock)](https://redis.io/docs/latest/develop/use/patterns/distributed-locks/)
- [Baeldung - Java Concurrency Locks](https://www.baeldung.com/java-concurrent-locks)
- 관련 노트: [JPA @Lock 어노테이션](../jpa/lock.md)

---

**학습 날짜**: 2026-05-25
**계기**: JPA @Lock을 공부하다가 "낙관적/비관적/분산 락"의 관계와, 그 외 어떤 락들이 있는지 전체 그림이 궁금해서 정리

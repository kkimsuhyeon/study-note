# 데드락(Deadlock, 교착 상태) 정리

> **한 줄 요약**: 둘 이상의 주체가 서로가 가진 자원을 기다리며 아무도 진행하지 못하고 영원히 멈추는 상태. OS·스레드·DB 락 모두에 동일하게 적용되는 CS 고전 개념.

관련 노트: [락 개념 종합](./locks.md) · [JPA @Lock](../jpa/lock.md)

---

## 1. 데드락이란

> 둘 이상의 트랜잭션/스레드가 **서로가 점유한 자원을 기다리며** 무한정 멈추는 상태.

가장 쉬운 비유 — **좁은 골목에서 마주친 두 차**:
```
차 A: 앞으로 가려면 B가 비켜야 함 → B를 기다림
차 B: 앞으로 가려면 A가 비켜야 함 → A를 기다림
→ 둘 다 영원히 멈춤 💥
```

---

## 2. 데드락 발생 4가지 조건 (Coffman Conditions)

운영체제 이론의 고전. **아래 4가지가 동시에 성립할 때만** 데드락이 발생한다. 락이든 스레드든 DB든 동일.

| 조건 | 의미 | 락에 대입하면 |
|------|------|---------------|
| **① 상호 배제** (Mutual Exclusion) | 자원을 한 번에 하나만 점유 가능 | 배타락은 하나만 가짐 |
| **② 점유와 대기** (Hold and Wait) | 자원을 쥔 채 다른 자원을 기다림 | A락 쥐고 B락 대기 |
| **③ 비선점** (No Preemption) | 남이 쥔 자원을 강제로 뺏지 못함 | 락은 보유자가 풀어야만 해제 |
| **④ 순환 대기** (Circular Wait) | 대기가 원형으로 물림 (A→B→A) | T1→T2, T2→T1 |

> **핵심**: 4개 중 **하나라도 깨면** 데드락은 발생하지 않는다. 그래서 예방법은 보통 ②(점유와 대기)나 ④(순환 대기)를 깨는 방향이다.

---

## 3. 락에서의 데드락 사례

### 사례 1 — 락 획득 순서 꼬임 (순환 대기)
```
T1: lock(A) 쥠 → lock(B) 대기  ┐
T2: lock(B) 쥠 → lock(A) 대기  ┘ → 원형으로 물림 💥
```

### 사례 2 — 공유락 → 배타락 업그레이드
> **전제: 둘이 "같은 행(row)"을 다룬다.** 같은 행을 둘 다 읽고(S락 공존) 둘 다 그 행을 수정(X락)하려 할 때 생긴다. (다른 행이라도 읽기/쓰기 대상이 트랜잭션 간에 엇갈리면 여전히 데드락 — 아래 참고.)
```
T1: SELECT ... WHERE id=1 FOR SHARE   → id=1 에 S락
T2: SELECT ... WHERE id=1 FOR SHARE   → id=1 에 S락 (S끼리 호환 → 공존 OK)
T1: UPDATE ... WHERE id=1             → X락 필요, but T2의 S락 대기 ⏳
T2: UPDATE ... WHERE id=1             → X락 필요, but T1의 S락 대기 ⏳
→ 같은 행의 S락을 서로 놓길 기다림 → 순환 대기 → 데드락 💥
```
- S락은 **같은 행에 공존 가능** → 둘 다 읽기는 성공.
- X락(수정)은 다른 S락과 **호환 X** → 서로 상대의 S락 해제를 기다림.

#### "다른 행이면 안전"은 대칭 케이스 한정 — 엇갈리면(cross) 다른 행이어도 데드락

진짜 기준은 "다른 행이냐"가 아니라 **"순환 대기가 생기느냐"** 다.

```
[엇갈림(cross) — 다른 행인데도 데드락]
T1: SELECT id=1 FOR SHARE → S(1)
T2: SELECT id=2 FOR SHARE → S(2)          (다른 행, 여기까진 충돌 없음)
T1: UPDATE id=2 → X(2) 필요 → T2의 S(2) 대기 ⏳
T2: UPDATE id=1 → X(1) 필요 → T1의 S(1) 대기 ⏳   → 순환 대기 → 데드락 💥
```
이건 결국 **사례 1(락 획득 순서 꼬임)과 같은 구조** — T1은 1→2, T2는 2→1 순서로 자원을 건드려 원형으로 물린다. 먼저 잡은 락이 읽기(S)락일 뿐.

| 케이스 | 데드락? |
|--------|---------|
| T1: 1읽고 **1**수정 / T2: 2읽고 **2**수정 (각자 자기 행) | 안 남 — 자원이 안 겹침 |
| T1·T2 둘 다 1읽고 1수정 (같은 행 업그레이드) | **데드락** (위 사례 2) |
| T1: 1읽고 **2**수정 / T2: 2읽고 **1**수정 (엇갈림) | **데드락** (사례 1 구조) |

> ⚠️ **"없는 행"으로도 순환 대기가 생긴다 (MySQL InnoDB, REPEATABLE READ).** `FOR UPDATE`가 빈 범위를 잡으면 행이 아니라 **갭 락**이 걸리고, 갭 락끼리는 서로 호환이라 두 트랜잭션이 같은 빈 범위를 동시에 잠글 수 있다. 그다음 둘 다 그 범위에 INSERT하면 각자의 삽입이 **상대의 갭 락**에 막혀 데드락 — "없으면 만들기"(check-then-insert) 코드의 단골 사고. 해법은 유니크 제약 + 중복 예외 처리, 또는 READ COMMITTED(갭 락 대부분 비활성). (격리 수준별 동작 → [트랜잭션 격리](../jpa/transaction-isolation.md))

> 그래서 "읽고 곧 수정할 거면 처음부터 `FOR UPDATE`(배타락)를 써라"가 정설. → 읽기 단계에서 이미 배타락으로 진입하면 S락 공존이 없어 업그레이드 충돌이 사라지고, **락 획득 순서까지 통일**하면 엇갈림(순환 대기)도 막힌다.

### 사례 3 — JVM 락 (ReentrantLock / synchronized)
DB만의 문제가 아니다. 멀티스레드에서도 동일하게 발생.
```java
// T1
synchronized (lockA) { synchronized (lockB) { ... } }
// T2
synchronized (lockB) { synchronized (lockA) { ... } }  // 순서 반대 → 데드락 위험
```

---

## 4. 예방법

### ④ 순환 대기 깨기 — "락 획득 순서 통일" (가장 흔하고 효과적)
```
규칙: 항상 ID 오름차순으로 락을 잡는다
T1: lock(A) → lock(B)   ┐ 둘 다 A를 먼저 잡으니
T2: lock(A) → lock(B)   ┘ 원형이 생기지 않음 ✅
```

**순서는 "목록을 만든 순서"가 아니라 "키"에 묶는다.** 여러 행을 한 트랜잭션에서 잠글 때 잠글 대상을 **키(복합 키면 전체 키)로 정렬**한 뒤 차례로 잠근다.

```java
buckets.stream()
       .sorted(Comparator.comparing(Bucket::subject)     // 복합 기본 키 (subject, start, kind) 전체로 정렬
               .thenComparing(Bucket::start)              // → 어떤 두 키도 순서가 하나로 정해짐(전순서)
               .thenComparing(Bucket::kind))
       .forEach(this::lockAndConsume);
```

- 코드를 짠 순서(`List.of(세션 시간, 세션 일, IP 시간, IP 일)`)가 우연히 모든 요청에서 같다면 정렬 없이도 지금은 안전할 수 있다. 하지만 그 보장은 **목록을 만드는 코드 한 곳**에 기대고 있어서, 다른 코드 경로(예: 재시도 API가 IP 버킷부터 만들기)나 버킷 종류 추가 한 번에 조용히 깨진다. 키로 정렬하면 순서가 **데이터의 성질**이 되어 어느 코드 경로에서 잠가도 같다.
- 정렬 기준은 오름차순·내림차순 무엇이든 상관없다. **모든 곳이 같은 기준**이기만 하면 된다.
- DB는 데드락을 감지해 한쪽을 에러로 끝내 준다(PostgreSQL은 기본 1초 대기 후 검사). 그 에러가 사용자에게는 원인 모를 5xx로 보이므로, "감지되니 괜찮다"가 아니라 순서 통일로 **애초에 안 생기게** 한다.

### 그 외
| 방법 | 깨는 조건 | 설명 |
|------|-----------|------|
| 락 한 번에 모두 획득 | ② 점유와 대기 | 필요한 락을 한꺼번에 잡거나, 못 잡으면 가진 것도 다 놓기 |
| 락 타임아웃 | ③ 비선점 (유사) | 일정 시간 못 잡으면 포기 → 영원한 대기 방지 |
| 처음부터 배타락 | - | 공유락 업그레이드 데드락 회피 |
| 트랜잭션 짧게 유지 | - | 락 보유 시간이 짧으면 충돌·교착 확률 감소 |

```java
// 타임아웃 예시 (JPA) — ⚠️ 방언에 따라 무시된다 (아래)
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000")})
Optional<UserPoint> findByUserIdForUpdate(@Param("userId") Long userId);
```
> ⚠️ **`jakarta.persistence.lock.timeout`의 ms 값은 DB·Hibernate 버전에 따라 그냥 무시된다.**
> - **Hibernate 6.x + MySQL/PostgreSQL**: 두 DB 모두 `FOR UPDATE`에 "N초 대기" 문법이 없어 방언이 ms 값을 버린다. 반영되는 건 `0`(→ `NOWAIT`)과 `-2`(→ `SKIP LOCKED`)뿐. 위 3000은 효과 없음.
> - **Hibernate 7(7.1+)**: `ConnectionLockTimeoutStrategy`로 커넥션 단위 lock timeout 설정을 지원한다(incubating API).
> - 그래서 실제로 걸리는 건 **DB 기본값**: MySQL `innodb_lock_wait_timeout` = **50초**(무한 대기는 아님), PostgreSQL `lock_timeout` = **0(무제한)**. 짧게 끊으려면 DB/세션 설정(`SET lock_timeout = '3s'` 등)으로 건다.
```java
// 타임아웃 예시 (ReentrantLock)
if (lock.tryLock(3, TimeUnit.SECONDS)) {
    try { ... } finally { lock.unlock(); }
} else {
    // 락 획득 실패 처리 (데드락/경합 회피)
}
```

---

## 5. DB는 데드락을 자동 감지한다

다행히 **MySQL InnoDB, PostgreSQL은 데드락을 자동 탐지**한다. 순환 대기를 감지하면 **희생자(victim) 트랜잭션 하나를 강제 롤백**시켜 교착을 푼다.

```
순환 대기 감지 → 한 트랜잭션 강제 롤백 (나머지는 진행)
→ 롤백당한 쪽은 예외를 받음
   - MySQL: "Deadlock found when trying to get lock" (에러 1213)
   - Spring(+Hibernate): PessimisticLockingFailureException 계열 (CannotAcquireLockException 등 — 변환 경로에 따라 다름)
     ※ DeadlockLoserDataAccessException은 Spring 6.0.3부터 deprecated
→ 보통 재시도로 처리 (낙관적 락 재시도와 유사한 패턴)
```

> 즉 영원히 멈추진 않고 한 명이 희생되고 나머지는 진행된다. 단, **희생된 트랜잭션의 재시도 처리는 개발자 몫**이다.

> ⚠️ **JVM 데드락은 아무도 풀어주지 않는다.** DB와 달리 JVM은 `synchronized`/`ReentrantLock` 순환 대기를 감지해 희생자를 고르지 않는다 — 스레드들이 **영원히** 멈추고, 그 스레드를 쓰던 요청·풀 슬롯도 같이 묶인다. 감지는 사후에 사람이: `jstack <pid>`(스레드 덤프 끝에 "Found one Java-level deadlock")나 `ThreadMXBean.findDeadlockedThreads()`로 찾고, 복구는 사실상 재시작. 그래서 JVM 쪽은 **예방(순서 통일)과 `tryLock` 타임아웃**이 DB보다 더 중요하다.

### 데드락 vs 락 타임아웃 (헷갈리기 쉬움)
| | 데드락 | 락 타임아웃 |
|---|---|---|
| 원인 | 순환 대기 (서로 물림) | 단순히 오래 기다림 (락 못 잡음) |
| 감지 주체 | DB가 자동 감지 후 victim 롤백 | 설정한 시간 초과 시 예외 |
| DB 쪽 원인 | MySQL 1213 | MySQL 1205 (`innodb_lock_wait_timeout`) / PG `lock_timeout` |
| 앱이 받는 예외 (Spring+Hibernate) | `PessimisticLockingFailureException` 계열 | 같은 계열 — JPA `LockTimeoutException`은 Spring이 `CannotAcquireLockException`(그 하위)으로 변환 |

> 💡 예외 타입으로 둘을 정확히 가르기는 어렵다(DB·드라이버·Hibernate 버전마다 매핑이 다름) → 재시도 정책은 **`PessimisticLockingFailureException` 계열을 한 번에** 잡아 걸고, 원인 구분이 필요하면 원본 SQL 에러 코드를 본다.

---

## 6. 정리

- 데드락 = **서로가 가진 자원을 기다리며 멈춘 상태** (OS 고전 개념과 동일).
- **4조건(상호배제·점유와대기·비선점·순환대기)** 이 모두 성립할 때만 발생 → 하나만 깨면 예방.
- 실무 1순위 예방책: **락 획득 순서 통일**(순환 대기 제거) + **타임아웃** + **트랜잭션 짧게**.
- DB는 자동 감지해 한쪽을 롤백 → 개발자는 **재시도**로 대응. JVM은 감지·해소가 없다 → 예방과 `tryLock` 타임아웃이 전부.

> 💡 **판단 기준: "누가 풀어주나"로 대비 수준을 정한다.** DB 데드락은 DB가 victim을 골라 풀어주니 "재시도 가능한 예외"로 설계하면 되고, JVM 데드락은 아무도 안 풀어주니 "애초에 순환이 생기지 않는 순서 규칙 + 기다림의 상한"으로 설계한다.

---

## 7. 참고
- [Coffman conditions - Wikipedia](https://en.wikipedia.org/wiki/Deadlock#Necessary_conditions)
- [MySQL - Deadlocks in InnoDB](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html) · [InnoDB Locking (갭 락·삽입 의도 락)](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [Hibernate User Guide - Locking (11.4)](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html#locking) · [Hibernate 6.6 MySQLDialect 소스](https://github.com/hibernate/hibernate-orm/blob/6.6/hibernate-core/src/main/java/org/hibernate/dialect/MySQLDialect.java) — lock timeout은 NOWAIT/SKIP LOCKED만 SQL에 반영
- [MySQL - innodb_lock_wait_timeout (기본 50초)](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout) · [PostgreSQL - lock_timeout (기본 0)](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-LOCK-TIMEOUT)
- [Spring - DeadlockLoserDataAccessException (6.0.3 deprecated)](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/dao/DeadlockLoserDataAccessException.html)
- 관련 노트: [락 개념 종합](./locks.md) · [JPA @Lock](../jpa/lock.md)

---

**학습 날짜**: 2026-05-25
**계기**: JPA 락을 공부하다 공유락 업그레이드 데드락을 접하고, 데드락이 OS의 기본 개념(교착 상태)과 같은 것인지 궁금해서 정리

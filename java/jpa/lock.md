# @Lock - Spring Data JPA 락 어노테이션

> **한 줄 요약**: Spring Data JPA Repository 메서드에 락 모드를 지정해, 쿼리 실행 시 DB 레벨의 락을 걸 수 있게 해주는 어노테이션. (`@Lock`은 Spring Data JPA 것이고, 락 모드 `LockModeType`만 JPA 표준이다)

```java
import org.springframework.data.jpa.repository.Lock;  // Spring Data JPA (jakarta.persistence.Lock은 없다)
import jakarta.persistence.LockModeType;              // JPA 표준 enum

@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<UserPoint> findByUserId(@Param("userId") Long userId);
```

관련 노트: [영속성 컨텍스트 · flush · 더티 체킹](./persistence-context.md) · [@Lock 심화 개념](./lock-concepts.md) · [@Lock 실무 패턴](./lock-practical.md)

> 이 문서는 **@Lock 기본**(언제·종류·기본 사용·주의)만 다룬다. 헷갈리는 심화 개념(@Version vs @Lock, 공유/배타, FORCE_INCREMENT)은 [lock-concepts.md](./lock-concepts.md), 실무 적용 패턴(프록시·네이밍·테스트·재시도·벌크 등)은 [lock-practical.md](./lock-practical.md)로 분리.

---

## 1. 언제 쓰나

여러 트랜잭션이 **같은 데이터를 동시에 수정**할 가능성이 있을 때.

대표적인 시나리오:
- 포인트 충전/사용 (잔액 동시 변경)
- 재고 차감 (한정 수량 상품)
- 좌석 예매 (선착순)
- 계좌 이체

핵심은 **lost update(갱신 분실)** 문제를 막는 것.

```
[갱신 분실 예시]
T1: SELECT balance = 1000
T2: SELECT balance = 1000
T1: UPDATE balance = 1000 - 300  → 700
T2: UPDATE balance = 1000 - 500  → 500  ← T1의 차감이 사라짐
```

> 이걸 막으려면 **읽기·수정이 같은 트랜잭션**이어야 한다 → [Read-Modify-Write와 트랜잭션 경계](./read-modify-write.md)

---

## 2. LockModeType 종류

| 모드 | 동작 | 사용 SQL (대략) |
|------|------|-----------------|
| `NONE` | 락 없음 (기본값) | - |
| `OPTIMISTIC` | 낙관적 락. @Version 컬럼으로 충돌 감지 | 커밋 시점에 version 체크 |
| `OPTIMISTIC_FORCE_INCREMENT` | 낙관적 락 + 무조건 version 증가 | UPDATE ... version+1 |
| `PESSIMISTIC_READ` | 공유 락. 다른 트랜잭션의 쓰기만 차단 | `SELECT ... FOR SHARE` |
| `PESSIMISTIC_WRITE` | 배타 락. 다른 트랜잭션의 쓰기·락 읽기 차단 (일반 SELECT는 허용) | `SELECT ... FOR UPDATE` |
| `PESSIMISTIC_FORCE_INCREMENT` | 배타 락 + version 증가 | FOR UPDATE + version+1 |

> ⚠️ `PESSIMISTIC_WRITE`는 "아무도 읽지 못하게"가 아니다. 자세한 동작은 [@Lock 실무 패턴](./lock-practical.md)의 "PESSIMISTIC_WRITE가 모든 SELECT를 막는 것은 아니다" 참고.

### 비관적 락 (Pessimistic) vs 낙관적 락 (Optimistic)

| 구분 | 비관적 락 | 낙관적 락 |
|------|-----------|-----------|
| 전제 | "충돌은 자주 일어난다" | "충돌은 드물다" |
| 방식 | DB 락으로 선점 | version 컬럼 비교 |
| **충돌 시 동작** | **대기(blocking)**, 타임아웃 시 예외 | **즉시 예외**(`OptimisticLockException`), 대기 X |
| 해결 방법 | DB가 순서대로 처리 (재시도 불필요) | 애플리케이션에서 **재시도(retry)** |
| **누가 처리하나** | 대기는 **DB가 자동** / 개발자는 **타임아웃만** 챙김 | 재시도는 **개발자 책임**(`@Retryable` 등 직접 작성) |
| 비용 | DB 락 보유 → 성능 저하, 데드락 가능 | 충돌 시 재시도 비용 |
| 적합한 경우 | 충돌 빈도 높음, 정확성 최우선 | 충돌 빈도 낮음, 처리량 중요 |
| 예시 | 포인트 차감, 재고 | 게시글 수정, 프로필 변경 |

> **version이 틀리면 대기? 에러?** → 낙관적 락은 락을 실제로 걸지 않고 커밋 시점에 version만 비교한다. 안 맞으면 **기다리지 않고 즉시 예외** → 롤백 → 애플리케이션에서 재시도하거나 사용자에게 알려야 함. 반면 비관적 락은 DB가 row를 잠그므로 다른 트랜잭션이 **대기**하고, 락이 풀리면 이어서 진행된다 (대기 한도 초과·데드락일 때만 예외 → §4).

> ⚠️ 재시도는 **트랜잭션 바깥에서 새 트랜잭션으로** 해야 하고, Spring 환경에서 잡을 예외는 JPA `OptimisticLockException`이 아니라 Spring이 감싼 `ObjectOptimisticLockingFailureException`(상위 `OptimisticLockingFailureException`)이다 — JPA 예외로 catch하면 안 잡힌다. (왜·`@Retryable` 패턴 → [@Lock 실무 패턴 §6](./lock-practical.md))

> 위 모드/전략에서 헷갈리는 부분(@Version만 vs @Lock(OPTIMISTIC), 공유락 vs 배타락, OPTIMISTIC vs FORCE_INCREMENT)은 [@Lock 심화 개념](./lock-concepts.md)에서 따로 정리.

---

## 3. 사용 예시

### 비관적 락 (PESSIMISTIC_WRITE)

```java
public interface UserPointRepository extends JpaRepository<UserPoint, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT u FROM UserPoint u WHERE u.userId = :userId")
    Optional<UserPoint> findByUserIdForUpdate(@Param("userId") Long userId);
}
```

```java
@Service
@RequiredArgsConstructor
public class PointService {

    private final UserPointRepository userPointRepository;

    @Transactional  // 필수
    public UserPoint charge(Long userId, long amount) {
        UserPoint point = userPointRepository.findByUserIdForUpdate(userId)
            .orElseThrow();
        point.charge(amount);
        return point;  // 더티 체킹으로 UPDATE
    }
}
```

실행되는 SQL:
```sql
SELECT * FROM user_point WHERE user_id = ? FOR UPDATE;
```

→ 이 row를 잡은 트랜잭션이 커밋/롤백할 때까지 다른 트랜잭션은 같은 row에 대해 SELECT FOR UPDATE / UPDATE가 **block**된다.

### 낙관적 락 (`@Version`)

```java
@Entity
public class UserPoint {
    @Id private Long userId;
    private Long point;

    @Version  // 이것만으로 낙관적 락 동작 — 별도 @Lock 불필요
    private Long version;
}
```

```java
// 평범한 조회 + 더티 체킹이 곧 낙관적 락
UserPoint point = userPointRepository.findById(userId).orElseThrow();
point.charge(amount);   // flush 시 UPDATE ... WHERE user_id = ? AND version = ?
```

커밋(flush) 시점에 version이 안 맞으면(0건 갱신) 예외 → 재시도 또는 사용자 통지. `@Lock(LockModeType.OPTIMISTIC)`은 **수정하지 않고 읽기만 한 엔티티**까지 검증하고 싶을 때만 덧붙이는 보조 옵션이다 → [@Lock 심화 §1](./lock-concepts.md).

> 실무 적용 패턴(`@Lock` 동작 위치·메서드 네이밍·조회/수정 분리·동시성 테스트·재시도·벌크/조건부 UPDATE·인덱스)은 [@Lock 실무 패턴](./lock-practical.md) 참고.

---

## 4. 주의사항

### (1) @Transactional 필수
락은 트랜잭션 범위에서만 유지된다. 트랜잭션 없이 호출하면 락이 즉시 해제되거나 예외 발생.

### (2) 데드락 위험
여러 row를 락 거는 순서가 일관되지 않으면 데드락 발생 가능.

```
T1: lock(A) → lock(B) 시도 (대기)
T2: lock(B) → lock(A) 시도 (대기)
→ Deadlock
```

해결: **항상 같은 순서로 락 획득** (예: ID 오름차순). (자세히는 [데드락](../concurrency/deadlock.md))

### (3) 락 타임아웃 설정
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000")})
Optional<UserPoint> findByUserIdForUpdate(@Param("userId") Long userId);
```
- ⚠️ **이 힌트는 DB·Hibernate 버전에 따라 무시된다.** Hibernate 6.x의 MySQL·PostgreSQL 방언은 `0`(→ `NOWAIT`)·`-2`(→ `SKIP LOCKED`)만 SQL에 반영하고 그 밖의 ms 값은 버린다(`FOR UPDATE WAIT n`을 지원하는 Oracle 등만 반영). Hibernate 최신 소스(7.x)에는 커넥션 수준 설정(`set local lock_timeout` / `SET @@SESSION.innodb_lock_wait_timeout`)으로 적용하는 경로가 생겼다(도입 버전 확인 필요). → 실제 나가는 SQL·설정을 로그로 확인할 것.
- 힌트가 안 먹으면 **DB 기본값**을 따른다: MySQL `innodb_lock_wait_timeout` 기본 **50초**, PostgreSQL `lock_timeout` 기본 **0(무제한 대기)**. PostgreSQL에서 무한 대기 → 커넥션 고갈 → 장애 전파를 막으려면 별도로 설정해야 한다.

### (4) DB 종류별 지원 차이
- MySQL: `FOR UPDATE` 지원, `FOR SHARE`는 8.0+ (이전엔 `LOCK IN SHARE MODE`)
- PostgreSQL: `FOR UPDATE`, `FOR SHARE` 지원
- H2: 지원하지만 동작이 미묘하게 다름 → 테스트 시 주의

### (5) 트랜잭션 격리 수준과 무관
`@Lock`은 격리 수준과는 별도 개념. READ_COMMITTED여도 PESSIMISTIC_WRITE는 동작함.

---

## 5. 다른 동시성 제어와의 비교

JVM 락(`synchronized`, `ReentrantLock`, 사용자별 락을 담은 `ConcurrentHashMap<Long, ReentrantLock>`)은 **그 JVM 안에서만** 유효하다. 포인트 충전을 사용자별 `ReentrantLock`으로 직렬화하던 서버를 2대로 늘리면, 각 서버의 락이 따로 놀아 lost update가 다시 생긴다. 같은 로직을 §3의 `findByUserIdForUpdate` + `@Transactional`로 바꾸면 **DB가 행 단위로 줄을 세우므로** 서버 대수와 무관하게 막힌다. 대신 대기가 DB 커넥션을 쥔 채 일어나므로 성능 특성과 장애 양상(커넥션 고갈·데드락)이 달라진다.

→ 범위별 비교(JVM 락 / DB 락 / 분산 락)는 [락 개념 종합 §2](../concurrency/locks.md), JVM 도구는 [JVM 동시성 도구](../concurrency/jvm-concurrency-tools.md).

---

## 6. 💡 판단 기준

- **기본은 `@Version`(낙관).** 충돌이 드물고, 충돌 시 재시도나 "다시 시도하세요" 통지로 충분한 수정(프로필·게시글)이면 이것만 붙인다. `@Lock(OPTIMISTIC)`은 "안 고치는 엔티티 값에 내 판단이 걸릴 때"만 → [@Lock 심화 §1](./lock-concepts.md).
- **같은 행에 충돌이 잦고 실패 비용이 큰 경로(잔액·재고·좌석)만 `PESSIMISTIC_WRITE`.** 붙이는 순간 타임아웃이 실제로 먹는지(§4-(3))와 락 순서(데드락)를 같이 확인한다.
- **읽고 판단할 게 없는 단순 차감이면 락 대신 조건부 UPDATE** (`... WHERE stock >= ?` + 영향 행 수) → [@Lock 실무 패턴 §8](./lock-practical.md).
- **사람이 끼는 편집(GET 폼 → POST 저장)은 비관락 불가** — version을 화면에 실어 보냈다가 저장 때 비교하는 오프라인 낙관락 → [락 개념 종합](../concurrency/locks.md).
- **서버가 2대 이상이면 JVM 락은 답이 아니다** (§5).

---

## 7. 참고

- [Spring Data JPA - Locking 공식 문서](https://docs.spring.io/spring-data/jpa/reference/jpa/locking.html)
- [Hibernate ORM 6.6 User Guide - Locking (lock timeout 힌트·미지원 시 무시)](https://docs.hibernate.org/orm/6.6/userguide/html_single/#locking)
- [MySQL 8.0 - innodb_lock_wait_timeout (기본 50초)](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- [PostgreSQL - lock_timeout (기본 0 = 무제한)](https://www.postgresql.org/docs/current/runtime-config-client.html)
- [Baeldung - Pessimistic Locking in JPA](https://www.baeldung.com/jpa-pessimistic-locking)
- 관련 노트: [@Lock 심화 개념](./lock-concepts.md) · [@Lock 실무 패턴](./lock-practical.md) · [영속성 컨텍스트](./persistence-context.md) · [락 개념 종합](../concurrency/locks.md)

---

**학습 날짜**: 2026-05-25
**계기**: 포인트 충전 동시성을 `ReentrantLock`으로 제어하다가, 서버가 여러 대일 때 JPA에서는 어떻게 처리하는지 궁금해서 조사 (2026-05-26 심화/실무를 별도 문서로 분리, 2026-10-02 사실 보정·💡 추가)

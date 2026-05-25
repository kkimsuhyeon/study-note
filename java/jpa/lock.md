# @Lock - JPA 락 어노테이션

> **한 줄 요약**: JPA Repository 메서드에 락 모드를 지정해, 쿼리 실행 시 DB 레벨의 락을 걸 수 있게 해주는 어노테이션.

```java
import jakarta.persistence.Lock;
import jakarta.persistence.LockModeType;

@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<UserPoint> findByUserId(@Param("userId") Long userId);
```

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

---

## 2. LockModeType 종류

| 모드 | 동작 | 사용 SQL (대략) |
|------|------|-----------------|
| `NONE` | 락 없음 (기본값) | - |
| `OPTIMISTIC` | 낙관적 락. @Version 컬럼으로 충돌 감지 | 커밋 시점에 version 체크 |
| `OPTIMISTIC_FORCE_INCREMENT` | 낙관적 락 + 무조건 version 증가 | UPDATE ... version+1 |
| `PESSIMISTIC_READ` | 공유 락. 다른 트랜잭션의 쓰기만 차단 | `SELECT ... FOR SHARE` |
| `PESSIMISTIC_WRITE` | 배타 락. 읽기/쓰기 모두 차단 | `SELECT ... FOR UPDATE` |
| `PESSIMISTIC_FORCE_INCREMENT` | 배타 락 + version 증가 | FOR UPDATE + version+1 |

### 비관적 락 (Pessimistic) vs 낙관적 락 (Optimistic)

| 구분 | 비관적 락 | 낙관적 락 |
|------|-----------|-----------|
| 전제 | "충돌은 자주 일어난다" | "충돌은 드물다" |
| 방식 | DB 락으로 선점 | version 컬럼 비교 |
| 비용 | DB 락 보유 → 성능 저하, 데드락 가능 | 충돌 시 재시도 비용 |
| 적합한 경우 | 충돌 빈도 높음, 정확성 최우선 | 충돌 빈도 낮음, 처리량 중요 |
| 예시 | 포인트 차감, 재고 | 게시글 수정, 프로필 변경 |

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

### 낙관적 락 (OPTIMISTIC)

```java
@Entity
public class UserPoint {
    @Id private Long userId;
    private Long point;

    @Version  // 낙관적 락에 필요
    private Long version;
}
```

```java
@Lock(LockModeType.OPTIMISTIC)
@Query("SELECT u FROM UserPoint u WHERE u.userId = :userId")
Optional<UserPoint> findByUserId(@Param("userId") Long userId);
```

커밋 시점에 version이 안 맞으면 `OptimisticLockException` 발생 → 재시도 로직 필요.

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

해결: **항상 같은 순서로 락 획득** (예: ID 오름차순).

### (3) 락 타임아웃 설정
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000")})
Optional<UserPoint> findByUserIdForUpdate(@Param("userId") Long userId);
```
타임아웃 없으면 무한 대기 → 장애 전파.

### (4) DB 종류별 지원 차이
- MySQL: `FOR UPDATE`, `FOR SHARE` 지원 (8.0+)
- PostgreSQL: `FOR UPDATE`, `FOR SHARE` 지원
- H2: 지원하지만 동작이 미묘하게 다름 → 테스트 시 주의

### (5) 트랜잭션 격리 수준과 무관
`@Lock`은 격리 수준과는 별도 개념. READ_COMMITTED여도 PESSIMISTIC_WRITE는 동작함.

---

## 5. 다른 동시성 제어와의 비교

현재 진행 중인 hhplus-tdd-java 프로젝트는 **JPA 없이** `ConcurrentHashMap<Long, ReentrantLock>`으로 동시성을 제어 중. 같은 문제를 푸는 방식들 비교:

| 방식 | 범위 | 장점 | 단점 |
|------|------|------|------|
| `synchronized` / `ReentrantLock` | **JVM 단일 인스턴스** | 간단함, 빠름 | 서버 여러 대면 무용지물 |
| `ConcurrentHashMap` + `ReentrantLock` (userId별) | JVM 단일 인스턴스 | 유저별 락 → 다른 유저는 병렬 처리 가능 | 위와 동일 |
| **JPA `@Lock`** (DB 락) | **DB를 공유하는 모든 인스턴스** | 멀티 서버에서도 안전 | DB 부하, 데드락 가능 |
| Redis 분산 락 (Redisson 등) | 분산 환경 | DB 부하 낮음, 유연 | 인프라 의존성 추가 |

→ **단일 서버 학습 단계**에서는 ReentrantLock으로 충분. 실제 서비스(서버 N대)에서는 DB 락이나 분산 락이 필요.

### hhplus 프로젝트 현재 코드와의 연결

```java
// PointServiceImpl.java
private final Map<Long, ReentrantLock> userLocks = new ConcurrentHashMap<>();

ReentrantLock lock = userLocks.computeIfAbsent(userId, k -> new ReentrantLock());
lock.lock();
try {
    // 포인트 충전/사용 로직
} finally {
    lock.unlock();
}
```

이 코드를 JPA + `@Lock(PESSIMISTIC_WRITE)`로 바꾸면:
```java
@Transactional
public UserPoint charge(Long userId, long amount) {
    UserPoint point = userPointRepository.findByUserIdForUpdate(userId).orElseThrow();
    point.charge(amount);
    return point;
}
```
→ 코드는 깔끔해지지만, DB가 락을 관리하므로 성능 특성과 장애 시나리오가 달라진다.

---

## 6. 참고

- [Spring Data JPA - Locking 공식 문서](https://docs.spring.io/spring-data/jpa/reference/jpa/locking.html)
- [Baeldung - Pessimistic Locking in JPA](https://www.baeldung.com/jpa-pessimistic-locking)
- 관련 노트: (TODO) `transactional.md`, `version.md`

---

**학습 날짜**: 2026-05-25
**계기**: hhplus-tdd-java 프로젝트에서 ReentrantLock으로 동시성 제어 중, JPA에서는 어떻게 처리하는지 궁금해서 조사

# @Lock 실무 패턴 — 프록시·네이밍·테스트·재시도·벌크·조건부 UPDATE

> **한 줄 요약**: JPA `@Lock`을 실제 코드에 적용할 때의 패턴과 함정 — `@Lock`이 동작하는 위치(Spring Data 프록시), 메서드 네이밍, 조회/수정 분리, 동시성 테스트, `PESSIMISTIC_WRITE`의 SELECT 동작, 낙관적 락 재시도, 벌크 UPDATE 우회, 조건부 UPDATE, 인덱스.

관련 노트: [@Lock 기본](./lock.md) · [@Lock 심화 개념](./lock-concepts.md) · [영속성 컨텍스트](./persistence-context.md) · [테스트 작성 가이드](../test/test-writing-guide.md)

> 이 문서는 **@Lock 실무 적용**을 다룬다. 동시성 테스트가 깨지거나 `@Lock`이 안 먹는 것 같을 때, 재시도·벌크 UPDATE·조건부 UPDATE 중 무엇을 골라야 할지 볼 때 다시 연다.

---

## 1. `@Lock`은 어디서 동작하나 (Spring Data JPA 프록시)

`@Lock`은 단순 표시용 어노테이션이 아니다. **Spring Data JPA가 `JpaRepository` 인터페이스를 보고 구현체(프록시)를 생성할 때**, `@Lock`이 붙은 메서드에 락 쿼리를 적용하도록 처리한다.

→ 즉 `@Lock`은 **Spring Data JPA Repository 인터페이스에서만 동작**한다. 일반 클래스(예: 헥사고날의 Adapter)에 붙이면 **그냥 무시된다.**

```java
// ❌ 동작 안 함 — 일반 클래스라 Spring Data JPA가 프록시를 안 만듦
@Repository
public class ConcertRepositoryAdapter implements ConcertRepository {
    private final ConcertJpaRepository jpaRepository;

    @Lock(LockModeType.PESSIMISTIC_WRITE)  // 무시됨!
    public Optional<ConcertEntity> findById(String id) { ... }
}
```

### 헥사고날 아키텍처에서의 처리
락은 **JpaRepository 인터페이스에 메서드를 정의**하고, Adapter는 그걸 호출만 한다.

```java
// JpaRepository 인터페이스에 락 메서드 정의 (여기서만 @Lock이 먹힌다)
public interface ConcertJpaRepository extends JpaRepository<ConcertEntity, String> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT c FROM ConcertEntity c WHERE c.id = :id")
    Optional<ConcertEntity> findByIdForUpdate(@Param("id") String id);
}

// Adapter는 호출만
@Repository
@RequiredArgsConstructor
public class ConcertRepositoryAdapter implements ConcertRepository {
    private final ConcertJpaRepository jpaRepository;
    public Optional<ConcertEntity> findByIdForUpdate(String id) {
        return jpaRepository.findByIdForUpdate(id);
    }
}
```

### 대안: EntityManager 직접 사용
정 Adapter 레벨에서 직접 락을 걸어야 하면 `EntityManager`를 쓸 수 있다. (단, fetch join 등은 직접 처리해야 해서 권장도는 낮음)

```java
ConcertEntity concert = entityManager.find(
    ConcertEntity.class, concertId, LockModeType.PESSIMISTIC_WRITE
);
```

> **`@Lock` + `JOIN FETCH` 주의**: 어느 테이블 행까지 잠기는지는 **DB·방언·Hibernate 버전마다 다르다.** `OF` 없는 `FOR UPDATE`로 나가면 조인된 연관 테이블 행까지 잠기고(공유 코드 테이블 같은 행까지 잠가 경합이 커질 수 있음), `FOR UPDATE OF 별칭`이 붙으면 지정한 테이블만 잠긴다. "주 엔티티만 잠기겠지"라고 가정하지 말고 **실제 SQL을 로그로 확인**할 것. (`OF` 동작 → [SELECT FOR UPDATE](../../database/select-for-update.md))

---

## 2. 락 메서드 네이밍 컨벤션

락 조회 메서드 이름을 어떻게 지을지도 실무에서 자주 고민되는 부분.

### 후보별 비교
| 네이밍 | 평가 |
|--------|------|
| `findByIdForUpdate` | ✅ SQL `FOR UPDATE`에서 유래. 백엔드 개발자에게 가장 직관적, 사실상 표준 |
| `findByIdExclusive` | ✅ "배타적". 기술 중립적이라 구현(Redis 등) 바뀌어도 의미 유지 |
| `findByIdLocked` | △ 짧지만 덜 일반적 |
| `findByIdWithLock` | ❌ **`With` 충돌** — 아래 참고 |

### ⚠️ `WithLock`을 피해야 하는 이유
Spring Data JPA에서 **`With`는 이미 "연관 엔티티를 fetch join한다"는 강한 관례**를 가진다.
```java
findWithSchedulesById(...)   // "스케줄을 함께 가져온다"는 의미
```
여기에 `findByIdWithLock`을 쓰면 "Lock이라는 연관 엔티티를 함께 가져오나?"로 오해될 수 있다. 심하면 `findWithSchedulesByIdWithLock`처럼 `With`가 두 번 나오는 이상한 이름이 된다.

### 레이어별 네이밍 전략 (권장)
- **Repository(인프라)**: 기술 용어 그대로 → `findByIdForUpdate`
- **Service(도메인)**: 기술 중립/비즈니스 의도 → `getSeatExclusive` 또는 `getSeatForReservation`

```java
// Repository — 기술 용어
Optional<SeatEntity> findByIdForUpdate(String id);

// Service — 도메인 의도 (구현이 바뀌어도 이름이 유효)
SeatEntity getSeatExclusive(String seatId);       // 기술 중립
SeatEntity getSeatForReservation(String seatId);  // 비즈니스 의도(DDD 유비쿼터스 언어)
```

> 헥사고날/DDD 관점: 도메인 레이어는 "DB 락"이라는 기술 세부사항을 몰라야 한다. 그래서 Service에서는 `ForUpdate`보다 `Exclusive`/`ForReservation`이 더 적절. 단 **일관성이 최우선** — 한 번 정한 규칙은 프로젝트 전체에 동일 적용.

---

## 3. 락 메서드를 분리해야 하나? (조회용 vs 수정용)

> 관련: 쓰기 로직이 **왜 조회 서비스에 의존하면 안 되는지**(read-modify-write를 같은 트랜잭션에서 직접 읽는 이유)는 [Read-Modify-Write와 트랜잭션 경계](./read-modify-write.md) 참고.

"모든 조회에 락을 걸면 성능 저하" → 락 조회와 일반 조회를 **별도 메서드로 분리**하는 게 정석.

### 왜 분리하나
Spring Data JPA에서 `@Lock`은 **메서드 레벨**에 붙는다. 같은 메서드를 호출하면서 "이번엔 락 걸고, 다음엔 빼고"처럼 **조건부 제어가 불가능**하다. 그래서 분리가 강제된다.

```java
public interface SeatRepository {
    Optional<SeatEntity> findById(String id);            // 조회용 (락 X)
    Optional<SeatEntity> findByIdForUpdate(String id);   // 수정용 (락 O)
}
```

### 트랜잭션 안에서는 1차 캐시 — 락은 "처음 읽을 때" 건다
한 번 락 걸고 조회한 엔티티는 커밋까지 행 락이 유지되므로 **락은 처음 한 번만** 걸면 된다. 단 1차 캐시가 SQL을 아껴 주는 건 `findById`(id 조회)뿐이고, JPQL·파생 쿼리는 **항상 SQL을 실행**한 뒤 이미 영속성 컨텍스트에 있는 엔티티는 DB 값을 버리고 기존 인스턴스를 돌려준다 ([영속성 컨텍스트 §1](./persistence-context.md)).

> ⚠️ 그래서 **락 없이 먼저 읽은 엔티티를 나중에 `findByIdForUpdate`로 다시 읽으면** 행은 잠기지만 객체 상태는 처음 읽은 옛 값 그대로다. `@Version`이 있으면 Hibernate가 락을 올리는 순간 version을 검사해 잡아내지만([@Lock 심화 §4](./lock-concepts.md)), 없으면 옛 값으로 계산해 조용히 lost update가 난다. → 수정할 엔티티는 **처음부터 락 메서드로** 읽는다.

### CQRS로 자연스럽게 분리
조회 전용 Service와 명령 Service를 나누면 락 필요 여부가 저절로 갈린다.
```java
@Service @Transactional(readOnly = true)   // 조회 전용 → 락 불필요
public class ConcertQueryService { ... }

@Service @Transactional                     // 명령 → 필요시 락
public class ReservationService {
    public void reserve(...) {
        SeatEntity seat = seatService.getSeatExclusive(seatId);  // 여기서만 락
    }
}
```

---

## 4. 락 테스트 작성법

동시성 문제는 **여러 스레드가 실제로 동시에 접근할 때만** 드러나므로, 테스트도 그 상황을 시뮬레이션해야 한다. 3개 레벨로 나뉜다.

### 레벨 1 — 단위 테스트 (Mock): "올바른 메서드를 부르는가"
락 동작 자체가 아니라 **흐름**만 검증. 빠르고 안정적.
```java
verify(seatService, times(1)).getSeatExclusive("1");  // 락 메서드 호출 확인
verify(seatService, never()).getSeat(any());          // 일반 조회 안 썼는지
```

### 레벨 2·3 — 동시성 통합 테스트 (가장 중요): "정확히 1개만 성공하는가"
`@SpringBootTest` + 실제 DB(Testcontainers)에서 N개 스레드를 `CountDownLatch`로 **동시에 출발**시켜 결과로 락 동작을 역추적한다. 테스트 골격(latch 3개·`finally`의 countDown·테스트에 `@Transactional` 금지·셋업 commit)은 정본인 [동시성 테스트 작성법](../test/concurrency-test.md)을 따른다. 락 노트에서 챙길 검증 포인트만:

- **같은 자원(같은 좌석) 동시 요청 → 성공 정확히 1, 나머지 N-1은 업무 예외**("이미 예약됨"). 비관 락 + 합계형이면 최종 합계로 검증(결정적), 낙관 락이면 1성공/N충돌(타이밍 의존).
- **다른 자원(다른 좌석) 동시 요청 → 전부 성공** — 락이 과하게(테이블 단위로) 걸리지 않는지 확인. 조건 컬럼에 인덱스가 없으면 다른 좌석끼리도 서로 기다려(직렬화) 이 테스트가 느려지거나 타임아웃으로 드러난다(§9).
- **락을 빼면 이 테스트가 실패하는지** 한 번 확인해야 테스트가 락을 증명한다.
- Testcontainers라 Docker 실행 필수, 락 대기만큼 실행 시간이 늘어난다.

---

## 5. PESSIMISTIC_WRITE가 모든 SELECT를 막는 것은 아니다

`PESSIMISTIC_WRITE`는 보통 `SELECT ... FOR UPDATE`로 동작한다. 이 락은 다른 트랜잭션의 `UPDATE` / `DELETE` / `SELECT ... FOR UPDATE`(또는 `FOR SHARE`)를 막지만, **MVCC DB(MySQL InnoDB, PostgreSQL)에서는 잠금 없는 일반 `SELECT`는 과거 커밋된 스냅샷을 그대로 읽을 수 있다.**

```
T1: SELECT ... FOR UPDATE  (id=1 배타 락) 🔒
T2: SELECT * FROM ... WHERE id=1           → 스냅샷 읽기 OK (안 막힘) ✅
T2: UPDATE ... WHERE id=1                   → 대기 ⏳
T2: SELECT ... FOR UPDATE WHERE id=1        → 대기 ⏳
```

> 즉 "아무도 읽지 못하게 한다"가 아니라, **"다른 트랜잭션이 같은 row를 수정하거나 락을 잡지 못하게 한다"**에 가깝다. (MVCC 스냅샷 읽기 때문. 예외: MySQL `SERIALIZABLE`은 autocommit이 꺼져 있으면 일반 SELECT를 `FOR SHARE`로 바꾸므로 이때는 막힌다)

### "일반 SELECT 허용"의 정확한 의미 — 안 막히지만 "미커밋 값"은 못 본다

"허용"은 **"대기하지 않고 즉시 읽힌다"**는 뜻이다. 단, 읽히는 값은 **락 잡은 트랜잭션이 수정 중인 최신값이 아니라, 그 직전에 마지막으로 커밋된 스냅샷**이다. (MVCC가 커밋된 과거 버전을 언두 로그로 유지하기 때문)

```
stock=10 인 상품을 T1이 잠그고 차감 중:
T1: SELECT ... FOR UPDATE   (stock=10 읽고 배타 락) 🔒
T1: UPDATE stock = 5        (변경했지만 아직 COMMIT 전!)
T2: SELECT stock ...        → 즉시 읽힘 ✅  but 값은 10 ❗ (커밋 전 스냅샷, 5 아님)
T1: COMMIT                  (이제 stock=5 확정)
T2: SELECT stock ...        → READ COMMITTED면 5 (문장마다 새 스냅샷)
                              REPEATABLE READ(MySQL 기본)면 여전히 10 (첫 읽기 스냅샷 유지)
```

> 일반 SELECT / `FOR SHARE` / `FOR UPDATE` / `UPDATE`가 상대 락에 막히는지 한눈에 보는 표 → [@Lock 심화 §2 호환성 표](./lock-concepts.md).

> 핵심: 배타락은 **"읽기를 막는 락"이 아니라 "쓰기·락 획득을 막는 락"**이다. 따라서 "내가 읽은 값으로 정확히 판단하고 수정"하려면 읽는 쪽도 반드시 `FOR UPDATE`로 읽어 **같은 락 경쟁에 참여**해야 한다. 일반 SELECT로 읽으면 락의 보호를 못 받는다. (→ 비관적 락에서 조회 메서드를 `findById` / `findByIdForUpdate`로 분리하는 이유)

---

## 6. 낙관적 락 재시도는 "새 트랜잭션"에서 해야 한다

흔한 실수: `@Transactional` 안에서 `catch` 후 그대로 재시도. 두 가지 이유로 안 된다.

1. **예외는 보통 메서드 안이 아니라 commit 때 터진다.** 더티 체킹 UPDATE의 version 검증은 **UPDATE가 실제로 나가는 flush 시점**에 일어나고, 중간 flush가 없으면 그건 **메서드 종료 후 프록시가 commit하는 순간**이다. 그래서 본문 안에 `try-catch`를 둬도 **그 catch엔 안 잡힌다**. (본문에서 `em.flush()`·`saveAndFlush()`나 같은 테이블을 조회하는 JPQL이 auto-flush를 일으키면 그 줄에서 터진다 → [영속성 컨텍스트 §5](./persistence-context.md))
2. **잡혀도 rollback-only라 재사용 불가.** 충돌 난 트랜잭션은 이미 **rollback-only**로 마킹돼, 같은 트랜잭션을 이어 쓰면 영속성 컨텍스트가 꼬이거나 커밋이 실패한다.

```java
// ❌ 트랜잭션 안에서 잡아 재시도 — 위 1·2 때문에 동작 안 함
@Transactional
public void decreaseStock(Long id, int qty) {
    try {
        Product p = repo.findById(id).orElseThrow();
        p.decrease(qty);
    } catch (OptimisticLockException e) { /* 안 잡힘 + 트랜잭션 이미 rollback-only */ }
}
```

→ **재시도 단위는 트랜잭션 바깥**이어야 한다. 실패한 트랜잭션을 잇는 게 아니라, **최신 데이터를 다시 읽는 새 트랜잭션**으로 재시도. (즉 "catch 금지"가 아니라 "트랜잭션 안에서 같은 트랜잭션으로 재시도"가 금지. 바깥에서 새 트랜잭션으로 잡는 `@Retryable`은 정상)

```java
@Retryable(
    retryFor = ObjectOptimisticLockingFailureException.class,  // JPA 예외를 Spring이 감싼 타입
    maxAttempts = 3,
    backoff = @Backoff(delay = 100)
)
@Transactional
public void decreaseStock(Long productId, int qty) {
    Product product = productRepository.findById(productId).orElseThrow();
    product.decrease(qty);   // 더티 체킹 → 커밋 시 version 검증
}
```

**주의점**
- 위 코드는 **Spring Retry**(`org.springframework.retry`, `@EnableRetry` 필요) 기준이다. Spring Framework 7부터는 코어에 `@Retryable`(`org.springframework.resilience.annotation`, `@EnableResilientMethods`로 활성화)이 따로 생겼다 — 속성 이름(`maxRetries` 등)이 다르고, `@Transactional`과의 적용 순서는 확인 필요.
- 예외 타입: JPA의 `OptimisticLockException`은 Spring 데이터 환경에서 `ObjectOptimisticLockingFailureException`으로 변환되어 올라온다. `@Retryable`에는 후자(또는 상위 `OptimisticLockingFailureException`)를 잡는 게 안전.
- `@Retryable`이 `@Transactional`보다 **바깥쪽**에서 동작해야(재시도마다 새 트랜잭션) 의미가 있다.
- **self-invocation 금지**: 같은 클래스 내부 메서드 호출은 프록시를 안 타서 재시도/트랜잭션이 적용되지 않는다.

### 그런데 재시도하면 "또 같은 version으로 또 실패"하는 거 아닌가?

**아니다. 재시도는 `UPDATE`만 다시 하는 게 아니라 `findById`(조회)부터 통째로 다시 한다.** 그래서 매 시도마다 **그 시점의 최신 version을 새로 읽어온다.** 같은 version으로 무한 박치기하는 게 아니다.

마지막 재고 2개를 T1, T2가 동시에 차감하는 예:
```
[1차 시도]
T1: findById → version=1 읽음
T2: findById → version=1 읽음          (둘 다 version=1을 봄)
T2: UPDATE ... version=2 WHERE version=1  → 성공 ✅  (DB는 이제 version=2)
T1: UPDATE ... version=2 WHERE version=1  → 0건! ❌  (이미 2라 안 맞음)
    → ObjectOptimisticLockingFailureException → T1 롤백

[2차 시도 — T1이 새 트랜잭션으로 다시]
T1: findById → version=2 읽음           ← 이번엔 "최신값"을 읽음! (T2가 바꾼 결과)
T1: UPDATE ... version=3 WHERE version=2  → 성공 ✅
```

**왜 결국 성공하나?** 충돌은 "두 트랜잭션이 겹친 그 찰나"에만 일어난다. 재시도 시점에는 앞선 T2가 **이미 커밋을 끝낸 뒤**라, 다시 읽으면 최신 version을 가져오고 그걸로 `UPDATE`하면 조건이 맞아 통과한다.

**그럼 무한히 실패할 수도 있나?** 두 경우를 구분해야 한다.

| 상황 | 재시도하면 |
|------|-----------|
| **version 충돌** (동시 접근이 찰나에 겹침) | 다시 읽으면 최신 version → 보통 1~2회 내 성공 |
| **비즈니스 실패** (예: 재고가 진짜 0) | 재시도해도 계속 실패 → 이건 version 문제가 아니라 "재고 부족"으로 정상 처리해야 함 |

- 충돌이 **너무 잦으면**(같은 row에 수천 명 동시) `maxAttempts`를 다 쓰고도 실패할 수 있다.
- 그래서 **낙관적 락은 "충돌이 드문 경우"에 적합**하고, 충돌이 빈번하면 비관적 락(처음부터 잠그고 줄 세우기)이 낫다.

### 그런데 retry가 항상 답은 아니다 — "자동 재시도 vs 사용자 통지"

| 상황 | 충돌 시 대응 |
|------|-------------|
| **서버 내 read-modify-write** (예: 재고 차감) | **자동 retry** — 다시 읽어 재계산해도 결과가 같으니 안전 |
| **2-요청 편집** (GET으로 폼 → POST로 저장) | **사용자에게 통지** — 사용자가 **낡은 화면 값** 기준으로 입력했으므로 몰래 retry하면 안 됨. "다른 사용자가 먼저 수정했습니다. 새로고침 후 다시 시도". ⚠️ 이 충돌은 GET에서 받은 version을 클라이언트가 들고 와 비교해야 감지된다(POST에서 새로 읽은 version끼리는 항상 일치) → [Optimistic Offline Lock](../concurrency/locks.md) |

> retry는 "기계가 다시 읽어 자동 처리해도 안전할 때"만. 사람의 판단이 낡은 값에 묶여 있으면 retry가 아니라 **충돌 통지**가 맞다.
>
> ⚠️ **부수효과 주의**: 이메일 발송·외부 API 호출 등이 트랜잭션 안에 있으면 retry 시 **중복 실행**될 수 있다(멱등성 확인 필요).

> 참고 — `@Version`을 일반 도메인에 기본 장착하는 건 lost update 안전망으로 합리적이나, **붙이는 순간 동시 수정이 "조용한 덮어쓰기"가 아니라 예외가 된다 → 반드시 위 둘 중 하나로 처리**해야 한다(안 하면 동시성 상황에서 500). (`@Version` 기본 동작은 [@Lock 심화 개념 §1](./lock-concepts.md))

---

## 7. 벌크 UPDATE는 @Version을 우회한다

JPQL `@Modifying` bulk update는 **엔티티를 로딩하지 않고 DB에 직접** 쿼리를 날린다. 영속성 컨텍스트·더티 체킹을 거치지 않으므로 **`@Version` 자동 증가·검증이 일어나지 않는다.**

```java
// ❌ version 검증/증가 안 됨 — 낙관적 락이 무력화됨
@Modifying
@Query("UPDATE Product p SET p.stock = p.stock - :qty WHERE p.id = :id")
int decreaseStock(@Param("id") Long id, @Param("qty") int qty);
```

낙관적 락을 유지하려면 **version을 직접 SET하고 WHERE에 넣어야** 한다.

```java
@Modifying(clearAutomatically = true)  // bulk 후 1차 캐시가 stale → 비우기
@Query("""
    UPDATE Product p
       SET p.stock = p.stock - :qty,
           p.version = p.version + 1
     WHERE p.id = :id
       AND p.version = :version
""")
int decreaseStockWithVersion(@Param("id") Long id,
                             @Param("qty") int qty,
                             @Param("version") Long version);
// 반환값(영향 row 수)이 0이면 충돌 → 직접 예외 처리
```

---

## 8. 락 대신 "조건부 UPDATE"도 좋은 선택

단순 재고 차감처럼 "읽고 복잡한 판단"이 필요 없다면, 락 없이 **원자적 조건부 UPDATE** 한 방으로 끝낼 수 있다.

```sql
UPDATE product
   SET stock = stock - 1
 WHERE id = ?
   AND stock >= 1;   -- 재고 있을 때만 차감
```

- **영향받은 row 수 = 1** → 성공, **= 0** → 재고 부족
- `SELECT ... FOR UPDATE`보다 **락 보유 시간이 짧고** 코드도 간결
- 엄밀히는 락이 아니라 **DB의 원자적 연산**을 이용하는 방식 (낙관적/비관적 락과 구분되는 제3의 선택지)

> 정리: 단순 차감 = 조건부 UPDATE, 읽어서 복잡한 검증·여러 작업이 필요 = 비관적 락.

---

## 9. 비관적 락은 인덱스가 중요하다

비관적 락은 "어떤 row를 찾는가"가 곧 "어디에 락을 거는가"다. InnoDB는 **인덱스 레코드에 락**을 거는데, 조건 컬럼에 인덱스가 없으면 DB가 많은 row를 풀스캔하면서 **스캔한 모든 row에 락**이 걸려 사실상 테이블 락처럼 번질 수 있다.

```java
// user_id 에 인덱스가 없으면, 이 FOR UPDATE가 예상보다 넓은 범위를 잠글 수 있다
Optional<UserPoint> findByUserIdForUpdate(@Param("userId") Long userId);
```

> `FOR UPDATE`를 쓰는 조회 조건(WHERE 컬럼)에는 **인덱스가 있는지 확인하는 습관**이 필요하다. 인덱스가 없으면 락 경합과 성능 문제가 급격히 커진다.

---

## 10. 💡 판단 기준

- **차감 한 줄로 끝나면 조건부 UPDATE, 읽고 여러 검증을 거쳐야 하면 `FOR UPDATE`** (§8). 재고 1 차감은 `WHERE stock >= 1` 한 방, "회원 등급·쿠폰·한도를 보고 차감"은 락 조회 후 엔티티 메서드.
- **충돌 시 자동 재시도는 "기계가 다시 읽어 다시 계산해도 같은 의도"일 때만**, 사람이 낡은 화면을 보고 입력한 경우는 통지 (§6). 재시도 경로에 이메일·외부 API가 있으면 멱등성부터.
- **락 조회와 일반 조회는 메서드를 나누고, 수정할 엔티티는 처음부터 락 메서드로 읽는다** (§3). 나중에 락으로 다시 읽어도 1차 캐시의 옛 상태는 안 바뀐다.
- **`FOR UPDATE`를 붙였으면 SQL 로그로 잠기는 범위(조인 테이블·인덱스)를 확인한다** (§1, §9). "다른 좌석 동시 요청이 기다림 없이 둘 다 성공"하는 테스트가 그 확인을 대신한다 (§4).

---

## 참고
- [Spring Data JPA - Locking 공식 문서](https://docs.spring.io/spring-data/jpa/reference/jpa/locking.html)
- [MySQL 8.0 - Consistent Nonlocking Reads (RR은 첫 읽기 스냅샷 유지)](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html)
- [MySQL 8.0 - Transaction Isolation Levels (SERIALIZABLE은 일반 SELECT를 FOR SHARE로)](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
- [Spring Framework 7 - Resilience Features (`@Retryable`)](https://docs.spring.io/spring-framework/reference/core/resilience.html)
- 관련 노트: [@Lock 기본](./lock.md) · [@Lock 심화 개념](./lock-concepts.md) · [영속성 컨텍스트](./persistence-context.md) · [데드락](../concurrency/deadlock.md) · [동시성 테스트 작성법](../test/concurrency-test.md)

---

**학습 날짜**: 2026-05-25 (2026-05-26 lock.md에서 실무 패턴만 분리, 2026-10-02 사실 보정·§4 요약화·💡 추가)

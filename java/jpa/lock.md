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
| **충돌 시 동작** | **대기(blocking)**, 타임아웃 시 예외 | **즉시 예외**(`OptimisticLockException`), 대기 X |
| 해결 방법 | DB가 순서대로 처리 (재시도 불필요) | 애플리케이션에서 **재시도(retry)** |
| 비용 | DB 락 보유 → 성능 저하, 데드락 가능 | 충돌 시 재시도 비용 |
| 적합한 경우 | 충돌 빈도 높음, 정확성 최우선 | 충돌 빈도 낮음, 처리량 중요 |
| 예시 | 포인트 차감, 재고 | 게시글 수정, 프로필 변경 |

> **version이 틀리면 대기? 에러?** → 낙관적 락은 락을 실제로 걸지 않고 커밋 시점에 version만 비교한다. 안 맞으면 **기다리지 않고 즉시 `OptimisticLockException`** → 롤백 → 애플리케이션에서 재시도해야 함. 반면 비관적 락은 DB가 row를 잠그므로 다른 트랜잭션이 **대기**하고, 락 해제 시 순서대로 진행된다 (무한 대기 방지용 타임아웃 초과 시에만 `LockTimeoutException`).

```java
// 낙관적 락 재시도 예시
int retry = 0;
while (true) {
    try {
        return pointService.charge(userId, amount);
    } catch (OptimisticLockException e) {
        if (++retry >= 3) throw e;
        // 다시 읽어서 최신 version으로 재시도 (스프링은 @Retryable로도 처리)
    }
}
```

---

## 2-1. 심화: 헷갈리는 개념 정리

### (1) `@Version`만 vs `@Lock(OPTIMISTIC)` — 차이가 뭔가?

핵심: **`@Version`은 낙관적 락의 "기반 도구(필수)"**, **`@Lock`은 그 락을 "언제/어떻게 발동시킬지" 지정**하는 것.

**`@Version`만 있을 때 (자동 동작)**

`@Version`을 붙이면 엔티티를 **수정(UPDATE)할 때만** 자동으로 version 체크가 일어난다.
```sql
UPDATE product SET stock = 9, version = 2
WHERE id = 1 AND version = 1;
-- version이 1이 아니면 (누가 먼저 바꿨으면) 0건 업데이트 → OptimisticLockException
```
즉 수정 시에는 별도 `@Lock` 없이도 낙관적 락이 걸린다.

**그럼 `@Lock(OPTIMISTIC)`은 왜 필요한가? → "읽기만 하는 경우" 때문**

데이터를 **읽기만 하고 수정 안 하면** UPDATE 쿼리가 안 나가니 version 체크도 안 일어난다.
```java
// 주문 검증: 상품을 읽어서 재고 충분한지 확인만 함 (상품 자체는 수정 안 함)
Product product = em.find(Product.class, 1L);
if (product.getStock() < orderQty) throw ...;
// 이 트랜잭션 동안 다른 사람이 stock을 바꿔도 나는 모름
```
이때 `@Lock(LockModeType.OPTIMISTIC)`을 쓰면 **읽기만 했어도 커밋 시점에 version이 그대로인지 검증**한다.
```sql
-- 커밋 직전에 강제로 한 번 확인, 처음 읽은 version과 다르면 예외
SELECT version FROM product WHERE id = 1;
```

| | `@Version`만 | `@Lock(OPTIMISTIC)` 추가 |
|---|---|---|
| 수정(UPDATE)할 때 | version 체크 됨 | version 체크 됨 |
| **읽기만 할 때** | **체크 안 됨** | **커밋 시점에 체크 됨** |

> 요약: `@Version`은 락의 전제 조건(필수). `@Lock(OPTIMISTIC)`은 "읽기만 하는 데이터도 트랜잭션 끝까지 안 변했는지 보장"하고 싶을 때 추가로 명시.

**그럼 `@Version` 없이 `@Lock(OPTIMISTIC)`만 쓰면? → 예외 발생 (동작 자체 불가)**

낙관적 락은 version 컬럼을 비교하는 게 본질이라, 비교할 version이 없으면 락이 성립하지 않는다. JPA는 무시하고 넘어가는 게 아니라 **예외를 던진다.**
```java
@Entity
public class Product {
    @Id private Long id;
    private int stock;
    // @Version 없음!
}

@Lock(LockModeType.OPTIMISTIC)  // version 없는데 낙관적 락 → 예외
Optional<Product> findById(Long id);
```
```
PersistenceException / IllegalArgumentException
  → "Entity has no version attribute" 류 (구현체·버전마다 메시지 다름)
```

**비관적 락과의 결정적 차이**

| | 무엇으로 락을 거나 | `@Version` 필요? |
|---|---|---|
| 낙관적 락 (`OPTIMISTIC`) | 엔티티의 version 컬럼 비교 | **필수** (없으면 예외) |
| 비관적 락 (`PESSIMISTIC_WRITE` 등) | DB의 물리적 락 (`FOR UPDATE`) | 불필요 |

- **낙관적 락**: 충돌 감지 기준이 version밖에 없음 → 없으면 동작 불가능
- **비관적 락**: DB가 직접 row를 잠그므로 애플리케이션 레벨 값이 불필요 → `@Version` 없이도 동작

> 즉 `@Version`은 "있으면 좋은 것"이 아니라, **없으면 낙관적 락 자체가 불가능한 필수 전제**. 반면 비관적 락은 `@Version` 없이도 DB 락으로 동작한다.

**반대로 `@Version`만 쓰고 `@Lock`은 안 써도 되나? → 된다. 오히려 가장 일반적인 패턴.**

`@Version`만 붙여도 **수정(UPDATE) 시 자동으로 낙관적 락이 동작**한다. `@Lock(OPTIMISTIC)`은 "읽기만 하는 값도 검증"하고 싶은 특수한 경우에만 추가로 쓰는 보조 장치일 뿐.

```java
@Transactional
public void decreaseStock(Long id, int qty) {
    Product p = repository.findById(id).orElseThrow();  // @Lock 없음
    p.setStock(p.getStock() - qty);                     // 수정함
    // 커밋 시 자동: UPDATE ... WHERE id=? AND version=1 → 안 맞으면 OptimisticLockException
}
```

**세 경우 비교**

| 조합 | 결과 |
|---|---|
| `@Version`만 | ✅ 정상 — 수정 시 자동 낙관적 락 (**가장 흔한 패턴**) |
| `@Version` + `@Lock(OPTIMISTIC)` | ✅ 정상 — 읽기 검증까지 확장 |
| `@Lock(OPTIMISTIC)`만 (Version 없음) | ❌ 예외 — 비교할 version이 없음 |

> `@Version`이 주인공, `@Lock(OPTIMISTIC)`은 읽기 전용 보조. `@Version`만 써도 의미 있고(수정 시 동작), 오히려 `@Lock`만 쓰는 게 불가능한 조합이다.

### (2) 공유 락(Shared) vs 배타 락(Exclusive)

비관적 락(DB 레벨 락) 얘기.

**공유 락 (`PESSIMISTIC_READ`, `SELECT ... FOR SHARE`)**
> "여러 명이 같이 **읽는 건** OK. 단, 아무도 **수정은 못 함**"
- 여러 트랜잭션이 동시에 공유 락을 가질 수 있음 (읽기끼리 공존)
- 쓰기(배타 락)는 차단됨
- 용도: "내가 읽는 동안 이 값이 바뀌면 안 돼. 근데 남이 같이 읽는 건 괜찮아"

**배타 락 (`PESSIMISTIC_WRITE`, `SELECT ... FOR UPDATE`)**
> "내가 잡으면 **나만 접근 가능**. 다른 사람은 읽기도 쓰기도 대기"
- 단 하나의 트랜잭션만 획득 가능
- 다른 트랜잭션의 읽기/쓰기 모두 차단
- 용도: "내가 이거 수정할 거니까 아무도 건드리지 마" (재고 차감 등 경쟁 상황)

**호환성 표**

| | 상대가 공유 락 | 상대가 배타 락 |
|---|---|---|
| **공유 락 요청** | ✅ 가능 | ⛔ 대기 |
| **배타 락 요청** | ⛔ 대기 | ⛔ 대기 |

> 한 줄 요약: 공유 락 = "같이 읽기 OK, 쓰기 금지", 배타 락 = "나만, 완전 독점". 재고·포인트 차감 같은 동시성 처리는 보통 `PESSIMISTIC_WRITE`(배타)를 쓴다.

### (3) `OPTIMISTIC` vs `OPTIMISTIC_FORCE_INCREMENT`

둘 다 낙관적 락. 차이는 **version을 증가시키냐 마냐**.

- **`OPTIMISTIC`**: 읽은 엔티티가 안 바뀌었는지 **확인만** 함. 내가 수정 안 했으면 version 그대로.
- **`OPTIMISTIC_FORCE_INCREMENT`**: 그 엔티티 자체를 수정하지 않고 **읽기만 해도 version을 강제로 +1** 한다. ← 이게 핵심 포인트.

**읽기만 했을 때 두 모드의 차이**

| | 읽기만 했을 때 (수정 X) | 커밋 시 실제 SQL |
|---|---|---|
| `OPTIMISTIC` | version **체크만** (그대로 유지) | `SELECT version ...` (확인) |
| `OPTIMISTIC_FORCE_INCREMENT` | version **강제 +1** | `UPDATE ... SET version = version + 1` |

**왜 강제 증가가 필요한가? → "연관된 자식이 바뀌면 부모의 버전도 올리고 싶을 때"**

대표 예시: 게시글(부모) - 댓글(자식)
```java
// 댓글을 추가하면 게시글 컬럼 자체는 안 바뀌지만, 게시글의 상태는 논리적으로 바뀐 것
Post post = em.find(Post.class, 1L, LockModeType.OPTIMISTIC_FORCE_INCREMENT);
post.addComment(new Comment("새 댓글"));  // post 테이블 컬럼은 안 바뀜
```
```sql
INSERT INTO comment ...;
UPDATE post SET version = version + 1 WHERE id = 1 AND version = 1;  -- 강제 증가
```
→ "게시글에 댓글이 달리는 동안 다른 사람이 게시글을 동시에 건드리는" 충돌을 막을 수 있다. **자식의 변경을 부모 버전에 반영**(집합체 일관성 보장).

| | `OPTIMISTIC` | `OPTIMISTIC_FORCE_INCREMENT` |
|---|---|---|
| 엔티티 안 바뀌면 | version 유지 | **version 강제 +1** |
| 용도 | 읽은 값 변경 감지 | 연관 객체 변경 시 부모 버전도 올림 |

> `PESSIMISTIC_FORCE_INCREMENT`는 **배타 락 + version 강제 증가**를 합친 것 (DB 락으로 독점하면서 동시에 version도 올림).

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

## 3-1. `@Lock`은 어디서 동작하나 (Spring Data JPA 프록시)

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

> **`@Lock` + `JOIN FETCH` 주의**: 락은 주 엔티티에만 걸린다. fetch join으로 가져온 연관 엔티티에는 락이 안 걸리므로, 자식에도 락이 필요하면 별도 처리해야 함.

---

## 3-2. 락 메서드 네이밍 컨벤션

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

## 3-3. 락 메서드를 분리해야 하나? (조회용 vs 수정용)

"모든 조회에 락을 걸면 성능 저하" → 락 조회와 일반 조회를 **별도 메서드로 분리**하는 게 정석.

### 왜 분리하나
Spring Data JPA에서 `@Lock`은 **메서드 레벨**에 붙는다. 같은 메서드를 호출하면서 "이번엔 락 걸고, 다음엔 빼고"처럼 **조건부 제어가 불가능**하다. 그래서 분리가 강제된다.

```java
public interface SeatRepository {
    Optional<SeatEntity> findById(String id);            // 조회용 (락 X)
    Optional<SeatEntity> findByIdForUpdate(String id);   // 수정용 (락 O)
}
```

### 트랜잭션 안에서는 1차 캐시 활용
같은 트랜잭션 안에서 한 번 락 걸고 조회한 엔티티는 **1차 캐시**에 있으므로, 이후 다시 조회해도 DB를 안 친다. 즉 **락은 처음 한 번만** 걸면 된다.

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

## 3-4. 락 테스트 작성법

동시성 문제는 **여러 스레드가 실제로 동시에 접근할 때만** 드러나므로, 테스트도 그 상황을 시뮬레이션해야 한다. 3개 레벨로 나뉜다.

### 레벨 1 — 단위 테스트 (Mock): "올바른 메서드를 부르는가"
락 동작 자체가 아니라 **흐름**만 검증. 빠르고 안정적.
```java
verify(seatService, times(1)).getSeatExclusive("1");  // 락 메서드 호출 확인
verify(seatService, never()).getSeat(any());          // 일반 조회 안 썼는지
```

### 레벨 2·3 — 동시성 통합 테스트 (가장 중요): "정확히 1개만 성공하는가"
`ExecutorService` + `CountDownLatch`로 N개 스레드를 **동시에 출발**시켜, 같은 좌석 예약 시 하나만 성공하는지 검증. (실제 DB 필요 → Testcontainers)

```java
@SpringBootTest
@Import(TestcontainersConfiguration.class)
class ReservationConcurrencyTest {
    @Test
    void 동시_예약_시_한_명만_성공() throws InterruptedException {
        int threadCount = 10;
        ExecutorService pool = Executors.newFixedThreadPool(threadCount);
        CountDownLatch startLatch = new CountDownLatch(1);          // 동시 출발 신호
        CountDownLatch endLatch = new CountDownLatch(threadCount);  // 전원 종료 대기
        AtomicInteger success = new AtomicInteger();
        AtomicInteger fail = new AtomicInteger();

        for (int i = 0; i < threadCount; i++) {
            pool.submit(() -> {
                try {
                    startLatch.await();   // 모든 스레드가 여기서 대기하다 동시에 출발
                    reservationUseCase.execute(command);
                    success.incrementAndGet();
                } catch (BusinessException e) {
                    fail.incrementAndGet();   // 이미 예약됨
                } finally {
                    endLatch.countDown();
                }
            });
        }
        Thread.sleep(100);          // 전 스레드가 대기 상태 들어갈 시간
        startLatch.countDown();     // 동시 출발!
        endLatch.await(30, TimeUnit.SECONDS);

        assertThat(success.get()).isEqualTo(1);                 // 딱 1명 성공
        assertThat(fail.get()).isEqualTo(threadCount - 1);      // 나머지 실패
    }
}
```

**핵심 포인트**
- `CountDownLatch startLatch`로 모든 스레드를 **같은 순간 출발**시켜야 진짜 동시성 재현
- 락이 없으면 이 테스트는 success가 2 이상 나오며 **실패**한다 (→ 락이 필요함을 증명)
- 서로 다른 좌석이면 둘 다 성공해야 정상 (락이 과하게 안 걸리는지 확인)
- Testcontainers라 **Docker 실행 필수**, 락 대기로 실행 시간이 길어질 수 있음

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

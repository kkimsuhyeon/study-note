# 영속성 컨텍스트 · flush · 더티 체킹 — "커밋 시점"의 정체

> **한 줄 요약**: JPA는 `setter`로 값을 바꿔도 즉시 `UPDATE`를 날리지 않고, 변경을 **영속성 컨텍스트**에 모아뒀다가 **flush**(보통 트랜잭션 commit 직전) 때 한꺼번에 SQL로 내보낸다. 락 문서에서 자주 나오는 "**커밋 시점에 version 체크**"는 바로 이 flush 동작 때문이다.

관련 노트: [JPA @Lock](./lock.md) · [락 개념 종합](../concurrency/locks.md) · [@Transactional](../spring/transactional.md) · [준영속 수정: merge 함정](./merge-vs-dirty-checking.md) · [키 생성 전략(IDENTITY는 쓰기 지연 무력화)](./id-generation.md)

---

## 0. 왜 이걸 알아야 하나

낙관적 락 문서를 읽다 보면 이런 문장이 나온다:

> "낙관적 락은 **커밋 시점에** version을 비교한다. 조회 시점엔 안 한다."

여기서 막힌다면, 원인은 락이 아니라 그 아래 깔린 **JPA의 동작 방식(영속성 컨텍스트)** 을 모르기 때문이다. 이걸 잡으면 락·더티체킹·`OptimisticLockException`이 왜 그 타이밍에 터지는지 한 번에 풀린다.

---

## 1. 영속성 컨텍스트 (Persistence Context)

> 엔티티를 보관·관리하는 **JPA의 1차 캐시 같은 메모리 공간.** 트랜잭션 동안 "관리 대상(영속 상태)" 엔티티들이 여기 올라가 있다.

- 생명주기 ≈ **트랜잭션 범위** (보통 `@Transactional` 메서드 시작~끝). 단 스프링 부트 기본값인 OSIV가 켜져 있으면 **요청이 끝날 때까지** 산다 → [OSIV](./osiv.md).
- `find`/JPQL로 조회하면 엔티티가 여기에 **올라가면서 그 순간의 값을 스냅샷으로 같이 저장**한다.
- 한 번 올라온 엔티티는 **JPA가 계속 감시**한다.

> ⚠️ **1차 캐시가 SQL을 아껴 주는 건 id 조회(`em.find`/`findById`)뿐이다.** JPQL·파생 쿼리는 **항상 SQL을 실행**하고, 결과 행 중 이미 영속성 컨텍스트에 있는 엔티티는 **DB에서 읽은 값을 버리고 기존 인스턴스를 돌려준다**(같은 트랜잭션 안 `==` 동일성 보장). 그래서 트랜잭션 도중 남이 커밋한 최신 값이 필요하면 쿼리를 다시 날리는 게 아니라 `em.refresh(entity)`나 처음부터 락 조회가 필요하다 ([@Lock 실무 패턴 §3](./lock-practical.md)).

```
[엔티티 상태]
비영속(new)  →  영속(managed)  →  준영속(detached) / 삭제(removed)
   new            em.persist /        em.detach /
                  find로 조회          트랜잭션 종료
```

핵심: **영속 상태인 동안에만** 더티 체킹·쓰기 지연·version 체크가 작동한다.

---

## 2. 쓰기 지연 (Write-Behind) — UPDATE는 바로 안 나간다

JPA에서 가장 직관에 어긋나는 부분.

```java
@Transactional
void charge(Long id) {
    Product p = em.find(Product.class, id);  // ① SELECT 1번 나감, version=1 기억
    p.setStock(9);                           // ② 자바 객체만 바뀜. DB엔 아무 일 없음!
}                                            // ③ 메서드 끝 → commit → 이제서야 UPDATE 발사
```

- `setStock(9)`는 그냥 **자바 객체의 필드를 바꾼 것**일 뿐, `UPDATE` SQL이 나가지 않는다. 이 순간엔 **아무것도 예약되지 않는다** — UPDATE는 flush 때 더티 체킹이 스냅샷과 비교해 **그때 만든다**(§4).
- 반면 `em.persist()`/`em.remove()`는 호출 시점에 INSERT/DELETE 작업이 영속성 컨텍스트의 **쓰기 지연 저장소**(Hibernate의 ActionQueue)에 쌓인다 (IDENTITY 전략 INSERT는 예외 → [키 생성 전략](./id-generation.md)).
- 모아둔 SQL은 **flush 시점에** 한꺼번에 DB로 나간다. (단, `SELECT`는 미뤄지지 않고 호출 즉시 나간다 — `FOR UPDATE` 락 조회 포함)
- **`@Query`로 직접 쓴 쿼리는 쓰기 지연 대상이 아니다.** `@Modifying` 벌크 UPDATE·DELETE·네이티브 INSERT는 호출하는 순간 `executeUpdate()`로 바로 나간다(직전에 flush 한 번). 쓰기 지연은 "엔티티 객체를 통한 변경"에만 적용되는 장치이고, 직접 쓴 쿼리는 영속성 컨텍스트를 거치지 않기 때문이다. 결과(영향받은 행 수)를 지금 돌려줘야 하니 미룰 수도 없다.
- 헷갈리지 말 것: **"SQL이 DB에 도착하는 시점"과 "확정(커밋)되는 시점"은 다르다.** 즉시 나간 네이티브 INSERT도, 나중에 flush된 UPDATE도 **커밋은 트랜잭션 끝에 같이** 된다. 롤백되면 둘 다 사라지고, 커밋 전에는 둘 다 다른 트랜잭션에 보이지 않는다.

| 코드 | SQL이 나가는 시점 | 확정 |
| --- | --- | --- |
| 엔티티 필드 변경(더티 체킹) | flush 때 | 커밋 때 |
| `persist`·`save`(새 엔티티) | flush 때 (IDENTITY 키 전략이면 즉시) | 커밋 때 |
| `@Query` 조회(JPQL·네이티브) | 호출 즉시 (직전 자동 flush) | — |
| `@Modifying @Query` 변경 | 호출 즉시 (직전 flush) | 커밋 때 |

> 그래서 `save()`를 명시적으로 안 불러도 값이 반영된다(= dirty checking). 반대로, UPDATE가 "언제" 나가는지는 내 코드 줄이 아니라 **flush 타이밍**이 결정한다.

---

## 3. flush — "모아둔 SQL을 DB로 내보내는" 순간

flush = 영속성 컨텍스트의 변경 내용을 DB에 동기화(SQL 발사). **commit과는 다른 개념**이지만 보통 같이 일어난다.

**flush가 일어나는 시점 (3가지)**
1. **트랜잭션 commit 직전** ← 가장 흔함. "커밋 시점"이라는 말의 정체.
2. **JPQL/HQL 실행 직전** (조회 결과 정합성을 위해, AUTO 모드 기본) — 단 Hibernate는 그 쿼리가 **대기 중인 변경과 테이블이 겹칠 때만** flush한다. 무관한 테이블 조회면 flush하지 않는다. 네이티브 SQL은 동기화 대상(쿼리 공간)을 등록하지 않았으면 항상 flush.
3. **`em.flush()` 직접 호출**

```
flush  = SQL을 DB로 보냄 (트랜잭션은 아직 안 끝남, 롤백 가능)
commit = 트랜잭션을 확정 (flush 포함 → 되돌릴 수 없음)
```

> "커밋 시점에 version 체크"를 더 정확히 말하면 **"commit이 트리거하는 flush 시점에, 그때 발사되는 UPDATE SQL 안에서"** 체크된다.

### flush는 여러 번, commit은 딱 1번 (헷갈림 주의)

| | 한 트랜잭션 내 횟수 | 의미 |
|---|---|---|
| **flush** | **여러 번 가능** | 모아둔 SQL을 중간중간 DB로 내보냄 (아직 롤백 가능) |
| **commit** | **딱 1번** (맨 끝) | 트랜잭션 최종 확정 (되돌릴 수 없음) |

- **스프링 `@Transactional`의 commit = DB의 commit = 여기서 말하는 commit** — 다른 게 아니라 같은 것. 선언적 트랜잭션은 결국 commit/rollback 한 번을 호출하는 추상화 껍데기다.
- **한 트랜잭션은 commit(또는 rollback) 1회로 끝난다.** 한 트랜잭션 안에 commit이 여러 번 있는 게 아니다 — 여러 번인 건 flush다.
- 예외처럼 보이는 `REQUIRES_NEW`(전파)는 **별도 트랜잭션**이 새로 생기는 것이라, "한 트랜잭션 안 여러 commit"이 아니라 "트랜잭션이 여러 개(각자 commit 1번)"인 경우다.

### ⚠️ flush 전이면 누가 옛값을 읽지 않나?

엔티티 값을 바꾼 뒤 flush 전까지 DB에는 옛값이 있다. 그래도 문제가 되지 않는 이유는 **읽는 쪽이 누구냐**에 따라 다르다.

| 읽는 쪽 | 무엇을 보나 | 왜 안전한가 |
| --- | --- | --- |
| 같은 트랜잭션의 `em.find`·`findById` | 1차 캐시의 **메모리 객체** | DB까지 가지 않고 바뀐 값 그대로를 돌려준다 |
| 같은 트랜잭션의 JPQL | flush 후의 DB | 실행 직전 자동 flush(테이블이 겹칠 때) — 결과도 1차 캐시의 객체로 맞춰진다 |
| 같은 트랜잭션의 네이티브 SQL | flush 후의 DB | 실행 직전 전체 flush(쿼리 공간 미등록 시). `@Modifying(flushAutomatically = true)`로 명시하기도 한다 |
| **다른 트랜잭션** | 커밋된 값만 | flush돼도 **커밋 전이면 안 보인다**(READ COMMITTED). 같은 행을 `FOR UPDATE`로 읽으려 하면 내 커밋까지 **기다렸다가** 최신 커밋값을 읽는다 |

정리하면 "flush 전이라 옛값을 읽는" 일은 같은 트랜잭션에서는 위 장치들 때문에 생기지 않고, 다른 트랜잭션은 원래 커밋된 값만 본다. 다른 트랜잭션이 **락 없이** 읽고 그 값으로 판단·쓰기를 하면 그건 flush가 아니라 [Read-Modify-Write](./read-modify-write.md) 경쟁의 문제다.

---

## 4. 더티 체킹 (Dirty Checking, 변경 감지)

> flush 시점에 JPA가 **"조회할 때 찍어둔 스냅샷"과 "지금 엔티티의 값"을 비교**해, 바뀐 게 있으면 **자동으로 `UPDATE` SQL을 생성**하는 기능.

```
조회 시점:  product = {stock:10, version:1}  ← 스냅샷 저장
수정:       product.setStock(9)              ← 현재 {stock:9, version:1}
flush 시점: 스냅샷(10) ≠ 현재(9)  → 변경 감지 → UPDATE 생성
```

- 그래서 `em.update()` 같은 메서드가 없다. **값만 바꾸면 알아서 반영**된다.
- 단, 영속 상태여야 함. 준영속(detached) 엔티티는 더티 체킹 대상이 아니다.

---

## 5. 그래서 @Version 체크는 어디서? (락 문서와의 연결고리)

version 검증은 **별도 동작이 아니라**, 더티 체킹이 만든 `UPDATE`의 `WHERE`에 **얹혀서 함께 나간다.**

```sql
-- flush 시점에 생성·발사되는 UPDATE
UPDATE product SET stock = 9, version = 2
WHERE id = 1 AND version = 1;   -- ← 조회 때 기억해둔 version
-- DB의 version이 이미 2면 → 매칭 0건 → OptimisticLockException
```

| 단계 | 무슨 일 |
|------|---------|
| 조회 시점 | version 값을 **읽어서 스냅샷에 저장만** 함 (검증 X) |
| flush(commit) 시점 | 더티 체킹이 UPDATE 생성 → `WHERE version=...` 얹음 → 발사 → 0건이면 예외 |

| 기능 | 역할 | 타이밍 |
|------|------|--------|
| 더티 체킹 | 바뀐 걸 찾아 **UPDATE를 만든다** | flush |
| version 체크 | 그 UPDATE의 `WHERE`로 **충돌을 감지한다** | flush (같이) |

> 둘은 **같은 flush 타이밍에 한 묶음**으로 일어난다. 그래서 "수정할 때(=UPDATE 나갈 때) version 체크한다"는 말이 맞다.

**자주 당하는 함정**: `OptimisticLockException`은 `setter`를 친 줄이 아니라 **트랜잭션이 끝나는 commit 시점**에 터진다. try-catch를 setter 주변에 둬봐야 안 잡힌다 — 트랜잭션 경계(서비스 호출부)에서 잡아야 한다.

### 더 정확히: 에러 기준은 "커밋"이 아니라 "flush"

version 체크는 **flush마다** 일어난다. flush는 commit 때만이 아니라 그 전에도 발생할 수 있으므로, `OptimisticLockException`은 **커밋 전에도, 커밋 단계에서도** 날 수 있다.

```java
@Transactional
void x() {
    Product p = em.find(Product.class, 1L);   // version=1 기억
    p.setStock(9);                             // UPDATE 아직 안 나감

    em.createQuery("select p from Product p where ...").getResultList();
    //  ↑ Product 테이블과 겹치는 JPQL → 실행 직전 auto-flush → 모아둔 UPDATE 먼저 발사
    //    WHERE version=1 인데 DB가 이미 2 → 0건 → 여기서 예외 💥 (commit 전)
}                                              // commit 때 flush되면 → 그때 예외
```

> ⚠️ 위 쿼리가 `select o from Order o`처럼 **무관한 테이블**이면 auto-flush가 일어나지 않아, 예외는 commit 때 난다 (Hibernate User Guide의 AUTO flush 예제).

| flush 시점 | 에러가 나는 위치 |
|------------|------------------|
| JPQL/쿼리 실행 직전(auto-flush) | **커밋 전** (트랜잭션 중간) |
| `em.flush()` 직접 호출 | **커밋 전** |
| 트랜잭션 commit 직전 | **커밋 단계** |

> 메커니즘은 셋 다 동일(UPDATE의 `WHERE version=...`이 0건). **타이밍만 다르다.** 그래서 "버전 충돌은 커밋 때 난다"보다 "**flush 때 난다**"가 더 정확하다.

---

## 6. 트랜잭션과의 관계 (한눈에)

```
@Transactional 시작
 │  영속성 컨텍스트 생성
 ├─ find/JPQL 조회   → 엔티티 영속화 + 스냅샷 저장 (version 기억)
 ├─ setter로 값 변경 → 자바 객체만 변경 (아무것도 예약 안 됨 — flush 때 감지)
 │
 └─ 메서드 정상 종료 → COMMIT
        └─ flush (이 순간!)
             ├─ 더티 체킹: 스냅샷 vs 현재 비교 → UPDATE 생성
             ├─ version 조건 얹어 UPDATE 발사
             ├─ 0건이면 OptimisticLockException → 롤백
             └─ DB 트랜잭션 확정
```

- 영속성 컨텍스트 = **트랜잭션 단위로 생성·소멸** (OSIV off 기준. OSIV on이면 요청 단위로 살고 트랜잭션은 그 안에서 열고 닫힌다 → [OSIV](./osiv.md)).
- 롤백되면 모아둔 SQL은 안 나간 셈(또는 되돌림). flush 했어도 commit 전이면 롤백 가능.
- **트랜잭션을 짧게 유지**하라는 락 조언도 결국 이것 때문 — 영속성 컨텍스트가 길게 열려 있으면 락·충돌 구간이 길어진다.

---

## 6-1. flush는 JPA(ORM) 고유 — MyBatis/JDBC와 비교

`flush`·더티 체킹·쓰기 지연은 **영속성 컨텍스트를 가진 ORM(JPA/Hibernate)** 에만 있는 개념이다. **MyBatis엔 영속성 컨텍스트가 없다.**

- MyBatis: `mapper.update(...)`를 호출하면 SQL이 **즉시** DB로 나간다. "모았다가 나중에"가 없다. (예외: `ExecutorType.BATCH`는 모았다 flush — 기본 SIMPLE은 즉시)
- 그래서 "값만 바꾸면 알아서 UPDATE"(더티 체킹)도 없고, **내가 SQL을 호출한 그 순간이 곧 실행 시점.**

### "락 획득"과 "락 해제"는 기준이 다르다 (헷갈림 주의)

| | 락 획득 / version 체크가 일어나는 시점 | 락 해제 |
|---|---|---|
| **JPA — `@Lock(PESSIMISTIC_*)` 조회** | **조회 호출 즉시** (SELECT는 쓰기 지연 대상이 아님) | **commit/rollback** |
| **JPA — 더티 체킹 UPDATE** (행 락·version 체크) | **flush 때** (UPDATE가 그때 나가므로) | **commit/rollback** |
| **MyBatis** | **mapper 호출 즉시** (SQL이 바로 나가므로) | **commit/rollback** |

- **락 해제(= 트랜잭션 종료)는 commit/rollback이 기준** — ORM이든 MyBatis든 **DB의 보편 규칙**으로 동일. `FOR UPDATE`로 잡은 row 락은 commit/rollback 전까지 안 풀린다.
- **락 획득(= 락 거는 행위 / 충돌 감지)은 "SQL이 DB에서 실행되는 순간"** 이 기준. JPA는 INSERT/UPDATE/DELETE만 flush로 미뤄지고, SELECT(`FOR UPDATE` 포함)는 MyBatis처럼 즉시.

> JPA에서 "커밋 시점"이 자꾸 등장한 건, **더티 체킹 UPDATE(와 거기 얹힌 version 체크)가 commit 직전 flush로 미뤄지기 때문**이다. 명시적 비관 락은 조회한 그 줄에서 바로 걸린다. MyBatis는 미룸 자체가 없으니 모든 SQL이 **호출 즉시** 실행되고, 해제만 commit 때 된다.

```java
// MyBatis 낙관적 락 — 호출한 그 줄에서 바로 판가름
int updated = mapper.update(...);  // WHERE id=? AND version=? (UPDATE 즉시 실행)
if (updated == 0) throw new OptimisticLockException();  // 영향 행 0 → 충돌, 직접 처리
```

> MyBatis는 "매 호출이 곧 flush"인 셈. JPA가 변경을 모았다 flush로 한 번에 터뜨리는 것과 달리, 호출마다 SQL이 바로 나간다.

### "lazy"는 두 종류 — 헷갈리지 말 것

| | JPA | MyBatis |
|---|---|---|
| **쓰기 지연** (write-behind, UPDATE 모았다 나중에) | 있음 | **없음** |
| **지연 로딩** (lazy loading, 연관 데이터 읽기를 미룸) | 있음 | **있음** (`lazyLoadingEnabled` 설정 기반) |

> "MyBatis엔 lazy가 없다"는 **쓰기 지연** 한정으로만 맞다. 읽기 쪽 지연 로딩은 MyBatis도 설정으로 지원한다(자동/강력하진 않음).

### 즉시 실행돼도 commit 전이면 통째로 롤백된다 (원자성)

MyBatis에서 SQL이 "즉시" 나가도, `@Transactional`이 `autocommit=false`로 묶고 있어 **commit 전까진 미확정**이다. 그래서 중간에 예외가 나면 이미 실행된 SQL까지 전부 rollback된다.

```
@Transactional 시작 (autocommit=false)
  update(A) → 즉시 실행 (미확정)
  update(B) → 즉시 실행 (미확정)
  update(C) → 💥 예외 → rollback → A·B도 전부 취소
```

> ⚠️ "하나라도 에러나면 다 롤백"이 항상 참은 아니다 — 스프링은 기본적으로 unchecked 예외만 롤백한다 → [@Transactional §5 롤백 규칙](../spring/transactional.md).

---

## 7. 정리

- **"커밋 시점" = 트랜잭션 commit이 트리거하는 flush 시점.** 이때 모아둔 SQL이 DB로 나간다.
- JPA는 `setter`로 즉시 UPDATE를 안 날리고 **flush 때 변경을 감지해** 발사한다(쓰기 지연). persist/remove는 호출 시 쌓아뒀다가 flush 때 발사.
- **더티 체킹**이 변경을 감지해 UPDATE를 만들고, **version 체크**는 그 UPDATE의 `WHERE`에 얹혀 함께 나간다 → 그래서 "수정할 때 체크"가 맞고, "조회 시점엔 안 함".
- 이 모든 게 **트랜잭션(영속성 컨텍스트) 위에서** 돌아간다. 락은 이 기반 위에 올라탄 것.

### 💡 판단 기준

- **"UPDATE·예외가 언제 나나?"는 코드 줄이 아니라 flush 트리거(commit·겹치는 JPQL·`em.flush()`)를 찾아서 답한다.** setter 옆에 둔 `try-catch`가 `OptimisticLockException`을 못 잡는 건 그 줄에서 SQL이 안 나가기 때문이다(§5).
- **같은 트랜잭션에서 "방금 커밋된 최신 값"이 필요하면 1차 캐시를 먼저 의심한다.** 쿼리를 다시 날려도 이미 올라온 엔티티는 옛 상태다 → `em.refresh()`, 또는 처음부터 락 조회(§1).
- **값을 바꿨는데 반영이 안 되면 "이 엔티티가 지금 영속 상태인가(트랜잭션 안·같은 컨텍스트인가)"부터 본다** → [준영속 수정: merge 함정](./merge-vs-dirty-checking.md) · [OSIV](./osiv.md).

---

## 8. 참고
- [Hibernate ORM 6.6 User Guide - Flushing (AUTO flush는 겹치는 쿼리만)](https://docs.hibernate.org/orm/6.6/userguide/html_single/#flushing-auto)
- [Hibernate User Guide - Flushing](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#flushing)
- [Baeldung - JPA Persistence Context](https://www.baeldung.com/jpa-hibernate-persistence-context)
- 관련 노트: [JPA @Lock](./lock.md) · [락 개념 종합](../concurrency/locks.md) · [OSIV](./osiv.md)

---

**학습 날짜**: 2026-05-26
**계기**: 낙관적 락의 "커밋 시점에 version 체크"가 더티체킹과 같은 건지, 트랜잭션과 무슨 관계인지 헷갈려서 그 기반인 영속성 컨텍스트/flush를 정리 (2026-10-02 auto-flush 조건·락 획득 시점·1차 캐시 함정 보정, 💡 추가)

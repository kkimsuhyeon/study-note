# 영속성 컨텍스트 · flush · 더티 체킹 — "커밋 시점"의 정체

> **한 줄 요약**: JPA는 `setter`로 값을 바꿔도 즉시 `UPDATE`를 날리지 않고, 변경을 **영속성 컨텍스트**에 모아뒀다가 **flush**(보통 트랜잭션 commit 직전) 때 한꺼번에 SQL로 내보낸다. 락 문서에서 자주 나오는 "**커밋 시점에 version 체크**"는 바로 이 flush 동작 때문이다.

관련 노트: [JPA @Lock](./lock.md) · [락 개념 종합](../concurrency/locks.md) · [@Transactional](../spring/transactional.md)

---

## 0. 왜 이걸 알아야 하나

낙관적 락 문서를 읽다 보면 이런 문장이 나온다:

> "낙관적 락은 **커밋 시점에** version을 비교한다. 조회 시점엔 안 한다."

여기서 막힌다면, 원인은 락이 아니라 그 아래 깔린 **JPA의 동작 방식(영속성 컨텍스트)** 을 모르기 때문이다. 이걸 잡으면 락·더티체킹·`OptimisticLockException`이 왜 그 타이밍에 터지는지 한 번에 풀린다.

---

## 1. 영속성 컨텍스트 (Persistence Context)

> 엔티티를 보관·관리하는 **JPA의 1차 캐시 같은 메모리 공간.** 트랜잭션 동안 "관리 대상(영속 상태)" 엔티티들이 여기 올라가 있다.

- 생명주기 ≈ **트랜잭션 범위** (보통 `@Transactional` 메서드 시작~끝).
- `find`/JPQL로 조회하면 엔티티가 여기에 **올라가면서 그 순간의 값을 스냅샷으로 같이 저장**한다.
- 한 번 올라온 엔티티는 **JPA가 계속 감시**한다.

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

- `setStock(9)`는 그냥 **자바 객체의 필드를 바꾼 것**일 뿐, `UPDATE` SQL이 나가지 않는다.
- 변경 내용은 영속성 컨텍스트의 **쓰기 지연 SQL 저장소**에 쌓인다.
- 모아둔 SQL은 **flush 시점에** 한꺼번에 DB로 나간다.

> 그래서 `save()`를 명시적으로 안 불러도 값이 반영된다(= dirty checking). 반대로, UPDATE가 "언제" 나가는지는 내 코드 줄이 아니라 **flush 타이밍**이 결정한다.

---

## 3. flush — "모아둔 SQL을 DB로 내보내는" 순간

flush = 영속성 컨텍스트의 변경 내용을 DB에 동기화(SQL 발사). **commit과는 다른 개념**이지만 보통 같이 일어난다.

**flush가 일어나는 시점 (3가지)**
1. **트랜잭션 commit 직전** ← 가장 흔함. "커밋 시점"이라는 말의 정체.
2. **JPQL/쿼리 실행 직전** (조회 결과 정합성을 위해, AUTO 모드 기본)
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

    em.createQuery("select o from Order o ...").getResultList();
    //  ↑ JPQL 실행 직전 auto-flush → 모아둔 UPDATE 먼저 발사
    //    WHERE version=1 인데 DB가 이미 2 → 0건 → 여기서 예외 💥 (commit 전)
}                                              // commit 때 flush되면 → 그때 예외
```

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
 ├─ setter로 값 변경 → 자바 객체만 변경 (쓰기 지연 저장소에 예약)
 │
 └─ 메서드 정상 종료 → COMMIT
        └─ flush (이 순간!)
             ├─ 더티 체킹: 스냅샷 vs 현재 비교 → UPDATE 생성
             ├─ version 조건 얹어 UPDATE 발사
             ├─ 0건이면 OptimisticLockException → 롤백
             └─ DB 트랜잭션 확정
```

- 영속성 컨텍스트 = **트랜잭션 단위로 생성·소멸**.
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
| **JPA** | **flush 때** (SQL이 그때 나가므로) | **commit/rollback** |
| **MyBatis** | **mapper 호출 즉시** (SQL이 바로 나가므로) | **commit/rollback** |

- **락 해제(= 트랜잭션 종료)는 commit/rollback이 기준** — ORM이든 MyBatis든 **DB의 보편 규칙**으로 동일. `FOR UPDATE`로 잡은 row 락은 commit/rollback 전까지 안 풀린다.
- **락 획득(= 락 거는 행위 / 충돌 감지)은 "SQL이 DB에서 실행되는 순간"** 이 기준. JPA는 flush로 미뤄지고, MyBatis는 즉시.

> JPA에서 "커밋 시점"이 자꾸 등장한 건, **JPA가 SQL을 commit 직전 flush로 미루기 때문**에 "락 거는 시점 ≈ 커밋 시점"이 됐던 것. MyBatis는 그 미룸이 없으니 락은 **호출 즉시** 걸리고, 해제만 commit 때 된다.

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

> ⚠️ **스프링 롤백 함정**: 스프링은 기본적으로 **`RuntimeException`/`Error`(unchecked)만 자동 롤백**한다. `checked exception`(`IOException` 등)은 던져도 **롤백하지 않고 commit**된다. 롤백시키려면 `@Transactional(rollbackFor = Exception.class)`를 명시해야 한다. → "하나라도 에러나면 다 롤백"이 항상 참은 아니다.

---

## 7. 정리

- **"커밋 시점" = 트랜잭션 commit이 트리거하는 flush 시점.** 이때 모아둔 SQL이 DB로 나간다.
- JPA는 `setter`로 즉시 UPDATE를 안 날리고 **모아뒀다가(쓰기 지연) flush 때** 발사한다.
- **더티 체킹**이 변경을 감지해 UPDATE를 만들고, **version 체크**는 그 UPDATE의 `WHERE`에 얹혀 함께 나간다 → 그래서 "수정할 때 체크"가 맞고, "조회 시점엔 안 함".
- 이 모든 게 **트랜잭션(영속성 컨텍스트) 위에서** 돌아간다. 락은 이 기반 위에 올라탄 것.

---

## 8. 참고
- [Hibernate User Guide - Flushing](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#flushing)
- [Baeldung - JPA Persistence Context](https://www.baeldung.com/jpa-hibernate-persistence-context)
- 관련 노트: [JPA @Lock](./lock.md) · [락 개념 종합](../concurrency/locks.md)

---

**학습 날짜**: 2026-05-26
**계기**: 낙관적 락의 "커밋 시점에 version 체크"가 더티체킹과 같은 건지, 트랜잭션과 무슨 관계인지 헷갈려서 그 기반인 영속성 컨텍스트/flush를 정리

# @Lock 심화 — 헷갈리는 개념 정리

> **한 줄 요약**: JPA 락에서 자주 헷갈리는 세 가지 — ① `@Version`만 vs `@Lock(OPTIMISTIC)`, ② 공유 락 vs 배타 락, ③ `OPTIMISTIC` vs `OPTIMISTIC_FORCE_INCREMENT`.

관련 노트: [@Lock 기본](./lock.md) · [@Lock 실무 패턴](./lock-practical.md) · [영속성 컨텍스트](./persistence-context.md)

---

## 1. `@Version`만 vs `@Lock(OPTIMISTIC)` — 차이가 뭔가?

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

---

## 2. 공유 락(Shared) vs 배타 락(Exclusive)

비관적 락(DB 레벨 락) 얘기.

**공유 락 (`PESSIMISTIC_READ`, `SELECT ... FOR SHARE`)**
> "여러 명이 같이 **읽는 건** OK. 단, 아무도 **수정은 못 함**"
- 여러 트랜잭션이 동시에 공유 락을 가질 수 있음 (읽기끼리 공존)
- 쓰기(배타 락)는 차단됨
- 용도: "내가 읽는 동안 이 값이 바뀌면 안 돼. 근데 남이 같이 읽는 건 괜찮아"

**배타 락 (`PESSIMISTIC_WRITE`, `SELECT ... FOR UPDATE`)**
> "내가 잡으면 **나만 락 가능**. 다른 사람의 락 시도·쓰기는 대기"
- 단 하나의 트랜잭션만 획득 가능
- 다른 트랜잭션의 **락 시도(`FOR SHARE`/`FOR UPDATE`)와 쓰기**를 차단 (단, 잠금 없는 일반 SELECT는 MVCC 스냅샷으로 읽힘 → [@Lock 실무 패턴](./lock-practical.md)의 "PESSIMISTIC_WRITE가 모든 SELECT를 막는 것은 아니다" 참고)
- 용도: "내가 이거 수정할 거니까 아무도 건드리지 마" (재고 차감 등 경쟁 상황)

**락 요청끼리의 호환성 표**

| | 상대가 공유 락 | 상대가 배타 락 |
|---|---|---|
| **공유 락 요청** (`FOR SHARE`) | ✅ 가능 (같이 잡음) | ⛔ 대기 |
| **배타 락 요청** (`FOR UPDATE`) | ⛔ 대기 | ⛔ 대기 |

**일반 SELECT까지 포함한 전체 표** (← 공유락/배타락 차이가 헷갈릴 때)

| 내가 하려는 것 | 상대가 **공유 락** 잡음 | 상대가 **배타 락** 잡음 |
|---|---|---|
| 일반 `SELECT` | ✅ 읽힘 | ✅ 읽힘 |
| `SELECT ... FOR SHARE` (공유락) | ✅ **같이 잡힘** | ⛔ 대기 |
| `SELECT ... FOR UPDATE` (배타락) | ⛔ 대기 | ⛔ 대기 |
| `UPDATE` / `DELETE` | ⛔ 대기 | ⛔ 대기 |

> **핵심 차이는 단 한 칸**: 공유락이 걸린 row는 다른 트랜잭션도 **공유락(`FOR SHARE`)을 동시에 잡을 수 있다**(읽기 락 공존). 배타락이 걸린 row는 **어떤 락도 못 잡는다**(독점). **일반 SELECT는 둘 다 허용**하므로, 일반 조회만 보면 둘의 차이가 안 보인다.

> 비유: 공유락 = 도서관 열람실(여러 명이 같이 펴서 읽기 가능, 단 가져가긴 불가) / 배타락 = 1인 대출(한 명만, 나머지는 다 대기) / 일반 SELECT = 멀리서 표지 사진 찍기(둘 다 가능, 단 스냅샷).

> 한 줄 요약: 공유 락 = "같이 읽기 락 OK, 쓰기 금지", 배타 락 = "락은 나만, 완전 독점". 재고·포인트 차감 같은 동시성 처리는 보통 `PESSIMISTIC_WRITE`(배타)를 쓴다.

**그럼 공유 락은 어디에 쓰나? (실무에선 배타 락보다 훨씬 덜 씀)**

배타 락이 "내가 수정할 거니까 비켜"라면, 공유 락은 **"나는 이 값을 안 바꾼다. 근데 이걸 기준으로 작업하는 동안 남이 바꾸면 안 됨. 단, 다른 읽기는 허용"**일 때 쓴다.

- **부모 행 보호 (FK 참조 무결성)** — 가장 현실적인 용도. 예: 게시글(부모)에 댓글(자식)을 추가하는 동안 게시글이 삭제/변경되면 안 되지만, 게시글 자체는 안 바꾸고, 여러 명이 동시에 댓글을 달 수 있어야 함 → 게시글을 `FOR SHARE`로 읽음. (InnoDB는 FK 검사 시 부모 행에 공유락을 자동으로 검)
- **읽은 값 기반 검증/집계** — 환율·설정값을 읽어 계산하는 동안 그 값이 바뀌면 안 되지만, 나는 수정 안 하고, 다른 읽기 작업과는 공존해야 할 때.

**선택 기준**

| 내 의도 | 선택 |
|---|---|
| 이 행을 **내가 수정할 것** | 배타 락 `FOR UPDATE` |
| 안 바꾸지만 읽는 동안 남이 못 바꾸게 + 다른 읽기는 허용 | 공유 락 `FOR SHARE` |
| 그냥 보기만 (남이 바꿔도 무관) | 일반 SELECT |

**⚠️ 왜 잘 안 쓰이나 — 공유락→배타락 업그레이드 데드락**
```
T1: FOR SHARE 읽음 (공유락) ┐ 둘이 공존
T2: FOR SHARE 읽음 (공유락) ┘
T1: UPDATE 시도 → T2의 공유락 해제 대기 ⏳
T2: UPDATE 시도 → T1의 공유락 해제 대기 ⏳   → 데드락 💥
```
그래서 **"읽고 나서 곧 수정할 거면 처음부터 `FOR UPDATE`를 써라"**가 정설. 읽기 정합성만 필요하면 보통 일반 SELECT(MVCC)나 낙관적 락(`@Version`)으로 해결하고, 공유락은 FK 보호 등 제한적 상황에만 신중히 쓴다. (데드락 상세는 [데드락](../concurrency/deadlock.md) 참고)

---

## 3. `OPTIMISTIC` vs `OPTIMISTIC_FORCE_INCREMENT`

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

## 참고
- 관련 노트: [@Lock 기본](./lock.md) · [@Lock 실무 패턴](./lock-practical.md) · [영속성 컨텍스트](./persistence-context.md) · [데드락](../concurrency/deadlock.md)

---

**학습 날짜**: 2026-05-25 (2026-05-26 lock.md에서 심화 개념만 분리)

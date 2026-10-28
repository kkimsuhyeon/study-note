# 준영속 엔티티 수정 — 변경 감지 vs merge

> **한 줄 요약**: 준영속 엔티티(DB 식별자를 가졌지만 영속성 컨텍스트가 관리하지 않는 객체)를 수정하는 방법은 ① **변경 감지**(트랜잭션 안에서 조회 후 값 변경 — 권장)와 ② **merge** 두 가지. merge의 함정은 **모든 필드를 통째로 교체**한다는 것 — 폼/DTO에 없는 필드는 **null로 덮어쓴다.** 그래서 실무 답은 "컨트롤러에서 엔티티를 만들지 말고, **id + 변경 데이터만 서비스로 넘겨 변경 감지**"다. (김영한: "정말 중요, 완벽 이해 필수")

관련 노트: [영속성 컨텍스트 · 더티 체킹](./persistence-context.md) · [변환 계층](../design/transform-layers.md)

---

## 1. 언제 만나는 문제인가 — 수정 폼 저장

```java
@PostMapping("items/{id}/edit")
String updateItem(@PathVariable Long id, @ModelAttribute BookForm form) {
    Book book = new Book();          // ⚠️ new로 만들었지만
    book.setId(form.getId());        //    DB 식별자를 갖고 있다 = "준영속 엔티티"
    book.setName(form.getName());
    book.setPrice(form.getPrice());
    itemService.saveItem(book);      // 이걸 어떻게 DB에 반영하지?
}
```

- **준영속 엔티티** = 이미 DB에 저장된 적 있는 식별자를 가진, 그러나 지금 영속성 컨텍스트가 관리하지 않는 객체. 트랜잭션이 끝나 detach된 것뿐 아니라 **위처럼 임의로 new한 객체도 포함**된다.
- 준영속이므로 [더티 체킹](./persistence-context.md) 대상이 아니다 — 값을 바꿔도 UPDATE가 저절로 안 나간다. 그래서 두 가지 방법이 등장한다.

## 2. 방법 ① 변경 감지 (권장)

```java
@Transactional
public void updateItem(Long itemId, String name, int price, int stockQuantity) {
    Item findItem = itemRepository.findOne(itemId);  // 영속 상태로 조회
    findItem.setName(name);                          // 값만 변경 (원하는 필드만!)
    findItem.setPrice(price);
    findItem.setStockQuantity(stockQuantity);
}   // 트랜잭션 commit → flush → 더티 체킹 → UPDATE
```

트랜잭션 안에서 **영속 엔티티를 조회해서 직접 값을 변경** — 커밋 시점에 변경 감지가 UPDATE를 만든다. **원하는 속성만 선택해서** 바꿀 수 있다.

## 3. 방법 ② merge — 동작과 함정

```java
@Transactional
public void update(Item itemParam) {       // itemParam: 준영속
    Item merged = em.merge(itemParam);     // 반환된 merged가 영속, itemParam은 여전히 준영속
}
```

merge의 내부 동작:

```
① 파라미터의 식별자로 1차 캐시/DB에서 영속 엔티티 조회
② 조회한 영속 엔티티에 준영속 객체의 값을 "전부" 밀어넣음 (모든 필드 교체)
③ 커밋 시 변경 감지로 UPDATE
```

> ⚠️ ①에서 **DB에 행이 없으면?** 예전엔 새로 INSERT했지만, **Hibernate 6.6부터는 "분명히 준영속인데 행이 없다"고 판단되면 `OptimisticLockException`**을 던진다(생성 전략 `@Id`이거나 non-primitive `@Version`이 있을 때). 다른 트랜잭션이 지운 행을 merge가 몰래 되살리던 동작을 막은 것.

### ⚠️ 핵심 함정 — 없는 필드는 null로 덮어쓴다

**merge는 부분 수정이 아니라 전체 교체다.** 수정 폼에 `price`가 없어서 준영속 객체의 price가 null이면 — **DB의 price가 null로 UPDATE된다.** "폼에 안 넣은 필드가 사라졌어요"류 사고의 정체. 변경 감지는 바꾼 필드만 반영하므로 이 위험이 없다.

### 곁가지 — save()의 정체

Spring Data JPA의 `save()`는 사실 이 분기다:

```java
if (entity.getId() == null) em.persist(entity);   // 식별자 없음 = 신규
else                        em.merge(entity);      // 식별자 있음 = merge!
```

`@GeneratedValue` 전제의 관례 — **"save = 저장"이 아니라 "식별자 있으면 merge"**라서, 수정 의도로 save를 부르면 위의 null 덮어쓰기 함정을 그대로 밟는다. (식별자를 직접 할당하는 엔티티면 이 분기 자체가 어긋난다)

- 정확히는 위 코드보다 한 단계가 더 있다: 엔티티에 **non-primitive `@Version`이 있으면 version이 null인지로 먼저** 신규를 판정하고, 없을 때만 id가 null인지 본다. 직접 할당 id라면 `Persistable.isNew()`를 구현해 판정을 넘긴다 (Spring Data JPA "Entity State-detection Strategies").
- 기준은 "단일키냐 복합키냐"가 아니라 **`save()` 호출 순간 id가 이미 채워져 있나**다. `@GeneratedValue(strategy = UUID)`는 값을 Java(Hibernate)가 만들어도 persist **도중에** 채우므로 판정 시점엔 null → persist. 반면 필드 초기화로 `UUID.randomUUID()`를 넣어 두면 단일키여도 merge로 빠진다.

### ⚠️ 직접 할당 키 + `save()` = 유니크 방어선이 조용히 사라진다

추가 SELECT보다 큰 문제는 **중복일 때**다. 복합키 `(user_id, idempotency_key)`처럼 "같은 조합이 두 번 들어오면 DB가 에러로 막아라"를 PK에 맡긴 테이블에서:

```
첫 저장   (u1, K) 없음      → merge: SELECT → 없음 → INSERT   (평소엔 정상, SELECT 하나 낭비)
중복 저장 (u1, K) 이미 있음 → merge: SELECT → 있음 → UPDATE  ← 중복 키 에러 대신 덮어씀 ❌
```

평소엔 저장도 되고 테스트도 통과한다. 락이 뚫리는 등 **방어선이 실제로 필요한 순간에만** 에러 없이 덮어쓴다 — 멱등키 기록, 1인 1매 예약 같은 "중복이면 실패해야 하는" 테이블에서 치명적. (Hibernate 6.6의 "행이 없으면 `OptimisticLockException`"은 생성 전략 `@Id`·`@Version`이 있을 때만이라, `@Version` 없는 직접 할당 키는 위 동작 그대로 — 확인 필요)

| 쓰기 방법 | 복합 PK | 대리키(`@GeneratedValue`) + `@UniqueConstraint` |
|---|---|---|
| `save()` | ❌ merge → 중복 시 UPDATE | ✅ persist → 중복 시 `DataIntegrityViolationException` |
| `entityManager.persist` / `Persistable.isNew()` / `@Version`(wrapper) | ✅ INSERT → 중복 시 에러 | ✅ |
| 직접 SQL (`JdbcTemplate`, 네이티브 INSERT) | ✅ | ✅ |

> 💡 판단 기준: **테이블 설계는 "업무 키를 PK로"가 깔끔하고, JPA 쓰기는 "생성 id"가 안전하다.** 둘이 부딪히면 쓰기를 실행하는 도구에 맞춘다 — `save()`를 쓰면 대리키 + 유니크 제약(JPA 기본값), 복합 PK를 고집하면 신규 저장은 `persist`, SQL을 직접 쓰면 복합 PK도 문제없다. 어느 쪽이든 **"같은 키로 두 번 저장하면 예외" 레포지토리 테스트**를 둬야 이 함정이 드러난다. (복합키를 쓰는 대표 경우: 컬럼이 붙은 다대다 연결 테이블 `@EmbeddedId`+`@MapsId`, 레거시 스키마. 복합키 매핑 문법은 [값 타입 §5-1](./value-types.md))
>
> 참고로 MySQL InnoDB는 행을 PK 순서로 저장(클러스터드 인덱스)하고 보조 인덱스엔 PK 값만 담으므로, 대리키 방식은 유니크 인덱스 → PK로 한 번 더 찾아간다. 그래도 둘 다 B-tree 탐색이라 대부분 테이블에선 결정 요인이 아니다.

## 4. 💡 권장 패턴 — 컨트롤러에서 엔티티를 만들지 마라

```java
// ❌ 컨트롤러에서 어설픈 엔티티 생성 → merge/save
Book book = new Book(); book.setId(id); ...; service.saveItem(book);

// ✅ id + 변경 데이터만 전달 → 서비스가 조회 후 변경 감지
itemService.updateItem(id, form.getName(), form.getPrice(), form.getStockQuantity());
// 파라미터가 많으면 UpdateItemDto로 묶는다
```

> **엔티티를 변경할 때는 항상 변경 감지를 사용하라** (강의 결론). merge는 "전 필드가 완비된 객체"가 보장될 때만 안전한데, 그 보장을 지키는 비용이 변경 감지보다 비싸다. 컨트롤러는 [변환 계층](../design/transform-layers.md)대로 DTO만 다루고, 엔티티의 생성·수정은 트랜잭션이 있는 서비스 계층 안에서.

---

## 5. 참고
- 김영한, 실전! 스프링 부트와 JPA 활용1 — 7장 "변경 감지와 병합(merge)"
- [Hibernate ORM 6.6 Migration Guide - Merge versioned entity when row is deleted](https://docs.hibernate.org/orm/6.6/migration-guide/#merge-versioned-deleted)
- [Spring Data JPA - Entity State-detection Strategies (save()의 신규 판정)](https://docs.spring.io/spring-data/jpa/reference/jpa/entity-persistence.html)
- [MySQL - Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html) (보조 인덱스에 PK 값 저장)
- 관련 노트: [영속성 컨텍스트](./persistence-context.md) (준영속·더티 체킹 메커니즘) · [값 타입 §5-1 복합 키](./value-types.md)

---

**학습 날짜**: 2026-08-13
**계기**: 활용1 7장 — "폼에 없는 필드가 null로 저장되는" 사고의 원인이 merge의 전체 교체 동작임을 이해하고, save()가 내부적으로 merge 분기라는 것까지 연결해서 정리. (2026-10-02 Hibernate 6.6 merge 동작 변경·save() 신규 판정 보강)

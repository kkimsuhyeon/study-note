# JPQL 심화 — 벌크 연산 · 묵시적 조인 · fetch join 한계

> **한 줄 요약**: JPQL의 실무 사고 3대 지점. ① **벌크 연산(executeUpdate)은 영속성 컨텍스트를 무시하고 DB로 직행** → 1차 캐시와 어긋나므로 직후 `em.clear()`(Spring Data는 `@Modifying(clearAutomatically=true)`). ② **경로 표현식은 탐색만 해도 묵시적 조인**을 만든다 → 명시적 조인만 쓸 것. ③ **fetch join 한계 3종**(별칭 지양·컬렉션은 하나만·컬렉션+페이징 불가)은 규칙으로 외워야 한다.

관련 노트: [N+1과 fetch 전략](./n-plus-one-fetch.md) · [Criteria·Specification·Pageable](./spring-data-query.md) · [영속성 컨텍스트](./persistence-context.md)

---

## 1. ⚠️ 벌크 연산 — 영속성 컨텍스트를 건너뛴다

여러 행을 한 번에 UPDATE/DELETE:

```java
int count = em.createQuery("update Member m set m.age = m.age + 1 where m.age >= :age")
              .setParameter("age", 20)
              .executeUpdate();   // 영향받은 행 수 반환
```

**함정**: 벌크 연산은 [쓰기 지연·더티 체킹](./persistence-context.md)의 세계 밖이다 — **영속성 컨텍스트를 무시하고 DB에 직접** 나간다. 그래서:

```java
Member m = em.find(Member.class, 1L);   // 1차 캐시에 age=20으로 올라옴
em.createQuery("update Member m set m.age = 21 where ...").executeUpdate();  // DB만 21
m.getAge();   // 💥 20 — 1차 캐시는 여전히 옛값! DB와 불일치
```

**규칙 (둘 중 하나)**:
1. 벌크 연산을 **가장 먼저** 실행 (컨텍스트에 뭐가 올라오기 전에)
2. 벌크 연산 **직후 `em.clear()`** — 컨텍스트를 비워서 다음 조회가 DB에서 새로 읽게

```java
// Spring Data JPA — clearAutomatically가 위 2번을 자동으로
@Modifying(clearAutomatically = true)
@Query("update Member m set m.age = m.age + 1 where m.age >= :age")
int bulkAgePlus(@Param("age") int age);
```

**`flushAutomatically`는 반대 방향이다 — 실행 *전에* 내보낸다.**

| 옵션 | 언제 | 막는 문제 |
| --- | --- | --- |
| `clearAutomatically = true` | 쿼리 **후** 영속성 컨텍스트를 비움 | 1차 캐시가 옛값을 들고 있는 것(위 예시) |
| `flushAutomatically = true` | 쿼리 **전** 쌓인 변경을 DB에 씀 | DB가 옛값인 채로 쿼리가 도는 것 — 더티 체킹으로 바뀐 필드, `persist`만 하고 아직 안 나간 INSERT가 DB에 없어서 벌크·네이티브 쿼리가 그걸 못 봄 |

JPA 기본 플러시 모드(AUTO)에서 Hibernate는 JPQL 앞에서는 **테이블이 겹칠 때만** flush하고, **네이티브 SQL** 앞에서는 어떤 테이블을 건드리는지 모르므로(쿼리 공간을 따로 등록하지 않았다면) **전부 flush**한다([영속성 컨텍스트](./persistence-context.md) 3절). 그래서 EntityManager로 쓰는 일반적인 Spring Data JPA 환경에서 `flushAutomatically = true`는 기본 동작을 **명시해 두는** 쪽에 가깝다 — 플러시 모드를 `COMMIT`으로 바꿨거나 다른 JPA 구현체를 쓸 때도 "이 쿼리 전엔 무조건 내보낸다"가 유지된다. `flushAutomatically`는 `@Modifying`의 속성이라 조회 쿼리에는 붙일 수 없다.

`@Modifying` 자체는 "이 `@Query`는 SELECT가 아니라 INSERT·UPDATE·DELETE다"라는 표시다. 없으면 Spring Data가 결과 목록을 읽으려 해서(`getResultList`) 실패한다.

**"없으면 넣기"를 `save()`로 못 만드는 이유 → 네이티브 `INSERT ... ON CONFLICT DO NOTHING`.**

- `save()`는 새 엔티티인지 먼저 판단한다. `@Version`이 없고 ID가 이미 채워진(복합 키·직접 지정 ID) 엔티티는 "기존 것"으로 보고 `merge()` → **SELECT 후 없으면 INSERT**. 확인과 삽입이 두 문장이라 동시 요청 둘이 모두 "없음"을 보고 둘 다 INSERT → 기본 키 위반.
- PostgreSQL은 문장 하나가 제약 위반으로 실패하면 **그 트랜잭션 전체를 중단 상태**로 만든다(이후 문장이 전부 거부됨). 예외를 catch해서 이어 가는 방식이 통하지 않는다.
- JPQL 표준에는 `INSERT ... VALUES` 자체가 없다.
- 그래서 PostgreSQL 문법을 네이티브로 쓴다. `ON CONFLICT DO NOTHING`은 "이미 있으면 에러 없이 넘어가라"를 **DB가 한 문장 안에서** 처리한다. 상대 트랜잭션이 같은 키를 넣는 중이면 끝날 때까지 기다렸다가 판단한다. 이어서 `SELECT ... FOR UPDATE`로 그 행을 잠그는 패턴은 [advisory lock 노트](../../database/postgres-advisory-lock.md) 판단 기준 2번.

- UPDATE/DELETE 표준 지원 (+Hibernate는 insert into ... select도).
- ⚠️ 벌크 UPDATE는 **@Version 증가·더티 체킹·cascade를 전부 우회**한다 — 낙관락이 지켜주지 않는 경로가 된다는 뜻 ([lock-practical](./lock-practical.md)의 벌크/조건부 UPDATE 논의와 연결).

## 2. ⚠️ 경로 표현식 — 점(.) 하나가 조인을 만든다

```java
select m.team.name from Member m   // ⚠️ m.team 탐색 → SQL엔 JOIN이 "묵시적으로" 생김
```

| 경로 종류 | 예 | 동작 |
|---|---|---|
| 상태 필드 | `m.username` | 탐색 끝 (조인 없음) |
| **단일 값 연관** | `m.team` | **묵시적 내부 조인 발생**, 계속 탐색 가능 |
| 컬렉션 값 연관 | `t.members` | 탐색 **불가** — 명시적 조인 + 별칭으로만 (`t.members.username` ❌) |

> **"묵시적 조인 대신 명시적 조인을 써라"** — 조인은 SQL 튜닝의 핵심 포인트인데, 묵시적 조인은 쿼리 어디서 조인이 생기는지 JPQL만 봐서는 안 보인다. `select m.team.name` 대신 `select t.name from Member m join m.team t`.

## 3. fetch join 정밀 규칙

### 일반 조인 vs fetch join

```java
select m from Member m join m.team t        // 일반 조인: SQL은 조인하지만 SELECT는 Member만
                                            //   → m.getTeam()은 여전히 지연 로딩(프록시)
select m from Member m join fetch m.team    // fetch join: Team까지 한 번에 조회 (영속화)
```

일반 조인은 **조건으로만 조인**하고 연관 엔티티를 가져오지 않는다 — "조인했는데 왜 N+1이 나지?"의 정체. 객체 그래프를 채우는 건 fetch join뿐이고, 글로벌 전략(LAZY)보다 우선한다.

### ⚠️ 한계 3종 (규칙으로 암기)

1. **fetch join 대상에 별칭을 주고 where로 거르지 말 것** — Hibernate는 허용하지만, 컬렉션을 걸러서 가져오면 "일부만 로딩된 컬렉션"이 영속성 컨텍스트에 올라가 정합성이 깨진다 (cascade·더티 체킹이 그 일부만 보고 동작).
2. **컬렉션 fetch join은 하나만** — 둘 이상이면 1:N:M으로 행이 곱해진다. `List` 2개면 `MultipleBagFetchException`, `Set`으로 바꾸면 Hibernate는 허용하지만 곱해진 행 비용은 그대로 → 나머지는 batch size·쿼리 분리 ([N+1 노트 §8](./n-plus-one-fetch.md)).
3. **컬렉션 fetch join + 페이징 불가** — 메모리 페이징(OOM 위험). 해법은 batch size → 상세는 [N+1 노트 §5](./n-plus-one-fetch.md).

### distinct와 Hibernate 6

1:N 조인의 행 뻥튀기 대응으로 JPQL `distinct`는 SQL DISTINCT + **애플리케이션에서 같은 식별자 엔티티 중복 제거**의 이중 동작을 한다. **Hibernate 6부터는 distinct 없이 자동 중복 제거** — 상세는 [N+1 노트 §8](./n-plus-one-fetch.md).

## 4. 자잘하지만 당하는 것들

- **`getSingleResult()`**: 결과 0건 → `NoResultException`, 2건 이상 → `NonUniqueResultException`. (Spring Data는 0건을 null/Optional로 감싸줘서 이 예외를 잊기 쉽다 — 순수 JPA/QueryDSL 쓸 때 복귀)
- **엔티티를 직접 파라미터로**: `where m = :member`는 SQL에서 `m.id = ?`로 (엔티티 = PK 취급). 연관 필드도 `m.team = :team` → `team_id = ?`.
- **FROM 절 서브쿼리는 JPQL 표준에 없다** — 표준은 WHERE/HAVING만. Hibernate는 SELECT 절 서브쿼리를 예전부터, FROM 절 서브쿼리를 6.x부터 HQL 확장으로 지원한다(스프링 부트 3+면 사용 가능, 도입 마이너 버전은 확인 필요). 표준만 쓰는 코드라면 조인으로 풀거나 네이티브.
- **Named 쿼리 / @Query의 진짜 장점**: **애플리케이션 로딩 시점에 문법 검증** — 오타가 배포 전에 죽는다. Spring Data `@Query`가 사실상 이것.
- **다형성**: `where type(i) in (Book, Movie)`, `treat(i as Book).author` (다운캐스팅).

## 5. 💡 판단 기준

> **"이 쿼리, 영속성 컨텍스트와 몇 번 어긋나나"를 세라.** 벌크 연산은 컨텍스트를 건너뛰고(→ clear), fetch join 별칭 필터는 일부만 로딩하고(→ 금지), 묵시적 조인은 SQL을 숨긴다(→ 명시적으로). JPQL 사고는 문법 실수가 아니라 전부 **"JPQL은 객체를 다루는 척하지만 실행은 SQL"**이라는 이중성에서 온다 — 항상 "이게 SQL로 뭐가 되나"를 그려볼 것.

---

## 6. 참고
- [Hibernate User Guide - HQL/JPQL](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#hql)
- 김영한, JPA 기본편 10장 (객체지향 쿼리 언어)
- 관련 노트: [N+1과 fetch 전략](./n-plus-one-fetch.md) · [스프링 데이터 쿼리](./spring-data-query.md)

---

**학습 날짜**: 2026-08-13
**계기**: 김영한 JPA 기본편 10장 — 벌크 연산이 영속성 컨텍스트를 우회한다는 것(1차 캐시 불일치), 경로 표현식의 묵시적 조인, fetch join 한계 3종을 "실무 사고 지점" 중심으로 정리. (2026-10-02 컬렉션 fetch join·FROM 서브쿼리 서술 보정)

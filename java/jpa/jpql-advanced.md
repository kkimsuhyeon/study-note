# JPQL 심화 — 벌크 연산 · 묵시적 조인 · fetch join 한계

> **한 줄 요약**: JPQL의 실무 사고 3대 지점. ① **벌크 연산(executeUpdate)은 영속성 컨텍스트를 무시하고 DB로 직행** → 1차 캐시와 어긋나므로 직후 `em.clear()`(Spring Data는 `@Modifying(clearAutomatically=true)`). ② **경로 표현식은 탐색만 해도 묵시적 조인**을 만든다 → 명시적 조인만 쓸 것. ③ **fetch join 한계 3종**(별칭 지양·컬렉션 2개 불가·컬렉션+페이징 불가)은 규칙으로 외워야 한다.

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
2. **컬렉션 fetch join은 둘 이상 불가** — 1:N:M으로 행이 곱해져 데이터 부정합. (`List` 2개면 `MultipleBagFetchException`)
3. **컬렉션 fetch join + 페이징 불가** — 메모리 페이징(OOM 위험). 해법은 batch size → 상세는 [N+1 노트 §5](./n-plus-one-fetch.md).

### distinct와 Hibernate 6

1:N 조인의 행 뻥튀기 대응으로 JPQL `distinct`는 SQL DISTINCT + **애플리케이션에서 같은 식별자 엔티티 중복 제거**의 이중 동작을 한다. **Hibernate 6부터는 distinct 없이 자동 중복 제거** — 상세는 [N+1 노트 §8](./n-plus-one-fetch.md).

## 4. 자잘하지만 당하는 것들

- **`getSingleResult()`**: 결과 0건 → `NoResultException`, 2건 이상 → `NonUniqueResultException`. (Spring Data는 0건을 null/Optional로 감싸줘서 이 예외를 잊기 쉽다 — 순수 JPA/QueryDSL 쓸 때 복귀)
- **엔티티를 직접 파라미터로**: `where m = :member`는 SQL에서 `m.id = ?`로 (엔티티 = PK 취급). 연관 필드도 `m.team = :team` → `team_id = ?`.
- **FROM 절 서브쿼리 불가** — JPQL 표준은 WHERE/HAVING만 (SELECT는 Hibernate 확장). FROM 절이 필요하면 조인으로 풀거나 네이티브. (Hibernate 6부터 FROM 서브쿼리 지원 시작)
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
**계기**: 김영한 JPA 기본편 10장 — 벌크 연산이 영속성 컨텍스트를 우회한다는 것(1차 캐시 불일치), 경로 표현식의 묵시적 조인, fetch join 한계 3종을 "실무 사고 지점" 중심으로 정리.

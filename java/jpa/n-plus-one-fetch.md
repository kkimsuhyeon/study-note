# N+1과 fetch 전략 — fetch join, EntityGraph, batch size

> **한 줄 요약**: N+1은 목록 1번 조회 후 연관 객체를 N번 추가 조회하는 문제다. 해결은 무조건 fetch join이 아니라, 화면이 필요한 데이터 모양에 맞춰 fetch join, EntityGraph, batch size, DTO 조회 중 고른다.

관련 노트: [영속성 컨텍스트](./persistence-context.md) · [Criteria · Specification · Pageable · Page](./spring-data-query.md) · [JPA Repository 테스트](../test/jpa-repository-test.md)

---

## 1. 언제 의심하나

목록 조회는 1번인데 로그에 비슷한 `select`가 반복해서 찍히면 N+1을 의심한다.

```java
List<Order> orders = orderRepository.findAll();

for (Order order : orders) {
    order.getMember().getName(); // 여기서 member 조회가 order 수만큼 추가될 수 있음
}
```

흐름:

1. `orders` 조회 1번
2. 각 `order.member` 접근 시 member 조회 N번
3. 총 1 + N번

---

## 2. 왜 생기나

JPA 연관관계는 보통 지연 로딩(`LAZY`)으로 둔다. 지연 로딩은 나쁜 게 아니다. 필요할 때만 읽으니 기본값으로 좋다.

> **선행지식 — 지연 로딩은 "프록시"로 동작한다.** LAZY면 연관 객체 자리에 **프록시(가짜 객체)**를 끼워두고, 실제로 그 필드에 접근하는 순간 쿼리를 날려 채운다(→ [영속성 컨텍스트](./persistence-context.md)). N+1은 바로 그 "접근 시 쿼리"가 루프를 돌며 N번 반복되는 것.

문제는 **목록 화면에서 연관 데이터를 매번 필요로 하는데도**, 그 사실을 쿼리에 알려주지 않을 때 생긴다.

---

## 3. 해결 선택지

| 방법 | 언제 쓰나 | 주의 |
|---|---|---|
| fetch join | 한 화면에서 반드시 필요한 연관을 같이 읽을 때 | 컬렉션 fetch join + 페이징은 위험 |
| `@EntityGraph` | Repository 메서드에 읽을 연관을 선언하고 싶을 때 | 복잡한 조건에는 한계 |
| batch size | 여러 프록시 초기화를 `IN (...)`으로 묶고 싶을 때 | N+1을 1+몇 번으로 줄이는 방식 |
| DTO 조회 | 화면/API 응답 모양이 명확할 때 | 변경 감지 대상 엔티티가 아님 |

---

## 4. fetch join

```java
@Query("""
    select o
    from Order o
    join fetch o.member
    where o.status = :status
""")
List<Order> findByStatusWithMember(OrderStatus status);
```

`Order`와 `Member`를 한 SQL로 가져온다. 단일 연관(`ManyToOne`, `OneToOne`)을 같이 가져올 때 특히 편하다.

---

## 5. ⚠️ 컬렉션 fetch join + 페이징 = 메모리 페이징(OOM)

`OneToMany` 컬렉션을 fetch join하면 row가 늘어난다. 주문 1개에 주문상품 3개면 SQL 결과 row는 3개 — 엔티티 개수(주문 1)와 row 수(3)가 안 맞는다.

그래서 Hibernate는 이때 **`LIMIT`/`OFFSET`을 SQL에 넣지 못한다.** `Pageable`(= `setFirstResult`/`setMaxResults`)을 같이 주면 잘리는 게 아니라 **전체 row를 메모리로 다 읽은 뒤 애플리케이션 메모리에서 페이징**한다 → 데이터가 많으면 **OOM**. 로그에 경고가 뜬다:

```
HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!
```

> ⚠️ 흔한 오해 교정: "주문이 아니라 row 기준으로 잘린다"가 아니라 **"페이징이 SQL에서 안 되고 전부 메모리에 올라온다(OOM 위험)"**가 진짜 함정. → 컬렉션은 fetch join과 페이징을 같이 쓰지 말 것.

실무 선택:

- 단일 연관(`ManyToOne`/`OneToOne`)은 fetch join (row 안 늘어남)
- 컬렉션 + 페이징은 batch size로 (컬렉션은 LAZY 두고 1+몇 번으로 — §6)
- 목록 API는 DTO 조회
- 딥페이지·대용량 목록 자체의 성능은 keyset/`Window` → [spring-data-query](./spring-data-query.md) §8 고려

---

## 5-1. ⚠️ 페이징이 없어도 컬렉션 fetch join은 비싸다 — 비용은 "행 × 컬럼"

§5는 페이징과 같이 쓸 때의 함정이고, **페이징이 없어도** 컬렉션 fetch join은 조용히 느려질 수 있다. N+1을 없앴다고 안심하는 지점이 여기다.

```
쿼리 횟수  = 1번   ✅ (N+1 해결)
행 수      = 부모 수 × 자식 수
행의 폭    = fetch join 한 모든 엔티티의 전체 컬럼
```

**Hibernate는 이 행들을 전부 엔티티로 만들고, 각각 더티 체킹용 스냅샷을 복사하고, 같은 PK끼리 묶어 중복을 제거한다.** 그 비용이 행 수 × 컬럼 수에 비례해 커진다.

### ⚠️ 안 쓰는 연관을 걸면 그 컬럼이 "전 행"에 중복으로 실린다

가장 놓치기 쉬운 부분. 부모의 `ManyToOne` 연관을 fetch join하면, **컬렉션 때문에 늘어난 모든 행에 그 컬럼이 반복**된다.

```java
.leftJoin(usr.assignments, asgn).fetchJoin()   // 컬렉션 → 행이 곱해짐
.leftJoin(usr.org, org).fetchJoin()            // ⚠️ org 는 화면에서 안 쓰는데
.leftJoin(usr.org.brnFile).fetchJoin()         // ⚠️ 전 행에 컬럼 40개가 반복됨
```

부모 27건 × 자식 3,173건 = **3,173행**인데, 전 직원이 같은 기관이라 `org` 컬럼 40개가 **3,173번 똑같이** 전송된다. 실측으로 이 두 줄을 빼는 것만으로 **1,500ms → 971ms**가 됐다.

> 💡 **"연관을 쓰나?"를 응답 조립 코드에서 확인하고 붙인다.** 습관적으로 fetch join을 늘리면 N+1은 사라져도 그만큼 행이 뚱뚱해진다.

### SQL 은 빠른데 `.fetch()` 가 느리다면 매핑 비용이다

두 숫자가 크게 벌어지면 원인이 DB가 아니라 애플리케이션이다.

```
p6spy [statement] 55 ms      ← SQL 실행 시간 (DB 가 결과를 만들기까지)
프로파일러 .fetch() 2,553ms  ← 실행 + 전송 + 엔티티 매핑 + 스냅샷
```

인덱스를 아무리 봐도 답이 안 나오는 이유가 여기 있다. **DB는 이미 빨리 답했고, 그걸 객체로 빚는 데 시간을 쓰고 있다.**

### 실제로 쓰는 게 컬럼 몇 개면 → 프로젝션

응답 조립 코드가 쓰는 값을 세어봤더니 컬럼 5~6개인데 엔티티 3종을 통째로 만들고 있었다면, 엔티티를 만들 이유가 없다.

```java
// 엔티티 조회 → 행마다 100+ 컬럼, 엔티티 생성·스냅샷·중복제거
.selectFrom(usr).leftJoin(...).fetchJoin()

// 프로젝션 → 행마다 6컬럼, 엔티티 생성 없음
.select(Projections.fields(UsrRow.class,
        usr.id.usrId, usr.usrNm, usr.usrEml,
        asgn.id.childId, mgmt.extlId, mgmt.channelId))
.from(usr)
.leftJoin(usr.assignments, asgn)        // fetchJoin 아님 — 엔티티를 안 만드니 필요 없다
.leftJoin(asgn.mgmt, mgmt)
```

행 수는 그대로지만 **행의 폭과 객체 생성이 사라진다.** 실측 **2,553ms → 수백 ms**. 부수 효과로 영속성 컨텍스트에 안 올라가니 스냅샷·더티 체킹도 없다.

### 프로젝션은 "평평한 행"을 준다 — 묶는 건 내 몫

엔티티 조회는 Hibernate가 부모 1개에 자식 N개를 묶어줬지만, 프로젝션은 **SQL 결과 그대로 평평한 표**가 온다. 그래서 직접 묶어야 한다.

```java
rows.stream()
    .collect(groupingBy(Row::getParentId, LinkedHashMap::new, toList()))   // 부모별로 묶기
    .values().stream()
    .map(this::toResponse)
```

이 그룹핑은 **메모리 연산이라 몇 ms면 끝난다**(실측 6ms). Hibernate가 하던 일을 뺏어온 셈인데, 필요한 것만 다루므로 훨씬 싸다.

> ⚠️ **`left join`이면 자식이 없는 부모도 한 행이 나온다**(자식 컬럼 전부 null). 그 행을 자식으로 세면 개수가 1로 틀어진다. **자식 식별자를 프로젝션에 포함시켜 `null` 여부로 판별**할 것.
> ```java
> this.childCount = rows.stream().filter(r -> r.getChildId() != null).count();
> ```

---

## 6. batch size

```yaml
spring:
  jpa:
    properties:
      hibernate.default_batch_fetch_size: 100
```

지연 로딩 자체를 없애지 않고, 프록시 초기화 쿼리를 `where id in (...)` 형태로 묶는다.

목록에서 여러 연관을 적당히 읽는 경우 fetch join보다 덜 위험할 때가 많다.

---

## 7. 판단 기준

| 질문 | 선택 |
|---|---|
| 단일 연관을 항상 같이 보여주나? | fetch join 또는 EntityGraph |
| 컬렉션을 페이징 목록에서 보여주나? | batch size 또는 DTO 조회 |
| API 응답 전용 화면인가? | DTO 조회 |
| 엔티티 수정까지 이어지나? | 엔티티 조회 + 필요한 연관 fetch |
| **응답에 쓰는 게 컬럼 몇 개뿐인가?** | **프로젝션** (→ §5-1) |
| **컬렉션 fetch join 인데 행이 수천 건인가?** | **프로젝션** — 페이징이 없어도 매핑 비용이 폭발 |

> ⭐ **"쿼리 1회 = 빠름"이 아니다.** N+1을 fetch join으로 없애면 *횟수*는 1이 되지만 **행 × 컬럼**이라는 새 비용이 생긴다. N+1을 잡은 다음 만나는 두 번째 함정이고, 증상이 "쿼리는 하난데 느리다"라 원인을 DB에서 찾다 시간을 버리기 쉽다.

---

## 8. 함정

- `EAGER`로 바꾸는 건 보통 해결책이 아니다. 모든 조회에서 강제로 따라와 다른 N+1이나 과조회가 생긴다.
- fetch join을 많이 붙이면 SQL row 수가 폭증할 수 있다. (상세 → §5-1)
- **안 쓰는 연관까지 fetch join 하면 그 컬럼이 전 행에 중복으로 실린다** — 습관적으로 늘리지 말 것 (→ §5-1)
- **p6spy 의 SQL 시간과 실제 메서드 시간이 크게 다르면 매핑 비용을 의심** — 인덱스를 봐도 답이 안 나온다 (→ §5-1)
- N+1은 테스트에서 SQL 로그를 보지 않으면 놓치기 쉽다.
- Repository 메서드 이름만 봐서는 fetch 여부가 안 보일 수 있으니 `WithMember`, `WithItems`처럼 의도를 드러내면 좋다.

---

## 9. 참고

- Hibernate User Guide: fetching
- Spring Data JPA EntityGraph
- 관련 노트: [영속성 컨텍스트](./persistence-context.md) · [Stream API(groupingBy·프로젝션 후 묶기)](../functional/stream-api.md) · [스레드 풀 내부](../concurrency/thread-pool.md)

---

**학습 날짜**: 2026-08-12 (§5-1 추가)
**계기**: 목록 조회 최적화에서 외부 API 호출을 40회→2회로 줄였는데 기대만큼 안 빨라져 프로파일링 → **전체의 80%가 DB 조회 한 줄**이었다. `p6spy`는 SQL 을 55ms 로 찍는데 `.fetch()` 는 2,553ms — 차이가 전부 엔티티 매핑이었다. 컬렉션 fetch join 이 만든 3,173행에 안 쓰는 연관 컬럼까지 실려 있었고, 실제로 필요한 건 컬럼 6개뿐이라 프로젝션으로 바꿔 해결. **"쿼리 1회니까 병목이 아니다"라고 넘겨짚은 게 가장 큰 시간 낭비**였다.

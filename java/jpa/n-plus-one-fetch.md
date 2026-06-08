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

## 5. 컬렉션 fetch join + 페이징 주의

`OneToMany` 컬렉션을 fetch join하면 row가 늘어난다.

예를 들어 주문 1개에 주문상품 3개가 있으면 SQL 결과 row는 3개다. 이 상태에서 DB 페이징을 걸면 "주문 기준" 페이징이 아니라 "join 결과 row 기준" 페이징이 되어 의도와 달라질 수 있다.

실무 선택:

- 단일 연관은 fetch join
- 컬렉션은 batch size나 별도 조회
- 목록 API는 DTO 조회 고려

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

---

## 8. 함정

- `EAGER`로 바꾸는 건 보통 해결책이 아니다. 모든 조회에서 강제로 따라와 다른 N+1이나 과조회가 생긴다.
- fetch join을 많이 붙이면 SQL row 수가 폭증할 수 있다.
- N+1은 테스트에서 SQL 로그를 보지 않으면 놓치기 쉽다.
- Repository 메서드 이름만 봐서는 fetch 여부가 안 보일 수 있으니 `WithMember`, `WithItems`처럼 의도를 드러내면 좋다.

---

## 9. 참고

- Hibernate User Guide: fetching
- Spring Data JPA EntityGraph

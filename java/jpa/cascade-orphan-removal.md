# 영속성 전이(cascade)와 고아 객체(orphanRemoval)

> **한 줄 요약**: `cascade`는 부모를 persist/remove할 때 자식도 **같이** 처리하는 편의 기능이고, `orphanRemoval=true`는 컬렉션에서 빠진 자식을 **DELETE**하는 기능. 사용 기준은 하나 — **"자식을 참조하는 부모가 단 하나 + 라이프사이클이 부모와 같을 때만."** 이 조건에서 `ALL + orphanRemoval=true`를 걸면 부모가 자식 생명주기를 완전히 관리한다 = DDD **애그리거트 루트** 구현 도구.

관련 노트: [애그리거트 소유권](../design/aggregate-ownership.md) · [연관관계 매핑](./relation-mapping.md) · [값 타입](./value-types.md)

---

## 1. 언제 쓰나

`Order → OrderItem, Delivery`처럼 **부모 없이는 존재 의미가 없는 자식**을, 부모 저장 한 번으로 같이 저장하고 싶을 때.

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> orderItems = new ArrayList<>();

    @OneToOne(fetch = LAZY, cascade = CascadeType.ALL)
    @JoinColumn(name = "delivery_id")
    private Delivery delivery;
}
```

```java
// cascade 없으면: 셋 다 각각 persist해야 함
em.persist(order); em.persist(orderItem1); em.persist(delivery);

// cascade ALL이면: 부모 하나로 전파
orderRepository.save(order);   // orderItems·delivery까지 INSERT
```

## 2. cascade 종류

| 종류 | 전파되는 것 |
|---|---|
| **PERSIST** | 저장 (실무 사용 빈도 높음) |
| **REMOVE** | 삭제 |
| **ALL** | 전부 (PERSIST+REMOVE+MERGE+REFRESH+DETACH) |
| MERGE / REFRESH / DETACH | 해당 연산만 |

- ⚠️ cascade는 **연관관계 매핑과 아무 관련 없다** — 매핑(FK)은 그대로 두고, "영속성 연산을 같이 하겠다"는 편의일 뿐. 걸었다고 관계가 달라지지 않는다.

## 3. orphanRemoval — 컬렉션에서 빠지면 DELETE

```java
order.getOrderItems().remove(0);   // 컬렉션에서 제거
// flush 시: DELETE FROM order_item WHERE id = ...  ← 자동 삭제!
```

- **부모와의 연결이 끊긴(고아) 자식을 삭제**한다. `@OneToOne`/`@OneToMany`에서만 가능.
- 부모를 제거하면 자식도 함께 제거된다 (`CascadeType.REMOVE`처럼 동작).

### cascade REMOVE vs orphanRemoval (혼동 주의)

| | cascade = REMOVE | orphanRemoval = true |
|---|---|---|
| 부모 삭제 시 | 자식도 삭제 ✅ | 자식도 삭제 ✅ (동일) |
| **컬렉션에서 제거만** 하면 | 아무 일 없음 | **DELETE 나감** ← 차이점 |

## 4. ⚠️ 사용 기준 — 소유자가 하나일 때만

> **자식을 참조하는 곳이 그 부모 하나뿐이고, 자식의 라이프사이클이 부모와 완전히 같을 때만** 건다.

- `Order → OrderItem`: OrderItem은 Order에서만 참조, 주문 없이 존재 의미 없음 → **OK**
- `Team → Member`: Member는 다른 곳에서도 조회·참조되는 독립 엔티티 → **금지.** 팀 삭제했다고 회원이 지워지면 사고.
- 판단이 애매하면 안 거는 쪽이 안전 — cascade는 "실수로 지워짐"의 폭발 반경을 키운다.

### ALL + orphanRemoval=true = 애그리거트 루트

두 개를 같이 걸면 자식의 생명주기를 부모가 **완전히 관리**한다 — 자식 리포지토리 없이 부모 저장/삭제/컬렉션 조작만으로 충분. [애그리거트 소유권](../design/aggregate-ownership.md)에서 말하는 "루트를 통해서만 내부에 접근"을 JPA로 구현하는 방법이 바로 이 조합이다.

## 5. 곁가지 — 정적 생성 메서드 패턴 (생성 로직 응집)

cascade로 묶인 애그리거트는 **생성도 한곳에서** 하는 게 자연스럽다:

```java
public class Order {
    public static Order createOrder(Member member, Delivery delivery, OrderItem... orderItems) {
        Order order = new Order();
        order.setMember(member);
        order.setDelivery(delivery);
        for (OrderItem item : orderItems) order.addOrderItem(item);  // 편의 메서드
        order.setStatus(OrderStatus.ORDER);
        order.setOrderDate(LocalDateTime.now());
        return order;
    }
}

public class OrderItem {
    public static OrderItem createOrderItem(Item item, int price, int count) {
        OrderItem oi = new OrderItem();
        oi.setItem(item); oi.setOrderPrice(price); oi.setCount(count);
        item.removeStock(count);   // 생성 시점 부수 로직(재고 차감)도 여기 응집
        return oi;
    }
}
```

- 생성 시점의 규칙(상태 초기화·재고 차감)이 **한곳에 응집** — 밖에서 `new`로 만들면 이 규칙이 흩어진다. 기본 생성자를 `protected`로 막아(`@NoArgsConstructor(access = PROTECTED)`) 생성 메서드 사용을 강제하면 완성. ([변환 계층](../design/transform-layers.md)의 Factory가 이것)

## 6. 💡 판단 기준

> **"이 자식, 다른 데서도 쓰나?"** — 아니오(부모 전용 + 같은 수명)면 `cascade = ALL, orphanRemoval = true`로 부모가 전권 관리(애그리거트 루트). 예(독립 엔티티)면 아무것도 걸지 말고 각자 리포지토리로. 중간은 없다고 생각하는 게 안전하다.

---

## 7. 참고
- [Hibernate User Guide - Cascading](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#pc-cascade)
- 관련 노트: [애그리거트 소유권](../design/aggregate-ownership.md) · [값 타입](./value-types.md) (값 타입 컬렉션 대체 시 이 조합 사용)

---

**학습 날짜**: 2026-08-13
**계기**: 김영한 JPA 기본편 08장 + 활용1 6장 — cascade를 "편하니까" 거는 게 아니라 "소유자 하나 + 수명 동일"일 때만 걸어야 하는 이유와, ALL+orphanRemoval이 애그리거트 루트의 JPA 구현이라는 연결을 정리.

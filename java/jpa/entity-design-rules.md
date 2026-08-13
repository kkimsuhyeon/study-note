# 엔티티 설계 실무 규칙 — Setter 금지 · LAZY 강제 · 컬렉션 초기화 · STRING enum

> **한 줄 요약**: 엔티티 클래스에 반복 적용하는 실무 기본값 모음 — **Setter 열지 않기, 모든 연관관계 LAZY, 컬렉션은 필드에서 초기화하고 교체 금지, `@Enumerated`는 무조건 STRING**. 여기에 `@Entity`의 스펙 제약(기본 생성자·final 불가)과 `hbm2ddl.auto` 환경별 설정까지, "새 엔티티 만들 때 체크리스트"로 쓰는 노트.

관련 노트: [연관관계 매핑](./relation-mapping.md) · [도메인 검증 위치](../design/domain-validation.md) (Setter 대신 비즈니스 메서드로 검증까지 옮기는 이야기) · [N+1과 fetch 전략](./n-plus-one-fetch.md)

---

## 1. Setter를 열지 않는다

- Setter가 다 열려 있으면 **값이 어디서 바뀌는지 추적이 안 된다.** `member.setStock(...)`이 30군데면 재고가 왜 틀어졌는지 30군데를 다 봐야 한다.
- JPA는 더티 체킹으로 동작하므로 setter 호출 = 곧 UPDATE 예약이다([영속성 컨텍스트](./persistence-context.md)). 변경 지점이 흩어지면 UPDATE 발생 지점도 흩어진다.
- 대신 **의도가 드러나는 비즈니스 메서드**를 제공:

```java
// ❌ setter — "왜 바꾸는지"가 코드에 없다
order.setStatus(OrderStatus.CANCEL);
item.setStockQuantity(item.getStockQuantity() - count);

// ✅ 비즈니스 메서드 — 검증·불변식까지 한곳에
public void cancel() {
    if (delivery.getStatus() == DeliveryStatus.COMP)
        throw new IllegalStateException("이미 배송완료된 상품은 취소가 불가능합니다.");
    this.setStatus(OrderStatus.CANCEL);  // (내부에선 private/protected setter 또는 필드 직접)
    ...
}

public void removeStock(int quantity) {
    int restStock = this.stockQuantity - quantity;
    if (restStock < 0) throw new NotEnoughStockException("need more stock");
    this.stockQuantity = restStock;
}
```

> 생성도 마찬가지 — 기본 생성자 + setter 조합 대신 **생성 메서드/생성자**로 필수값을 강제한다. 검증을 어디까지 엔티티에 두는지는 [도메인 검증 위치](../design/domain-validation.md) 참고.

---

## 2. 모든 연관관계는 LAZY

- `@ManyToOne` · `@OneToOne`(ToOne)의 fetch **기본값은 EAGER** — 그대로 두면 조회마다 연관 엔티티를 즉시 로딩하고, JPQL과 만나면 N+1로 직행한다.
- **모든 연관관계에 `fetch = FetchType.LAZY`를 명시**하고, 함께 필요한 경우만 fetch join·EntityGraph로 그때그때 가져온다.

```java
@ManyToOne(fetch = FetchType.LAZY)   // ToOne은 반드시 명시 (기본 EAGER)
@JoinColumn(name = "MEMBER_ID")
private Member member;
```

상세 메커니즘: [N+1과 fetch 전략](./n-plus-one-fetch.md) · 관계별 규칙: [연관관계 매핑](./relation-mapping.md)

---

## 3. 컬렉션은 필드에서 초기화하고, 교체하지 않는다

```java
@OneToMany(mappedBy = "member")
private List<Order> orders = new ArrayList<>();   // ✅ 필드 레벨 초기화로 고정
```

- 필드 초기화면 **null 걱정이 없다** (생성 직후에도 `getOrders().add(...)` 안전).
- 더 중요한 이유: **하이버네이트가 영속화 시점에 컬렉션을 자기 내장 컬렉션으로 감싼다.**

```java
Member member = new Member();
System.out.println(member.getOrders().getClass());
// class java.util.ArrayList

em.persist(member);
System.out.println(member.getOrders().getClass());
// class org.hibernate.collection.internal.PersistentBag  ← 교체됐다!
```

- 하이버네이트는 이 **PersistentBag을 통해 컬렉션 변경을 추적**한다. 임의로 `setOrders(new ArrayList<>())`처럼 **컬렉션을 통째로 갈아끼우면 추적 메커니즘이 깨진다.**
- → **컬렉션은 필드에서 한 번 초기화하고, 이후엔 절대 교체하지 말 것.** 내용만 add/remove로 다룬다.

---

## 4. @Enumerated는 STRING 강제

| | ORDINAL (기본값 ⚠️) | STRING |
|---|---|---|
| 저장값 | enum **순서** (0, 1, 2…) | enum **이름** ("USER", "ADMIN") |
| enum 중간에 상수 추가 시 | **기존 데이터 전부 의미가 밀림 → 데이터 파손** 💥 | 안전 |
| 저장 공간 | 작음 | 조금 큼 (그래도 이걸 쓴다) |

```java
// enum이 USER, ADMIN이던 시절 USER를 0으로 저장했는데,
// enum RoleType { GUEST, USER, ADMIN }  ← 맨 앞에 GUEST 추가하면
// DB의 0은 이제 GUEST로 읽힌다. 기존 USER 데이터가 전부 GUEST가 됨.

@Enumerated(EnumType.STRING)   // 무조건 STRING
private RoleType roleType;
```

기본값이 하필 위험한 ORDINAL이라 **명시를 습관화**해야 한다.

---

## 5. @Entity 스펙 제약 (왜 그런가까지)

- **기본 생성자 필수** — 파라미터 없는 `public` 또는 `protected` 생성자. JPA가 **리플렉션**으로 객체를 생성·프록시를 만들기 때문. 실무에선 `protected` 기본 생성자(+Lombok `@NoArgsConstructor(access = PROTECTED)`)로 두고, 외부에는 생성 메서드만 연다 — 1절과 결이 같다.
- **final 클래스, enum, interface, inner 클래스는 엔티티 불가** — 프록시는 엔티티를 **상속**해서 만들기 때문에 final 클래스는 프록시를 못 만든다.
- **저장할 필드에 final 불가** — 리플렉션으로 값을 채워야 하므로.

---

## 6. hbm2ddl.auto — 옵션 5종과 환경별 설정

`hibernate.hbm2ddl.auto` (스프링부트: `spring.jpa.hibernate.ddl-auto`)

| 옵션 | 동작 |
|------|------|
| `create` | 기존 테이블 **DROP 후 재생성** |
| `create-drop` | create + 애플리케이션 **종료 시점에 DROP** (테스트용) |
| `update` | 변경분만 반영 — 컬럼 추가만 되고 삭제는 안 됨 |
| `validate` | 엔티티↔테이블 매핑이 맞는지 **검증만** (틀리면 기동 실패) |
| `none` | 사용 안 함 |

**환경별 (기본편 04장 그대로):**

| 환경 | 설정 |
|------|------|
| 로컬/개발 초기 | `create` 또는 `update` |
| 테스트 서버(여럿이 공유) | `update` 또는 `validate` |
| **스테이징 · 운영** | **`validate` 또는 `none`** |

- ⚠️ **운영 장비에는 create / create-drop / update 절대 금지.** create는 테이블 DROP(데이터 전멸), update도 ALTER가 락을 잡아 서비스가 멈출 수 있다. 스키마 변경은 마이그레이션 도구(Flyway 등)로 DDL을 직접 관리.
- 자동 생성된 DDL은 **개발 장비에서만** 쓰고, 운영 반영 전엔 다듬어서 쓴다.

---

## 7. 네이밍 전략 — 카멜 → 언더스코어는 자동

- 스프링부트(Hibernate 기본 `PhysicalNamingStrategy`)는 **camelCase 필드를 snake_case 컬럼으로 자동 변환**한다: `orderDate` → `order_date`, `Member` → `member`.
- 그러니 `@Column(name = "order_date")`처럼 **변환 결과를 그대로 다시 적는 건 노이즈**다. 명시가 필요한 경우는 자동 규칙과 다른 이름을 써야 할 때(레거시 테이블 매핑 등)뿐.
- 회사 컨벤션이 다르면 `spring.jpa.hibernate.naming.physical-strategy`로 전략 교체.

---

## 8. 💡 판단 기준

- **새 엔티티 체크리스트**: ① `protected` 기본 생성자 + 생성 메서드 ② setter 대신 비즈니스 메서드 ③ 연관관계 전부 LAZY ④ 컬렉션 필드 초기화(교체 금지) ⑤ enum은 STRING — 다섯 개는 고민 없이 기계적으로 적용한다.
- 이 규칙들의 공통 뿌리는 하나다: **JPA는 "객체를 계속 감시(더티 체킹·프록시·컬렉션 래핑)"하는 프레임워크**라서, 감시를 깨는 행위(무분별한 setter, 컬렉션 교체, final)와 감시 비용을 키우는 행위(EAGER)를 막는 것.
- `hbm2ddl`은 "편의 기능은 로컬까지"로 선을 긋는다 — **운영 스키마는 코드가 아니라 마이그레이션이 관리**한다.

---

## 9. 참고

- 김영한, 자바 ORM 표준 JPA 프로그래밍 기본편 04장(엔티티 매핑) · 스프링부트와 JPA 활용1 2장(도메인 분석 설계 — 엔티티 설계 시 주의점)
- [Hibernate User Guide - Naming strategies](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#naming)
- [Vlad Mihalcea - The best way to map an enum](https://vladmihalcea.com/the-best-way-to-map-an-enum-type-with-jpa-and-hibernate/)
- 관련 노트: [연관관계 매핑](./relation-mapping.md) · [도메인 검증 위치](../design/domain-validation.md)

---

**학습 날짜**: 2026-08-12
**계기**: 활용1 2장 "엔티티 설계 시 주의점" + 기본편 04장을 합쳐, 새 엔티티를 만들 때마다 반복 적용할 규칙을 체크리스트로 정리 (특히 PersistentBag 때문에 컬렉션을 교체하면 안 되는 이유가 처음 납득돼서)

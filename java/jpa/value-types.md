# 값 타입(@Embeddable) — "식별자 없이 값으로만 사는 객체"의 설계 규칙

> **한 줄 요약**: 식별자를 갖고 추적·변경되는 건 **엔티티**, 값만 있으면 되는 건 **값 타입**이다. 값 타입은 `@Embeddable`로 묶어 엔티티 안에 임베드하되(테이블은 그대로), 반드시 **불변(immutable)** 으로 설계하고 **equals로 동등성 비교**해야 한다. 값 타입 **컬렉션**(@ElementCollection)은 변경 시 전체 삭제+재저장이라는 함정이 있어 실무에서는 일대다 엔티티로 대체하는 경우가 많다.

관련 노트: [영속성 전이와 고아 객체](./cascade-orphan-removal.md)

---

## 0. 왜 이걸 알아야 하나

- `Member`에 city/street/zipcode 세 필드가 널브러져 있다. "주소"라는 개념 하나로 묶고 싶다.
- 묶었더니 두 회원이 같은 `Address` 인스턴스를 공유하다가 **한 명 주소를 바꿨는데 둘 다 바뀌는** 사고가 났다.
- `List<Address>`를 `@ElementCollection`으로 매핑했더니 주소 하나 바꿨는데 **DELETE 후 전부 다시 INSERT**가 나간다.

값 타입은 "객체를 잘게 쪼개 쓰는" 편의 기능처럼 보이지만, **공유·변경을 어떻게 막느냐**가 본질이다.

---

## 1. 엔티티 vs 값 타입 — 판단이 먼저

| | 엔티티 타입 | 값 타입 |
|---|---|---|
| 정의 | `@Entity` | int, String, `@Embeddable` 등 |
| 식별자 | 있음 (id) | **없음** |
| 추적 | 값이 변해도 식별자로 계속 추적 | 변경하면 그냥 완전히 다른 값 |
| 생명주기 | 스스로 관리 (persist/remove) | **소유한 엔티티에 의존** |
| 공유 | 가능 (참조 공유가 정상) | **공유하면 안 됨** (복사해서 사용) |

> 💡 원본 강의의 기준: **"식별자가 필요하고, 지속해서 값을 추적·변경해야 한다면 그것은 값 타입이 아닌 엔티티."** 값 타입은 정말 값 타입이라 판단될 때만 쓴다. 예: "주소"는 바뀌면 그냥 새 주소지 이력 추적 대상이 아니면 값 타입, "주소 변경 이력을 남겨야 한다"면 엔티티다.

값 타입 세 분류: **기본값 타입**(int, Integer, String…) · **임베디드 타입**(@Embeddable, 복합 값) · **컬렉션 값 타입**(@ElementCollection). 이 노트의 주인공은 뒤의 둘.

---

## 2. @Embeddable / @Embedded 기본기

```java
@Embeddable
public class Address {
    private String city;
    private String street;
    private String zipcode;

    protected Address() {}  // 기본 생성자 필수 (JPA 스펙) — protected 권장

    public Address(String city, String street, String zipcode) { ... }

    // getter만. setter는 만들지 않는다 (4번 불변 설계)
}
```

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;

    @Embedded
    private Address homeAddress;
}
```

- `@Embeddable`: 값 타입을 **정의**하는 쪽에, `@Embedded`: **사용**하는 쪽에. (둘 중 하나만 있어도 동작하지만 둘 다 명시가 관례)
- **기본 생성자 필수.** 외부에서 무분별하게 못 쓰게 `protected`로 두는 게 관례.
- 값 타입도 `Period.isWork()`처럼 **자기 데이터를 쓰는 의미 있는 메서드**를 가질 수 있다 — 응집도가 임베디드 타입의 진짜 장점.

### 테이블 매핑은 그대로

- 임베디드 타입을 쓰기 **전과 후에 매핑하는 테이블은 같다.** DB 입장에서는 여전히 MEMBER 테이블의 CITY/STREET/ZIPCODE 컬럼일 뿐.
- 객체는 잘게(fine-grained) 쪼개고 테이블은 그대로 → **"잘 설계한 ORM 애플리케이션은 매핑한 테이블 수보다 클래스 수가 더 많다."**
- 임베디드 필드가 **null이면 매핑한 컬럼 전부 null**로 저장된다.

### 한 엔티티에서 같은 값 타입을 두 번 쓰면 — @AttributeOverrides

컬럼명이 중복되므로 재정의가 필요하다.

```java
@Embedded
private Address homeAddress;

@Embedded
@AttributeOverrides({
    @AttributeOverride(name = "city",    column = @Column(name = "WORK_CITY")),
    @AttributeOverride(name = "street",  column = @Column(name = "WORK_STREET")),
    @AttributeOverride(name = "zipcode", column = @Column(name = "WORK_ZIPCODE"))
})
private Address workAddress;
```

---

## 3. ⚠️ 공유 참조 부작용 — 값 타입 최대 함정

임베디드 타입은 자바 **객체**라서 참조가 그대로 공유된다.

```java
Address address = new Address("OldCity", "street", "10000");

member1.setHomeAddress(address);
member2.setHomeAddress(address);   // 같은 인스턴스 공유!

member1.getHomeAddress().setCity("NewCity");
// 의도: member1만 이사 / 실제: member2의 city도 UPDATE 됨 💥
```

- 기본 타입(int)은 대입하면 값이 **복사**되지만, 객체 타입은 **참조가 전달**된다. 자바 언어 차원에서 참조 대입 자체를 막을 방법은 없다.
- "복사해서 쓰면 되지"는 규약일 뿐 컴파일러가 강제해주지 못한다 → 사람이 실수한다.

### 해법: 불변 객체로 설계

> **"불변이라는 작은 제약으로 부작용이라는 큰 재앙을 막을 수 있다."** (기본편 09장)

- 생성자로만 값을 설정하고 **수정자(setter)를 만들지 않는다.** (Integer, String이 자바의 대표 불변 객체)
- 값을 바꾸고 싶으면 수정이 아니라 **새 인스턴스로 통째로 교체**한다:

```java
// setCity() 같은 부분 수정 대신
member.setHomeAddress(new Address("NewCity", old.getStreet(), old.getZipcode()));
```

- 부수 효과: 교체 방식은 더티 체킹도 명확해진다(필드 몇 개를 몰래 바꾸는 게 아니라 값 전체가 바뀜) → [persistence-context.md](./persistence-context.md)의 스냅샷 비교와 자연스럽게 맞물린다.

---

## 4. 값 타입의 비교 — equals/hashCode 재정의

- **동일성(identity)** 비교: 참조 비교, `==`
- **동등성(equivalence)** 비교: 값 비교, `equals()`

값 타입은 인스턴스가 달라도 **안의 값이 같으면 같은 것**으로 봐야 하므로 `equals()`를 재정의해 동등성 비교를 해야 한다(주로 모든 필드 사용, `hashCode()`도 함께 — HashMap/HashSet에서 깨지지 않게).

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Address)) return false;   // 프록시 대비 instanceof
    Address address = (Address) o;
    return Objects.equals(city, address.city)
        && Objects.equals(street, address.street)
        && Objects.equals(zipcode, address.zipcode);
}

@Override
public int hashCode() { return Objects.hash(city, street, zipcode); }
```

- 필드 접근을 getter 경유로 작성하면 프록시로 감싸진 경우에도 안전하다 (→ [proxy.md](./proxy.md)의 타입 비교 함정과 같은 맥락).

---

## 5. ⚠️ 값 타입 컬렉션 (@ElementCollection) — 알고 쓰면 무섭다

```java
@ElementCollection
@CollectionTable(name = "FAVORITE_FOOD",
    joinColumns = @JoinColumn(name = "MEMBER_ID"))
@Column(name = "FOOD_NAME")
private Set<String> favoriteFoods = new HashSet<>();

@ElementCollection
@CollectionTable(name = "ADDRESS",
    joinColumns = @JoinColumn(name = "MEMBER_ID"))
private List<Address> addressHistory = new ArrayList<>();
```

- 컬렉션은 한 테이블에 못 넣으므로 **별도 테이블**로 매핑된다. 값 타입이라 자체 생명주기가 없고, 사실상 **cascade ALL + 고아 객체 제거를 필수로 가진 것**처럼 동작하며 **지연 로딩**이 기본.

### 제약사항 (실무에서 안 쓰는 이유)

1. **변경 시 전체 삭제 + 재INSERT**: 값 타입은 식별자가 없어 어느 행이 바뀌었는지 추적할 수 없다. 그래서 컬렉션에 변경이 생기면 **주인 엔티티와 연관된 데이터를 전부 DELETE하고, 현재 컬렉션 값을 전부 다시 INSERT**한다. 주소 100개 중 1개 바꿨는데 DELETE 1번 + INSERT 100번.
2. **PK 구성 제약**: 매핑 테이블은 **모든 컬럼을 묶어 기본 키**를 구성해야 한다 (null 입력 불가, 중복 저장 불가). 식별자 컬럼(id)을 넣으면 그건 이미 값 타입이 아니라 엔티티다.

### 실무 대안: 일대다 엔티티로 승격

```java
@Entity
public class AddressEntity {   // 값 타입을 엔티티로 한 번 감싼다
    @Id @GeneratedValue
    private Long id;

    @Embedded
    private Address address;   // 값 타입은 안에서 재사용
}

// Member
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
@JoinColumn(name = "MEMBER_ID")
private List<AddressEntity> addressHistory = new ArrayList<>();
```

- id가 생기니 개별 행 UPDATE/DELETE가 가능해지고, cascade ALL + orphanRemoval로 **값 타입 컬렉션처럼 생명주기를 부모에 묶어** 쓴다 (→ [cascade-orphan-removal.md](./cascade-orphan-removal.md)).
- 값 타입 컬렉션은 **"치킨/피자 선호" 체크박스처럼 정말 단순한 것**(추적 불필요, 값 바뀌어도 전체 교체가 자연스러운 것)에만 쓴다.

### @ElementCollection vs 일대다 엔티티

| | @ElementCollection | @OneToMany + 엔티티 (cascade ALL, orphanRemoval) |
|---|---|---|
| 식별자 | 없음 | 있음 (id) |
| 변경 SQL | **전체 DELETE + 전체 재INSERT** | 해당 행만 UPDATE/DELETE |
| 테이블 PK | 모든 컬럼 묶음 (null·중복 불가) | id (유연) |
| 생명주기 | 부모에 완전 종속 (자동) | cascade/orphanRemoval로 동일하게 구성 |
| 적합한 곳 | 정말 단순한 다중 값 | 그 외 대부분의 실무 케이스 |

---

## 6. 💡 판단 기준

- **첫 질문은 항상 "이거 식별자 갖고 추적해야 하나?"** — 그렇다면 엔티티. 아니고 값이 통째로 바뀌는 게 자연스러우면 값 타입. (주소 이력을 남겨라 → 이미 엔티티라는 신호)
- **값 타입을 만들면 무조건 불변으로.** setter를 지우고 생성자+교체 방식으로 강제해야, "공유 참조 부작용"이라는 재현도 어려운 버그를 설계 단계에서 차단한다.
- **@ElementCollection은 기본 보류.** 변경이 조금이라도 있는 컬렉션이면 일대다 엔티티 + cascade ALL + orphanRemoval로 시작하는 게 안전하다. 값 타입 컬렉션은 "전체 갈아끼워도 아무렇지 않은 단순 값 목록"에만.
- 값 타입 도입의 실익은 컬럼 묶기가 아니라 **검증·행위의 응집**(생성자 검증, `isWork()` 같은 메서드)이다. 묶기만 하고 로직이 없다면 굳이 서두를 필요 없다.

---

## 7. 참고

- 김영한, 자바 ORM 표준 JPA 프로그래밍 - 기본편, 09장 값 타입
- [Hibernate User Guide - Embeddable types](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#embeddables)
- [Baeldung - JPA @Embedded and @Embeddable](https://www.baeldung.com/jpa-embedded-embeddable)
- [Vlad Mihalcea - @ElementCollection 성능 이슈](https://vladmihalcea.com/how-to-optimize-unidirectional-collections-with-jpa-and-hibernate/)
- 관련 노트: [영속성 전이와 고아 객체](./cascade-orphan-removal.md) · [영속성 컨텍스트](./persistence-context.md)

---

**학습 날짜**: 2026-08-12
**계기**: JPA 기본편 09장을 들으며 "엔티티 vs 값 타입" 판단 기준과, 공유 참조 부작용을 불변 설계로 막는 이유, 값 타입 컬렉션을 실무에서 일대다 엔티티로 대체하는 이유를 정리

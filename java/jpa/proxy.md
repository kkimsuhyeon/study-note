# 프록시와 지연 로딩 — getReference()가 돌려주는 "가짜 객체"의 정체

> **한 줄 요약**: `em.getReference()`(스프링 데이터의 `getReferenceById()`)는 DB 조회를 미루는 **가짜(프록시) 객체**를 반환한다. 지연 로딩(LAZY)도 연관 엔티티 자리에 이 프록시를 넣어두는 같은 메커니즘이다. 프록시는 **실제 사용되는 순간 영속성 컨텍스트를 통해 초기화**되므로, 영속성 컨텍스트가 닫힌 뒤(준영속) 건드리면 `LazyInitializationException`이 터진다.

관련 노트: [영속성 컨텍스트](./persistence-context.md) · [N+1과 fetch join](./n-plus-one-fetch.md) · [OSIV](./osiv.md)

---

## 0. 왜 이걸 알아야 하나

- `@ManyToOne(fetch = LAZY)`를 걸었더니 `member.getTeam()`이 이상한 클래스(`Team$HibernateProxy$...`)를 돌려준다.
- 트랜잭션 밖에서 지연 로딩을 건드렸다가 `LazyInitializationException`을 만났다.
- `findById` 대신 `getReferenceById`를 쓰면 쿼리가 안 나간다는데, 언제 써야 하는지 모르겠다.

셋 다 뿌리는 하나 — **프록시**다. 프록시의 초기화 메커니즘을 잡으면 지연 로딩·N+1·OSIV 논의가 전부 이 위에 올라간다.

---

## 1. em.find() vs em.getReference()

| | `em.find()` / `findById()` | `em.getReference()` / `getReferenceById()` |
|---|---|---|
| 반환 | DB를 조회한 **실제 엔티티** | DB 조회를 미루는 **프록시 객체** |
| SQL 시점 | 호출 즉시 SELECT | **실제 값을 사용하는 순간** SELECT |
| 대상이 없을 때 | null / `Optional.empty` | 사용 시점에 `EntityNotFoundException` |

```java
Member member = em.getReference(Member.class, 1L); // SQL 안 나감
member.getId();    // id는 이미 알고 있으므로 여전히 SQL 안 나감 (프록시가 들고 있음)
member.getName();  // ← 이 순간 초기화: SELECT 발사
```

### 프록시의 구조

- **실제 엔티티 클래스를 상속**받아 만들어진다 → 겉모양이 같아서 사용하는 쪽은 (이론상) 구분 없이 쓰면 된다.
- 내부에 실제 객체의 참조(`target`)를 보관하고, 메서드가 호출되면 `target`의 메서드로 **위임(delegate)** 한다.
- 처음엔 `target = null`. 실제 값이 필요해질 때 채운다.

### 초기화 과정 (핵심 그림)

```
member.getName() 호출
  1. 프록시: target이 null이네?
  2. 영속성 컨텍스트에 초기화 요청
  3. 영속성 컨텍스트가 DB 조회
  4. 실제 Entity 생성, 영속성 컨텍스트에 보관
  5. 프록시의 target에 연결 → 이후 target.getName()으로 위임
```

- 초기화는 **처음 사용할 때 한 번만** 일어난다.
- ⚠️ 초기화돼도 **프록시가 실제 엔티티로 교체되는 게 아니다.** 프록시 객체는 그대로 있고, 그 안의 target을 통해 실제 엔티티에 접근하게 될 뿐이다. → 그래서 초기화 이후에도 `getClass()`는 여전히 프록시 클래스다.
- 초기화의 주체는 프록시 자신이 아니라 **영속성 컨텍스트**다. 이게 아래 3번 함정(준영속)의 복선.

---

## 2. 지연 로딩 = 연관 필드에 프록시를 넣어두는 것

```java
@Entity
public class Member {
    @ManyToOne(fetch = FetchType.LAZY)  // 기본값은 EAGER라서 명시 필요
    @JoinColumn(name = "TEAM_ID")
    private Team team;
}
```

```java
Member member = em.find(Member.class, 1L); // Member만 SELECT, team 자리엔 프록시
Team team = member.getTeam();              // 아직 SQL 안 나감 (프록시 반환)
team.getName();                            // ← 실제 사용 시점에 초기화 (Team SELECT)
```

- `getReference()`를 내가 직접 부르는 일은 드물지만, **LAZY 연관관계는 JPA가 자동으로 같은 짓을 해준다.** 프록시를 이해해야 지연 로딩이 이해되는 이유.
- 즉시 로딩(EAGER)은 조회 시점에 조인으로 함께 가져오지만, **실무에서는 전부 LAZY로 깔고** 함께 필요한 경우만 fetch join 등으로 해결하는 게 원칙 (즉시 로딩은 JPQL에서 N+1을 일으킨다 → [n-plus-one-fetch.md](./n-plus-one-fetch.md)).
- `@ManyToOne`·`@OneToOne`은 **기본이 EAGER**라 직접 LAZY로 바꿔야 하고, `@OneToMany`·`@ManyToMany`는 기본이 LAZY다.

---

## 3. ⚠️ 프록시 3대 함정

### (1) 타입 비교는 == 이 아니라 instanceof

프록시는 원본을 **상속**한 별개 클래스다. `getClass()` 비교는 실패한다.

```java
Member m1 = em.find(Member.class, 1L);         // 실제 엔티티
Member m2 = em.getReference(Member.class, 2L); // 프록시

m1.getClass() == m2.getClass();  // false! (Member vs Member$HibernateProxy)
m2 instanceof Member;            // true ← 이걸 써야 함
```

equals를 직접 구현할 때도 `obj.getClass() == this.getClass()` 대신 `obj instanceof Member`로 비교해야 프록시가 섞여도 안전하다.

### (2) 영속성 컨텍스트에 이미 있으면 getReference도 실제 엔티티를 반환한다

JPA는 **같은 트랜잭션 안에서 같은 엔티티의 == 동일성을 보장**해야 한다. 그래서 반환 타입이 상황에 따라 달라진다.

```java
// 원본이 먼저 올라간 경우
Member m1 = em.find(Member.class, 1L);         // 실제 엔티티 영속화
Member m2 = em.getReference(Member.class, 1L); // 프록시가 아니라 실제 엔티티 반환!
m1 == m2; // true

// 역순: 프록시가 먼저면
Member p1 = em.getReference(Member.class, 1L); // 프록시
Member p2 = em.find(Member.class, 1L);         // find인데도 프록시 반환!
p1 == p2; // true (동일성 보장이 우선)
```

> 교훈: **"getReference니까 프록시겠지"라는 가정으로 코드를 짜면 안 된다.** 프록시일 수도, 실제 엔티티일 수도 있다는 전제로 (instanceof, isLoaded) 다뤄야 한다.

### (3) 준영속 상태에서 초기화하면 LazyInitializationException

초기화는 영속성 컨텍스트에게 부탁하는 것이므로, **영속성 컨텍스트가 닫혔거나(트랜잭션 종료) detach된 상태**면 초기화할 방법이 없다.

```java
Member member = em.getReference(Member.class, 1L);
em.detach(member); // 또는 em.close() / 트랜잭션 종료

member.getName(); // 💥 org.hibernate.LazyInitializationException
```

실무에서 만나는 형태는 대부분 이것이다:

```java
// 서비스: @Transactional 안에서 조회만 하고 반환
Member member = memberService.find(id); // 트랜잭션 끝 → 영속성 컨텍스트 닫힘

// 컨트롤러/뷰: 트랜잭션 밖에서 지연 로딩 접근
member.getTeam().getName(); // 💥 could not initialize proxy
```

> "**트랜잭션 밖에서 지연 로딩**"이 터지는 근본 원인. 스프링 부트에서 이게 뷰 렌더링까지 안 터지고 동작하는 건 OSIV(open-in-view) 기본값 덕분인데, 그 트레이드오프는 [osiv.md](./osiv.md)에서 정리.

---

## 4. 프록시 상태 확인·강제 초기화 유틸

```java
// 초기화 여부 확인 (초기화를 일으키지 않음)
emf.getPersistenceUnitUtil().isLoaded(entity);

// 프록시인지 클래스로 확인
entity.getClass().getName(); // ..HibernateProxy.. 가 섞여 있으면 프록시

// 강제 초기화 (Hibernate 제공)
org.hibernate.Hibernate.initialize(entity);
```

- JPA 표준에는 강제 초기화 메서드가 없다. 표준만으로는 `entity.getName()` 같은 **강제 호출**로 초기화한다.
- 트랜잭션 안에서 미리 초기화해서 내보내야 할 때 `Hibernate.initialize()`가 의도를 드러내기 좋다.

---

## 5. findById vs getReferenceById (실무 사용처)

| | `findById` | `getReferenceById` |
|---|---|---|
| SQL | 즉시 SELECT | 사용 전까지 없음 |
| 용도 | **엔티티의 데이터가 실제로 필요**할 때 | **FK 세팅 등 참조(id)만 필요**할 때 |
| 없는 id | Optional.empty로 바로 알 수 있음 | 사용(flush) 시점에야 예외 |

```java
// 주문 생성: member의 데이터는 안 읽고 FK만 채우면 됨
Order order = new Order();
order.setMember(memberRepository.getReferenceById(memberId)); // SELECT 없이 INSERT만
```

- 조회 쿼리 한 번을 아끼는 최적화. 단, **존재하지 않는 id여도 즉시 알 수 없다**는 트레이드오프가 있다(무결성은 FK 제약이 최종 방어).

---

## 6. 💡 판단 기준

- **연관관계는 전부 LAZY로 깔고 시작한다.** 함께 조회할 필요는 쿼리(fetch join, EntityGraph)로 그때그때 해결 — 로딩 전략을 엔티티에 박아두면(EAGER) 모든 쿼리가 영향을 받는다.
- **findById vs getReferenceById**: 그 엔티티의 필드 값을 읽어야 하면 find, FK 채우기용 참조만 필요하면 getReference.
- **LazyInitializationException을 만나면** "왜 여기서 초기화가 안 되지?"가 아니라 **"왜 트랜잭션(영속성 컨텍스트) 밖에서 엔티티를 건드리고 있지?"** 를 물어야 한다. 해법은 강제 초기화가 아니라 경계 설계(트랜잭션 안에서 필요한 데이터를 다 갖춘 DTO로 변환해 내보내기)가 정석.
- 프록시 여부에 의존하는 코드(== 타입 비교, getClass 스위칭)는 깨진다. **프록시일 수도 있다는 전제**(instanceof, isLoaded)로 작성한다.

---

## 7. 참고

- 김영한, 자바 ORM 표준 JPA 프로그래밍 - 기본편, 08장 프록시와 연관관계 관리
- [Hibernate User Guide - Proxies and lazy fetching](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#BytecodeEnhancement)
- [Baeldung - JPA Proxy Objects](https://www.baeldung.com/hibernate-proxy-load-method)
- 관련 노트: [영속성 컨텍스트](./persistence-context.md) · [N+1과 fetch join](./n-plus-one-fetch.md) · [OSIV](./osiv.md)

---

**학습 날짜**: 2026-08-12
**계기**: JPA 기본편 08장을 들으며 지연 로딩이 실제로 어떻게 동작하는지(프록시 초기화), 그리고 트랜잭션 밖 지연 로딩에서 LazyInitializationException이 터지는 근본 원인을 정리

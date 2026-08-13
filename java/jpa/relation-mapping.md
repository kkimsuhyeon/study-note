# 연관관계 매핑 — 주인 · mappedBy · 양방향

> **한 줄 요약**: 객체의 "양방향"은 사실 **서로 다른 단방향 2개**고, 테이블은 **FK 하나**로 끝난다. 그래서 두 참조 중 **누가 FK를 관리하나(= 연관관계의 주인)** 를 정해야 하며, 판별 기준은 단 하나 — **FK가 있는 쪽(N쪽)이 주인**. 주인이 아닌 쪽에만 값을 넣으면 **FK가 null로 INSERT**되는 게 이 주제의 최다 사고다.

관련 노트: [애그리거트 소유권 & 참조 방향](../design/aggregate-ownership.md) (설계 관점에서 "어느 쪽이 참조를 갖나") · [N+1과 fetch 전략](./n-plus-one-fetch.md) · [영속성 컨텍스트](./persistence-context.md)

---

## 0. 왜 이걸 알아야 하나

- **테이블**: FK 하나(`MEMBER.TEAM_ID`)로 양쪽 어디서든 JOIN 가능. 애초에 **방향이라는 개념이 없다.**
- **객체**: 참조 필드가 있는 쪽으로만 갈 수 있다. `member.getTeam()`은 되지만, Team에 컬렉션이 없으면 `team.getMembers()`는 불가능.

객체를 양방향으로 만들려면 참조를 **양쪽에 하나씩(단방향 × 2)** 만들어야 하는데, 테이블엔 FK가 하나뿐이니 **"둘 중 누가 그 FK를 건드리나"** 를 JPA에게 알려줘야 한다. 이게 연관관계의 주인(Owner)이고, `mappedBy`가 그 지정 수단이다.

---

## 1. 기본 매핑 (N:1 단방향 → 양방향)

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)   // ⚠️ ToOne 기본값은 EAGER — LAZY 명시 (아래 7절)
    @JoinColumn(name = "TEAM_ID")        // FK 컬럼 지정 → 이 필드가 연관관계의 주인
    private Team team;
}

@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "team")        // "주인은 Member.team이다" — 나는 읽기 전용 거울
    private List<Member> members = new ArrayList<>();
}
```

- 단방향(`Member.team`)만으로 **연관관계 매핑은 이미 완료**다. 테이블 모습은 양방향으로 만들어도 전혀 달라지지 않는다.
- 양방향은 거기에 **반대 방향 조회(객체 그래프 탐색) 기능이 추가된 것뿐**이다.

---

## 2. 연관관계의 주인 — 규칙과 판별

**양방향 매핑 규칙** (기본편 05장):

| | 주인 (owner) | 주인이 아닌 쪽 (mappedBy 붙은 쪽) |
|---|---|---|
| FK 등록/수정 | **O — 이 참조만 DB에 반영됨** | X (값을 바꿔도 DB에 아무 일 없음) |
| 조회 | O | O (읽기만 가능) |
| mappedBy | 사용 안 함 | `mappedBy = "주인 필드명"` 필수 |

**누가 주인인가 = FK가 있는 곳.** N:1이면 항상 N쪽(FK 보유)이 주인이다.

- ⚠️ **비즈니스 중요도로 정하는 게 아니다.** Team이 비즈니스상 더 중요해 보여도 주인은 Member다. 김영한의 비유: **자동차(Team)와 바퀴(Member)** — 자동차가 중요하지만 FK는 바퀴 테이블에 있으니 바퀴가 주인. 주인이라는 말은 "지위가 높다"가 아니라 "**FK 관리 책임이 있다**"는 뜻일 뿐이다.
- 주인을 반대(1쪽)로 잡으면: Team을 수정했는데 MEMBER 테이블에 UPDATE가 나가는, 읽는 사람이 예측 못 하는 구조가 된다. → 그래서 "FK 위치 기준"이 규칙이다.

---

## 3. ⚠️ 최다 실수 — 주인이 아닌 쪽에만 값 세팅

```java
Team team = new Team();
team.setName("TeamA");
em.persist(team);

Member member = new Member();
member.setName("member1");

team.getMembers().add(member);   // ⚠️ 역방향(주인 아닌 쪽)만 설정
em.persist(member);
```

| ID | USERNAME | TEAM_ID |
|----|----------|---------|
| 1 | member1 | **null** 💥 |

`Team.members`는 mappedBy가 붙은 **가짜 매핑(읽기 전용)** 이라, 아무리 add해도 JPA는 무시한다. FK를 채우는 건 오직 주인 `member.setTeam(team)` 뿐이다.

```java
team.getMembers().add(member);  // (순수 객체를 위해)
member.setTeam(team);           // ★ 주인에 값 설정 — 이 줄이 있어야 TEAM_ID가 채워진다
em.persist(member);
```

---

## 4. 양쪽 다 세팅 + 연관관계 편의 메서드

"주인에만 넣으면 DB는 맞는데, 왜 양쪽 다 넣으라 하나?" → **순수 객체 상태**를 위해서다.

- flush 전에(같은 영속성 컨텍스트에서) `team.getMembers()`를 읽으면, 주인에만 넣은 경우 컬렉션이 비어 있다 — DB는 맞는데 **메모리의 객체 그래프는 깨진 상태**.
- 테스트 코드는 JPA 없이 순수 객체로도 돌아가야 한다.

두 줄을 매번 짝으로 부르는 건 잊기 쉬우니 **한 메서드로 묶는다**(연관관계 편의 메서드):

```java
@Entity
public class Member {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "TEAM_ID")
    private Team team;

    // 편의 메서드 — 원자적으로 양쪽 동기화 (한쪽에만 두기. 양쪽에 두면 무한루프)
    public void changeTeam(Team team) {
        this.team = team;
        team.getMembers().add(this);
    }
}
```

> 편의 메서드를 어느 엔티티에 둘지는 설계 판단(주로 주도하는 쪽) — [애그리거트 소유권](../design/aggregate-ownership.md) 참고.

### ⚠️ 양방향 무한루프 3형제

서로가 서로를 참조하므로 **양쪽을 다 순회하는 코드는 전부 무한루프** 위험:

- `toString()` — 서로 호출하다 StackOverflowError
- **Lombok** `@ToString`, `@EqualsAndHashCode` — 자동 생성이라 더 잘 당함 (연관 필드 exclude 필수)
- **JSON 직렬화** — 컨트롤러에서 엔티티를 직접 반환하면 Jackson이 무한 순회. → 엔티티를 API 응답으로 직접 내보내지 말고 DTO로 변환

---

## 5. 실무 지침 — 설계는 단방향으로 끝내라

- **단방향 매핑만으로 설계를 완료**한다. (JPA 매핑은 단방향으로 이미 완성)
- 양방향은 "반대 방향 탐색이 실제로 필요해졌을 때" **나중에 추가** — 테이블에 전혀 영향을 주지 않으므로 필요 시점에 필드만 얹으면 된다.
- JPQL에서 역방향 탐색을 자주 쓰게 되면 그때가 추가할 때다.

---

## 6. 관계별 실무 규칙 (06장 종합)

| 관계 | 실무 규칙 | 이유 / 함정 |
|------|-----------|------------|
| **N:1** (`@ManyToOne`) | **기본으로 선택.** 필요하면 양방향 | FK 있는 쪽이 주인이라 구조가 직관적. 가장 많이 쓰는 형태 |
| **1:N 단방향** (`@OneToMany`+`@JoinColumn`) | **쓰지 말 것** | 주인(1쪽)이 관리하는 FK가 **남의 테이블**에 있음 → 저장 시 **추가 UPDATE SQL** 발생. ⚠️ `@JoinColumn` 생략하면 **조인 테이블(중간 테이블)이 자동 생성**되는 함정 |
| 1:N 양방향 | 공식적으로 없음 | `@JoinColumn(insertable=false, updatable=false)` 읽기 전용 트릭이 있으나 그냥 **N:1 양방향을 써라** |
| **1:1** (`@OneToOne`) | 주 테이블에 FK 권장 | **대상 테이블에 FK**가 있으면 프록시 한계로 **지연 로딩 설정해도 항상 즉시 로딩**됨. (대상 테이블 FK 단방향은 JPA 지원 자체가 없음) |
| **N:M** (`@ManyToMany`) | **실무 금지** | 연결 테이블에 주문시간·수량 같은 컬럼을 못 얹음(필드 추가 X). → **연결 테이블을 엔티티로 승격**해 `@OneToMany` + `@ManyToOne`으로 풀기 |

---

## 7. ToOne은 전부 LAZY 명시

fetch 기본값이 어노테이션마다 다르다:

| 어노테이션 | fetch 기본값 |
|-----------|-------------|
| `@ManyToOne`, `@OneToOne` (ToOne) | **EAGER** ⚠️ |
| `@OneToMany`, `@ManyToMany` (ToMany) | LAZY |

ToOne을 기본값대로 두면 조회마다 연관 엔티티를 즉시 끌고 오고, JPQL과 만나면 N+1로 직행한다. → **모든 연관관계에 `fetch = FetchType.LAZY` 명시**가 실무 표준. 상세: [N+1과 fetch 전략](./n-plus-one-fetch.md)

---

## 8. 💡 판단 기준

- **"어느 쪽이 주인?" → 고민하지 말고 FK 있는 쪽(N쪽).** 비즈니스 중요도·주도권은 판단 재료가 아니다.
- **값 세팅은 "주인 필수, 양쪽 권장"** — DB를 채우는 건 주인이고, 객체 그래프를 위해 양쪽을 편의 메서드로 묶는다. `team.getMembers().add()`만 있는 코드를 보면 FK null 사고를 의심하라.
- **설계 단계에선 단방향만.** 양방향·`mappedBy`는 매핑 설계가 아니라 "조회 편의 옵션"이다 — 필요해질 때 추가해도 테이블은 그대로다.
- 관계 선택이 고민되면: **N:1 양방향으로 수렴**시키는 게 06장의 결론이다 (1:N 단방향·N:M은 함정 쪽).

---

## 9. 참고

- 김영한, 자바 ORM 표준 JPA 프로그래밍 기본편 — 05장 연관관계 매핑 기초 · 06장 다양한 연관관계 매핑
- [Hibernate User Guide - Associations](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#associations)
- [Baeldung - Hibernate One to Many](https://www.baeldung.com/hibernate-one-to-many)
- 관련 노트: [애그리거트 소유권](../design/aggregate-ownership.md) · [N+1과 fetch 전략](./n-plus-one-fetch.md)

---

**학습 날짜**: 2026-08-12
**계기**: 김영한 JPA 기본편 05·06장 수강 — mappedBy가 왜 필요한지(객체 양방향 = 단방향 2개 vs 테이블 FK 1개)와, 역방향에만 값 넣어 FK가 null로 들어가는 최다 실수를 정리

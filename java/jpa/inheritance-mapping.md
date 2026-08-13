# 상속 매핑 — JOINED / SINGLE_TABLE / TABLE_PER_CLASS + @MappedSuperclass

> **한 줄 요약**: 객체의 상속을 테이블로 옮기는 전략은 3가지 — **JOINED**(정규화, 조인 비용), **SINGLE_TABLE**(기본값, 빠르지만 컬럼 전부 null 허용), **TABLE_PER_CLASS**(둘 다 비추천). 그리고 **@MappedSuperclass는 상속 매핑이 아니라** 등록일/수정일 같은 공통 매핑 정보만 물려주는 도구(BaseEntity 패턴)다 — 둘을 혼동하지 말 것.

관련 노트: [엔티티 설계 실무 규칙](./entity-design-rules.md) · [연관관계 매핑](./relation-mapping.md)

---

## 1. 언제 쓰나

`Item ← Book/Album/Movie`처럼 **"is-a" 관계의 엔티티들**을 매핑할 때. RDB에는 상속이 없으므로 JPA가 3가지 흉내 전략을 제공한다.

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)   // 전략 선택 (부모에)
@DiscriminatorColumn                              // DTYPE 컬럼 (자식 구분자)
public abstract class Item {
    @Id @GeneratedValue private Long id;
    private String name;
    private int price;
}

@Entity
@DiscriminatorValue("B")                          // DTYPE에 들어갈 값 (기본: 클래스명)
public class Book extends Item {
    private String author;
    private String isbn;
}
```

## 2. 3가지 전략 비교

| | JOINED (조인) | SINGLE_TABLE (단일, **기본값**) | TABLE_PER_CLASS |
|---|---|---|---|
| 테이블 | 부모 1 + 자식별 1 | 전부 한 테이블 | 자식별 1 (부모 없음) |
| 조회 | 부모-자식 **조인** 필요 | 조인 없음 → **빠름** | 부모 타입 조회 시 **UNION** 💥 |
| 저장 | **INSERT 2번** (부모+자식) | 1번 | 1번 |
| 무결성 | ✅ 정규화·FK·저장공간 효율 | ❌ 자식 컬럼 **전부 null 허용** | — |
| DTYPE | 사용 (@DiscriminatorColumn) | **필수** (없으면 행 구분 불가) | 불필요 |

- **JOINED**: 정석. 테이블 정규화, `not null` 제약 유지 가능, 저장공간 효율. 대가는 조회 조인 + INSERT 2번.
- **SINGLE_TABLE**: 성능 단순함이 장점. 대가는 자식 전용 컬럼이 모두 null 허용이 되고, 자식이 많아지면 테이블이 비대해져 오히려 느려질 수 있음.
- **TABLE_PER_CLASS**: **"DB 설계자와 ORM 전문가 둘 다 추천하지 않는다"** (강의 원문). 부모 타입으로 조회하면 자식 테이블 전부를 UNION — 쓰지 말 것.

## 3. @MappedSuperclass — 상속 매핑이 아니다 (BaseEntity 패턴)

```java
@MappedSuperclass
public abstract class BaseEntity {
    private LocalDateTime createdDate;
    private LocalDateTime lastModifiedDate;
    private String createdBy;
}

@Entity
public class Member extends BaseEntity { ... }   // 컬럼만 물려받음
```

- **테이블과 매핑되지 않는다** — 자식 엔티티에 **매핑 정보(컬럼)만 물려주는** 용도. 등록일/수정일/등록자처럼 전 엔티티 공통 필드에 쓴다.
- 엔티티가 아니다 → `em.find(BaseEntity.class, ...)` **불가**, 조회·검색 대상 아님.
- 직접 생성할 일이 없으므로 **추상 클래스 권장**.
- ⚠️ `@Entity` 클래스는 **@Entity 또는 @MappedSuperclass가 붙은 클래스만** 상속할 수 있다 — 아무 POJO나 부모로 못 쓴다.

### 상속 매핑 vs @MappedSuperclass (혼동 주의)

| | @Inheritance (상속 매핑) | @MappedSuperclass |
|---|---|---|
| 관계 | 부모도 **엔티티** — "Book **is a** Item" | 부모는 엔티티 아님 — "공통 필드 복사" |
| 부모 타입 조회 | ✅ `em.find(Item.class, id)` | ❌ 불가 |
| 용도 | 진짜 도메인 상속 구조 | 등록일·수정일 등 반복 필드 제거 |

## 4. 💡 판단 기준

> **기본은 JOINED, 단순하고 자식 필드가 적으면 SINGLE_TABLE, TABLE_PER_CLASS는 금지.** 그리고 그 전에 — **정말 엔티티 상속이어야 하나?** 부터 의심하라. 상속 구조는 테이블에 영구히 박제되고 자식 추가마다 스키마가 흔들린다. 공통 필드 재사용이 목적이면 @MappedSuperclass, 행동 차이가 목적이면 상속 대신 컴포지션/enum 분기도 후보다.

---

## 5. 참고
- [Hibernate User Guide - Inheritance](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#entity-inheritance)
- 관련 노트: [엔티티 설계 실무 규칙](./entity-design-rules.md)

---

**학습 날짜**: 2026-08-13
**계기**: 김영한 JPA 기본편 07장 — 상속 매핑 3전략의 트레이드오프와, BaseEntity(@MappedSuperclass)가 상속 매핑과 전혀 다른 도구라는 것을 정리.

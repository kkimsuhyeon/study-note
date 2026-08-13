# 기본 키 생성 전략 — IDENTITY / SEQUENCE / allocationSize

> **한 줄 요약**: `@GeneratedValue` 전략 중 **IDENTITY는 식별자를 DB가 만들기 때문에 `em.persist()` 순간 즉시 INSERT가 나간다** — [쓰기 지연](./persistence-context.md)이 무력화되는 유일한 지점. SEQUENCE는 시퀀스에서 번호만 미리 받아오므로 쓰기 지연이 유지되고, `allocationSize`(기본 50)로 시퀀스 호출 자체도 묶는다. 식별자는 **Long + 대체키 + 생성 전략**이 정석 — 자연키(주민번호 등)는 쓰지 않는다.

관련 노트: [영속성 컨텍스트 · flush](./persistence-context.md) · [엔티티 설계 실무 규칙](./entity-design-rules.md)

---

## 1. 언제 쓰나

엔티티의 `@Id`를 직접 할당하지 않고 자동 생성할 때. 사실상 모든 엔티티에서 쓰는 코드지만, **전략에 따라 INSERT가 나가는 "시점"이 달라진다**는 걸 모르면 persistence-context 노트의 "쓰기 지연" 이해와 충돌한다.

```java
@Id @GeneratedValue(strategy = GenerationType.IDENTITY)  // MySQL AUTO_INCREMENT
private Long id;

@Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "member_seq")
@SequenceGenerator(name = "member_seq", sequenceName = "member_seq", allocationSize = 50)
private Long id;
```

## 2. 4가지 전략

| 전략 | 방식 | DB |
|---|---|---|
| **IDENTITY** | 기본 키 생성을 DB에 위임 (AUTO_INCREMENT) | MySQL, PostgreSQL(serial) |
| **SEQUENCE** | DB 시퀀스 오브젝트에서 번호 채번 | Oracle, PostgreSQL, H2 |
| TABLE | 키 생성 전용 테이블 흉내 | 모든 DB (성능↓, 잘 안 씀) |
| AUTO | 방언(dialect)에 따라 위 셋 중 자동 선택 (기본값) | — |

## 3. ⚠️ IDENTITY는 persist 즉시 INSERT — 쓰기 지연 무력화

영속성 컨텍스트가 엔티티를 관리하려면 **1차 캐시의 키인 식별자(@Id)가 반드시 필요**하다. 그런데 IDENTITY는 식별자를 **DB가 INSERT를 실행해야만** 알 수 있다:

```java
em.persist(member);   // IDENTITY: 이 순간 INSERT 즉시 실행! (식별자를 받아와야 하므로)
                      // SEQUENCE: 시퀀스에서 번호만 받고, INSERT는 flush까지 미룸
```

- IDENTITY: "모아뒀다 flush 때 발사"([쓰기 지연](./persistence-context.md) §2)가 **이 전략에서만 예외적으로 깨진다.** persist 호출 줄에서 SQL 로그가 바로 찍히는 이유.
- SEQUENCE: `call next value for member_seq`로 **번호만** 미리 받아 1차 캐시에 등록 → INSERT는 평소처럼 flush 때. 쓰기 지연·배치 INSERT 유지.
- 이 차이 때문에 **JDBC 배치 INSERT(hibernate.jdbc.batch_size)도 IDENTITY에선 동작하지 않는다** — 대량 INSERT 성능이 필요하면 SEQUENCE(또는 직접 채번) 고려.

## 4. SEQUENCE의 allocationSize — 시퀀스 호출도 묶는다

매 persist마다 시퀀스를 부르면 네트워크 왕복이 늘어난다. `allocationSize=50`(기본값)이면:

```
첫 persist: 시퀀스를 2번 호출해 1~50 범위를 통째로 확보 (메모리에 보관)
이후 49번의 persist: DB 호출 없이 메모리에서 번호 할당
51번째: 다시 시퀀스 호출해 51~100 확보
```

- ⚠️ **DB 시퀀스가 "1씩 증가"로 만들어져 있으면 `allocationSize=1`로 맞춰야 한다** — 안 맞으면 충돌/점프. (반대로 allocationSize=50을 쓰려면 DB 시퀀스도 `INCREMENT BY 50`)
- 서버 여러 대여도 안전 — 각자 자기 범위를 확보하므로 겹치지 않는다 (번호에 구멍은 생길 수 있음 — 정상).

## 5. 💡 판단 기준 — 식별자는 무엇으로?

> **기본 키 조건: null 아님 · 유일 · 변하면 안 됨.** "변하면 안 됨"을 미래까지 만족하는 자연키는 거의 없다 — 주민번호도 정책 변경(보관 금지)으로 못 쓰게 된 사례. 그래서 **비즈니스와 무관한 대체키(Long + @GeneratedValue)가 정석**이고, 자연키(이메일·주민번호)는 유니크 제약으로만 건다.

| 상황 | 선택 |
|---|---|
| MySQL 계열 | IDENTITY (persist 즉시 INSERT 감수) |
| 시퀀스 지원 DB + 대량 INSERT/배치 필요 | SEQUENCE + allocationSize |
| DB를 못 정하는 라이브러리성 코드 | AUTO |

---

## 6. 참고
- [Hibernate User Guide - Identifier generation](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#identifiers-generators)
- 관련 노트: [영속성 컨텍스트](./persistence-context.md)

---

**학습 날짜**: 2026-08-13
**계기**: 김영한 JPA 기본편 04장 — "쓰기 지연" 원칙의 유일한 예외가 IDENTITY(식별자를 DB가 만들어서 persist 즉시 INSERT)라는 것과, allocationSize의 정체(시퀀스 호출 묶기)를 정리.

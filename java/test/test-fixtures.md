# 테스트 픽스처 (Object Mother / Test Data Builder) — 사용법

> **한 줄 요약**: 테스트용 객체 생성 코드를 한 곳에 모으는 패턴. 두 형태가 있다 — **Object Mother**(이름 붙은 정적 팩토리: `UserFixture.withBalance(1000)`)와 **Test Data Builder**(체이닝: `aUser().balance(1000).build()`). 핵심 원칙은 **"그 테스트가 *바꾸는 값*만 받고, 나머지는 안에서 기본값으로 채운다."**

관련 노트: [JPA repository 테스트](./jpa-repository-test.md) · [테스트 작성 가이드](./test-writing-guide.md) · [AssertJ 사용법](./assertj.md)

---

## 1. Object Mother — 이름 붙은 정적 팩토리 (기본형)

가장 단순하고 흔한 형태. 의도가 메서드 이름에 드러난다.

```java
public class UserFixture {
    private static final String EMAIL = "test@test.com";    // 안 변하는 기본값은 안에 숨김
    private static final String PASSWORD = "password123";
    private static final BigDecimal DEFAULT_BALANCE = BigDecimal.ZERO;

    public static User withBalance(BigDecimal balance) {     // 잔액이 포인트인 테스트용
        return User.of(null, EMAIL, PASSWORD, balance, UserRole.USER);
    }

    public static User withEmail(String email) {            // 이메일이 포인트인 테스트용
        return User.of(null, email, PASSWORD, DEFAULT_BALANCE, UserRole.USER);
    }
}
```

```java
// 호출부 — 관심 있는 값만 보임
User a = UserFixture.withBalance(BigDecimal.valueOf(1000));
User b = UserFixture.withEmail("b@test.com");
```

---

## 2. Test Data Builder — 체이닝 (변형이 많아질 때)

필드 조합이 많아 정적 팩토리가 폭발하면(`withBalance`, `withEmail`, `withBalanceAndRole`...) 빌더로 전환한다.

```java
public class UserBuilder {
    private String email = "test@test.com";        // 기본값
    private BigDecimal balance = BigDecimal.ZERO;
    private UserRole role = UserRole.USER;

    public static UserBuilder aUser() { return new UserBuilder(); }   // 진입점

    public UserBuilder email(String v)   { this.email = v;   return this; }   // 체이닝 위해 this 반환
    public UserBuilder balance(BigDecimal v) { this.balance = v; return this; }
    public UserBuilder role(UserRole v)  { this.role = v;    return this; }

    public User build() { return User.of(null, email, password, balance, role); }
}
```

```java
// 필요한 필드만 골라 덮어쓰기 — 나머진 빌더 기본값
User admin = aUser().email("admin@test.com").role(UserRole.ADMIN).build();
```

| | Object Mother | Test Data Builder |
|---|---|---|
| 형태 | `UserFixture.withX(...)` | `aUser().x(..).y(..).build()` |
| 적합 | 변형 2~3개, 의도 이름이 중요 | 필드 조합 다양, 일부만 덮어쓰기 |
| 비용 | 가벼움 | 빌더 클래스 작성 |

> 시작은 Object Mother. 메서드가 5~6개로 늘면 그때 Builder. 미리 Builder부터 만들 필요 없음(YAGNI).

---

## 3. ⭐ 핵심 원칙: "변하는 축만 받는다"

```java
withBalance(BigDecimal balance)   // balance만 받음 — email/password/role은 기본값
withEmail(String email)           // email만 받음 — 나머지는 기본값
```

- `password`를 **안 받는다** → 어떤 테스트도 password 값을 검증/변경 안 하니까. 받으면 `"password123"`이 모든 호출에 반복되는 노이즈.
- 받는 기준: **"이 테스트가 이 값을 바꿔야 하나?"** → 아니면 숨긴다.

| 안티패턴 | 이유 |
|---|---|
| `of(id, email, password, balance, role)` 다 받기 | 노이즈 그대로, 픽스처 의미 없음 |
| 처음부터 다인자/빌더로 "유연하게" | 과한 확장성 (YAGNI) |

---

## 4. 여러 건 적재 (필터·목록 테스트)

### `@BeforeEach`로 공통 시딩

```java
@BeforeEach
void setUp() {
    em.persistAndFlush(UserEntity.create(UserFixture.withEmail("a@test.com")));
    em.persistAndFlush(UserEntity.create(UserFixture.withEmail("b@test.com")));
}
```
> `@DataJpaTest`는 테스트마다 롤백 → `@BeforeEach` 시딩도 매 테스트 깨끗이 초기화된다.

### 가변인자 헬퍼로 여러 건

```java
private void persistUsers(String... emails) {
    for (String email : emails) {
        em.persistAndFlush(UserEntity.create(UserFixture.withEmail(email)));
    }
}
// persistUsers("a@test.com", "b@test.com", "c@test.com");
```

> ⚠️ unique 컬럼(email) 주의 — 같은 값으로 여러 건 넣으면 flush에서 제약 위반. 그래서 `withEmail(email)`처럼 **변하는 값을 받게** 해야 함.

---

## 5. static 픽스처 vs 인스턴스 헬퍼

픽스처는 **순수 팩토리** = 테스트 인프라(`EntityManager`)에 의존하면 안 된다.

| | 무엇 | 어디에 | 이유 |
|---|---|---|---|
| static 픽스처 | 순수 객체 생성 | `UserFixture` | 의존성 없음 → 재사용 |
| 인스턴스 헬퍼 | DB 적재(`em.persistAndFlush`) | 테스트 클래스 private 메서드 | `em`이 인스턴스 필드라 static이 못 가짐 |

```java
public static User withEmail(String email) { ... }          // static 픽스처
private User persistedUser(String email) {                   // 인스턴스 헬퍼 (em 필요)
    return em.persistAndFlush(UserEntity.create(UserFixture.withEmail(email))).toModel();
}
```

---

## 6. 픽스처를 안 쓰는 경우

팩토리/생성자 **자체를 검증**하는 테스트는 픽스처로 우회하지 말고 직접 호출.

```java
// User.create()가 기본값(잔액 0, role USER)을 세팅하는지 = create가 테스트 대상
User user = User.create(EMAIL, PASSWORD);   // 픽스처(withBalance 등)로 우회하면 create 검증 못 함
assertThat(user.getRole()).isEqualTo(UserRole.USER);
```

---

## 7. 라이브러리 (참고)

수작업 대신 랜덤/자동 생성 라이브러리도 있다.

| 라이브러리 | 특징 |
|---|---|
| **Instancio** | 모든 필드 자동 채움 + 일부만 지정: `Instancio.of(User.class).set(field(User::getEmail), "a@test.com").create()` |
| **EasyRandom** | 객체 그래프를 랜덤 값으로 채움 |

> 도메인 의미가 중요한 테스트(잔액 0 vs 1000)는 수작업 픽스처가 더 명확. 대량/무의미 데이터엔 자동 생성이 편함.

---

## 8. 위치
- `src/test`에 둔다 (`src/main` 아님 — 배포물에 테스트 코드 섞이면 안 됨).
- 대상 도메인 경로 미러링: `src/test/.../domain/user/UserFixture.java`.

---

## 9. 참고
- 관련 노트: [JPA repository 테스트](./jpa-repository-test.md) · [테스트 작성 가이드](./test-writing-guide.md)
- [Martin Fowler - Object Mother](https://martinfowler.com/bliki/ObjectMother.html)

---

**학습 날짜**: 2026-06-05
**계기**: server-java User 테스트 리팩토링 중 `UserFixture`를 만들며 — Object Mother/Test Data Builder 두 형태, "변하는 축만 받기" 원칙, static 픽스처 vs em 필요한 인스턴스 헬퍼 구분을 정리.

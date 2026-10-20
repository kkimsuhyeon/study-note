# 테스트 픽스처 (Object Mother / Test Data Builder) — 사용법

> **한 줄 요약**: 테스트용 객체 생성 코드를 한 곳에 모으는 패턴. 두 형태가 있다 — **Object Mother**(이름 붙은 정적 팩토리: `UserFixture.withBalance(1000)`)와 **Test Data Builder**(체이닝: `aUser().balance(1000).build()`). 핵심 원칙은 **"그 테스트가 *바꾸는 값*만 받고, 나머지는 안에서 기본값으로 채운다."**

관련 노트: [JPA repository 테스트](./jpa-repository-test.md) · [JUnit 5 라이프사이클](./junit-lifecycle.md) · [테스트 작성 가이드](./test-writing-guide.md) · [AssertJ 사용법](./assertj.md)

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
    private String password = "password123";       // 기본값 (안 바꾸는 축이라 체이닝 메서드 없음 — §3)
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
> `@DataJpaTest`는 테스트마다 롤백 → `@BeforeEach` 시딩도 매 테스트 깨끗이 초기화된다. 라이프사이클 상세 → [JUnit 5 라이프사이클](./junit-lifecycle.md)

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

### 통합 테스트에서 객체 픽스처를 저장까지 쓸 때 ⚠️

통합 테스트(`@SpringBootTest`)는 픽스처로 만든 객체를 **실제 DB에 저장**한다. 단위 테스트에서는 문제없던 기본값이 여기서 깨진다.

```java
// 통합 테스트 안의 적재 헬퍼 — 객체는 픽스처, 저장은 테스트 클래스(리포지토리 필요)
private User savedUser(BigDecimal balance) {
    return userRepository.save(UserFixture.aUser()
            .id(null)                                         // DB가 id를 만들게
            .email("u" + System.nanoTime() + "@test.com")     // unique 컬럼은 매번 다르게
            .balance(balance)                                 // 이 테스트가 바꾸는 값
            .build());
}
```

| 함정 | 왜 | 대응 |
|---|---|---|
| 기본 id가 채워진 픽스처를 그대로 저장 | id가 있으면 JPA가 "이미 있는 행"으로 보고 `merge`(SELECT 후 INSERT/UPDATE)로 처리하거나, 다른 테스트가 넣은 같은 id 행을 덮어쓸 수 있다 | 저장용은 id를 비우거나, 저장 전용 팩토리를 따로 둔다 |
| unique 컬럼 기본값(`test@test.com`) | 롤백하지 않는 통합 테스트·동시성 테스트에서 두 번째 저장부터 제약 위반 | 테스트마다 유일한 값 생성 |
| 셋업을 production 서비스로(`reservationService.create(...)`) | 아래 5-1과 같은 문제 + "만료된 예약"처럼 서비스로는 못 만드는 상태가 있다 | 원하는 상태를 리포지토리로 직접 저장 |

"Fixture"라는 이름은 두 가지를 섞어 부르는 경우가 많다 — **객체를 만들기만 하는 픽스처**(static/builder, 스프링 없음)와 **DB에 적재까지 하는 헬퍼**(리포지토리 필요, 통합 테스트용). 여러 통합 테스트 클래스가 같은 적재를 반복하면 적재 헬퍼를 테스트 소스의 `@Component`로 빼서 주입받기도 한다.

---

## 5-1. 💡 셋업을 production 메서드에 의존시키지 않기 (관점)

상태를 만들 때 production 로직을 타면, 그 로직이 깨질 때 **무관한 테스트가 같이 실패**한다(실패 지점이 번짐).

```java
// ❌ deductBalance 테스트인데 상태를 addBalance로 만듦 → addBalance 깨지면 같이 실패
User user = UserFixture.withBalance(BigDecimal.ZERO);
user.addBalance(BigDecimal.valueOf(1000));
user.deductBalance(...);

// ✅ 픽스처/of로 원하는 상태를 직접 주입 → deductBalance만 검증
User user = UserFixture.withBalance(BigDecimal.valueOf(1000));
user.deductBalance(...);
```

- 픽스처/`of`로 **원하는 상태를 직접** 만들면 그 테스트는 한 메서드만 검증한다.
- repo 테스트의 `em.persistAndFlush` vs `save()`도 같은 원칙(어댑터 `save` 우회). → [JPA repository 테스트 §6-1](./jpa-repository-test.md)

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
| **Fixture Monkey** (Naver) | `FixtureMonkey.create().giveMeOne(Product.class)` 한 줄로 랜덤 객체, `giveMeBuilder(Product.class).set("price", 29_900L).sample()`로 일부만 지정. 국내 회사 코드에서 자주 보인다 |

> 도메인 의미가 중요한 테스트(잔액 0 vs 1000)는 수작업 픽스처가 더 명확. 대량/무의미 데이터엔 자동 생성이 편함.

---

## 8. 위치
- `src/test`에 둔다 (`src/main` 아님 — 배포물에 테스트 코드 섞이면 안 됨).
- 대상 도메인 경로 미러링: `src/test/.../domain/user/UserFixture.java`.
- 멀티 모듈에서 다른 모듈의 테스트가 같은 픽스처를 써야 하면 Gradle `java-test-fixtures` 플러그인을 쓴다. `src/testFixtures/java` 소스 세트가 생기고(main 클래스를 볼 수 있음, 그 모듈의 test가 볼 수 있음), 다른 모듈은 `testImplementation(testFixtures(project(":lib")))`로 가져다 쓴다. 단일 모듈이면 `src/test`로 충분하다.

---

## 9. 참고
- 관련 노트: [JPA repository 테스트](./jpa-repository-test.md) · [테스트 작성 가이드](./test-writing-guide.md)
- [Martin Fowler - Object Mother](https://martinfowler.com/bliki/ObjectMother.html)
- [Gradle: Using test fixtures](https://docs.gradle.org/current/userguide/java_testing.html#sec:java_test_fixtures)
- [Fixture Monkey](https://naver.github.io/fixture-monkey/)

---

**학습 날짜**: 2026-06-05
**계기**: User 테스트 리팩토링 중 `UserFixture`를 만들며 — Object Mother/Test Data Builder 두 형태, "변하는 축만 받기" 원칙, static 픽스처 vs em 필요한 인스턴스 헬퍼 구분을 정리.
**보강**: 2026-10-20 — 회사 코드에서 `createUser` 같은 헬퍼를 Fixture로 빼는 걸 보고 "그게 일반적인가" 질문. 통합 테스트에서 객체 픽스처를 저장할 때의 함정(기본 id·unique 기본값·서비스로 셋업), "Fixture" 두 의미, Fixture Monkey, Gradle `java-test-fixtures`(공식 문서 확인) 추가.

# JUnit 5 라이프사이클 · 구조 어노테이션 — 사용법

> **한 줄 요약**: 테스트 전/후에 코드를 자동 실행하는 `@BeforeEach`/`@BeforeAll`/`@AfterEach`/`@AfterAll` + 그룹화 `@Nested`·`@DisplayName`. 핵심은 **`@BeforeEach`는 매 테스트 직전 실행**이라 공통 셋업(유저 미리 심기 등)에 쓰고, **`@BeforeAll`은 클래스당 1번 + `static` 필수**.

관련 노트: [JPA repository 테스트](./jpa-repository-test.md) · [테스트 픽스처](./test-fixtures.md) · [테스트 작성 가이드](./test-writing-guide.md)

---

## 1. 라이프사이클 어노테이션

| 어노테이션 | 실행 시점 | static? | 횟수 |
|---|---|---|---|
| `@BeforeAll` | 모든 테스트 **전 1번** | **필수** | 1 |
| `@BeforeEach` | **각 테스트 직전** | 아니오 | 테스트 수만큼 |
| `@AfterEach` | 각 테스트 직후 | 아니오 | 테스트 수만큼 |
| `@AfterAll` | 모든 테스트 **후 1번** | **필수** | 1 |

```java
class FooTest {
    @BeforeAll  static void initAll()  { }   // 무거운 1회성 준비 (static)
    @BeforeEach       void setUp()     { }   // 매 테스트 공통 셋업
    @AfterEach        void tearDown()  { }   // 매 테스트 정리
    @AfterAll   static void cleanAll() { }
}
```

### 실행 순서

```
@BeforeAll
 ├─ @BeforeEach → @Test 1 → @AfterEach
 ├─ @BeforeEach → @Test 2 → @AfterEach
 └─ ...
@AfterAll
```

> 테스트 인스턴스는 기본적으로 **메서드마다 새로 생성**(`@TestInstance(PER_METHOD)`)된다. 그래서 인스턴스 필드로 테스트 간 상태 공유가 안 되고, `@BeforeAll`은 인스턴스가 없는 시점이라 `static`이어야 한다.

---

## 2. 셋업 사용 예시 — 테스트 전에 데이터 심기

"테스트 실행 전에 유저가 셋팅되면 좋겠다" → `@BeforeEach`.

```java
@BeforeEach
void setUp() {
    em.persistAndFlush(UserEntity.create(UserFixture.withEmail("a@test.com")));
    em.persistAndFlush(UserEntity.create(UserFixture.withEmail("b@test.com")));
    em.clear();   // 캐시 비워 각 테스트가 진짜 SELECT 하도록
}

@Test
void findByEmail_found() {
    // 이미 a@, b@ 가 DB에 있음 → 바로 조회
    assertThat(userRepository.findByEmail("a@test.com")).isPresent();
}
```

모든 `@Test` 직전에 `setUp()`이 돌아 **항상 같은 시작 상태**가 보장된다.

---

## 3. 구조 어노테이션 — `@Nested` · `@DisplayName`

```java
@DisplayName("UserRepository")
class UserRepositoryAdapterTest {

    @Nested
    @DisplayName("findAllByCriteria (필터)")
    class FilterTest {

        @BeforeEach
        void seed() { /* 이 그룹 테스트들만 데이터 심김 */ }

        @Test
        @DisplayName("이메일로 필터하면 해당 유저만 조회된다")
        void filterByEmail() { }
    }
}
```

| 어노테이션 | 역할 |
|---|---|
| `@Nested` | inner **non-static** 클래스로 관련 테스트 그룹화 |
| `@DisplayName` | 리포트에 표시될 한글 설명 (메서드명 대신) |

### `@Nested`와 `@BeforeEach` 순서

바깥 `@BeforeEach` → 안쪽 `@BeforeEach` → 테스트. (바깥→안쪽 순)

```
바깥 @BeforeEach
 └─ 안쪽 @BeforeEach
      └─ @Test
```

---

## 4. ⚠️ 함정

- **`@BeforeAll`은 `static`** — 안 붙이면 `JUnitException`. 인스턴스가 아직 없는 시점이라.
- **`@DataJpaTest`는 매 테스트 롤백** → 셋업은 `@BeforeEach`. `@BeforeAll`로 한 번 심으면 롤백으로 사라져 안 맞음. ([JPA repository 테스트](./jpa-repository-test.md))
- **`@BeforeEach` 범위** — 클래스 최상단에 두면 그 클래스 **모든 테스트**에 셋업이 적용됨. 일부 테스트엔 노이즈(개수 세기 등)가 될 수 있음 → **셋업 필요한 것만 `@Nested`로 묶고 그 안에 `@BeforeEach`**.
- **`@BeforeAll`을 non-static으로 쓰고 싶다** → 클래스에 `@TestInstance(Lifecycle.PER_CLASS)`를 붙이면 인스턴스를 클래스당 1개로 유지해 가능(상태 공유도 됨). 단 테스트 간 격리가 약해지니 주의.

---

## 5. 💡 판단 기준

| 상황 | 선택 |
|---|---|
| 여러 테스트가 **같은 데이터셋** 공유 | `@BeforeEach`로 공통 셋업 |
| **한 테스트만** 특수 데이터 필요 | 그 테스트 안에서 직접 심기 (given) |
| 공용 셋업이 일부 테스트엔 노이즈 | `@Nested`로 범위 좁히기 |
| 무겁고 **변하지 않는** 1회성 준비(컨테이너 등) | `@BeforeAll` (static) |

> 한 줄: **"매 테스트 같은 상태로 시작해야 하면 `@BeforeEach`, 비싼 1회성이면 `@BeforeAll`."** 셋업이 일부에만 필요하면 `@Nested`로 가둬서 다른 테스트를 오염시키지 않는다.

---

## 6. 참고
- [JUnit 5 User Guide - Test Lifecycle](https://junit.org/junit5/docs/current/user-guide/#writing-tests-test-instance-lifecycle)
- 관련 노트: [JPA repository 테스트](./jpa-repository-test.md) · [테스트 픽스처](./test-fixtures.md)

---

**학습 날짜**: 2026-06-05
**계기**: repository 테스트에서 "테스트 실행 전에 유저를 미리 심고 싶다" → `@BeforeEach` 사용법을 찾으며, `@BeforeAll`과의 차이·`@DataJpaTest` 롤백과의 관계·`@Nested`로 셋업 범위 좁히기를 정리.

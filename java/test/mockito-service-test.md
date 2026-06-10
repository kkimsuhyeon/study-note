# Mockito — 서비스 테스트 (Mock + 상호작용 검증)

> **한 줄 요약**: 서비스 테스트는 **DB를 안 띄우고** repository(포트)를 **Mock**으로 바꿔, 서비스의 **분기/예외 · 흐름 조합 · 상호작용**을 검증한다. `given().willReturn()`으로 repo 응답을 정해주고, `verify`/`ArgumentCaptor`로 "무엇을 어떻게 불렀나"를 본다. 핵심은 **도메인 규칙·DB는 안 보고**(각자 다른 테스트가 책임), 서비스의 판단만 본다는 것.

관련 노트: [테스트 작성 가이드](./test-writing-guide.md) · [JPA repository 테스트](./jpa-repository-test.md) · [테스트 픽스처](./test-fixtures.md) · [AssertJ 사용법](./assertj.md)

---

## 1. 기본 골격

```java
@ExtendWith(MockitoExtension.class)        // JUnit5 + Mockito 연결 (스프링 부팅 X)
class UserCommandServiceTest {

    @Mock        private UserRepository repository;     // 가짜 repo
    @InjectMocks private UserCommandService service;    // 위 @Mock을 주입받은 실제 서비스
}
```

| 어노테이션 | 역할 |
|---|---|
| `@ExtendWith(MockitoExtension.class)` | Mockito를 JUnit5에 연결 (스프링 컨텍스트 없음 → 빠름) |
| `@Mock` | 가짜 객체 생성 (모든 메서드 기본 null/0/empty 반환) |
| `@InjectMocks` | 테스트 대상(SUT)에 `@Mock`들을 생성자/필드 주입 |
| `@Spy` | 부분 mock (스텁 안 한 메서드는 **실제 동작**) — 가끔만 |

---

## 2. 스터빙 — "이렇게 응답해라" (given/when)

```java
// BDD 스타일 (권장 — given/when/then 구조와 맞음)
given(repository.findByIdForUpdate("u1")).willReturn(Optional.of(user));
given(repository.existsByEmail("a@")).willReturn(true);

// 클래식 스타일 (동일 동작)
when(repository.findById("u1")).thenReturn(Optional.of(user));

// 예외를 던지게
given(repository.findById("u1")).willThrow(new BusinessException(...));

// 받은 인자를 그대로 반환 (update가 받은 user를 돌려주게)
given(repository.update(any())).willAnswer(inv -> inv.getArgument(0));
```

---

## 3. 검증 — "무엇을 어떻게 불렀나" (verify)

```java
// BDD 스타일
then(repository).should().update(any(User.class));        // 1번 호출됐나
then(repository).should(times(2)).save(any());            // 2번
then(repository).should(never()).update(any());           // ⭐ 호출 안 됐나 (실패 경로 핵심)

// 클래식 스타일
verify(repository).findByIdForUpdate("u1");
verify(repository, never()).update(any());
```

### ArgumentMatchers — 인자 매칭

```java
any()  any(User.class)  anyString()  eq("u1")
```
> ⚠️ **한 인자라도 matcher를 쓰면 모든 인자를 matcher로.** `update(eq("u1"), any())` ✅ / `update("u1", any())` ❌(에러).

### ArgumentCaptor — 넘긴 인자를 붙잡아 검증

```java
// (A) inline — forClass
ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
then(repository).should().update(captor.capture());
assertThat(captor.getValue().getBalance()).isEqualByComparingTo(BigDecimal.valueOf(1000));

// (B) @Captor 필드 — MockitoExtension이 초기화. 반복/공유·제네릭에 유리
@Captor private ArgumentCaptor<User> userCaptor;
...
then(repository).should().update(userCaptor.capture());
userCaptor.getValue();        // 마지막 1개
userCaptor.getAllValues();    // 여러 번 호출됐으면 전부
```
"서비스가 repo.update에 **어떤 user를 넘겼나**"를 확인 → 흐름 조합(load→도메인메서드→저장) 검증에 씀. 패키지는 `org.mockito.ArgumentCaptor`.

> ⚠️ **제네릭은 `@Captor`가 유리**: `forClass(List.class)`는 `ArgumentCaptor<List<User>>`의 제네릭을 못 살려 unchecked 경고 → `@Captor ArgumentCaptor<List<User>>` 필드로 두면 깔끔. 단일 타입(`User`)이면 둘 다 무방.

> 💡 **captor는 "참조를 못 가지는 인자"일 때만.** 인자가 **메서드 안에서 새로 만들어져** 테스트가 참조를 못 가질 때(예: `create`가 command로 User를 만들어 `save`에 넘김) captor로 붙잡는다. 반대로 **stub으로 준 객체를 그대로** 넘기는 경우(예: `addBalance`가 조회한 user를 mutate해 `update`)는 그 참조로 `assertThat(user...)` + `verify(repo).update(user)`가 더 단순 — captor 불필요. 남발 ✗, 필요할 때만. (필드로 빼는 `@Captor` 어노테이션 방식도 있음)

---

## 4. 실제 예시 — `addBalance` (성공 + NOT_FOUND)

대상 코드: `findByIdForUpdate → orElseThrow(NOT_FOUND) → user.addBalance → repository.update`

```java
@Test
@DisplayName("잔액 충전 시 조회한 유저에 더해 update를 호출한다")
void addBalance_success() {
    // given — repo가 잔액 0인 유저를 준다고 stub
    User user = UserFixture.aUser().balance(BigDecimal.ZERO).build();
    given(repository.findByIdForUpdate("u1")).willReturn(Optional.of(user));

    // when
    service.addBalance("u1", BigDecimal.valueOf(1000));

    // then — update에 넘긴 user의 잔액이 1000인가 (load→addBalance→update 조합 검증)
    ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
    then(repository).should().update(captor.capture());
    assertThat(captor.getValue().getBalance()).isEqualByComparingTo(BigDecimal.valueOf(1000));
}

@Test
@DisplayName("없는 유저면 NOT_FOUND, update는 호출하지 않는다")
void addBalance_notFound() {
    // given — repo가 빈 Optional
    given(repository.findByIdForUpdate("u1")).willReturn(Optional.empty());

    // when & then
    assertThatThrownBy(() -> service.addBalance("u1", BigDecimal.valueOf(1000)))
            .isInstanceOf(BusinessException.class)
            .extracting("errorCode").isEqualTo(UserErrorCode.NOT_FOUND);

    then(repository).should(never()).update(any());   // ⭐ 예외 후 저장 안 함
}
```

> `getUser`도 동일 패턴: `findById` stub → 성공 반환 / `empty`면 `NOT_FOUND`.

---

## 5. 💡 관점 — 계층마다 보는 게 다르다

| 계층 | 도구 | 검증 | 안 봄 |
|---|---|---|---|
| 모델 (`UserTest`) | 순수 | 도메인 규칙(addBalance 음수 등) | — |
| Repo 어댑터 (`@DataJpaTest`) | 실제 H2 | 쿼리·매핑·제약 | 서비스 로직 |
| **서비스** | **Mockito** | **분기/예외 · 흐름 조합 · 상호작용** | **DB · 도메인 규칙** |

서비스 테스트가 보는 3가지:
1. **분기/예외** — repo가 `empty` 주면 `NOT_FOUND` 던지나
2. **흐름 조합** — 조회→도메인 메서드→저장 순서로 엮였나 (ArgumentCaptor)
3. **상호작용** — repo의 무엇을 몇 번/어떤 인자로 부르나 (verify), **실패 시 `never()`**

안 보는 것:
- **도메인 규칙 재검증** — `addBalance` 음수 가드는 `UserTest`가 함. 서비스는 "호출한다"까지만.
- **DB/매핑** — repo 어댑터 테스트가 함.
- **순수 위임 메서드**(`save`/`update`/`create`) — 분기 없어 가치 낮음.

> 💡 한 줄: **repo를 가짜로 세팅하고, 서비스가 올바른 분기·호출을 하는지 본다. "실패하면 저장 안 함"(`should(never())`)은 서비스에서만 잡히는 포인트.**

> 💡 **분기 없는 메서드는 fail 테스트가 없다 — 실패 테스트는 규칙을 소유한 계층에 산다.** `create`처럼 `register→save` 위임만 하면(if·throw 없음) 서비스 레벨 실패 경로가 0 → 성공 테스트 하나(위임+저장 확인)면 충분. "중복 이메일 → DUPLICATE_EMAIL" 실패 테스트는 그 규칙을 소유한 `UserRegistration`(도메인 서비스)의 테스트에 있다 — **규칙을 다른 계층으로 추출하면 실패 테스트도 따라 이사간다.** 위임 계층에서 `when(register).thenThrow(...)`로 또 만들면 "mock이 던지라고 시킨 예외를 자바가 전파하나"를 검증하는 것 = **mock과 언어 스펙 테스트**(우리 로직 아님). 단 위임 메서드가 예외를 **잡아서 변환/복구**하기 시작하면 그때 자기 분기가 생긴 것 → fail 테스트 부활. (어떤 검증이 어느 계층에 사는지 → [도메인 검증 위치](../design/domain-validation.md)) "이 메서드는 왜 실패 테스트가 없지?"는 보통 "이 메서드엔 판단이 없다"는 신호.

---

## 6. ⚠️ 함정

- **`UnnecessaryStubbingException`** — `MockitoExtension`은 기본 **strict**라, `given(...)` 해놓고 안 쓰면 에러. **쓸 것만 stub.**
- **matcher 혼용 금지** — 한 인자에 matcher 쓰면 나머지도 matcher(§3).
- **`@Mock`은 진짜 동작 안 함** — stub 안 한 메서드는 null/empty 반환. 그래서 흐름에 필요한 응답은 `given`으로 정해줘야 함.
- **도메인 규칙을 서비스 테스트에서 또 검증하지 말 것** — 중복. 계층 책임 분리.

---

## 7. 참고
- [Mockito 공식 - Getting Started](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html)
- 관련 노트: [JPA repository 테스트](./jpa-repository-test.md) · [테스트 작성 가이드](./test-writing-guide.md)
- 상세 방법론: server-java `docs/TEST_GUIDE.md` (§5 서비스/유스케이스 테스트)

---

**학습 날짜**: 2026-06-07
**계기**: `UserCommandService`(addBalance/deductBalance) 서비스 테스트를 시작하며 — "서비스는 어떤 관점으로 테스트하나"가 출발점. Mock으로 repo 세팅 → 분기/조합/상호작용 검증, 도메인·DB는 각 계층이 책임진다는 관점 + Mockito API(@Mock/@InjectMocks/given/verify/ArgumentCaptor/never) 정리.

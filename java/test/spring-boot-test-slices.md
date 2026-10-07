# Spring Boot 테스트 슬라이스 — @SpringBootTest, @WebMvcTest, @DataJpaTest

> **한 줄 요약**: Spring Boot 테스트는 많이 띄울수록 실제 환경에 가깝지만 느리고, 적게 띄울수록 빠르지만 검증 범위가 좁다. 테스트 목적에 맞춰 슬라이스를 고른다.

관련 노트: [JPA Repository 테스트](./jpa-repository-test.md) · [Mockito 서비스 테스트](./mockito-service-test.md) · [테스트 작성 가이드](./test-writing-guide.md)

---

## 1. 큰 기준

| 테스트 | 띄우는 범위 | 주로 검증 |
|---|---|---|
| 순수 단위 테스트 | Spring 없음 | 도메인, 서비스 분기, 계산 |
| `@WebMvcTest` | MVC 계층 | 컨트롤러, 요청/응답, validation, advice |
| `@DataJpaTest` | JPA 계층 | Repository, 매핑, 쿼리 |
| `@SpringBootTest` | 전체 컨텍스트 | 통합 흐름, 설정, 빈 연결 |

> **선행지식 — "슬라이스"란.** `@SpringBootTest`가 전체 자동설정을 켜는 반면, `@WebMvcTest`/`@DataJpaTest`는 **그 계층에 필요한 자동설정만 부분 적용**한다(슬라이스 = 잘라낸 한 조각). 그래서 빠르지만, 슬라이스 밖의 빈(예: `@Service`)은 안 떠서 `@MockitoBean`으로 채워야 한다.

---

## 2. 순수 단위 테스트

```java
@ExtendWith(MockitoExtension.class)
class PointServiceTest {
    @Mock PointRepository repository;
    @InjectMocks PointService service;
}
```

Spring을 띄우지 않는다. 빠르고 실패 원인이 좁다. 서비스 로직 대부분은 이 방식으로 먼저 검증한다.

> ⚠️ 엄밀히는 두 가지가 섞여 있다 — 위 `@ExtendWith(MockitoExtension)` + `@Mock`은 **mock 기반 단위 테스트**(서비스 분기·상호작용)이고, **진짜 "순수" 도메인 테스트**는 확장 없이 `new`로 찍어 검증한다(계산·불변식). 전자의 상세(given/verify/ArgumentCaptor) → [Mockito 서비스 테스트](./mockito-service-test.md).

---

## 3. @WebMvcTest

```java
@WebMvcTest(PointController.class)
class PointControllerTest {
    @Autowired MockMvc mockMvc;
    @MockitoBean PointService pointService;   // org.springframework.test.context.bean.override.mockito
}
```

> ⚠️ **`@MockBean` → `@MockitoBean` (버전 변화)**: Boot의 `@MockBean`/`@SpyBean`은 **3.4에서 deprecated, 4.0에서 삭제**됐다. 대체는 Spring Framework 6.2+의 `@MockitoBean`/`@MockitoSpyBean`. 차이 하나 — 새 어노테이션은 테스트 클래스 필드에만 쓰고 **`@Configuration` 클래스 안에는 못 둔다**(공유 mock 세트는 테스트 클래스·공통 부모·인터페이스 쪽으로 옮긴다). Boot 4.0은 슬라이스도 모듈로 쪼개져 `spring-boot-webmvc-test`·`spring-boot-data-jpa-test`(스타터는 `spring-boot-starter-webmvc-test` 등)가 필요하다 — 어노테이션의 import 패키지도 모듈별로 옮겨졌을 수 있으니 마이그레이션 가이드로 확인(확인 필요).

컨트롤러의 HTTP 매핑, JSON 요청/응답, validation, `@ControllerAdvice`를 검증할 때 쓴다.

서비스는 보통 mock으로 둔다. 서비스 로직을 여기서 다시 검증하지 않는다.

---

## 4. @DataJpaTest

```java
@DataJpaTest
class PointRepositoryTest {
    @Autowired PointRepository repository;
    @Autowired TestEntityManager em;
}
```

엔티티 매핑, Repository 쿼리, flush/clear 이후 DB 왕복 검증에 좋다. (persist≠INSERT, `TestEntityManager`, `replace` 옵션, Testcontainers 상세 → [JPA Repository 테스트](./jpa-repository-test.md))

DB 방언, 락, 동시성처럼 H2로 재현이 애매한 것은 Testcontainers를 고려한다.

---

## 5. @SpringBootTest

```java
@SpringBootTest
class PointIntegrationTest {
}
```

전체 Spring 컨텍스트를 띄운다. 가장 무겁다.

다음처럼 "연결"을 검증할 때 쓴다.

- 실제 빈 주입이 맞는지
- 설정 파일이 맞는지
- Controller → Service → Repository 흐름이 이어지는지
- 트랜잭션, 이벤트, 스케줄러 등 여러 계층이 같이 필요한지

---

## 6. 💡 판단 기준

| 질문 | 선택 |
|---|---|
| 계산/분기만 검증하나? | 순수 단위 테스트 |
| HTTP 요청/응답 모양을 검증하나? | `@WebMvcTest` |
| JPA 쿼리와 매핑을 검증하나? | `@DataJpaTest` |
| 여러 계층 wiring을 검증하나? | `@SpringBootTest` |
| 어떤 걸 써야 할지 모르겠나? | 더 작은 테스트부터 시작 |

---

## 7. ⚠️ 함정

- 모든 테스트를 `@SpringBootTest`로 쓰면 느리고 실패 원인이 흐려진다.
- `@WebMvcTest`에서 서비스 로직까지 검증하려 하면 mock 설정만 복잡해진다.
- `@DataJpaTest`에서 `save()`만 호출하고 끝내면 실제 INSERT/UPDATE 검증이 약하다. flush/clear 후 다시 조회한다.
- 슬라이스 테스트는 일부 빈만 뜨므로 필요한 의존성은 mock이나 test configuration으로 명시해야 한다. (그래서 `@WebMvcTest`는 서비스를 `@MockitoBean`으로 올려야 `@Autowired`가 실패하지 않는다.)
- **`@WebMvcTest` + Spring Security 함정**: 시큐리티 필터가 같이 떠서 인증 없는 요청이 401/403으로 막힌다 → 테스트에서 `@WithMockUser`로 인증을 주거나 `@AutoConfigureMockMvc(addFilters = false)`로 필터를 끈다. ⚠️ **POST/PUT/DELETE는 `@WithMockUser`를 줘도 CSRF 때문에 403** — `mockMvc.perform(post("/x").with(csrf()))`로 토큰을 실어 보낸다.
- ⚠️ **mock 구성이 다르면 컨텍스트를 새로 띄운다** — `@MockitoBean`(bean override)은 컨텍스트 캐시 키의 일부라, 테스트 클래스마다 mock 조합이 조금씩 다르면 캐시를 못 타고 매번 부팅한다. 통합 테스트가 느려지는 1순위 원인 → 공통 부모 클래스로 mock 구성을 통일한다.

---

## 8. 참고

- [Spring Boot - Testing Spring Boot Applications (Test Slices)](https://docs.spring.io/spring-boot/reference/testing/spring-boot-applications.html)
- [Spring Framework - @MockitoBean and @MockitoSpyBean](https://docs.spring.io/spring-framework/reference/testing/annotations/integration-spring/annotation-mockitobean.html)
- [Spring Boot 3.4 Release Notes - Deprecation of @MockBean and @SpyBean](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.4-Release-Notes) · [Spring Boot 4.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)
- [Spring Framework - Context Caching](https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/ctx-management/caching.html) — bean override가 캐시 키에 포함
- [Spring Security - Testing with CSRF Protection](https://docs.spring.io/spring-security/reference/servlet/test/mockmvc/csrf.html)

---

**학습 날짜**: 2026-06-08 (2026-10-02 `@MockitoBean` 전환·CSRF·컨텍스트 캐시 보강)
**계기**: 테스트 노트(Mockito 서비스·JPA Repository 테스트)를 묶는 교차 가이드로, "어떤 검증에 어떤 슬라이스를 띄우나"를 한 장으로 정리.

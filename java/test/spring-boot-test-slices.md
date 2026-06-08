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

---

## 3. @WebMvcTest

```java
@WebMvcTest(PointController.class)
class PointControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean PointService pointService;
}
```

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

엔티티 매핑, Repository 쿼리, flush/clear 이후 DB 왕복 검증에 좋다.

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

## 6. 판단 기준

| 질문 | 선택 |
|---|---|
| 계산/분기만 검증하나? | 순수 단위 테스트 |
| HTTP 요청/응답 모양을 검증하나? | `@WebMvcTest` |
| JPA 쿼리와 매핑을 검증하나? | `@DataJpaTest` |
| 여러 계층 wiring을 검증하나? | `@SpringBootTest` |
| 어떤 걸 써야 할지 모르겠나? | 더 작은 테스트부터 시작 |

---

## 7. 함정

- 모든 테스트를 `@SpringBootTest`로 쓰면 느리고 실패 원인이 흐려진다.
- `@WebMvcTest`에서 서비스 로직까지 검증하려 하면 mock 설정만 복잡해진다.
- `@DataJpaTest`에서 `save()`만 호출하고 끝내면 실제 INSERT/UPDATE 검증이 약하다. flush/clear 후 다시 조회한다.
- 슬라이스 테스트는 일부 빈만 뜨므로 필요한 의존성은 mock이나 test configuration으로 명시해야 한다.

---

## 8. 참고

- Spring Boot Testing
- Spring Boot Test Slices

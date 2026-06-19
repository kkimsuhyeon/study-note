# Study Note

개발하면서 학습한 내용을 정리하는 공간

> ☑️ **읽음 표시**: `- [x]` 읽음 / `- [ ]` 안 읽음. **새 문서는 `- [ ]`(안 읽음)로 추가**되고, 읽으면 체크(`- [x]`)한다. (Obsidian에서 체크박스 클릭으로 토글)

## Java

### Concurrency (동시성)
- [ ] [스레드 기초 - 프로세스/스레드/플랫폼 스레드(OS 1:1)/생명주기 / 힙 공유→race](./java/concurrency/threads.md)
- [ ] [가상 스레드 - Java 21 JEP444 / M:N·캐리어·unmount / pinning(JEP491) / 락은 그대로 필요](./java/concurrency/virtual-threads.md)
- [ ] [락 개념 종합 - 낙관적/비관적/분산 락](./java/concurrency/locks.md)
- [ ] [JVM 동시성 도구 - synchronized/ReentrantLock/Semaphore/Latch/Barrier 등](./java/concurrency/jvm-concurrency-tools.md)
- [ ] [동시성 도구 목적별 정리 & 선택 가이드 - 만들기/조합(CompletableFuture)/대기/세기/정합성 / 최신≠최선](./java/concurrency/concurrency-tool-guide.md)
- [ ] [데드락(교착 상태) - Coffman 4조건/예방/DB 자동 감지](./java/concurrency/deadlock.md)

### JPA
> 추천 읽는 순서: [영속성 컨텍스트](./java/jpa/persistence-context.md) → [@Transactional](./java/spring/transactional.md) → [트랜잭션 롤백 예제](./java/spring/transaction-rollback-example.md) → [Read-Modify-Write](./java/jpa/read-modify-write.md) → [@Lock 기본](./java/jpa/lock.md) → [@Lock 심화](./java/jpa/lock-concepts.md) → [@Lock 실무](./java/jpa/lock-practical.md)

- [ ] [영속성 컨텍스트 · flush · 더티 체킹 - "커밋 시점"의 정체](./java/jpa/persistence-context.md)
- [ ] [트랜잭션 격리 수준 - Dirty/Non-repeatable/Phantom/Lost Update와 락 관계](./java/jpa/transaction-isolation.md)
- [ ] [@Lock 기본 - 락 어노테이션 (언제·종류·사용·주의)](./java/jpa/lock.md)
- [ ] [@Lock 심화 개념 - @Version·공유/배타·FORCE_INCREMENT](./java/jpa/lock-concepts.md)
- [ ] [@Lock 실무 패턴 - 프록시·네이밍·테스트·재시도·벌크/조건부 UPDATE](./java/jpa/lock-practical.md)
- [ ] [Read-Modify-Write와 트랜잭션 경계 - 쓰기 서비스가 조회 서비스에 의존하면 안 되는 이유](./java/jpa/read-modify-write.md)
- [ ] [Criteria · Specification · Pageable · Page - 출처(Spring Data/JPA/자작)와 동적 조회·페이징](./java/jpa/spring-data-query.md)
- [ ] [N+1과 fetch 전략 - fetch join, EntityGraph, batch size 판단 기준](./java/jpa/n-plus-one-fetch.md)

### Spring
- [ ] [@Transactional - 선언적 트랜잭션·전파(propagation)·롤백 규칙·프록시 함정](./java/spring/transactional.md)
- [ ] [예제로 보는 트랜잭션 전파·롤백 - a→b→c 워크스루](./java/spring/transaction-rollback-example.md)
- [ ] [@Valid · @Validated - Bean Validation 동작·위치, 중첩 cascade 함정](./java/spring/validation.md)
- [ ] [Spring 예외 처리 - @ControllerAdvice, ErrorCode, Validation 예외 흐름](./java/spring/exception-handling.md)

### BigDecimal
- [ ] [BigDecimal - 돈·정밀 계산, equals vs compareTo, scale, 반올림](./java/bigdecimal/bigdecimal.md)

### Collections (컬렉션)
- [ ] [리스트 생성 - singletonList/List.of/Arrays.asList, "불변 × 원소 개수"는 별개 축](./java/collections/list-creation.md)

### Test (테스트 도구 사용법)
- [ ] [AssertJ 사용법 - assertThat / isEqualByComparingTo / assertThatThrownBy](./java/test/assertj.md)
- [ ] [테스트 방법론 핵심 요약 - 판단 공식 + 막힌 케이스 누적 (상세는 프로젝트 가이드)](./java/test/test-writing-guide.md)
- [ ] [테스트 픽스처(Object Mother) - 변하는 값만 받기 / static 픽스처 vs 인스턴스 헬퍼](./java/test/test-fixtures.md)
- [ ] [JPA repository 테스트 - @DataJpaTest / persist≠INSERT / flush·clear 왕복 / H2·Testcontainers](./java/test/jpa-repository-test.md)
- [ ] [JUnit 5 라이프사이클 - @BeforeEach·@BeforeAll·@Nested / 테스트 전 데이터 셋업](./java/test/junit-lifecycle.md)
- [ ] [파라미터화 테스트 - @ParameterizedTest / ValueSource·EnumSource·CsvSource·MethodSource / 검증 같을 때만](./java/test/parameterized-test.md)
- [ ] [Mockito 서비스 테스트 - @Mock·@InjectMocks·given·verify·ArgumentCaptor / 분기·조합·상호작용 검증](./java/test/mockito-service-test.md)
- [ ] [동시성 테스트 작성법 - ExecutorService·CountDownLatch 3개 / @Transactional 금지 / 비관(합계)·낙관(1성공) 검증](./java/test/concurrency-test.md)
- [ ] [Spring Boot 테스트 슬라이스 - @SpringBootTest, @WebMvcTest, @DataJpaTest 선택 기준](./java/test/spring-boot-test-slices.md)

### 설계 (DDD / 도메인 모델)
- [ ] [도메인 검증 위치 - 엔티티(불변식) vs 도메인 서비스(유니크)·체커 주입 / 규칙=도메인·조회=인프라](./java/design/domain-validation.md)
- [ ] [변환 계층 - Factory(생성)/Mapper(web→Command)/Assembler(Command→Model) + Command/Query](./java/design/transform-layers.md)
- [ ] [포트와 어댑터 - 콘센트(규격)/플러그(구현) 비유 / 인터페이스는 의존 역전 필요할 때만 / 포트 소유권](./java/design/ports-and-adapters.md)

### 보안 (Security)
- [ ] [비밀번호 - PasswordEncoder(단방향 해시) vs AttributeConverter(양방향 암호화) / 복호화 여부가 갈림길](./java/security/password-encoding.md)

### Jackson
- [ ] [Jackson 어노테이션 종합 정리](./java/jackson/annotations.md)

## Database
- [ ] [SQL 케이스 쿡북 - 상황별 해법 (날짜 범위 조회, 인덱스/sargable 등)](./database/sql-cookbook.md)
- [ ] [PostgreSQL 날짜 함수 - to_date/make_date/EXTRACT/date_trunc](./database/postgresql-date-functions.md)
- [ ] [인덱스와 실행 계획 - EXPLAIN, range scan, full scan, sargable 조건](./database/index-explain.md)

## Infra / 분산 환경
- [ ] [스케일 아웃 & 배포 모델 - 1 JVM/인스턴스 복제/로드밸런서 vs 오토스케일러/무상태](./infra/scaling.md)

---

## 작성 규칙
- 한 어노테이션/개념당 한 파일
- 파일명은 kebab-case (`lock.md`, `json-property.md`)
- 새 글 추가 시 이 README에 링크 등록 — **`- [ ]`(안 읽음) 체크박스로** 추가 (안 그러면 잊어버림). 읽으면 `- [x]`로 체크
- **사용법/함정으로 시작하되, 끝에 판단 기준(관점) 한 줄을 남긴다** — API 문법은 다시 찾으면 되지만 "어느 쪽을 골라야 하나"는 한 번 잡으면 비슷한 상황 전부에 적용된다. (가장 오래 살아남는 층)
- **관점은 반드시 구체 케이스에 붙여서** — "이 상황에서 이렇게 막혔다 → 그래서 이 판단" 형태로. 관점만 따로 모은 추상 격언집("좋은 테스트를 짜라")은 죽은 문서가 된다.

> 📐 **문서 3층 구조** (아래로 갈수록 오래 산다): ① API 사용법(어떻게 쓰지) → ② 함정/메커니즘(왜 이렇게 동작하지) → ③ **관점/판단 기준(어느 쪽을 골라야 하지)**. API는 ③에 도달하기 위한 예시고, 관점이 알맹이.

## 글 템플릿
1. 한 줄 요약 (+ 끝에 핵심 함정/판단 한 줄)
2. 언제 쓰나
3. 사용 예시 (문법/API)
4. 종류/옵션 비교
5. 함정/메커니즘 (⚠️)
6. 💡 판단 기준 (관점) — "그래서 어느 쪽" 한 줄. 구체 케이스에 붙여서
7. 참고 링크 + 학습 날짜 + 계기

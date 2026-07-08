# Study Note

개발하면서 학습한 내용을 정리하는 공간

> ☑️ **읽음 표시**: `- [x]` 읽음 / `- [ ]` 안 읽음. **새 문서는 `- [ ]`(안 읽음)로 추가**되고, 읽으면 체크(`- [x]`)한다. (Obsidian에서 체크박스 클릭으로 토글)

## Java

### Concurrency (동시성)
- [ ] [스레드 기초 - 프로세스/스레드/플랫폼 스레드(OS 1:1)/생명주기 / 힙 공유→race](./java/concurrency/threads.md)
- [ ] [가상 스레드 - Java 21 JEP444 / M:N·캐리어·unmount / pinning(JEP491) / 락은 그대로 필요](./java/concurrency/virtual-threads.md)
- [ ] [메모리 가시성 - volatile·happens-before(JMM) / 가시성≠원자성 / volatile로 count++ 안 됨](./java/concurrency/memory-visibility.md)
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
- [ ] ["배치"의 세 층위 - 쿼리 배치화(N+1→IN+groupingBy) / Spring Batch(스케줄러 없음!) / JDBC 쓰기 배치](./java/spring/batch-three-meanings.md)
- [ ] [SseEmitter 서버 구현 - 서블릿 async(스레드 즉시 반납) / send·event()빌더·콜백3종 / 두 패턴(세션푸시+저장소·Pub/Sub / relay+구독) / 이벤트루프 블로킹 금지·boundedElastic](./java/spring/sse-emitter.md)

### BigDecimal
- [ ] [BigDecimal - 돈·정밀 계산, equals vs compareTo, scale, 반올림](./java/bigdecimal/bigdecimal.md)

### Collections (컬렉션)
- [ ] [리스트 생성 - singletonList/List.of/Arrays.asList, "불변 × 원소 개수"는 별개 축](./java/collections/list-creation.md)
- [ ] [Map 주요 메서드 - getOrDefault/putIfAbsent/computeIfAbsent(캐시)/merge(카운팅) / 람다 null·재진입 함정](./java/collections/map-methods.md)
- [ ] [컬렉션 계층 구조 - List/Set/Queue는 Collection, Map은 별도 / Set 집합연산=Collection 공통 / equals·hashCode](./java/collections/collection-hierarchy.md)

### Generics (제네릭)
- [ ] [가변인자(varargs) & @SafeVarargs - 제네릭 varargs의 heap pollution(타입안전성 오염) / 붙일 수 있는 위치 / 읽기만·노출 금지](./java/generics/varargs-safevarargs.md)

### Reactive (Reactor)
- [ ] [Flux/Mono 기초 - 본질은 논블로킹(실시간 아님) / lazy·신호 3종·에러=신호 / 이벤트 루프 블로킹·ThreadLocal 금지 / MVC선 가장자리만](./java/reactive/flux-mono-basics.md)

### 함수형 / 람다 (Functional)
- [ ] [람다 ≠ 비동기 - 람다는 "코드 값"일 뿐, 실행 타이밍은 받는 메서드가 정함 / forEach=즉시·on~/then~=콜백·submit/Async=다른 스레드 / 스트림 lazy·콜백 스레드엔 ThreadLocal 없음](./java/functional/lambda-execution-timing.md)

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
- [ ] [애그리거트 소유권 & 참조 방향 - source of truth / 자기 사실만 판정 / 1:1 FK는 나중 생긴 쪽이 단방향](./java/design/aggregate-ownership.md)
- [ ] [포트와 어댑터 - 콘센트(규격)/플러그(구현) 비유 / 인터페이스는 의존 역전 필요할 때만 / 포트 소유권](./java/design/ports-and-adapters.md)
- [ ] [일급 컬렉션 - 컬렉션 하나만 감싼 클래스 / 응집·불변(방어적 복사) / vs 값 객체(Tell Don't Ask·널 객체)](./java/design/first-class-collection.md)

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

### Redis
- [ ] [Redis 기초 - "자료구조 공용 메모리 서버" / 명령어로 대화 / TTL / RedisTemplate opsFor* 매핑](./infra/redis/redis-basics.md)
- [ ] [Redis Pub/Sub - 저장 없는 방송(유실=스펙) / 구독=연결 열어두기 / Spring 3부품(convertAndSend·ListenerContainer·onMessage) / 스케일아웃 SSE](./infra/redis/redis-pubsub.md)

### 네트워크 (Network)
- [ ] [실시간 통신 기법 비교 - Polling/Long Polling/SSE/WebSocket 진화 / relay(중계) 패턴 / "양방향 필요한가"가 갈림길](./infra/network/realtime-communication.md)
- [ ] [SSE - text/event-stream 포맷 / EventSource(GET 전용·자동 재연결) vs POST fetch 스트리밍 / heartbeat·프록시 버퍼링·UTF-8 함정](./infra/network/sse.md)
- [ ] [WebSocket - Upgrade 핸드셰이크(101) / STOMP / 재연결·스케일아웃 세션 공유가 내 숙제](./infra/network/websocket.md)
- [ ] [포트와 listen - 연결 거부(refused) vs 응답 없음(timeout) / localhost vs 0.0.0.0 바인딩](./infra/network/ports-and-listen.md)
- [ ] [SSH 포트 포워딩 - -L/-R/-D / 중간 호스트는 원격 기준 해석 / 막힌 포트 우회](./infra/network/ssh-port-forwarding.md)
- [ ] [SSH config - Host 별칭 / LocalForward·IdentityFile / 작업·터널 별칭 분리](./infra/network/ssh-config.md)

## Git
- [ ] [git worktree - clone 없이 여러 폴더에 동시 체크아웃 / .git 공유·커밋 실시간 공유 / MR·PR ref 활용 / 동일 브랜치 금지](./git/worktree.md)

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

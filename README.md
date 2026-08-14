# Study Note

개발하면서 학습한 내용을 정리하는 공간

> ☑️ **읽음 표시**: `- [x]` 읽음 / `- [ ]` 안 읽음. **새 문서는 `- [ ]`(안 읽음)로 추가**되고, 읽으면 체크(`- [x]`)한다. (Obsidian에서 체크박스 클릭으로 토글)

---

## 📍 하루 루틴 (틈틈이 공부용)

**한 번에 노트 1개.** 대부분 5~10분이면 끝난다. 틈날 때 하나씩이 주말에 몰아 읽는 것보다 낫다. (코테는 [로드맵](./algorithm/roadmap.md)에서 별도로 진행 — 읽기 순서와 섞지 않는다)

1. **읽기 전 30초 — 인출 먼저.** 아래 목록의 *한 줄 요약만* 보고 "내가 아는 걸 말해봐". 그다음 본문을 편다. 아는 부분은 스킵하고, **어긋난 부분이 그날의 진짜 학습**이다. (코테 로드맵의 "30초 루틴"과 같은 방식)
2. **읽고 나면 `- [x]` 체크 + 맨 끝 💡을 자기 말로 다시 써본다.** 못 쓰면 안 읽은 것 — 체크하지 말 것.
3. **막히면 그 자리에 `❓ ...` 한 줄 남긴다.** 다음 세션에 그 `❓`만 주면 노트를 보강한다. *노트는 완성품이 아니라 계속 자라는 문서.*
4. **트랙 하나가 끝나면 제목만 보고 💡 재생** → 안 나오는 노트만 다시 읽는다. (읽은 걸 다시 읽는 건 낭비, 못 꺼내는 걸 찾는 게 복습)

> 🕐 **2회분으로 쪼갤 큰 노트**: [도메인 검증](./java/design/domain-validation.md)(제일 큼) · [@Lock 실무](./java/jpa/lock-practical.md) · [@Transactional](./java/spring/transactional.md) · [Map 메서드](./java/collections/map-methods.md) · [JVM 동시성 도구](./java/concurrency/jvm-concurrency-tools.md) · [락 개념](./java/concurrency/locks.md) · [@Lock 심화](./java/jpa/lock-concepts.md) · [Jackson](./java/jackson/annotations.md) · [신호→도구](./algorithm/data-structure-selection.md)

## 🗺️ 추천 학습 순서 (트랙 1 → 9)

**카테고리 목록(아래)은 "찾아보는 사전", 이 순서는 "읽어나가는 길".** 순서는 노트끼리 실제로 서로를 참조하는 방향(선행 → 후행)으로 뽑았다. 앞 트랙을 건너뛰면 뒤 트랙에서 그 노트를 결국 다시 열게 된다.

> 🕒 **최근에 쓴 노트는 오히려 뒤로.** 방금 쓴 걸 읽는 건 인출이 아니라 재독이라 "안다"는 착각만 굳는다. 2~3주 지나 아래 순서에서 제 자리로 만날 때 읽는 게 낫다. (2026-08-03~04 작성분: [Stream API](./java/functional/stream-api.md) · [Spring Cache](./java/spring/spring-cache.md) · [메서드 보안](./java/security/method-security.md) · [계산 파이프라인](./java/design/calculation-pipeline.md) · [LATERAL](./database/lateral-join-top-n-per-group.md) · [LEFT JOIN ON/WHERE](./database/left-join-on-vs-where.md))

**트랙 1 · Java 기본기 (5개, 워밍업)** — *짧고, 나머지 트랙이 쓰는 어휘. 여기서 페이스를 만든다.*
[컬렉션 계층](./java/collections/collection-hierarchy.md) → [오토박싱·래퍼 캐시](./java/basics/autoboxing-wrapper-cache.md) → [Map 메서드](./java/collections/map-methods.md) → [리스트 생성](./java/collections/list-creation.md) → [람다 실행 타이밍](./java/functional/lambda-execution-timing.md)
> [SortedSet](./java/collections/sorted-navigable-set.md) · [BigDecimal](./java/bigdecimal/bigdecimal.md) · [varargs](./java/generics/varargs-safevarargs.md) · [Stream API](./java/functional/stream-api.md)는 **레퍼런스로 미룬다** — 다른 노트가 이들을 전제하지 않아서, 순서에 끼우면 본선만 늦어진다. 필요할 때 꺼내 읽는 쪽이 맞다.
> 람다 → Stream 순서만은 지킬 것. "람다는 코드 값일 뿐"을 먼저 잡아야 스트림의 lazy가 이상하지 않다.

**트랙 2 · 트랜잭션 · JPA (10개)** — *이 볼트의 최대 허브. 다른 노트 10개 이상이 여기를 참조한다.*
[영속성 컨텍스트](./java/jpa/persistence-context.md) → [@Transactional](./java/spring/transactional.md) → [롤백 예제](./java/spring/transaction-rollback-example.md) → [격리 수준](./java/jpa/transaction-isolation.md) → [Read-Modify-Write](./java/jpa/read-modify-write.md) → [@Lock 기본](./java/jpa/lock.md) → [@Lock 심화](./java/jpa/lock-concepts.md) → [@Lock 실무](./java/jpa/lock-practical.md) → [Criteria·Pageable](./java/jpa/spring-data-query.md) → [N+1](./java/jpa/n-plus-one-fetch.md)

**트랙 3 · 테스트 (8개)** — *트랙 2에서 배운 걸 "손으로 확인하는" 수단. 그래서 트랙 2 다음.*
[테스트 방법론](./java/test/test-writing-guide.md) → [JUnit 라이프사이클](./java/test/junit-lifecycle.md) → [AssertJ](./java/test/assertj.md) → [픽스처](./java/test/test-fixtures.md) → [파라미터화](./java/test/parameterized-test.md) → [Mockito](./java/test/mockito-service-test.md) → [repository 테스트](./java/test/jpa-repository-test.md) → [테스트 슬라이스](./java/test/spring-boot-test-slices.md)

**트랙 4 · 동시성 (8개)** — *락 노트가 JPA 락을 참조하므로 트랙 2 뒤가 자연스럽다.*
[스레드 기초](./java/concurrency/threads.md) → [메모리 가시성](./java/concurrency/memory-visibility.md) → [JVM 동시성 도구](./java/concurrency/jvm-concurrency-tools.md) → [락 개념](./java/concurrency/locks.md) → [데드락](./java/concurrency/deadlock.md) → [동시성 테스트](./java/test/concurrency-test.md) → [가상 스레드](./java/concurrency/virtual-threads.md) → [도구 선택 가이드](./java/concurrency/concurrency-tool-guide.md)

**트랙 5 · Spring 웹 · 프록시 계열 (9개)**
[@Valid·@Validated](./java/spring/validation.md) → [@AssertTrue](./java/spring/assert-true-cross-field.md) → [예외 처리](./java/spring/exception-handling.md) → [Jackson](./java/jackson/annotations.md) → [커스텀 어노테이션](./java/annotation/custom-annotation.md) → [Spring Cache](./java/spring/spring-cache.md) → [메서드 보안](./java/security/method-security.md) → ["배치"의 세 층위](./java/spring/batch-three-meanings.md) → [비밀번호 인코딩](./java/security/password-encoding.md)
> 💡 **묶어서 읽으면 한 번에 잡히는 것**: [커스텀 어노테이션](./java/annotation/custom-annotation.md) · [Spring Cache](./java/spring/spring-cache.md) · [메서드 보안](./java/security/method-security.md) · [@AssertTrue](./java/spring/assert-true-cross-field.md) · [@Transactional](./java/spring/transactional.md) — 전부 같은 함정을 공유한다. **① 활성화(`@Enable~`) 안 하면 예외가 아니라 "조용히 무시" ② 프록시라서 자기호출(this.method())은 안 먹는다.** 어노테이션 이름만 다를 뿐 같은 메커니즘이라, 붙여 읽으면 다섯 번 배울 걸 한 번에 배운다.

**트랙 6 · 설계 · DDD (6개)** — *제일 오래 사는 ③층. 단, 앞 트랙의 구체 경험이 있어야 와닿는다. 먼저 읽으면 격언집이 된다.*
[변환 계층](./java/design/transform-layers.md) → [포트와 어댑터](./java/design/ports-and-adapters.md) → [도메인 검증](./java/design/domain-validation.md) → [애그리거트 소유권](./java/design/aggregate-ownership.md) → [일급 컬렉션](./java/design/first-class-collection.md) → [계산 파이프라인](./java/design/calculation-pipeline.md)

**트랙 7 · 분산 · 실시간 (9개)** — *분산 락은 "락 + Redis + 스케일아웃"을 전부 요구해서 제일 뒤.*
[스케일 아웃](./infra/scaling.md) → [Redis 기초](./infra/redis/redis-basics.md) → [Pub/Sub](./infra/redis/redis-pubsub.md) → [Redisson 분산 락](./infra/redis/redisson-distributed-lock.md) → [실시간 통신 비교](./infra/network/realtime-communication.md) → [SSE](./infra/network/sse.md) → [SseEmitter](./java/spring/sse-emitter.md) → [WebSocket](./infra/network/websocket.md) → [Flux/Mono](./java/reactive/flux-mono-basics.md)

**트랙 8 · SQL 심화 (5개)** — *트랙 2(JPA)를 끝낸 뒤에 오면 "N+1을 쿼리로 푸는" 쪽이 보인다.*
[인덱스·실행계획](./database/index-explain.md) → [SQL 쿡북](./database/sql-cookbook.md) → [LEFT JOIN ON vs WHERE](./database/left-join-on-vs-where.md) → [LATERAL·top-N per group](./database/lateral-join-top-n-per-group.md) → [PG 날짜 함수](./database/postgresql-date-functions.md)

**트랙 10 · Spring 프록시 → AOP (김영한 고급편, 작성 중)** — *챕터의 한계가 다음 챕터를 부르는 서사가 있는 트랙. 강의 순서대로 읽는다.*
[ThreadLocal](./java/concurrency/thread-local.md) → [템플릿 메서드/전략/콜백](./java/design/template-method-strategy-callback.md) → [프록시/데코레이터](./java/design/proxy-decorator-pattern.md) → [동적 프록시](./java/design/dynamic-proxy.md) → [ProxyFactory](./java/design/proxy-factory.md) → [빈 후처리기](./java/design/bean-post-processor.md) → [@Aspect AOP](./java/design/aspect-aop.md) → [AOP 개념](./java/design/aop-concepts.md) → [AOP 구현](./java/design/aop-implementation.md) → [AOP 포인트컷](./java/design/aop-pointcut.md)

**트랙 9 · 레퍼런스 (8개, 순서 무관 — 필요할 때 꺼내 읽기)**
*도구*: [포트와 listen](./infra/network/ports-and-listen.md) → [SSH 포워딩](./infra/network/ssh-port-forwarding.md) → [SSH config](./infra/network/ssh-config.md) / [git worktree](./git/worktree.md)
*기본기 잔여*: [Stream API](./java/functional/stream-api.md) · [SortedSet](./java/collections/sorted-navigable-set.md) · [BigDecimal](./java/bigdecimal/bigdecimal.md) · [varargs](./java/generics/varargs-safevarargs.md) — 다른 노트의 선행이 아니라, 그 API를 실제로 쓸 때 펴는 쪽이 남는다

> ⚠️ **트랙을 섞지 말 것.** 매일 다른 트랙을 뽑아 읽으면 전부 "반씩 아는" 상태가 된다 — 코테 로드맵에 적어둔 것과 같은 이유. 한 트랙을 끝내고 다음으로.

---

## Java

### Concurrency (동시성)
- [ ] [스레드 기초 - 프로세스/스레드/플랫폼 스레드(OS 1:1)/생명주기 / 힙 공유→race](./java/concurrency/threads.md)
- [ ] [스레드 풀 내부 - core/max/queue 처리 순서(⚠️큐가 max보다 먼저→maxPoolSize 死) / Executors 팩토리는 큐·스레드 무한(OOM) / queueCapacity=0=SynchronousQueue 트릭 / 거부정책 4종·CallerRuns 백프레셔(shutdown 시 무음 유실) / @Async 폴백은 풀이 아님·자기호출 함정 / ⚠️join은 CompletionException으로 감싸 예외 핸들러·운영알림이 바뀜 / "병렬화보다 호출 묶기가 먼저"](./java/concurrency/thread-pool.md)
- [ ] [ThreadLocal - 쓰레드 전용 저장소 / 싱글톤 빈 필드 동시성 문제 해결 / 쓰레드 풀 재활용→remove() 필수(안 하면 타 사용자 데이터 노출) / 비동기·가상 스레드에선 배신](./java/concurrency/thread-local.md)
- [ ] [가상 스레드 - Java 21 JEP444 / M:N·캐리어·unmount / pinning(JEP491) / 락은 그대로 필요](./java/concurrency/virtual-threads.md)
- [ ] [메모리 가시성 - volatile·happens-before(JMM) / 가시성≠원자성 / volatile로 count++ 안 됨](./java/concurrency/memory-visibility.md)
- [ ] [락 개념 종합 - 낙관적/비관적/분산 락](./java/concurrency/locks.md)
- [ ] [JVM 동시성 도구 - synchronized/ReentrantLock/Semaphore/Latch/Barrier 등](./java/concurrency/jvm-concurrency-tools.md)
- [ ] [동시성 도구 목적별 정리 & 선택 가이드 - 만들기/조합(CompletableFuture)/대기/세기/정합성 / 최신≠최선](./java/concurrency/concurrency-tool-guide.md)
- [ ] [데드락(교착 상태) - Coffman 4조건/예방/DB 자동 감지](./java/concurrency/deadlock.md)
- [ ] [동시성 컬렉션 - ConcurrentHashMap(개별 연산만 원자적·check-then-act는 computeIfAbsent로)/CopyOnWriteArrayList(읽기多쓰기小)/BlockingQueue(스레드 풀 큐의 정체)](./java/concurrency/concurrent-collections.md)

### JPA
> 추천 읽는 순서: [영속성 컨텍스트](./java/jpa/persistence-context.md) → [@Transactional](./java/spring/transactional.md) → [트랜잭션 롤백 예제](./java/spring/transaction-rollback-example.md) → [Read-Modify-Write](./java/jpa/read-modify-write.md) → [@Lock 기본](./java/jpa/lock.md) → [@Lock 심화](./java/jpa/lock-concepts.md) → [@Lock 실무](./java/jpa/lock-practical.md)

- [ ] [영속성 컨텍스트 · flush · 더티 체킹 - "커밋 시점"의 정체](./java/jpa/persistence-context.md)
- [ ] [트랜잭션 격리 수준 - Dirty/Non-repeatable/Phantom/Lost Update와 락 관계](./java/jpa/transaction-isolation.md)
- [ ] [@Lock 기본 - 락 어노테이션 (언제·종류·사용·주의)](./java/jpa/lock.md)
- [ ] [@Lock 심화 개념 - @Version·공유/배타·FORCE_INCREMENT](./java/jpa/lock-concepts.md)
- [ ] [@Lock 실무 패턴 - 프록시·네이밍·테스트·재시도·벌크/조건부 UPDATE](./java/jpa/lock-practical.md)
- [ ] [Read-Modify-Write와 트랜잭션 경계 - 쓰기 서비스가 조회 서비스에 의존하면 안 되는 이유](./java/jpa/read-modify-write.md)
- [ ] [Criteria · Specification · Pageable · Page - 출처(Spring Data/JPA/자작)와 동적 조회·페이징 / ⚠️파생 쿼리는 OR 괄호 표현 불가 → 조용히 틀린 결과](./java/jpa/spring-data-query.md)
- [ ] [N+1과 fetch 전략 - fetch join, EntityGraph, batch size 판단 기준 / ⚠️"쿼리 1회=빠름"이 아니다 — 페이징 없어도 컬렉션 fetch join은 행×컬럼으로 비쌈(안 쓰는 연관이 전 행에 중복) / SQL은 55ms인데 fetch()가 2,553ms면 매핑 비용 → 프로젝션 + 직접 groupingBy](./java/jpa/n-plus-one-fetch.md)
- [ ] [연관관계 매핑 - 연관관계 주인·mappedBy(양방향=단방향 2개, FK는 하나) / ⚠️역방향에만 값 넣으면 FK null / 편의 메서드 / 1:N 단방향·N:M 금지 / ToOne은 기본 EAGER→LAZY 명시](./java/jpa/relation-mapping.md)
- [ ] [엔티티 설계 실무 규칙 - Setter 금지·모든 연관관계 LAZY·컬렉션 필드 초기화 후 교체 금지(PersistentBag)·@Enumerated STRING 강제 / @Entity 제약·hbm2ddl 환경별 규칙](./java/jpa/entity-design-rules.md)
- [ ] [기본 키 생성 전략 - IDENTITY는 persist 즉시 INSERT(쓰기 지연 무력화) / SEQUENCE+allocationSize / Long+대체키가 정석](./java/jpa/id-generation.md)
- [ ] [상속 매핑 - JOINED/SINGLE_TABLE(기본값)/TABLE_PER_CLASS(금지) 트레이드오프 / @MappedSuperclass는 상속 매핑이 아니라 BaseEntity(공통 필드) 도구](./java/jpa/inheritance-mapping.md)
- [ ] [프록시와 지연 로딩 - getReference 동작 원리 / ⚠️3대 함정: == 비교·find/getReference 순서별 동일성·LazyInitializationException(트랜잭션 밖 지연 로딩의 근본 원인)](./java/jpa/proxy.md)
- [ ] [영속성 전이와 고아 객체 - cascade·orphanRemoval은 "소유자 하나+수명 동일"일 때만 / ALL+orphanRemoval=애그리거트 루트 구현 / 정적 생성 메서드 패턴](./java/jpa/cascade-orphan-removal.md)
- [ ] [값 타입 - @Embeddable/@Embedded / 공유 참조 부작용→불변 설계 / ⚠️값 타입 컬렉션은 전체 DELETE+재INSERT→일대다 엔티티로 대체](./java/jpa/value-types.md)
- [ ] [준영속 엔티티 수정: 변경 감지 vs merge - ⚠️merge는 전체 교체(폼에 없는 필드 null 덮어쓰기) / save()의 정체=id 있으면 merge / "id+DTO 넘겨 변경 감지"가 정답](./java/jpa/merge-vs-dirty-checking.md)
- [ ] [JPQL 심화 - ⚠️벌크 연산은 영속성 컨텍스트 우회(@Modifying clearAutomatically) / 묵시적 조인 금지 / fetch join 한계 3종(별칭·컬렉션 2개·페이징) / getSingleResult 예외](./java/jpa/jpql-advanced.md)
- [ ] [OSIV - 기본 ON: 응답 완료까지 영속성 컨텍스트+커넥션 유지→실시간 트래픽에서 커넥션 고갈 / "고객 API는 OFF, ADMIN은 ON" / OFF 대응=커맨드·쿼리 서비스 분리](./java/jpa/osiv.md)

### MyBatis
- [ ] [resultMap 중첩 매핑 - association(단수)·collection(1:N) / `<id>`=정체성 판별(없으면 전컬럼 비교·그룹핑 오동작) / "이미 JOIN하면 컬럼+매핑 추가가 0비용" / 중첩 select는 N+1 / collection+LIMIT 함정](./java/mybatis/resultmap-association-collection.md)

### Spring
- [ ] [@Transactional - 선언적 트랜잭션·전파(propagation)·롤백 규칙·프록시 함정](./java/spring/transactional.md)
- [ ] [예제로 보는 트랜잭션 전파·롤백 - a→b→c 워크스루](./java/spring/transaction-rollback-example.md)
- [ ] [@Valid · @Validated - Bean Validation 동작·위치, 중첩 cascade 함정](./java/spring/validation.md)
- [ ] [@AssertTrue 필드 조합 검증 - boolean getter에 붙는 cross-field 규칙 / is·get 네이밍 아니면 조용히 무시 / null 가드 필수·Jackson 노출 / vs 클래스 레벨 제약](./java/spring/assert-true-cross-field.md)
- [ ] [Spring 예외 처리 - @ControllerAdvice, ErrorCode, Validation 예외 흐름](./java/spring/exception-handling.md)
- [ ] ["배치"의 세 층위 - 쿼리 배치화(N+1→IN+groupingBy) / Spring Batch(스케줄러 없음!) / JDBC 쓰기 배치](./java/spring/batch-three-meanings.md)
- [ ] [SseEmitter 서버 구현 - 서블릿 async(스레드 즉시 반납) / send·event()빌더·콜백3종 / 두 패턴(세션푸시+저장소·Pub/Sub / relay+구독) / 이벤트루프 블로킹 금지·boundedElastic](./java/spring/sse-emitter.md)
- [ ] [Spring Cache - @Cacheable·@CachePut·@CacheEvict 3형제 / 어노테이션=정책·CacheManager=저장소 분리 / @EnableCaching 없으면 조용히 무시 / 자기호출 우회→캐시 전용 컴포넌트 / ConcurrentMap·Caffeine·Redis "사본 허용?"](./java/spring/spring-cache.md)

### 기본 / 박싱 (Basics)
- [ ] [오토박싱 & 래퍼 캐시 - `Integer`·`Long`끼리 `==` 금지 / -128~127 캐시(JLS 5.1.7) → 작은 값엔 우연히 맞고 커지면 조용히 틀림 / JPA `Long id` 비교도 같은 메커니즘](./java/basics/autoboxing-wrapper-cache.md)
- [ ] [switch 문 vs 식 - Java14 JEP361 / 식은 enum 전체 커버 강제(exhaustiveness) = 상수 추가 시 컴파일 에러로 매핑 누락 잡는 안전망 / default는 "모든 미래 값에 공통 처리가 옳을 때"만(아니면 버그 은닉처) / 암묵 default→ICCE / values() 순회 검증도 함께 오염](./java/basics/switch-expression-exhaustiveness.md)

### BigDecimal
- [ ] [BigDecimal - 돈·정밀 계산, equals vs compareTo, scale, 반올림](./java/bigdecimal/bigdecimal.md)

### Collections (컬렉션)
- [ ] [리스트 생성 - singletonList/List.of/Arrays.asList, "불변 × 원소 개수"는 별개 축 / ⚠️불변 컬렉션은 null 거부→`List.of().contains(null)`·`Map.of().get(null)`도 NPE](./java/collections/list-creation.md)
- [ ] [Map 주요 메서드 - getOrDefault/putIfAbsent/computeIfAbsent(캐시)/merge(카운팅) / 람다 null·재진입 함정](./java/collections/map-methods.md)
- [ ] [컬렉션 계층 구조 - List/Set/Queue는 Collection, Map은 별도 / Set 집합연산=Collection 공통 / equals·hashCode](./java/collections/collection-hierarchy.md)
- [ ] [SortedSet · NavigableSet - TreeSet의 진짜 인터페이스 / first·last vs Collections.min·max / floor·ceiling 근처탐색 / ⚠️같음=compare==0(정렬기준이 중복기준)](./java/collections/sorted-navigable-set.md)

### Generics (제네릭)
- [ ] [가변인자(varargs) & @SafeVarargs - 제네릭 varargs의 heap pollution(타입안전성 오염) / 붙일 수 있는 위치 / 읽기만·노출 금지](./java/generics/varargs-safevarargs.md)

### Reactive (Reactor)
- [ ] [Flux/Mono 기초 - 본질은 논블로킹(실시간 아님) / lazy·신호 3종·에러=신호 / 이벤트 루프 블로킹·ThreadLocal 금지 / MVC선 가장자리만](./java/reactive/flux-mono-basics.md)

### 함수형 / 람다 (Functional)
- [ ] [람다 ≠ 비동기 - 람다는 "코드 값"일 뿐, 실행 타이밍은 받는 메서드가 정함 / forEach=즉시·on~/then~=콜백·submit/Async=다른 스레드 / 스트림 lazy·콜백 스레드엔 ThreadLocal 없음](./java/functional/lambda-execution-timing.md)
- [ ] [Stream API 종합 - 파이프라인(소스→중간lazy→최종) / 중간·최종·Collectors 연산 지도 / 1회용·peek=디버깅용·toMap 중복키 예외 / map vs flatMap(1:N은 Stream 반환 강제·Optional·thenCompose와 한 형제) / 필드 기준 중복제거는 distinct 아님→groupingBy+toSet / ⚠️lazy가 try-catch 무력화·CompletableFuture를 순차로 / 람다 중단점·Stream Trace 디버깅 / "for문의 목적을 말로 하면 연산 이름"](./java/functional/stream-api.md)

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
- [ ] [계산 파이프라인 구조 - Pipes and Filters + supports-execute + Collecting Parameter / 불변 Factor·정책·순서 고정 / ⚠️순서가 곧 스펙·supports 스킵은 무음 / 조회는 밖에서·계산 코어에 I/O 금지](./java/design/calculation-pipeline.md)
- [ ] [집계를 SQL vs 애플리케이션 - 판단 축은 건수와 유지보수성 / 수천 건 이하면 앱(테스트 가능·읽힘) / ⚠️"읽을 수 없는 SQL은 느려져도 아무도 못 고친다" — 성능 문제 방치의 진짜 원인 / 기간 오프바이원(N-1 시작·배타 상한 +1)](./java/design/aggregation-placement.md)
- [ ] [템플릿 메서드/전략/템플릿 콜백 - 변하는·변하지 않는 코드 분리 / 상속→위임→위임+람다 발전사 / V1(선조립=DI) vs V2(실행시 콜백=xxxTemplate) / 함수형 인터페이스=추상 메서드 1개→람다 / ⚠️원본 수정은 여전→프록시(4장) 예고](./java/design/template-method-strategy-callback.md)
- [ ] [프록시 패턴/데코레이터 패턴 - 같은 인터페이스 대리인(대체 가능성)→원본·클라이언트 수정 0 / 구조는 동일, **의도**로 구분(접근제어=프록시·기능추가=데코레이터) / 접근제어 3종(권한차단·캐싱·지연로딩)=@PreAuthorize·@Cacheable·getReference의 정체 / 클래스 기반 제약 3종(super(null)·final 클래스·final 메서드) ⚠️final 클래스는 기동시 시끄럽게 실패하지만 final 메서드는 CGLIB가 조용히 스킵→@Transactional 무음 사망 / V3 컴포넌트 스캔엔 못 끼움→빈 후처리기(7장) / 프록시 폭발→동적 프록시(5장)](./java/design/proxy-decorator-pattern.md)
- [ ] [동적 프록시 - 프록시 클래스를 런타임에 자동 생성→핸들러 1개로 대상 전부 / JDK(implements·InvocationHandler·Proxy.newProxyInstance) vs CGLIB(extends·MethodInterceptor·Enhancer) / Method는 메서드가 아니라 **메타정보 객체**—이게 파라미터라서 클래스 폭발이 풀림 / ⚠️ **`@Transactional`·`@Cacheable`이 바로 이 기술** — 자기호출 시 `this`는 프록시가 아니라 **target**이라 무음 실패 / ⚠️ Spring Boot 2.0+는 인터페이스가 있어도 **CGLIB이 기본** / Objenesis로 기본 생성자 제약은 사라짐 / equals·hashCode·toString도 invoke()로 들어옴 / invoke vs invokeSuper 무한루프](./java/design/dynamic-proxy.md)
- [ ] [ProxyFactory - 동적 프록시 추상화(인터페이스 있으면 JDK·없으면 CGLIB 자동 선택) / **내부에서 Advice를 호출하는 전용 InvocationHandler·MethodInterceptor를 자동 생성**—개발자는 Advice 하나만 / **Advisor = Pointcut + Advice**(짝을 보장하는 단위) / Pointcut = ClassFilter + MethodMatcher 둘 다 true여야 적용 / ⚠️ MethodInterceptor 이름 충돌(aopalliance vs cglib) / ⚠️ Advisor N개라도 **프록시는 1개**—Ch.4 데코레이터 체이닝(프록시 N개)과 대비 / addAdvisor 등록 순서=실행 순서 / Spring Boot는 proxyTargetClass=true 기본→항상 CGLIB / 남은 문제=설정 지옥·컴포넌트 스캔→빈 후처리기(Ch.7)](./java/design/proxy-factory.md)
- [ ] [빈 후처리기(BeanPostProcessor) - 빈 저장소 등록 **직전**에 가로채 다른 객체로 바꿔치기→설정 지옥·컴포넌트 스캔 둘 다 해결(드디어 V3 적용) / ⚠️ **Before/After는 "빈 등록" 기준이 아니라 `@PostConstruct` 기준** — 둘 다 등록 *전*에 실행 / 프록시 바꿔치기를 After에서 하는 이유=초기화된 완성품을 감싸야 안전 / **`@PostConstruct`도 `CommonAnnotationBeanPostProcessor`가 Before에서 호출하는 것**—스프링 내부도 같은 메커니즘 / ⚠️ **BPP에는 `@Order`가 안 먹힌다**(`Ordered` 인터페이스만) · order 값보다 그룹(PriorityOrdered>Ordered>나머지)이 우선 / **포인트컷이 두 번 쓰인다**—①클래스 단위=프록시 생성 여부 ②메서드 단위=어드바이스 적용 여부 / ⚠️ Advisor N개라도 프록시 1개이고 **각 Advisor의 포인트컷은 독립 판단**—전부 적용/미적용이 아님 / 이름만 보는 포인트컷은 스프링 내부 빈에 오폭→AspectJ 표현식 / "객체 생성+DI" ≠ "빈 저장소 등록"·순환 참조는 Boot 2.6부터 기본 금지](./java/design/bean-post-processor.md)
- [ ] [AOP 개념 - 횡단 관심사(cross-cutting concerns)를 애스펙트로 모듈화 / 위빙 3방식(컴파일·클래스 로딩·런타임) / 스프링=프록시 방식(런타임)→메서드 실행 제한 / 용어 체계(조인포인트·포인트컷·타깃·위빙) / AspectJ 문법 차용≠직접 사용 / ⚠️프록시 제약의 근본=메서드 오버라이딩](./java/design/aop-concepts.md)
- [ ] [@Aspect AOP - `@Around` 표현식=Pointcut·메서드 본문=Advice→**Advisor로 자동 변환**(Ch.6 수동 조립과 1:1 대응) / 핵심은 **`AnnotationAwareAspectJAutoProxyCreator` 2가지 역할**—①@Aspect→Advisor 변환·캐싱(`BeanFactoryAspectJAdvisorsBuilder`) ②프록시 생성(Ch.7) / `findCandidateAdvisors()`에서 @Bean Advisor와 @Aspect 변환분이 **한 리스트로 합류** / ⚠️ **ProceedingJoinPoint ≠ Pointcut** — Ch.6 `MethodInvocation`에 대응하는 호출 핸들(Join Point/Pointcut/PJP 3층 구분) / ⚠️ **빈 등록 안 하면 예외없이 조용히 무시**(조회 대상이 아니면 변환 자체가 안 일어남) / ⚠️ `proceed()` 빠뜨리면 target 미실행+null 반환 / ⚠️ **@Aspect 클래스 자신은 프록시 제외**(`isInfrastructureClass`)→안에 @Transactional 안 먹힘 / ⚠️ **@Order는 BPP에는 안 먹고 @Aspect에는 먹는다**—단 클래스 단위만·같은 aspect 내 타입 우선순위는 정의되고 같은 타입끼리만 미보장 / 자기호출 함정은 그대로 / Advice 5종·횡단 관심사·Ch.4~8 발전사](./java/design/aspect-aop.md)
- [ ] [스프링 AOP 구현 - `@Pointcut` 분리(void·빈 바디)·`&&` 조합·외부 `Pointcuts` 클래스 공용화(FQCN 참조·public 필수) / ⚠️ **`@Order`는 클래스 단위**—메서드에 붙이면 무음 무시→aspect를 static class로 쪼개야 순서 제어 / 값 작을수록 우선·바깥쪽 껍질 / Advice 5종(`@Around`만 `ProceedingJoinPoint`+`proceed()` 필수, 나머지는 `JoinPoint`) / ⚠️ `returning`·`throwing`은 **이름 매칭 + 타입 필터** 2역할—타입 좁으면 타겟은 정상 실행되고 **advice만 조용히 스킵** / ⚠️ `@AfterReturning`은 반환값 읽기만 가능·교체 불가(바꾸려면 `@Around`) / `@After`=finally의미라 `@AfterReturning` **뒤**에 호출 / ⚠️ `@Pointcut` 파라미터는 금지가 아니라 **바인딩 기능** / 무음 실패 4종 → `AopUtils.isAopProxy()`부터 4단계 진단](./java/design/aop-implementation.md)
- [ ] [스프링 AOP 포인트컷 - 지시자 10종을 **판단 축 3개**로 정리: ①정적(선언된 시그니처) vs 동적(런타임 실제 객체) ②프록시 vs 타깃 ③인스턴스의 클래스 vs 메서드 선언 클래스 / `execution`은 부모 타입 허용·`within`은 정확한 타입만 / ⚠️ 부모 타입 선언 시 **부모에 없는 메서드는 조용히 빠짐** / ⚠️ `.`과 `..` 하나 차이로 AOP 무음 미적용 / **`execution(* *(Object))` 실패 vs `args(Object)` 성공**—시그니처 vs `instanceof` / 🔴 **`@target`은 인스턴스 클래스·`@within`은 선언 클래스**—프록시 이야기가 아니다(this/target과 혼동 주의) / 🔴 **this vs target 8칸표—X는 딱 한 칸**(JDK 동적 프록시에서 `this(구체클래스)` 지정) / ⚠️ `args`·`@args`·`@target` **단독 사용 금지**—동적 판단→프록시 필요→로딩 시점 판단 불가→전체 빈에 프록시 시도→final 내부 빈에서 기동 실패 / 파라미터 바인딩(`@annotation(annotation)`+`.value()`가 실무 표준) / 공식 원칙 **kinded+scoping 최소 2종 포함** / `AspectJExpressionPointcut`으로 표현식 학습 테스트](./java/design/aop-pointcut.md)

### 보안 (Security)
- [ ] [비밀번호 - PasswordEncoder(단방향 해시) vs AttributeConverter(양방향 암호화) / 복호화 여부가 갈림길](./java/security/password-encoding.md)
- [ ] [메서드 보안 - @PreAuthorize·@EnableMethodSecurity / SpEL(#param·hasRole 접두사 자동) / 활성화 안 하면 조용히 무시·프록시 자기호출 / URL=경로 관문·메서드=개별 규칙](./java/security/method-security.md)

### Annotation (어노테이션)
- [ ] [커스텀 어노테이션 - @interface·메타 어노테이션(@Retention 기본=CLASS⚠️) / 처리기 3방식(리플렉션·AOP·컴파일타임) / SpEL 동적 값·@Order 순서·프록시 함정](./java/annotation/custom-annotation.md)

### Jackson
- [ ] [Jackson 어노테이션 종합 정리](./java/jackson/annotations.md)

## Database
- [ ] [SQL 케이스 쿡북 - 상황별 해법 (날짜 범위 조회, 인덱스/sargable 등)](./database/sql-cookbook.md)
- [ ] [PostgreSQL 날짜 함수 - to_date/make_date/EXTRACT/date_trunc](./database/postgresql-date-functions.md)
- [ ] [인덱스와 실행 계획 - EXPLAIN, range scan, full scan, sargable 조건](./database/index-explain.md)
- [ ] [LATERAL 조인과 top-N per group - FROM 절 for-each 루프 / MAX()론 "최신 행의 다른 컬럼" 못 뽑음 / LIMIT 없으면 조용히 중복 / N+1을 쿼리 안으로 / 절 평가 순서(FROM은 바깥 참조 X·SELECT는 O) / DISTINCT ON·윈도우함수 비교 / 계산식 재사용·집합반환함수](./database/lateral-join-top-n-per-group.md)
- [ ] [NULL 비교와 IS DISTINCT FROM - `!=`는 NULL이면 unknown→행이 조용히 사라짐 / NULL 안전 비교 / NOT IN 서브쿼리 NULL 함정(결과 0건) / COUNT·GROUP BY·UNIQUE의 NULL 취급 / 이종 테이블 COALESCE 병합 패턴](./database/null-comparison-is-distinct-from.md)
- [ ] [LEFT JOIN 자식 조건 ON vs WHERE - ON=매칭 규칙(부모 안전)·WHERE=생존 규칙(부모도 죽음) / NULL 비교로 조용히 INNER化 / soft delete 조건이 단골 사고](./database/left-join-on-vs-where.md)
- [ ] [SELECT FOR UPDATE - row 락으로 read-then-act 직렬화 / OF=JOIN 락 범위 지정 / 락 해제 후 재평가는 "잠긴 row가 변경된 경우만" → 잠근 row≠바뀌는 row 함정 / NOWAIT·SKIP LOCKED](./database/select-for-update.md)

## Infra / 분산 환경
- [ ] [스케일 아웃 & 배포 모델 - 1 JVM/인스턴스 복제/로드밸런서 vs 오토스케일러/무상태](./infra/scaling.md)

### Redis
- [ ] [Redis 기초 - "자료구조 공용 메모리 서버" / 명령어로 대화 / TTL / RedisTemplate opsFor* 매핑](./infra/redis/redis-basics.md)
- [ ] [Redis Pub/Sub - 저장 없는 방송(유실=스펙) / 구독=연결 열어두기 / Spring 3부품(convertAndSend·ListenerContainer·onMessage) / 스케일아웃 SSE](./infra/redis/redis-pubsub.md)
- [ ] [Redisson 분산 락 내부 동작 - 락="먼저 키 쓴 쪽이 주인" 관례(SETNX) / hash+Lua(재진입·주인식별) / pub/sub 대기(블로킹, true=즉시·false=waitTime 소진) / leaseTime 만료=보호 소멸⚠️ / DB 제약 이중 방어](./infra/redis/redisson-distributed-lock.md)

### 네트워크 (Network)
- [ ] [실시간 통신 기법 비교 - Polling/Long Polling/SSE/WebSocket 진화 / relay(중계) 패턴 / "양방향 필요한가"가 갈림길](./infra/network/realtime-communication.md)
- [ ] [SSE - text/event-stream 포맷 / EventSource(GET 전용·자동 재연결) vs POST fetch 스트리밍 / heartbeat·프록시 버퍼링·UTF-8 함정](./infra/network/sse.md)
- [ ] [WebSocket - Upgrade 핸드셰이크(101) / STOMP / 재연결·스케일아웃 세션 공유가 내 숙제](./infra/network/websocket.md)
- [ ] [포트와 listen - 연결 거부(refused) vs 응답 없음(timeout) / localhost vs 0.0.0.0 바인딩](./infra/network/ports-and-listen.md)
- [ ] [SSH 포트 포워딩 - -L/-R/-D / 중간 호스트는 원격 기준 해석 / 막힌 포트 우회](./infra/network/ssh-port-forwarding.md)
- [ ] [SSH config - Host 별칭 / LocalForward·IdentityFile / 작업·터널 별칭 분리](./infra/network/ssh-config.md)

## Git
- [ ] [git worktree - clone 없이 여러 폴더에 동시 체크아웃 / .git 공유·커밋 실시간 공유 / MR·PR ref 활용 / 동일 브랜치 금지](./git/worktree.md)

## Algorithm (코딩테스트)
- [ ] [코테 로드맵 - 프로그래머스 STEP 1~7 문제 목록 · 체크박스로 진행 추적 · 복습 큐(3일 뒤 재풀이)](./algorithm/roadmap.md)
- [ ] [자료구조 선택 - "신호 → 도구" 매핑 (누적 문서) / 풀이가 아닌 **신호**를 남긴다 / 개수=Map·존재=Set·최단거리=BFS](./algorithm/data-structure-selection.md)
- [ ] [Big-O와 입력 크기 - 풀기 전에 N 제약으로 복잡도 **역산**하기 / "N=10만이면 이중 for 불가" / 스트림 vs for는 상수 차이일 뿐](./algorithm/big-o-and-input-size.md)

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

# 면접 질문 은행 — 셀프 테스트

> **사용법**: 질문만 보고 **소리 내어(또는 글로) 답해본다** → 링크된 노트로 확인 → 막힘 없이 답했으면 `- [x]` 체크. 막히면 체크하지 말고 노트를 다시 읽는다.
> - **답은 여기 안 적는다** — 바로 보이면 인출 연습이 안 된다. 답은 링크된 노트에 있다.
> - ⭐ = 면접 단골. **꼬리:** = 실제 면접에서 이어지는 꼬리 질문(본 질문에 답하면 이것도 이어서).
> - 퀴즈 세션([quiz-log.md](./quiz-log.md))이 진행되면서 계속 추가된다.

---

## 스레드 기초

- [ ] ⭐ **스레드와 프로세스의 차이는? 스레드끼리 메모리는 어떻게 공유되나?** (힙/스택 구분까지)
  - 꼬리: 그럼 지역변수는 왜 스레드 안전한가? → [threads.md](./java/concurrency/threads.md)
- [ ] **run()과 start()의 차이는?** → [threads.md](./java/concurrency/threads.md)
- [ ] ⭐ **플랫폼 스레드는 왜 비싼가? 그래서 스레드 풀을 왜 쓰나?**
  - 꼬리: corePoolSize의 core는 CPU 코어 수와 관련 있나? → [threads.md](./java/concurrency/threads.md) · [thread-pool.md](./java/concurrency/thread-pool.md)
- [ ] **스레드 풀의 작업 처리 순서는? (core→queue→max)** 큐가 먼저라는 게 왜 함정인가? → [thread-pool.md](./java/concurrency/thread-pool.md)
- [ ] **데몬 스레드란? 어떤 작업을 맡기면 안 되나?** → [threads.md](./java/concurrency/threads.md)

## 동시성 문제와 제어

- [ ] ⭐ **동시성 문제는 어떤 조건에서 생기나?** (3요소) 단일 스레드 테스트로는 왜 못 잡나? → [thread-local.md §1](./java/concurrency/thread-local.md)
- [ ] ⭐ **lost update란? `count++`가 왜 위험한가?** (read-modify-write 3단계) → [jvm-concurrency-tools.md](./java/concurrency/jvm-concurrency-tools.md) · [read-modify-write.md](./java/jpa/read-modify-write.md)
- [ ] ⭐ **synchronized / volatile / Atomic의 차이는?** — 각각 어떤 병(원자성/가시성)을 고치나?
  - 꼬리: volatile 붙이면 count++ 안전해지나? 왜 아닌가? → [memory-visibility.md](./java/concurrency/memory-visibility.md) · [jvm-concurrency-tools.md](./java/concurrency/jvm-concurrency-tools.md)
- [ ] **가시성 문제란? 원인과 해법은?** (while 플래그가 안 멈추는 예) → [memory-visibility.md](./java/concurrency/memory-visibility.md)
- [ ] **check-then-act란? AtomicInteger를 써도 못 막는 이유는?** 해법은? → [jvm-concurrency-tools.md §7](./java/concurrency/jvm-concurrency-tools.md)
- [ ] **CAS의 동작 원리와 한계는?** (재시도 폭탄) → [jvm-concurrency-tools.md](./java/concurrency/jvm-concurrency-tools.md)
- [ ] ⭐ **데드락의 발생 조건과 실무 예방법은?** (락 획득 순서 통일)
  - 꼬리: DB에서 데드락 나면 영원히 대기하나? → [deadlock.md](./java/concurrency/deadlock.md)
- [ ] ⭐ **ThreadLocal은 언제 쓰나?** ("필드의 접근성 + 지역변수의 격리")
  - 꼬리: 스레드 풀 환경에서 remove() 안 하면? 어디서 remove 하나? 키가 약한 참조인데 왜 릭이 나나? → [thread-local.md](./java/concurrency/thread-local.md)
- [ ] **Semaphore와 synchronized의 차이는?** ("한 번에 몇 명") → [jvm-concurrency-tools.md §5](./java/concurrency/jvm-concurrency-tools.md)
- [ ] **ConcurrentHashMap을 쓰면 동시성 문제가 다 해결되나?** (개별 연산만 원자적) → [concurrent-collections.md](./java/concurrency/concurrent-collections.md)

## 락 — DB와 분산 환경까지

- [ ] ⭐ **낙관적 락 vs 비관적 락 — 각각 언제 선택하나?** (충돌 빈도 기준, 이유까지) → [locks.md](./java/concurrency/locks.md)
- [ ] ⭐ **@Version 낙관락의 충돌은 언제, 어떻게 감지되나? 충돌 후 수습은 누가 하나?** → [lock-concepts.md](./java/jpa/lock-concepts.md) · [locks.md](./java/concurrency/locks.md)
- [ ] ⭐ **서버를 2대로 늘리면 synchronized는 왜 무력해지나? 대안 배치는?** (검문소 위치 × 태도 매트릭스) → [locks.md](./java/concurrency/locks.md)
- [ ] ⭐ **선착순 쿠폰 100장, 동시성 어떻게 처리할 건가?** (설계형 단골)
  - 꼬리 체인: JVM 락으로 되나? → 2대면? → 낙관락이 탈락하는 이유는? → FOR UPDATE vs 조건부 UPDATE 한 문장? → [locks.md](./java/concurrency/locks.md) · [lock-practical.md §8](./java/jpa/lock-practical.md)
- [ ] **사용자가 폼을 열어두고 한참 뒤 저장하는 충돌 — 비관락 걸면 안 되는 이유는?** (오프라인 락) → [locks.md](./java/concurrency/locks.md)

## 가상 스레드

- [ ] ⭐ **가상 스레드란? 플랫폼 스레드와 뭐가 다른가?** (M:N, 캐리어, unmount)
  - 꼬리: 캐리어 스레드는 새로운 종류의 스레드인가? → [virtual-threads.md](./java/concurrency/virtual-threads.md)
- [ ] ⭐ **가상 스레드 도입 시 주의점은?** (커넥션 풀 병목→Semaphore · pinning · 풀링 금지 · CPU 바운드 무익) → [virtual-threads.md](./java/concurrency/virtual-threads.md)
- [ ] **pinning이란? Java 21에서의 처방과 Java 24의 변화는?** → [virtual-threads.md §5](./java/concurrency/virtual-threads.md)
- [ ] **가상 스레드가 해결해 주지 않는 것은?** (race·latency — throughput의 도구) → [virtual-threads.md §6](./java/concurrency/virtual-threads.md)

## JPA

- [ ] ⭐ **영속성 컨텍스트란? 뭘 해주나?** (1차 캐시·동일성·쓰기 지연·더티 체킹) → [persistence-context.md](./java/jpa/persistence-context.md)
- [ ] ⭐ **save() 호출 없이 UPDATE가 나가는 이유는?** (더티 체킹 — 감지 방법·시점까지) → [persistence-context.md](./java/jpa/persistence-context.md)
- [ ] ⭐ **지연 로딩은 어떻게 동작하나?** (프록시 초기화 4단계)
  - 꼬리: 프록시가 직접 쿼리를 날리나? "no Session"의 Session은 누구? → [proxy.md](./java/jpa/proxy.md)
- [ ] ⭐ **LazyInitializationException은 왜 발생하고, 어떻게 해결하나?** (강제 초기화 말고 경계 설계) → [proxy.md](./java/jpa/proxy.md) · [osiv.md](./java/jpa/osiv.md)
- [ ] ⭐ **N+1 문제란? 발생 경로와 해결책 비교는?** (EAGER+JPQL 포함) → [n-plus-one-fetch.md](./java/jpa/n-plus-one-fetch.md) · [relation-mapping.md](./java/jpa/relation-mapping.md)
- [ ] ⭐ **연관관계 주인이란? mappedBy 쪽에만 값을 넣으면 무슨 일이 생기나?** 왜 양쪽을 다 세팅하나? → [relation-mapping.md](./java/jpa/relation-mapping.md)
- [ ] **@ManyToOne의 기본 fetch 전략은? 실무 규칙과 이유는?** → [relation-mapping.md](./java/jpa/relation-mapping.md) · [entity-design-rules.md](./java/jpa/entity-design-rules.md)
- [ ] **@Transactional 메서드 안 try-catch로 커밋 시점 예외를 못 잡는 이유는?** 그럼 어디서 잡나? → [transactional.md](./java/spring/transactional.md) · [persistence-context.md](./java/jpa/persistence-context.md)
- [ ] **준영속 엔티티 수정 — merge의 함정과 정석은?** → [merge-vs-dirty-checking.md](./java/jpa/merge-vs-dirty-checking.md)
- [ ] **OSIV란? 기본값 ON의 트레이드오프는?** → [osiv.md](./java/jpa/osiv.md)

## 스프링 — 싱글톤·프록시·AOP

- [ ] ⭐ **싱글톤 빈에 상태 필드를 두면 안 되는 이유는? 해결책 2가지는?** → [thread-local.md](./java/concurrency/thread-local.md)
- [ ] ⭐ **템플릿 메서드 패턴 vs 전략 패턴 — 뭐가 다르고, 왜 위임(전략) 쪽이 표준이 됐나?** (상속의 강결합·단일 상속) → [template-method-strategy-callback.md](./java/design/template-method-strategy-callback.md)
- [ ] **JdbcTemplate·TransactionTemplate은 무슨 패턴인가?**
  - 꼬리: GOF 패턴인가? 전략 패턴과의 관계는? (전략의 변형 — 실행 시점에 파라미터로) → [template-method-strategy-callback.md](./java/design/template-method-strategy-callback.md)
- [ ] **함수형 인터페이스란? 람다로 구현할 수 있는 조건은?** (추상 메서드 딱 1개)
  - 꼬리: ⭐ 람다가 바깥 지역변수를 쓸 때 왜 effectively final이어야 하나? (스택·캡처=복사) → [template-method-strategy-callback.md §3-1·§4](./java/design/template-method-strategy-callback.md)
- [ ] **부가 기능 템플릿(또는 AOP 어드바이스)의 catch에서 예외를 삼키고 null을 반환하면 무슨 일이 벌어지나?** (실패의 성공 둔갑·롤백 무산·NPE 현장 이탈) → [template-method-strategy-callback.md §4](./java/design/template-method-strategy-callback.md)
- [ ] ⭐ **@Transactional은 어떻게 동작하나?** (프록시)
  - 꼬리: 같은 클래스 안에서 자기 메서드 호출하면 왜 안 먹나? → [transactional.md](./java/spring/transactional.md) · [dynamic-proxy.md](./java/design/dynamic-proxy.md)
- [ ] **횡단 관심사란? AOP가 해결하는 문제는?** → [aspect-aop.md](./java/design/aspect-aop.md) · [aop-concepts.md](./java/design/aop-concepts.md)
- [ ] **JDK 동적 프록시와 CGLIB의 차이는? 스프링은 뭘 쓰나?** → [dynamic-proxy.md](./java/design/dynamic-proxy.md)
- [ ] ⭐ **@Aspect를 붙이면 프록시가 적용되기까지 무슨 일이 일어나나?** (자동 프록시 생성기의 2가지 역할) → [aspect-aop.md](./java/design/aspect-aop.md)
- [ ] **Join Point / Pointcut / ProceedingJoinPoint를 구분해서 설명해보라** → [aspect-aop.md §3](./java/design/aspect-aop.md)

---

**만든 날**: 2026-08-19 · **계기**: 퀴즈 세션 중 "면접에서 이 정도까지 물어보나?" 질문 → 지금까지 학습한 노트 범위에서 면접 단골·꼬리 체인을 셀프 테스트용으로 집계. 새 트랙 진행 시 퀴즈 마스터가 계속 추가.

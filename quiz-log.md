# 퀴즈 오답노트 (quiz log)

> 퀴즈 세션에서 틀렸거나 막힌 문제를 누적하는 파일. 다음 세션은 여기 `- [ ]` 문제 재출제부터 시작.
> `- [ ]` = 재출제 필요 / `- [x]` = 통과

## 2026-08-11 — 동시성 (threads.md부터)

- [x] **Q1. 스레드 메모리 구조** — 힙 공유 / 스택 각자. 처음엔 반대로 답함(힙 각자·스택 공유). 힌트(지역변수 vs 필드) 후 정답. → 지역변수가 스택 프레임에 산다는 걸 몰랐음. → [threads.md §1](java/concurrency/threads.md) "지역변수는 왜 안전한가" 보강함. **[8/12 재출제(싱글턴 서비스 변형) 힌트 없이 통과 ✅]**
- [x] **Q2. lost update 시나리오** — `balance += 100` = 읽기→계산→쓰기 3단계(read-modify-write). 겹침 시나리오는 정확히 설명, "3단계로 쪼개진다"는 명명만 몰랐음.
- [x] **Q3. run() vs start()** — run()=현재 스레드에서 그냥 메서드 호출(동시성 X), start()=새 스레드 생성. 감은 정확.
- [ ] **Q4. 플랫폼 스레드 비용 + 대안** — 오답 2개:
  - 플랫폼 스레드를 "Spring 기본 스레드"로 혼동 → 자바 기본 Thread(OS 1:1)임.
  - 비싼 이유(스택 ~1MB 고정, 컨텍스트 스위칭) 못 답함.
  - 대안으로 CompletableFuture 답함 → 그건 "조합" 도구. "만들기"는 ExecutorService(풀).
  - **[8/12 재출제: 1:1·스위칭·ExecutorService 정답(지난 오답 교정 ✅). 스택 ~1MB 메모리 이유만 누락 — 이것만 한 번 더.]** "1:1 = 모든 비용이 OS 스레드의 비용" 사슬로 재정리해줌.
- 파생 질문: 스레드 ≠ CPU 코어, `corePoolSize`의 core는 CPU 코어와 무관 → [threads.md §3-1](java/concurrency/threads.md) 보강함.
- [x] **Q5. 임무별 도구 매칭** — 만들기=ExecutorService / 조합=CompletableFuture / 대기=CountDownLatch / 세기=AtomicInteger. 전부 정답.
- 파생 질문: Thread/ExecutorService/CompletableFuture 층 관계 (supplyAsync에 넘기는 건 스레드가 아니라 풀) → [concurrency-tool-guide.md §2-1](java/concurrency/concurrency-tool-guide.md) 보강함. CompletableFuture = 진동벨(결과 상자 + 후속 작업 예약).
- [ ] **Q6. 가시성 문제 (while(running) 안 멈춤)** — 오답: private/Async 추측. 가시성 개념 자체를 처음 접함. 원인(CPU 캐시 + JIT 호이스팅)과 해법(`volatile`) 학습. → [memory-visibility.md](java/concurrency/memory-visibility.md) 재출제 필요.
  - JIT 호이스팅은 "오늘 이해 무리, 다음에 다시" — 별도 재학습 항목.
  - **[8/12 재출제(ready 플래그 변형): 현상 이름(가시성)·캐시 메커니즘 정답 ✅. `volatile` 키워드 회상만 실패 — 단어만 한 번 더.]**
- [x] **Q7. volatile ≠ 원자성** — "volatile은 아무도 줄 세우지 않는다, 같은 값을 읽는 문제는 그대로"라고 스스로 정확히 추론. 가시성×원자성 표 + CAS(AtomicInteger) 학습.
- [x] **Q8. 상황별 도구 선택** — 플래그=volatile / 카운터=AtomicInteger / 이체(여러 변수)=synchronized 3/3 정답. 단 synchronized 사용법은 처음 배움: 블록=한 번에 한 스레드, "여러 변수 묶기"=블록 안 여러 줄이 한 덩어리(불변식 보호), 동기화≠동기/비동기(Async) 용어 구분.
- 파생 질문: synchronized 자물쇠 객체의 정체 — 능력 있는 키가 아니라 "같은 객체로 잠근 블록끼리만 배타"라는 이름표. this 노출 문제·자물쇠 분리 때문에 private lock 객체 사용. → [jvm-concurrency-tools.md §1](java/concurrency/jvm-concurrency-tools.md) "자물쇠 객체의 정체" 보강함 (보호하려면 전부 보호 + 임계영역 짧게 포함).
- [ ] **Q9. check-then-act (Atomic도 못 막는 틈)** — 힌트 받고 스스로 도달("차감 전에 B가 if 통과"). CAS 재시도 범위가 "그 연산 하나뿐"임을 처음 배움. 원자적 연산 2개 ≠ 원자적. → [jvm-concurrency-tools.md §7-2](java/concurrency/jvm-concurrency-tools.md) 보강함. 재출제 필요.
  - **[8/12 재출제(선착순 1명 변형): 메커니즘 정답("Atomic은 한 줄만 책임진다") ✅. 패턴 이름(check-then-act)·`compareAndSet` 해법은 처음 배움 — 한 번 더.]**
- [x] **Q10. synchronized로 고치기** — 메서드/블록 두 방식 모두 스케치 성공, "Atomic 불필요"(락=원자성+가시성)까지 스스로 추론. 배운 것: 모든 접근이 같은 자물쇠 통과해야 함, synchronized는 동기/비동기와 무관(병렬→한 명씩 직렬), 임계 영역은 짧게.
- 파생 학습(노트 통독 중): Thread 객체=리모컨·start()=시스템 콜(1:1의 정확한 의미) → [threads.md §3](java/concurrency/threads.md) 보강. new Thread 개수 제한은 OS가 강제·풀은 규율 도구 → [threads.md §3](java/concurrency/threads.md) 보강. 스케일 아웃 시대의 JVM 도구 용도(로컬 자원/프레임워크 내부/조율/테스트) → [jvm-concurrency-tools.md §0-1](java/concurrency/jvm-concurrency-tools.md) 보강.
- [x] **Q11. 생명주기 + 데몬** — NEW/BLOCKED/TIMED_WAITING/WAITING 4/4, 데몬 보너스(유저 스레드 끝나면 JVM 즉시 종료)까지 정답. 추가 학습: 데몬은 finally 보장 없이 죽음 → 중요 작업 금지.

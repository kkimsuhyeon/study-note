# 퀴즈 오답노트 (quiz log)

> 퀴즈 세션에서 틀렸거나 막힌 문제를 누적하는 파일. 다음 세션은 여기 `- [ ]` 문제 재출제부터 시작.
> `- [ ]` = 재출제 필요 / `- [x]` = 통과

## 🤝 세션 인수인계 (새 세션이 이어받는 법)
- **역할**: 퀴즈 마스터. 노트 기반 출제(한 문제씩) → 답변 → 틀리면 힌트→재시도→해설 → 결과를 이 파일에 기록. 모르는 내용·파생 질문이 나오면 해당 노트에 ⚠️/💡 보강(캡처 모드, CLAUDE.md 참조). 문제 유형: 사용법/메커니즘/판단형 믹스, 판단형 비중 높게. 세션 시작 = 이 파일의 `- [ ]` 스윕부터(간격 반복).
- **현재 위치 (2026-08-18)**: 트랙 10(스프링 프록시→AOP) 복습 진행 중 — "스프링 기본기 같이 배우기" 모드(챕터마다 스프링 연결 고리 문항 포함). **ThreadLocal 챕터 완료(Q27·Q28)**. 8/18 주간 회독 1회차(동시성 W1~W3)도 완료. 다음 챕터: 템플릿 메서드/전략/콜백(Q29부터). 사용자 목적 확인(8/18): ①노트 전체 지도 인지 ②면접 준비(말로 설명 연습 포함) ③실전 감각 — 면접 각도 변형·챕터말 지도 연결·노트에 없는 내용 적극 보강+보고.
- **일시 중단된 트랙**: JPA 퀴즈(Q26까지, persistence-context→relation-mapping→proxy 완료. 다음: cascade/merge/OSIV 확인 퀴즈). 동시성 잔여 스윕 항목은 8/12 섹션 하단 참조.
- **주간 회독 (8/18 신설, 사용자 합의)**: 주 1회(주초 세션) 지난주 학습 범위 전체를 재스윕 — 로그 오답 재탕이 아니라 **변형 + 노트 기반 신규 문항** 위주로. 1일→4일→7일 간격 반복 사다리 완성 목적. 첫 대상: 동시성 전체(8/11~14) — 잔여 스윕 항목(8/12 섹션 하단)을 신규 문항으로 흡수해서 진행.
- **면접 질문 은행 (8/19 신설)**: [interview-questions.md](interview-questions.md) — 셀프 테스트용 질문만 모은 파일(답은 노트 링크로). 퀴즈 진행 중 면접급 문항·꼬리 체인이 나오면 **여기에도 추가**할 것.

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

## 2026-08-12 — 락 개념 (locks.md)

- [x] **Q12. 낙관 vs 비관 전략** — synchronized=비관/CAS=낙관, 충돌 빈도 직감까지 정답. 보충: 왜 빈도로 갈리나(낙관=재시도 폭탄 위험, 비관=헛일 없지만 불필요한 줄 세우기).
- [ ] **Q13. 두 축(전략×범위) 직교** — "범위가 다르다"는 감은 잡았으나 "분산락이 낙관/비관을 커버하는 상위 버전 아닌가" 혼동. 학습: 4칸 매트릭스(낙관·비관 × DB·분산), **@Version은 DB 범위라 스케일 아웃에도 동작**(무력해지는 건 JVM 락뿐). 재출제 필요.
  - 후속 정리에서 "검문소 위치(JVM/DB/Redis) × 태도(잠그기/사후검증)" 모델로 재정립. JVM 범위에도 낙관(CAS)·비관(synchronized)이 있음을 연결.
- [x] **Q14. 상황별 락 배치** — JVM 락/FOR UPDATE/@Version/분산락 4/4 정답. 판단 기준(데이터가 어디 사나×충돌 빈도×DB 부하) 적용 성공.
- [ ] **Q15. @Version 충돌의 실제 흐름** — ①비교 시점을 "조회 시"로 혼동 → 저장 시 UPDATE WHERE절이 비교기, "0건=충돌 신호". ②"자동 재시도되지 않나" → 예외(OptimisticLockException)+롤백뿐, 자동 재시도 없음. ③수습은 앱 책임(@Retryable도 앱 레벨) — 정답. 판단: 재시도 vs 안내 = "남의 변경을 덮어쓰나"로 결정. ①② 재출제 필요.
- 파생 질문: "누가 줄을 세워주나" — 낙관=줄 없음(예외→앱), 비관=DB가 조용히 대기시킴(에러 안 남, 타임아웃·데드락 때만 예외), 분산=Redisson이 pub/sub+재시도로 대기 구현. → [locks.md](java/concurrency/locks.md) 표 보강함.
- [x] **Q16. 폼에 비관락 걸면? (Optimistic Offline Lock)** — 핵심(사용자가 관리자 올 때까지 대기) 정답. 보충: 무한 로딩(에러조차 없음)+커넥션 풀 고갈+웹은 폼열기/저장이 다른 트랜잭션이라 구조적 불가 → 사람 think-time엔 낙관락이 유일.
- 파생 교정: pub/sub은 Redis 내장 기능(PUBLISH/SUBSCRIBE). Redis=부품(SET NX·pub/sub·TTL), Redisson=부품을 조립한 완성품 락(+watchdog).
- 파생 질문: 커넥션 풀 고갈 — 대기자도 커넥션을 쥔 채 기다려서 점점 잠식→전체 마비 (transactional.md에 기존 내용 있음). 오프라인 락 vs 일반 낙관락 = "version이 여행하는 거리"(서버 메모리 vs 브라우저 왕복), 클라이언트 version = 충돌 판정 기준점(없으면 감지 못 하고 덮어씀).
- [x] **Q17. 데드락 시나리오** — ①서로 대기 ②데드락 정답. ③"한 번에 모두 잠그기"(점유와 대기 깨기) — 유효한 답, 실무 표준인 **락 획득 순서 통일**(순환 대기 깨기, min ID부터)은 신규 학습. 보충: DB가 자동 감지해 victim 롤백(영원히 대기 아님).
- ⭐ 사용자 통찰: 일반 낙관락=트랜잭션 겹침(동시) 충돌, 오프라인 락="본 시점 이후"의 모든 변경(동시 아니어도) — "낡은 화면 덮어쓰기 방지"가 목적. → [locks.md](java/concurrency/locks.md) 💡 섹션으로 캡처.
- [x] **Q18. S락 업그레이드 데드락** — ①S끼리 공존 ③처음부터 FOR UPDATE 정답. ②"서로 상대의 S락 대기"의 원형 구조(S락은 트랜잭션 끝나야 풀림 ↔ 트랜잭션은 UPDATE가 끝나야 종료)는 마지막 조각 보충. ReadWriteLock 업그레이드 불가와 같은 구조임을 연결.

## 2026-08-12 — 가상 스레드 (virtual-threads.md)

- [x] **Q19. 블로킹 낭비와 unmount** — ①스택·자원 쥐고 아무것도 안 함(낭비) ②"실행 중인 것만 있으면 된다"는 정확한 추론(정답: 코어 수 — 실행의 상한이 코어라서). 캐리어 기본값=코어 수 이유 학습.
- 파생 질문: "가상 스레드 = 스레드 대기열?" → 교정: 완전한 실행 흐름(힙에 스택+상태), 없는 건 전용 OS 스레드뿐. 2단 스케줄링 프레임(OS: 스레드→코어 / JVM: 가상→캐리어) → [virtual-threads.md §2](java/concurrency/virtual-threads.md) 보강함.
- [x] **Q20. 가상 스레드가 안 해주는 것** — (a)race는 무관(정합성=락의 일) 정답. (b)CPU 바운드 무익도 스스로 의심으로 도달. 핵심 구분 학습: **latency가 아니라 throughput의 도구** (DB 200ms는 그대로, 동시 수용량만 증가).
- 파생 질문: ExecutorService 풀(core/max) vs 가상 스레드 → 풀의 전제(비쌈·개수 위험)가 사라져 **풀링 금지**, 작업마다 새로. 동시 수 제한은 Semaphore로.

### 동시성 7개 노트 1회독 완료 (8/12)
- 남은 스윕 항목: 재출제(Q13 두 축, Q15①②), 단어 회상(스택 1MB·volatile·check-then-act·compareAndSet), jvm-tools 각론(ReentrantLock·Semaphore·CyclicBarrier·StampedLock), happens-before·DCL, Coffman 명칭, JIT 호이스팅(보류), pinning(virtual-threads §5 — 미출제)
- 다음 세션 계획: ①스윕 → ②[동시성 컬렉션](java/concurrency/concurrent-collections.md) 신규 노트(8/12 작성) 읽기 → 확인 퀴즈

## 2026-08-14 — 스윕 (하루 간격 복습)
- ⚡ 라이트닝: L1 스택 1MB ❌(힙이라 답함 — "각자 갖는 건 스택" 앵커로 교정) / L2 volatile △(개념 회상 성공, 스펠링만) / L3 check-then-act ❌(해결책 CAS와 혼동 — "병명 vs 처방" 구분) / L4 compareAndSet △(getAndSet도 선착순엔 동작하나 조건형은 CAS만). **L1·L3 재출제 예약.**
- [x] **Q13 재출제(2×2 표 배치)** — 5칸 배치 + "무력해지는 건 JVM 범위뿐" 전부 정답 ✅. JVM 낙관 칸(CAS)만 보충. 통과.
- [x] **Q15 재출제(동료 말 교정)** — ①비교는 조회가 아니라 UPDATE 때 ②JPA는 예외+롤백까지, 수습은 앱 — 둘 다 스스로 교정 ✅. 통과.

## 2026-08-18 — 스윕 (4일 간격)
- ⚡ S1 스택 1MB ✅(힙→스택 교정 완료, L1 통과) / S2 check-then-act ✅(표기만 교정, L3 통과) / S3 LazyInitializationException △(개념 완벽·이름만 실패, "optimistic"과 혼선 — 이름만 한 번 더) / S4 101번 ✅(Q25② 통과) / S5 mappedBy=읽기 전용 거울 ✅(자기 말로 "JPA는 주인만 보고 UPDATE" — Q23① 통과).
- ⚡ L5 LazyInitializationException 이름 ✅ — "lazyinitialization" 회상 성공 (S3 잔여 해소, 통과).
- ⚡ L6 Q26② 재출제 — ①비어있는 건 target ✅, "가로채기=상속 오버라이드" 확인 질문으로 해소. ②초기화 요청 대상(영속성 컨텍스트)은 2회 회상 실패 → 해설: **"no Session의 Session = 영속성 컨텍스트"** 앵커 + "프록시가 직접 쿼리를 날릴 수 있다면 이 예외는 존재할 이유가 없다" 논증 → [proxy.md §3(3)](java/jpa/proxy.md) 앵커 보강.
- 남은 재출제: Q26②(다음 스윕: "no Session의 Session은 누구?" 각도로 1회), ThreadLocal 정리 위치 명칭(필터 try-finally/인터셉터 afterCompletion — Q28③b), lost update 이름(앵커: "누락"=lost — W1, 2회째), 가시성↔원자성 오명명 교정 확인(W1).

## 2026-08-14~18 — 트랙 10: 스프링 프록시→AOP 복습 (ThreadLocal부터)
- [x] **Q27. ThreadLocal 스토리 복원(+스프링 웜업)** — ⓪싱글톤 1개 ✅(이유는 보충: 재사용 효율 ↔ **무상태 계약**, 위반이 곧 ②) ①파라미터 오염 ✅ ②싱글톤 필드 공유→덮어씀 ✅ ③두 축 표는 처음엔 막힘 → **"값은 ThreadLocal이 아니라 각 Thread 객체(ThreadLocalMap)에 산다"** 반전 + 사물함 비유로 해소. "필드의 접근성 + 지역변수의 격리".
- ⭐ 파생 통찰(스스로 도달): 공유 진영(synchronized/Atomic/volatile = 안전하게 공유) vs 격리 진영(ThreadLocal = 애초에 공유 안 함) — 노트 §5 구도를 자력 재발견. "톰캣 스레드 = 트랙 4의 그 플랫폼 스레드" 합류 확인. "뒤섞임 = 여러 번이 아니라 **동시에**"(순차 1,000번은 무사) 구분.
- [x] **Q28. remove 시나리오 ①②③** — ①풀 반납→재활용→A 데이터 잔존 사슬 스스로 복원 ✅ ②보안 사고 즉답 ✅, 메모리 릭은 GC 힌트 후 "계속 쌓인다"로 도달 ③(a) finally·level 0 시점 도달 ✅ — 단 "스레드가 끝났다"→"**요청**이 끝났다(스레드는 풀로 돌아갈 뿐)"로 교정 (b) 위치 명칭(필터 try-finally/인터셉터 afterCompletion)은 힌트로 노출됨 → **라이트닝 회상 1회 예약**. 파생 보강: 릭의 참조 사슬 + "키는 약한 참조인데 왜 릭?"(고아 엔트리, 면접 꼬리 질문) → [thread-local.md §4·§5](java/concurrency/thread-local.md) 추가. **ThreadLocal 챕터 완료.**

## 2026-08-18 — 주간 회독 1회차: 동시성 전체 (8/11~14 범위)
- [x] **W1. 선착순 쿠폰 종합(신규 시나리오)** — ①현상 재현 완벽(++ 겹침→1만 증가, if 통과 후 초과 발급) ✅, 단 "가시성"으로 오명명 → **가시성(안 보임, volatile) vs 원자성(겹침)** 재교정: 이 코드는 volatile로 안 고쳐진다 반증. 병2 check-then-act ✅ / **병1 lost update 이름 회상 실패(Q2 이후 2회째)** — 앵커: 본인 표현 "값이 누락"의 누락=lost. 라이트닝 예약. ②synchronized(어순: public synchronized void)+CAS 조건형 루프, 서버 2대 무력 이유(JVM 락 범위+필드 자체가 서버별) ✅ ③(a)분산락만으론 불가—값부터 공유 저장소로(각 서버 100장=200장), 값이 DB 가면 분산락 없이 DB 락으로 충분 (b)DB 비관락 FOR UPDATE, 초고충돌→낙관은 재시도 폭탄이라 탈락 ✅. 보너스: 단순 차감이면 **조건부 UPDATE 한 문장**([lock-practical.md §8](java/jpa/lock-practical.md)) — check+act를 SQL 한 문장으로.
- [ ] **W2. 가상 스레드 × 커넥션 풀 50개 (Semaphore 신규 학습)** — ①"가상 스레드는 싸다" 직감 ✅이나 핵심(톰캣 200이 자연 상한이었는데 소멸 → 수만 손이 커넥션 50개에 몰림 → 타임아웃 폭탄)은 신규 학습. Q20 "throughput 도구, 한정 자원은 그대로" 연결. ②Semaphore 회상 실패(첫 출제) — permit N장, acquire/release(finally 필수), "풀에서 재사용 빼고 개수 제한만 남긴 도구" 프레임 학습. ③synchronized=1명 vs Semaphore=N명은 **스스로 추론 성공** ✅ (Semaphore(1)≈synchronized, 주차장 차단기 비유). 재진술: ②③ 통과, ①은 "커넥션이 늘었다"로 인과 뒤집음 → **"커넥션은 50개 그대로, 늘어난 건 줄 서는 손"** 교정 — ① 사슬+Semaphore 이름 재출제 예약. 노트: [jvm-concurrency-tools.md §5](java/concurrency/jvm-concurrency-tools.md). ⭐ 파생 통찰(스스로 도달): "스레드를 늘려서 좋을 건 항상 없다 — 상한은 병목이 정한다"(플랫폼=스택 비용, 가상=하류 한정 자원) + 커넥션 풀=DB의 방패(구조상 Semaphore) → [virtual-threads.md §7](java/concurrency/virtual-threads.md) ⭐ 캡처.
- 파생 교정(W2 후속 문답): ①"풀링 금지"는 스레드 풀만 — 커넥션 풀(HikariCP)은 유지, 판단 기준 "그 자원이 아직 비싼가" → [virtual-threads.md §4](java/concurrency/virtual-threads.md) ⚠️ 보강 ②층 그림: 예전 요청→스레드풀(200)→커넥션풀(50)→DB에서 스레드 풀 층 소멸이 사고 원인, Semaphore=사라진 검문소 재건 ③ExecutorService는 계속 씀(newVirtualThreadPerTaskExecutor도 ExecutorService, 내용물이 풀→매번 새로).
- [ ] **W3. pinning (미출제 신규)** — ①②③ 전부 회상 실패 + **"캐리어 스레드가 뭐야?" 용어 공백 발견**(Q19 통과했으나 증발) → 캐리어=플랫폼 스레드의 역할 이름(새 종류 아님)·버스 비유로 재구축 → [virtual-threads.md §2](java/concurrency/virtual-threads.md) ⚠️ 보강. ①pinning 해설(synchronized 안 블로킹→버스 깔고 앉음). ②"캐리어를 기다린다" 스스로 도달 ✅, 단 피해 프레임 교정: 부하가 아니라 **정지**(CPU 놀면서 처리량 0, 대기 자체는 거의 공짜 — 힙에 상태만). ③해설: Java 21=블로킹 임계구역 ReentrantLock / Java 24 JEP 491=synchronized pinning 해소. **전체 재출제 예약**(pinning 이름·메커니즘·처방). [8/19 후속: 캐리어 모델 자기 말 재진술 성공 ✅("실행할 때만 타는 버스, 대기 발생 시 하차") — 재출제 범위는 pinning 이름·처방만으로 축소.] 파이프라인 재구성(스스로): 예전 요청→스레드풀→커넥션풀→DB / 지금 요청→가상스레드 생성→캐리어 탑승→(Semaphore)→커넥션풀→DB ✅.
- 파생 문답(그림 요청, 8/19): "세마포어 대신 캐리어로 제어하나?" 혼동 → **층 구분** 해설: 파이프라인(코드가 무엇을 하나: Semaphore→커넥션풀→DB) vs 캐리어(그 코드가 어디서 실행되나). 캐리어=이동 수단 아니라 "엔진 달린 좌석", 검문소 아님. **모든 대기(세마포어·커넥션·DB 응답)가 하차(unmount) 지점** → [virtual-threads.md §2](java/concurrency/virtual-threads.md) 보강. 요청 하나의 여정(탑승/하차 스윔레인) 다이어그램 제공.
- **주간 회독 1회차 종료** — W1 통과 / W2·W3 신규 학습(재출제 예약). 다음 회차 후보: ReentrantLock 각론·happens-before·DCL·Coffman 4조건 명칭·concurrent-collections·CountDownLatch/CyclicBarrier.

## 2026-08-14 — JPA 퀴즈 재개 (persistence-context)
- [x] **Q21. 더티 체킹** — 이름·시점(③ commit 직전 flush) 정답. ③ 감지 방법은 "영속화"까지 접근 → **스냅샷 비교**(조회 때 찍은 사진 vs flush 때 현재)로 완성. merge 노트 자발적으로 읽음 — merge 조회 기준=PK 질문.
- [x] **Q22. try-catch 헛스윙** — "트랜잭션 끝나고 터지니까 안 잡힌다" 스스로 도달 ✅. 파생 통찰(베스트 질문): "바깥에도 @Transactional 있으면 거기서도 안 잡히지 않나?" → 맞음, 같은 트랜잭션 참여 시 commit은 최외곽 1번 → **트랜잭션 없는 곳에서 잡기**(컨트롤러/파사드 또는 @Retryable=새 트랜잭션 재시도) 학습.
- [ ] **Q23. 연관관계 주인 (FK null)** — ①주인/mappedBy 역할 구분 처음 배움(주인=FK 쓰기 권한, mappedBy 쪽=읽기 전용 거울, JPA는 주인만 봄) ②양쪽 세팅은 경험으로 정답 ✅ ③편의 메서드 "재귀" 공포 해소 — **한쪽에만 만들고 반대편은 단순 대입/add로**(상대 편의 메서드 호출 금지). ① 재출제 필요.
- [x] **Q24. 주인 판별 즉석 확인** — 4/4 정답. 스스로 규칙 도출("FK 들고 있는 쪽이 주인") → @ManyToOne 쪽이 항상 주인으로 확정. 파생: 주인만 세팅 시 DB는 정상·같은 영속성 컨텍스트 안 컬렉션만 빈 상태(→양쪽 세팅 이유), 편의 메서드=공식 출입문 하나.
- [ ] **Q25. ToOne 기본 EAGER + JPQL N+1** — ①EAGER ✅ ③LAZY 명시 규칙 △. ②"조인해서 1번"이라 답함 ❌ — **JPQL은 쓴 대로 번역 후 EAGER 계약 이행으로 1+N 강제**(em.find만 JOIN 한 방) 학습. LAZY vs EAGER = 제어권 차이. ② 재출제 필요.
- [ ] **Q26. 프록시 메커니즘** — ①프록시 ✅ ③에러 방향 ✅(예외 이름은 몰랐음). ②초기화 메커니즘 처음 배움: 상속한 가짜(id+target), 메서드 가로채기 → **영속성 컨텍스트에 초기화 요청** → target 연결·위임. ③=LazyInitializationException("no Session") — 영속성 컨텍스트가 닫혀 요청할 곳이 없어서. OSIV ON이 이걸 가리는 이유까지 연결. ②③ 재출제 필요.

## 2026-08-13 — JPA 트랙 시작
- **Q21 (미답변·보류)**: 더티 체킹 — save() 없이 UPDATE 나가는 이름/시점/감지 방법. persistence-context.md 기반. 다음 세션 재출제.
- 김영한 PDF 3종(기본편·활용1·활용2) 전수 리뷰 → JPA 신규 노트 10개 작성(연관관계 매핑·엔티티 설계 규칙·키 생성·상속 매핑·프록시·cascade·값 타입·merge·JPQL 심화·OSIV) + 기존 노트 5개 보강(n-plus-one 권장순서·distinct, transform-layers 직렬화 근거, domain-validation 패턴 용어·check-then-act 링크, transactional readOnly 근거, persistence-context 링크). 전부 README 등록.
- JPA 퀴즈는 노트 읽기와 병행 예정 — 추천 순서: persistence-context(기존) → relation-mapping → proxy → cascade → merge → 이후 @Lock 트랙(동시성 선행지식 활용).

## 2026-08-14 — @Aspect AOP (고급편 Ch.8)

> 세션 시작 질문: "어디까지 알아야 하고 외워야 하는지 감이 안 잡힌다" → 결론: 외울 건 없고 **`AnnotationAwareAspectJAutoProxyCreator`의 2가지 역할** 하나. → [aspect-aop.md](java/design/aspect-aop.md)

- [ ] **Q22. 자동 프록시 생성기의 2가지 역할** — "@Aspect를 어드바이저로 만든다"와 "프록시를 만든다" **둘 중 어느 쪽인지 헷갈려 함** — 정답은 **둘 다**. ①@Aspect→Advisor 변환·캐싱(`BeanFactoryAspectJAdvisorsBuilder`) ②Advisor 기반 프록시 생성(Ch.7 그대로). 이름의 `AnnotationAware`가 ①을 가리킨다는 암기법 제시. **면접 단골 — 재출제 필요.**
- [x] **Q23. @Around 메서드는 무엇으로 변환되나** — "표현식=포인트컷, 안에 어드바이스"까지 감으로 정답. 다만 **"둘을 합쳐 Advisor로 자동 변환된다"는 결론까지는 도달 못 함**. Ch.6 `new DefaultPointcutAdvisor(pointcut, advice)`와 1:1 대응표로 정리.
- [ ] **Q24. ProceedingJoinPoint의 정체** — ⚠️ 오답: **"프록시 생성 여부를 결정하는 조건"(=Pointcut)으로 혼동**. 실제는 Ch.6 `MethodInvocation`에 대응하는 **호출 핸들**(proceed()·getTarget·getArgs·getSignature). 이름의 `Point` 때문에 생긴 혼동으로 보임 → **Join Point(후보 지점) / Pointcut(고르는 규칙) / ProceedingJoinPoint(골라진 그 순간의 컨텍스트)** 3층 구분을 [aspect-aop.md §3](java/design/aspect-aop.md)에 별도 정리. **재출제 필요.**
- [ ] **Q25. @Aspect만 붙이면 동작하나** — 방법(@Bean 직접 등록 / @Component 스캔)은 **정확히 답함**. 다만 이유가 "스프링이 관리해야 하니까"로 막연 → 구체화: 자동 프록시 생성기가 **"컨테이너에서 @Aspect 빈을 조회"**하므로, 빈이 아니면 조회 대상이 아니고 → 변환 자체가 안 일어난다. ⚠️ **예외도 경고도 없이 조용히 무시** — @EnableCaching·@EnableMethodSecurity와 같은 실패 모양. 이유 재진술 필요.
- [x] **Q26. 횡단 관심사 설명** — "각 함수의 공통 작업과 고유 작업을 분리해 목적만 남긴다"로 **내용은 정확**. 본인이 "그림은 알겠는데 말로 설명하기가 어렵다"고 표현 → 정의 정리: **"특정 기능 하나에 속하지 않고 여러 기능을 가로질러(cross) 걸쳐(cutting) 있는 관심사"**. → 용어 회상 연습만 필요.

### 세션 후 조사로 추가된 미출제 항목 (다음 세션 출제 후보)
- `@Aspect` 클래스 자신은 `isInfrastructureClass`로 **프록시 대상에서 제외** → 안에 `@Transactional` 붙여도 안 먹힘
- `@Order`가 **BPP에는 안 먹고(Ch.7) `@Aspect`에는 먹는다** — 단 **클래스 단위만**. 같은 aspect 안에서는 advice **타입 우선순위는 정의되고**(@Around>@Before>@After>@AfterReturning>@AfterThrowing, 5.2.7+) **같은 타입끼리만 미정의**(해결: 한 메서드로 합치거나 aspect 클래스 분리)
- `proceed()` 빠뜨리면 target이 **아예 실행 안 되고 null 반환** (예외 없음)
- Advice 5종 중 `ProceedingJoinPoint`는 **`@Around` 전용**, 나머지는 `JoinPoint`

### 다음 세션 계획
- 스윕: **Q22(2가지 역할 — 면접 단골)**, Q24(PJP 3층 구분), Q25(빈 등록 이유)
- Ch.9 스프링 AOP 개념 → Ch.10 구현 → Ch.11 포인트컷 순서로 진행 예정

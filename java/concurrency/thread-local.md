# ThreadLocal — 쓰레드 전용 저장소 / 싱글톤 빈 동시성 해결 / remove() 필수

> **한 줄 요약**: `ThreadLocal`은 **해당 쓰레드만 접근할 수 있는 전용 저장소** — 같은 `ThreadLocal` 객체에 접근해도 내부적으로 쓰레드별로 값이 분리된다. **싱글톤 빈의 필드에 상태를 저장하면 동시성 문제**(여러 쓰레드가 값을 덮어씀)가 터지는데, 필드를 `ThreadLocal`로 감싸면 해결. 단 **WAS 쓰레드 풀은 쓰레드를 재활용**하므로 **요청이 끝날 때 반드시 `remove()`** — 안 하면 다음 사용자가 이전 사용자의 데이터를 본다(보안 사고).

관련 노트: [스레드 기초](./threads.md) · [가상 스레드](./virtual-threads.md) · [메모리 가시성](./memory-visibility.md) · [락 개념](./locks.md) · [JVM 동시성 도구](./jvm-concurrency-tools.md) · [템플릿 메서드/전략/콜백](../design/template-method-strategy-callback.md) (트랙 10 다음 챕터 — 이 로그 추적기를 패턴으로 분리)

---

## 0. 등장인물 — TraceId / TraceStatus / LogTrace (이 코드가 전제)

이 노트의 예제는 김영한 고급편의 **로그 추적기**다. 목표: 모든 메서드 호출을 아래처럼 남기기.

```
[f8477cfc] OrderController.request()          ← 같은 요청 = 같은 ID
[f8477cfc] |-->OrderService.orderItem()       ← 호출 깊이만큼 level 증가
[f8477cfc] | |-->OrderRepository.save()
[f8477cfc] | |<--OrderRepository.save() time=1004ms
```

```java
// 지금 이 로그가 "어느 요청의, 몇 번째 깊이인가"를 담는 값 객체
public class TraceId {
    private String id;    // 트랜잭션ID (예: "f8477cfc") — HTTP 요청 하나를 구분
    private int level;    // 호출 깊이 (Controller=0, Service=1, Repository=2)

    public TraceId() {
        this.id = UUID.randomUUID().toString().substring(0, 8);
        this.level = 0;
    }

    private TraceId(String id, int level) { this.id = id; this.level = level; }

    public TraceId createNextId()     { return new TraceId(id, level + 1); } // 같은 id, 깊이+1
    public TraceId createPreviousId() { return new TraceId(id, level - 1); }
    public boolean isFirstLevel()     { return level == 0; }  // ★ remove() 타이밍 판단
    // getter 생략
}

// begin()이 반환 — 시작 상태를 들고 있다가 end()에서 소요시간 계산
public class TraceStatus {
    private TraceId traceId;
    private Long startTimeMs;
    private String message;
    // 생성자·getter 생략
}

// 로그 추적기 인터페이스 — 구현체 교체가 이 챕터의 스토리
public interface LogTrace {
    TraceStatus begin(String message);                 // 시작 로그 + 상태 반환
    void end(TraceStatus status);                      // 정상 종료 로그 (+소요시간)
    void exception(TraceStatus status, Exception e);   // 예외 로그
}
// 구현체: FieldLogTrace(필드 → 🔴 동시성 문제) → ThreadLocalLogTrace(✅ 해결)
```

사용하는 쪽(모든 계층에 반복)은 이런 모양:

```java
@Service
@RequiredArgsConstructor
public class OrderServiceV3 {
    private final OrderRepositoryV3 orderRepository;
    private final LogTrace trace;   // 싱글톤 빈 주입 — 모든 계층이 같은 인스턴스 사용!

    public void orderItem(String itemId) {
        TraceStatus status = null;
        try {
            status = trace.begin("OrderService.orderItem()");
            orderRepository.save(itemId);   // 핵심 로직 — TraceId 파라미터 없음!
            trace.end(status);
        } catch (Exception e) {
            trace.exception(status, e);
            throw e;
        }
    }
}
```

### ⭐ 근데 왜 TraceId를 "필드"에 둔 거지? 지역변수면 공유 안 되는데?

지역변수는 쓰레드 격리가 공짜지만 **그 메서드 안에서만 산다.** Controller의 지역변수를 Service가 볼 방법은 파라미터 전달뿐 — **즉 "지역변수 방식" = "파라미터 침투 방식"(V2)이고, 그게 싫어서 시작된 게 이 챕터다.** 파라미터 없이 Controller→Service→Repository가 같은 값을 보려면 공용 장소(주입된 빈의 필드)가 필요했고 → 싱글톤 필드라 동시성 문제 → ThreadLocal.

| | 접근 범위 (계층 간 공유) | 쓰레드 격리 |
|---|---|---|
| 지역변수 (=파라미터 전달) | ❌ 메서드 하나 | ✅ 스택별 분리 |
| 싱글톤 빈 필드 | ✅ 콜스택 전체 | ❌ 전 쓰레드 공유 |
| **ThreadLocal** | ✅ 콜스택 전체 | ✅ 쓰레드별 분리 |

> 한 줄: **ThreadLocal = "필드의 접근 범위 + 지역변수의 격리"를 동시에 갖는 쓰레드 스코프 변수.** Spring Security가 로그인 정보를 파라미터 없이 어디서든 꺼내게 하는 `SecurityContextHolder`도 같은 이유로 ThreadLocal이다.

---

## 1. 언제 쓰나 — 문제의 발단: 싱글톤 빈 + 필드 상태

파라미터로 값(예: 로그 추적용 `TraceId`)을 모든 계층에 넘기면 **메서드 시그니처가 오염**된다(코드 침투). 그래서 값을 빈의 **필드**에 저장하는 방식으로 바꾸면:

```java
public class FieldLogTrace implements LogTrace {
    private TraceId traceIdHolder; // ← 필드에 저장 (문제의 씨앗)
    // begin()에서 traceIdHolder를 읽고/쓰고, end()에서 level을 되돌림
}
```

- 스프링 빈은 기본 **싱글톤** → 인스턴스 1개를 **모든 쓰레드가 공유** → 필드 `traceIdHolder`도 공유.
- 동시 요청 시 thread-A가 처리 중인 값을 thread-B가 **덮어씀** → 트랜잭션ID·level이 뒤섞인 로그:

```
기대: [aaaa] request()          실제: [aaaa] request()
      [bbbb] request()               [aaaa] | | |-->request()  ← B가 A의 상태를 이어받음
```

- 단일 쓰레드 테스트에선 절대 안 잡히고, **동시 요청이 들어오는 운영에서만 터진다** — 동시성 버그의 전형.
- 조건: 여러 쓰레드 + 같은 인스턴스의 필드(또는 static 변수) + **쓰기**가 있을 때. (지역변수는 쓰레드별 스택에 생기므로 안전 → [스레드 기초 §1](./threads.md))

**이럴 때 ThreadLocal**: "쓰레드마다 다른 값을 유지해야 하는데, 파라미터로 넘기긴 싫을 때" — 요청 컨텍스트(인증 정보·트랜잭션·traceId)를 콜스택 전체에서 꺼내 쓰는 용도.

---

## 2. 사용 예시 — 변경은 선언부 + get/set 뿐

```java
public class ThreadLocalLogTrace implements LogTrace {

    private ThreadLocal<TraceId> traceIdHolder = new ThreadLocal<>(); // ① 감싸기

    private void syncTraceId() {
        TraceId traceId = traceIdHolder.get();          // ② 조회
        if (traceId == null) {
            traceIdHolder.set(new TraceId());           // ③ 저장
        } else {
            traceIdHolder.set(traceId.createNextId());
        }
    }

    private void releaseTraceId() {
        TraceId traceId = traceIdHolder.get();
        if (traceId.isFirstLevel()) {
            traceIdHolder.remove();                     // ④ ★ 요청 끝 = 반드시 제거
        } else {
            traceIdHolder.set(traceId.createPreviousId());
        }
    }
}
```

| 메서드 | 역할 |
|---|---|
| `new ThreadLocal<>()` | 선언. `ThreadLocal.withInitial(Supplier)`로 초기값 지정도 가능 |
| `set(value)` | **현재 쓰레드의** 전용 저장소에 저장 |
| `get()` | **현재 쓰레드의** 전용 저장소에서 조회 (없으면 null 또는 초기값) |
| `remove()` | **현재 쓰레드의** 값 제거 — 요청 종료 시 필수 |

동작 원리: 값은 `ThreadLocal` 객체 안이 아니라 **각 `Thread` 객체의 `ThreadLocalMap` 필드**에 저장된다(키가 ThreadLocal 인스턴스). 그래서 같은 코드를 실행해도 `Thread.currentThread()`가 다르면 다른 값을 본다.

---

## 3. 필드 vs ThreadLocal 비교

| | 싱글톤 빈 필드 | ThreadLocal |
|---|---|---|
| 저장 위치 | 인스턴스 1곳 (전 쓰레드 공유) | 쓰레드별 `ThreadLocalMap` |
| 동시 요청 | 서로 덮어씀 → 데이터 오염 | 격리됨 |
| 정리 책임 | 없음 | **`remove()` 직접 호출** |
| 파라미터 전달 | 불필요 | 불필요 (둘 다 코드 침투 해결) |

---

## 4. ⚠️ 함정 — remove() 안 하면 "다른 사용자의 데이터"가 보인다

WAS(톰캣)는 쓰레드 생성 비용 때문에 **쓰레드 풀**을 쓴다: 요청 처리 후 쓰레드를 **제거하지 않고 풀에 반납 → 재활용**. ([스레드 기초 §3](./threads.md)의 ExecutorService와 같은 원리, 톰캣 기본 최대 200개)

```
사용자A 요청 → thread-A 할당 → ThreadLocal에 A 데이터 저장 → 응답 → 풀에 반납 (remove 안 함!)
사용자B 요청 → 풀에서 하필 thread-A 재할당 → get() → A의 데이터가 나옴 🔴
```

- 로그가 꼬이는 수준이 아니라 **다른 사람의 개인정보 노출** = 보안 사고로 직결.
- 메모리 릭도 발생: 풀의 쓰레드는 애플리케이션과 수명을 같이하므로, remove 안 한 값은 GC 대상이 안 됨.
- ⚠️ 추가 함정: 콜백/비동기로 **다른 쓰레드에서 실행되는 코드는 ThreadLocal이 안 보인다** — 값은 쓰레드에 붙어 있으니까. (`@Async`, CompletableFuture, 이벤트 루프 등 → [람다 실행 타이밍](../functional/lambda-execution-timing.md), [Flux/Mono](../reactive/flux-mono-basics.md))
- ⚠️ [가상 스레드](./virtual-threads.md) 환경: 가상 쓰레드는 풀링·재사용이 없어 "남은 데이터" 문제는 없지만, **수백만 개가 각자 ThreadLocal 사본을 들면 힙이 터진다** → ThreadLocal로 비싼 객체 캐싱 금지(JEP 444). Java 25의 `ScopedValue`(JEP 506)가 대체제(불변·스코프 종료 시 자동 정리).

실무에서 ThreadLocal 기반인 것들: Spring Security `SecurityContextHolder` · MDC 로깅 · `TransactionSynchronizationManager` · `RequestContextHolder` — 전부 프레임워크가 요청 끝에 정리를 대신 해주고 있는 것.

---

## 5. 💡 판단 기준

### 첫 질문: "이 값, 쓰레드끼리 공유해야 하는 값인가?"

`volatile`·`Atomic*`·`synchronized`/락은 전부 **"공유는 해야 하는데, 안전하게 공유하자"** 진영의 도구다. ThreadLocal만 혼자 **"애초에 공유하지 말자"(격리)** 진영 — 해결하는 문제의 차원이 다르다.

- **공유하면 안 되는 값** (요청별 TraceId, 로그인 사용자 정보) → **ThreadLocal**. 끝. 락도 volatile도 불필요 — 충돌할 대상 자체를 없앴다.
- **공유해야 하는 값** (재고, 잔액, 카운터) → 두 번째 질문 "정확히 뭐가 문제냐"로:
  - 복합 연산(읽기→계산→쓰기)을 한 덩어리로 → `synchronized`/락 ([JVM 동시성 도구 §0-1](./jvm-concurrency-tools.md))
  - 단일 변수 연산 하나(count++)만 원자적으로 → `Atomic*` (CAS)
  - 원자성이 아니라 가시성 문제(바뀐 값이 안 보임) → `volatile` ([메모리 가시성](./memory-visibility.md))

| | 전제 | 해결하는 문제 |
|---|---|---|
| `synchronized`/락 | 공유함 | 복합 연산의 원자성 (한 번에 하나) |
| `Atomic*` | 공유함 | 단일 연산의 원자성 (락 없이 CAS) |
| `volatile` | 공유함 | 가시성 (바뀐 값이 보이게) |
| **ThreadLocal** | **공유 안 함** | 공유 자체를 제거 (격리) |

> ⚠️ TraceId에 `synchronized`를 걸면? 동작은 하지만 **잘못된 설계** — 요청A와 요청B의 TraceId는 애초에 섞이면 안 되는 별개 값인데, 필드 하나를 두고 순서만 조율하면 "B가 A의 값을 이어받는" 논리 오류는 그대로다. 락은 "같은 값을 다 같이 봐야 할 때", ThreadLocal은 "각자 다른 값을 가져야 할 때".

### 그 다음 판단들

- **"싱글톤 빈에 상태 필드를 둬야 하나?" → 두지 마라.** 상태가 필요하면 ① 파라미터로 넘기거나 ② 쓰레드별 상태면 ThreadLocal. `FieldLogTrace`가 단일 테스트를 통과하고 운영에서 터진 게 그 증거 — **테스트 통과 ≠ 동시성 안전**.
- **ThreadLocal을 쓰기로 했다면 `remove()`는 옵션이 아니라 세트다.** "level이 0으로 돌아오는 지점(요청의 시작점이 끝나는 곳)"처럼 라이프사이클이 닫히는 위치를 정해서 반드시 제거. 서블릿 필터/인터셉터의 `finally`가 단골 위치.
- 값이 **쓰레드에 붙는다**는 걸 항상 의식할 것 — 쓰레드가 바뀌는 순간(비동기·리액티브·가상 쓰레드 대량 생성) ThreadLocal은 배신한다.

---

## 참고

- 김영한, 스프링 핵심 원리 고급편 — Ch.2 쓰레드 로컬
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444) — ThreadLocal 주의사항
- [Oracle Virtual Threads 문서](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)

**학습 날짜**: 2026-08-11 · **계기**: 김영한 고급편 Ch.2 수강 후 Claude 소크라테스 복습 세션 — 동시성 문제의 원인(싱글톤 필드 공유)과 쓰레드 풀 재활용 메커니즘을 본인 말로 재구성함

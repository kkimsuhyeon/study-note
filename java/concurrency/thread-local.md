# ThreadLocal — 쓰레드 전용 저장소 / 싱글톤 빈 동시성 해결 / remove() 필수

> **한 줄 요약**: `ThreadLocal`은 **해당 쓰레드만 접근할 수 있는 전용 저장소** — 같은 `ThreadLocal` 객체에 접근해도 내부적으로 쓰레드별로 값이 분리된다. **싱글톤 빈의 필드에 상태를 저장하면 동시성 문제**(여러 쓰레드가 값을 덮어씀)가 터지는데, 필드를 `ThreadLocal`로 감싸면 해결. 단 **WAS 쓰레드 풀은 쓰레드를 재활용**하므로 **요청이 끝날 때 반드시 `remove()`** — 안 하면 다음 사용자가 이전 사용자의 데이터를 본다(보안 사고).

관련 노트: [스레드 기초](./threads.md) · [가상 스레드](./virtual-threads.md) · [메모리 가시성](./memory-visibility.md) · [락 개념](./locks.md)

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

- **"싱글톤 빈에 상태 필드를 둬야 하나?" → 두지 마라.** 상태가 필요하면 ① 파라미터로 넘기거나 ② 쓰레드별 상태면 ThreadLocal. `FieldLogTrace`가 단일 테스트를 통과하고 운영에서 터진 게 그 증거 — **테스트 통과 ≠ 동시성 안전**.
- **ThreadLocal을 쓰기로 했다면 `remove()`는 옵션이 아니라 세트다.** "level이 0으로 돌아오는 지점(요청의 시작점이 끝나는 곳)"처럼 라이프사이클이 닫히는 위치를 정해서 반드시 제거. 서블릿 필터/인터셉터의 `finally`가 단골 위치.
- 값이 **쓰레드에 붙는다**는 걸 항상 의식할 것 — 쓰레드가 바뀌는 순간(비동기·리액티브·가상 쓰레드 대량 생성) ThreadLocal은 배신한다.

---

## 참고

- 김영한, 스프링 핵심 원리 고급편 — Ch.2 쓰레드 로컬
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444) — ThreadLocal 주의사항
- [Oracle Virtual Threads 문서](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)

**학습 날짜**: 2026-08-11 · **계기**: 김영한 고급편 Ch.2 수강 후 Claude 소크라테스 복습 세션 — 동시성 문제의 원인(싱글톤 필드 공유)과 쓰레드 풀 재활용 메커니즘을 본인 말로 재구성함

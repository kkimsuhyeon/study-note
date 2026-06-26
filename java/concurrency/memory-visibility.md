# 메모리 가시성 — volatile · happens-before (JMM)

> **한 줄 요약**: 동시성 버그에는 두 축이 있다 — **원자성**(여러 단계가 끼어들기 없이 한 번에, 예: `count++`)과 **가시성**(한 스레드가 쓴 값을 다른 스레드가 *보느냐*). race condition은 원자성, 이 노트는 **가시성**. 원인은 CPU 캐시·컴파일러 재배열이고, `volatile`은 **가시성+순서는 보장하지만 원자성은 아니다** — 그래서 `count++` 같은 복합 연산엔 부족하다(→ `Atomic`/`synchronized`).

관련 노트: [스레드 기초](./threads.md) · [JVM 동시성 도구](./jvm-concurrency-tools.md) · [락 개념 종합](./locks.md)

---

## 1. 가시성 문제 — "분명히 바꿨는데 다른 스레드가 못 본다"

```java
class Worker {
    private boolean running = true;          // ← volatile 아님

    void run() {
        while (running) { /* 일함 */ }        // 워커 스레드: 이 루프가 안 끝날 수 있다
    }
    void stop() { running = false; }          // 다른 스레드: 멈추라고 false로
}
```

`stop()`이 `running = false`로 바꿔도 **워커 스레드의 루프가 안 멈출 수 있다.** 버그가 아니라 메커니즘이다:

- **CPU 캐시**: 각 코어가 변수를 자기 캐시에 들고 읽는다 → 다른 코어가 메인 메모리에 쓴 값을 못 본다.
- **컴파일러/JIT 재배열·최적화**: `running`이 루프 안에서 안 바뀐다고 판단해 **레지스터에 올려놓고** 매번 다시 안 읽거나(호이스팅), 순서를 바꾼다.

→ "쓰기"가 다른 스레드에 **언제 보이는지 보장이 없다.** 이게 가시성 문제.

---

## 2. `volatile` — 가시성과 순서 보장 (원자성은 아님)

```java
private volatile boolean running = true;     // ← 이 한 단어로 해결
```

`volatile` 변수는:

1. **읽기·쓰기가 항상 메인 메모리로** 간다(캐시에 묶이지 않음) → 한 스레드의 쓰기를 다른 스레드가 **즉시 본다**.
2. **재배열 금지(메모리 배리어)** → `volatile` 쓰기 이전의 일반 쓰기들도 그 시점에 같이 보인다(happens-before, §3).

> ⚠️ **`volatile`은 "원자성"을 주지 않는다.** `count++`는 *읽기 → +1 → 쓰기* 3단계라, `volatile`이어도 두 스레드가 동시에 들어오면 같은 값을 읽고 각각 +1 → **하나 유실**(lost update). 가시성만 보장하지 "한 번에"는 아니다.

```java
private volatile int count;
count++;          // ❌ 여전히 race — volatile은 못 막는다
```
→ 복합 연산은 `AtomicInteger`(CAS) 또는 `synchronized`. (CAS·Atomic → [JVM 동시성 도구](./jvm-concurrency-tools.md))

---

## 3. happens-before — JMM이 보장하는 "순서·가시성" 규칙

자바 메모리 모델(JMM)은 "A가 B보다 **happens-before**면, A의 결과가 B에게 보인다"를 정의한다. 주요 규칙:

| happens-before 관계 | 의미 |
|---|---|
| **프로그램 순서** | 한 스레드 안에서는 코드 순서대로 보인다 |
| **모니터 락** | `synchronized` **unlock** → 이후 같은 락 **lock** (락이 가시성도 준다) |
| **volatile** | volatile **쓰기** → 이후 그 변수 **읽기** |
| **스레드 시작** | `thread.start()` → 그 스레드의 모든 동작 |
| **스레드 종료** | 스레드의 모든 동작 → 다른 스레드의 `thread.join()` 반환 |
| **final 필드** | 생성자 완료 → 그 객체를 본 다른 스레드 (안전 발행) |

> 즉 **가시성은 "공짜"가 아니라 happens-before 관계가 있을 때만** 보장된다. 그냥 두 스레드가 공유 변수를 만지면 순서 보장이 없다.

---

## 4. 옵션 비교 — 가시성 × 원자성

| 도구 | 가시성 | 원자성 | 용도 |
|---|---|---|---|
| (아무것도 안 함) | ❌ | ❌ | 공유 변경 시 버그 |
| **`volatile`** | ✅ | ❌ (단일 읽기/쓰기만) | **플래그**, 한 번 쓰고 여럿이 읽기, DCL |
| **`Atomic*`** (CAS) | ✅ | ✅ (단일 변수) | 카운터·누적(`count++`), lock-free |
| **`synchronized`/`Lock`** | ✅ | ✅ (블록 전체) | 여러 변수·복합 불변식을 한 묶음으로 |
| **`final`** | ✅ (안전 발행) | — | 불변 객체 — 만든 뒤 안 바뀜 → 동기화 불필요 |

> 락(`synchronized`)은 **원자성과 가시성을 둘 다** 준다 — 그래서 락을 제대로 쓰면 가시성은 따로 신경 안 써도 된다. `volatile`은 "락 없이 가시성만" 싸게 얻는 도구.

---

## 5. ⚠️ 함정

- **`volatile`로 `count++`** — 안 된다(원자성 아님, §2). 카운터는 `AtomicInteger`.
- **`long`/`double`은 비-volatile이면 쓰기가 쪼개질 수 있다(tearing)** — JLS상 64비트는 32비트 두 번으로 나뉠 수 있어, 중간값이 보일 수 있다. `volatile long`이면 원자적 읽기/쓰기 보장. (대부분 현대 JVM은 이미 원자적이지만 표준 보장은 volatile)
- **더블체크 락킹(DCL)엔 `volatile` 필수** — 싱글턴 지연 초기화에서 `instance` 필드가 `volatile`이 아니면, 생성자가 끝나기 전의 **부분 초기화된 객체**가 다른 스레드에 보일 수 있다(재배열). `private static volatile Singleton instance;`
- **불변(`final`)이면 동기화가 아예 필요 없다** — 만든 뒤 안 바뀌니 가시성 문제 자체가 없다(안전 발행). "공유는 하되 안 바꾼다"가 가장 싼 해법.

---

## 6. 💡 판단 기준

| 상황 | 선택 |
|---|---|
| 멈춤 플래그, 한 번 세팅 후 여럿이 읽기 | **`volatile`** |
| 카운터·누적 등 단일 변수 복합 연산 | **`AtomicInteger`** (CAS) |
| 여러 변수를 한 불변식으로 함께 보호 | **`synchronized`/`Lock`** |
| 만든 뒤 안 바뀜 | **`final`** (불변 객체 — 동기화 불필요) |

> 한 줄: **race(원자성)와 가시성은 다른 축이다.** "값이 안 보인다"류 버그(`while(flag)`가 안 멈춤, 부분 초기화)는 락으로도 풀리지만, 단일 플래그면 `volatile`이 가장 싸다. 단 **`volatile`은 가시성·순서만, 원자성은 아니다** — 복합 연산은 `Atomic`/`synchronized`. 락을 쓰면 둘 다 공짜로 따라온다.

---

## 7. 참고
- [Java Language Specification - Memory Model (Ch.17)](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)
- [Java Concurrency in Practice - Visibility](https://jcip.net/)
- 관련 노트: [스레드 기초](./threads.md) · [JVM 동시성 도구](./jvm-concurrency-tools.md) · [락 개념 종합](./locks.md)

---

**학습 날짜**: 2026-06-26
**계기**: 동시성 노트(threads·jvm-concurrency-tools)가 "가시성이 필요하다"만 언급하고 *왜·어떻게*(volatile/happens-before)는 비어 있어, race(원자성)와 다른 축인 **가시성**을 선행지식으로 채움. 핵심은 "volatile = 가시성+순서지 원자성이 아니다".

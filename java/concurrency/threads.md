# 스레드 기초 — 프로세스 / 스레드 / 플랫폼 스레드 / 생명주기

> **한 줄 요약**: **프로세스** = 실행 중인 프로그램(자기 메모리 공간), **스레드** = 그 안에서 실제로 코드를 실행하는 흐름(한 프로세스 안 스레드들은 **힙 메모리를 공유**). 자바의 기본 스레드(**플랫폼 스레드**)는 **OS 스레드와 1:1**이라 비싸다(스택 ~1MB, 수천 개가 한계). 그래서 직접 만들기보다 **스레드 풀(`ExecutorService`)**로 재사용한다. 스레드들이 힙을 공유하므로 → **race condition** → 그래서 락/동기화가 필요해진다.

관련 노트: [JVM 동시성 도구](./jvm-concurrency-tools.md) · [락 개념](./locks.md) · [가상 스레드](./virtual-threads.md) · [메모리 가시성](./memory-visibility.md)

---

## 1. 프로세스 vs 스레드

| | 프로세스(Process) | 스레드(Thread) |
|---|---|---|
| 정의 | 실행 중인 프로그램 1개 | 프로세스 안의 실행 흐름 1개 |
| 메모리 | **독립** (자기 주소공간) | 같은 프로세스 안에서 **힙·코드 공유**, 스택만 각자 |
| 통신 | IPC(파이프·소켓 등) 필요, 무거움 | 그냥 같은 객체 참조 — 가볍지만 **충돌 위험** |
| 비용 | 생성·전환 비쌈 | 상대적으로 쌈(그래도 OS 스레드는 안 쌈, §3) |

- 한 JVM = 보통 한 프로세스. 그 안에서 **여러 스레드**가 동시에 코드를 실행.
- ⭐ **스레드끼리 힙(객체)을 공유**한다는 게 핵심 — 그래서 한 객체를 여러 스레드가 동시에 고치면 깨진다(race condition) → 동기화·락이 필요한 근본 이유. (스택·지역변수는 스레드별이라 안전)

### ⭐ 지역변수는 왜 안전한가 — 스택 프레임
```java
void charge(long amount) {
    long fee = amount / 100;   // 지역변수 → 내 스택 프레임 안 → 충돌 불가 ✅
    this.balance += amount;    // 힙의 객체 필드 → 모든 스레드가 같은 놈 → 깨질 수 있음 ❌
}
```
- 스레드가 메서드를 호출할 때마다 **자기 스택에 프레임이 하나 쌓이고**, 지역변수·파라미터는 그 프레임 안에 산다. 스택은 스레드별이므로 두 스레드가 같은 메서드를 동시에 실행해도 **각자 자기 `fee`**를 만진다.
- 반면 `this.balance`는 힙에 있는 객체의 필드 — 모든 스레드가 **같은 객체**를 바라본다. "공유되는 건 힙의 필드, 안전한 건 스택의 지역변수"로 구분하면 어떤 변수가 위험한지 바로 보인다.

---

## 2. 왜 스레드를 쓰나

1. **병렬 처리** — CPU 코어가 여러 개면 동시에 계산 (CPU 바운드)
2. **대기 중 다른 일** — DB·네트워크 응답을 기다리는 동안(블로킹) 다른 요청 처리 (I/O 바운드)
3. 웹 서버가 대표적 — **요청 1개당 스레드 1개**(thread-per-request)로 동시에 여러 사용자 처리.

> 그래서 동시성은 "빠르게"만이 아니라 "기다리는 시간을 낭비 안 하기"가 큰 목적.

---

## 3. 플랫폼 스레드 = OS 스레드와 1:1 (그래서 비싸다)

자바의 기본 `Thread`는 **플랫폼 스레드** — OS 스레드를 얇게 감싼 것. 1:1로 묶여 있다.

### "얇게 감쌌다 · 1:1"의 정확한 의미 — 두 세계
```
[자바 세계 (JVM)]                [OS 세계 (커널)]
Thread 객체                       OS 스레드
= 힙의 평범한 자바 객체(리모컨)    = 진짜 실행 흐름 (CPU에 실제로 오르는 놈, 스택 1MB도 얘 것)
```
- **"실행 흐름"이라는 진짜 자원은 OS만 만들 수 있다** (CPU 스케줄러가 OS 안에 있으므로). JVM은 직접 못 만들고 OS에 부탁한다.
- `new Thread(task)` = 자바 객체만 생성, **OS 스레드 아직 없음**(생명주기 NEW). `t.start()` = 시스템 콜로 OS 스레드 생성 → 이 순간부터 둘이 짝.
- **얇은 wrapper** = Thread 객체는 일을 안 한다. `interrupt()`/`join()` 등은 전부 "짝인 OS 스레드에게 전달"하는 리모컨 조작.
- **1:1** = Thread 객체 하나 ↔ OS 스레드 하나, 평생 고정 짝. 자바 스레드 1,000개 = OS 스레드 1,000개.
- → 그래서 비용(스택 1MB·컨텍스트 스위칭·생성)이 전부 **OS 스레드의 비용** — 자바가 최적화할 수 없다. [가상 스레드](./virtual-threads.md)가 바로 이 1:1을 깨는 것(M:N).
- **스택 메모리 ~1MB** 고정 할당 → 수만 개 만들면 메모리 폭발.
- **컨텍스트 스위칭** 비용 — OS가 스레드를 바꿔 끼울 때 CPU 레지스터·캐시 교체 → 많아질수록 오버헤드.
- 그래서 **현실적으로 수천 개가 한계.** "요청당 스레드"가 동시 접속 폭증 시 무너지는 이유. (→ 이 한계를 깨려고 나온 게 [가상 스레드](./virtual-threads.md))

> 💡 그래서 **직접 `new Thread()` 남발 대신 스레드 풀**(`ExecutorService`)로 **미리 만들어 재사용**한다(생성 비용·개수 제어). → [JVM 동시성 도구 §0](./jvm-concurrency-tools.md) · 풀의 **내부 동작(core/max/queue·거부 정책)**은 [스레드 풀 내부](./thread-pool.md)

> ⚠️ **`new Thread()`에 자바 차원의 개수 제한은 없다** — 지금도 무한 "시도"는 가능하고, 한계는 OS가 강제한다(`OutOfMemoryError: unable to create new native thread`). `ExecutorService`는 JVM이 강제하는 안전장치가 아니라 **개발자가 스스로 상한·큐를 정하는 규율 도구** — 풀을 써도 딴 데서 `new Thread()` 남발하면 못 막는다. (풀은 Java 5 `java.util.concurrent`부터 JDK 제공, 그 전엔 직접 구현하거나 Doug Lea 라이브러리 사용)

---

## 3-1. ⚠️ 헷갈림 주의 — 스레드 ≠ CPU 코어, `corePoolSize`의 "core"도 코어가 아니다

- **CPU 코어** = 하드웨어. "같은 순간 진짜로" 실행 가능한 계산 유닛 수 (8코어 = 물리적 동시 실행 8개).
- **스레드** = 소프트웨어 실행 흐름. OS 스케줄러가 스레드들을 코어에 **번갈아 태워서**(컨텍스트 스위칭) 실행 → 코어 8개에 스레드 수백 개 가능. "동시에 도는 것처럼" 보이는 이유.
- `new Thread().start()` = 코어를 점유하는 게 아니라 **OS 스레드를 하나 새로 만드는 것**. 어느 코어에서 언제 돌지는 OS 마음.
- **`corePoolSize`의 core는 CPU 코어와 무관** — 스레드 풀이 "평소에 유지하는 기본(core) 스레드 개수"라는 네이밍일 뿐 (`ThreadPoolTaskExecutor` 설정의 그 core).
  - `executor.submit(task)` → 풀의 core 스레드가 실행(재사용) / `new Thread(task).start()` → 풀과 무관하게 매번 새 스레드 생성.

---

## 4. 스레드 만드는 법

```java
// (1) Runnable — "할 일"을 넘김 (스레드 ≠ 할 일)
Runnable task = () -> System.out.println("hi");
new Thread(task).start();          // 직접 생성 (저수준, 잘 안 씀)

// (2) ExecutorService — 풀로 관리 (실무 기본)
ExecutorService pool = Executors.newFixedThreadPool(10);  // 스레드 10개 미리 생성
pool.submit(task);                 // "할 일"만 제출 → 노는 스레드가 실행
pool.shutdown();
```
- **`Runnable`/`Callable` = 할 일, `Thread` = 그 일을 실행하는 일꾼.** 둘은 다르다(혼동 주의).
- `start()` ≠ `run()`: `start()`라야 **새 스레드에서** 실행. `run()`을 직접 부르면 그냥 현재 스레드에서 메서드 호출(동시성 X).
- ⚠️ `Executors.newFixedThreadPool`은 **큐가 무한**이라 작업이 밀리면 OOM까지 간다. 실무에선 팩토리 대신 `ThreadPoolExecutor`로 경계를 명시한다. → [스레드 풀 내부 §4](./thread-pool.md)

---

## 5. 스레드 생명주기 (Thread.State)

```
NEW → RUNNABLE ⇄ (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED
```
| 상태 | 의미 |
|---|---|
| NEW | 생성됐지만 `start()` 전 |
| RUNNABLE | 실행 중 또는 실행 가능(OS 스케줄 대기 포함) |
| BLOCKED | `synchronized` 락을 기다리는 중 |
| WAITING | `join()`/`wait()`/`await()`로 무기한 대기 |
| TIMED_WAITING | `sleep(ms)`/타임아웃 대기 |
| TERMINATED | 실행 종료 |

> 락 노트의 "비관락 = 대기(blocking)"가 여기 **BLOCKED**와 연결. 락을 못 얻으면 BLOCKED로 멈춰 기다린다.

---

## 6. 데몬 스레드 vs 유저 스레드

- **유저 스레드**: 하나라도 살아있으면 JVM이 안 끝남(기본값).
- **데몬 스레드**(`setDaemon(true)`): 백그라운드용. **유저 스레드가 다 끝나면 JVM이 데몬을 안 기다리고 종료.** (예: GC, 모니터링)
- `main`은 유저 스레드. 풀의 작업 스레드 종류는 풀 설정에 따라.
- ⚠️ 데몬은 **작업 도중 그냥 죽는다** — `finally` 실행도 보장 없음 → 파일 쓰기·DB 커밋 같은 중요 작업 금지, "언제 죽어도 되는 일" 전용.

### 데몬으로 만드는 법 (+ 풀이 JVM을 안 죽게 하는 함정)
```java
// (1) 직접 — 반드시 start() 전에 setDaemon (후에 하면 IllegalThreadStateException)
Thread t = new Thread(task);
t.setDaemon(true);
t.start();

// (2) 풀 — ThreadFactory(일꾼 공장)를 갈아끼워서 데몬 일꾼 생산
ExecutorService pool = Executors.newFixedThreadPool(2, r -> {
    Thread th = new Thread(r);
    th.setDaemon(true);
    return th;
});
```
- ⚠️ **기본 ThreadFactory는 유저 스레드를 만든다** → 평범한 풀은 일꾼들이 큐 앞에서 대기하며 살아있는 유저 스레드라, `shutdown()` 안 부르면 **main이 끝나도 JVM이 영영 안 내려간다.** ("main 끝났는데 프로세스가 안 죽어요"의 단골 원인) → 해결: `shutdown()` 호출 또는 데몬 풀.

---

## 7. 스레드 → 동시성 문제로 이어진다

스레드들이 **힙을 공유**(§1)하므로, 같은 데이터를 동시에 건드리면:
- **race condition** — `count++`(읽기→+1→쓰기)가 겹쳐 갱신 분실 (→ [JVM 도구 §7-2 Atomic](./jvm-concurrency-tools.md))
- **가시성** — 한 스레드의 변경이 다른 스레드에 안 보임(`volatile`/메모리 배리어 필요) (→ [메모리 가시성](./memory-visibility.md))
- 해결: **동기화 도구·락** (→ [JVM 동시성 도구](./jvm-concurrency-tools.md), [락 개념](./locks.md))

---

## 8. 💡 정리

> **스레드 = 힙을 공유하며 도는 실행 흐름.** 공유하니까 빠르게 협업하지만 그래서 충돌(race)이 나고 → 락이 필요. 기본 플랫폼 스레드는 OS 스레드 1:1이라 비싸 → 풀로 재사용, 그래도 수천 개가 한계 → 그 벽을 깨는 게 [가상 스레드](./virtual-threads.md). 할 일(`Runnable`)과 일꾼(`Thread`)은 별개, `start()`라야 새 스레드.

---

## 9. 참고
- [Oracle - Java Threads](https://docs.oracle.com/javase/tutorial/essential/concurrency/)
- 관련 노트: [JVM 동시성 도구](./jvm-concurrency-tools.md) · [락 개념](./locks.md) · [가상 스레드](./virtual-threads.md)

---

**학습 날짜**: 2026-06-16
**계기**: 동시성 테스트·가상 스레드를 보다 "스레드 자체"를 정리한 적이 없어서 — 프로세스/스레드, 힙 공유→race, 플랫폼 스레드=OS 1:1(비쌈)→풀, 생명주기까지 기초를 잡음.

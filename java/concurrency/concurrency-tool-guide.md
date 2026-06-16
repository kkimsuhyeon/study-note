# 동시성 도구 — 목적별 정리 & 선택 가이드

> **한 줄 요약**: 동시성 도구는 "세대(최신일수록 우월)"가 아니라 **"각자 다른 임무를 가진 도구함"**이다. 스레드를 *만드는* 건 `ExecutorService`, 결과를 *조합*하는 건 `CompletableFuture`, *기다리는* 건 `CountDownLatch`, *세는* 건 `AtomicInteger`, *정합성*은 락, *블로킹 I/O 대량*은 가상 스레드. **"이 문제에 뭐가 맞나"로 고른다 — 최신 ≠ 최선.**

관련 노트: [스레드 기초](./threads.md) · [JVM 동시성 도구](./jvm-concurrency-tools.md) · [가상 스레드](./virtual-threads.md) · [락 개념](./locks.md) · [동시성 테스트](../test/concurrency-test.md)

---

## 1. 도구별 "한 가지 임무" (헷갈리면 이 표)

| 도구 | 등장 | **한 가지 임무** | 안 하는 것 |
|---|---|---|---|
| `Thread`/`Runnable` | Java 1 | 실행 흐름 1개 직접 만들기(저수준) | 재사용·결과관리 |
| `ExecutorService`(+`Executors`) | Java 5 | **스레드 풀** — 일꾼 만들고 작업 분배·재사용 | 결과 조합 |
| `Future` | Java 5 | submit한 작업의 **미래 결과 핸들**(`get()`으로 대기) | 조합·변환 |
| **`CompletableFuture`** | Java 8 | **비동기 실행+변환+합성+에러처리 파이프라인** | (스레드 직접 생성 X — 풀 위에서 돎) |
| `CountDownLatch` | Java 5 | **N개 끝날/모일 때까지 한 번 대기**(게이트, 일회성) | 결과 보관·세기 |
| `CyclicBarrier` | Java 5 | N개가 **매 단계** 서로 만날 때까지(재사용) | |
| `Semaphore` | Java 5 | **동시 접근 수를 N개로 제한**(자원 풀) | |
| `AtomicInteger` 등 | Java 5 | **락 없이 단일 변수** 세기·플래그·CAS | 여러 변수 묶기 |
| 락(`synchronized`/`ReentrantLock`/`@Version`/비관) | — | **공유 상태 정합성**(여러 변수·check-then-act) | |
| 가상 스레드 | Java 21 | **블로킹 I/O를 싸게 대량 동시** | CPU 바운드엔 무의미 |
| 구조적 동시성(`StructuredTaskScope`) | Java 21+ **preview** | 서브태스크 묶음을 **한 단위로** 실행/대기/에러전파/취소 | (아직 정식 아님) |

> ⭐ **"만들기 / 조합 / 기다리기 / 세기 / 정합성 / 제한"은 서로 다른 임무다.** 한 도구로 다 하려 하지 말고 임무에 맞는 걸 조합한다. (동시성 테스트가 그 예 — `ExecutorService`(만들기)+`CountDownLatch`(기다리기)+`AtomicInteger`(세기)를 같이 씀)

---

## 2. CompletableFuture — "비동기 조합"의 주력

`CountDownLatch`가 "대기"만 하는 데 비해, `CompletableFuture`는 **실행→변환→합성→에러**를 객체 하나에 담아 파이프라인으로 잇는다.

```java
// 실행 + 변환 + 에러처리 체인
CompletableFuture<Order> f = CompletableFuture
    .supplyAsync(() -> callPaymentApi(req))   // 비동기 실행 (풀에서)
    .thenApply(res -> toOrder(res))           // 결과 변환 (논블로킹 체인)
    .exceptionally(ex -> Order.failed());      // 에러 처리

// 두 결과 합치기
CompletableFuture<Combined> c = f1.thenCombine(f2, (a, b) -> merge(a, b));

// 의존 연쇄 (앞 결과로 다음 비동기 호출)
f1.thenCompose(a -> callNext(a));

// scatter-gather: N개 병렬 → 다 끝나면 진행 (CountDownLatch 대체)
CompletableFuture.allOf(f1, f2, f3).join();
```

| 메서드 | 역할 |
|---|---|
| `supplyAsync` / `runAsync` | 비동기 시작(결과 있음/없음) |
| `thenApply` | 결과를 **동기 변환** |
| `thenCompose` | 결과로 **다음 비동기** 호출(연쇄, flatMap) |
| `thenCombine` | **두 Future 결과 합치기** |
| `allOf` / `anyOf` | N개 **다/하나** 완료 대기 |
| `exceptionally` / `handle` | 에러 처리 |

> 💡 **서비스 코드에서 "여러 비동기 호출을 실행·합쳐서 반환"하면 보통 `CompletableFuture`.** scatter-gather(병렬 호출→집계)도 `allOf`가 `CountDownLatch`보다 흔하다. 단순 fire-and-forget이면 Spring `@Async`.

> ⚠️ 기본은 공용 `ForkJoinPool.commonPool`에서 돈다 — 블로킹 작업엔 **전용 Executor를 넘겨라**(`supplyAsync(task, executor)`). 안 그러면 공용 풀이 막힌다.

---

## 3. 선택 가이드 — "내가 뭘 하려나"로

| 하고 싶은 것 | 도구 |
|---|---|
| 스레드 여러 개 굴리고 재사용 | `ExecutorService` |
| 여러 비동기 결과를 **변환·합성**해 반환 | **`CompletableFuture`** |
| N개 작업 **끝날 때까지 한 번** 대기 | `CountDownLatch` / `allOf` |
| 단계마다 스레드들 **모으기(반복)** | `CyclicBarrier` |
| 동시 접근 **수 제한**(커넥션·API 한도) | `Semaphore` |
| 단일 **카운터·플래그** | `AtomicInteger` 등 |
| 공유 데이터 **정합성**(여러 변수·check-then-act) | 락(`@Version`/비관/`ReentrantLock`) |
| 블로킹 I/O **대량 동시**(웹 요청·외부호출) | 가상 스레드 |
| 서브태스크 fan-out/fan-in + 에러·취소 깔끔히 | 구조적 동시성(미래) |

---

## 4. 💡 최신 ≠ 최선 — 도구는 트렌드가 아니라 문제로 고른다

세대(`Latch→CompletableFuture→가상스레드`)는 **사다리가 아니라 도구함**. 위가 우월한 게 아니라 **다른 문제를 푼다.**

- **성숙도**: 최신일수록 버그·레퍼런스 적음. 구조적 동시성은 **아직 preview** → 프로덕션 부적합.
- **적합성**: 가상 스레드는 **I/O 바운드에만** 이득, CPU 계산엔 0. "최신"이어도 안 맞으면 손해.
- **복잡도/팀**: 단순 동기 코드에 `CompletableFuture` 체인을 끼우면 더 어려워짐. 팀이 모르는 최신 = 유지보수 부담.
- **YAGNI**: 안 필요한 정교함은 비용. "오래된" `CountDownLatch`가 동시성 테스트엔 **최적**인 것처럼.

> **기준: "이 문제에 뭐가 맞나" > "뭐가 최신이냐".** 새 도구는 *그게 푸는 특정 문제*가 내게 있을 때만 도입. **학습**은 최신까지 알아두되(언제 쓰는지 감), **프로덕션**은 성숙·적합·팀이 아는 것으로 보수적으로.

---

## 5. 참고
- 관련 노트: [스레드 기초](./threads.md) · [JVM 동시성 도구](./jvm-concurrency-tools.md) · [가상 스레드](./virtual-threads.md) · [락 개념](./locks.md) · [동시성 테스트](../test/concurrency-test.md)

---

**학습 날짜**: 2026-06-16
**계기**: CountDownLatch·CompletableFuture·가상 스레드의 관계를 보다 — "최신이 더 좋은 거냐"는 의문에서. 도구는 세대가 아니라 임무별 도구함이고(만들기/조합/대기/세기/정합성/제한), 비동기 조합의 주력은 CompletableFuture, 선택 기준은 "최신이 아니라 문제에 맞나"로 정리.

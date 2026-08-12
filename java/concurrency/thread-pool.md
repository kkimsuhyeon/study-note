# 스레드 풀 내부 — core / max / queue와 거부 정책, @Async

> **한 줄 요약**: 스레드 풀은 **일꾼(core·max)**과 **대기줄(queue)**로 이뤄지고, 작업이 오면 **core → 큐 → max → 거부** 순으로 처리한다. ⚠️ **큐가 max보다 먼저**라서 **큐를 크게 잡으면 `maxPoolSize`가 영원히 안 쓰인다** — `Executors.newFixedThreadPool`이 딱 그 함정(큐 무한 → OOM). 그래서 실무에선 팩토리 대신 `ThreadPoolExecutor`를 직접 만들고, **응답 지연이 문제면 `queueCapacity=0`(SynchronousQueue)로 큐를 없애 max를 살린다.** 꽉 차면 `CallerRunsPolicy`가 호출 스레드에 일을 떠넘겨 **백프레셔**를 만든다.

관련 노트: [스레드 기초](./threads.md) · [동시성 도구 선택 가이드](./concurrency-tool-guide.md) · [가상 스레드](./virtual-threads.md) · [람다 실행 타이밍](../functional/lambda-execution-timing.md)

> 전제: **스레드가 왜 비싼지(OS 1:1, 스택 ~1MB)와 그래서 풀을 쓴다**는 배경은 [스레드 기초 §3~§4](./threads.md). 여기서는 **그 풀의 내부**를 다룬다.

---

## 1. 언제 쓰나

- 외부 API·DB 호출을 **병렬로** 던지고 결과를 모을 때 (scatter-gather)
- `@Async`로 메일·알림 등을 **던지고 잊을** 때
- 요청마다 `new Thread()`를 만들지 않고 **개수를 통제**하고 싶을 때

> 풀을 "쓰는" 법(`submit`·`CompletableFuture`)은 [동시성 도구 가이드 §2](./concurrency-tool-guide.md). 이 노트는 **풀을 어떻게 설정하나**에 집중한다.

---

## 2. 부품 세 개

풀을 **은행 지점**으로 보면 그대로 대응된다.

| 파라미터 | 은행 비유 | 뜻 |
|---|---|---|
| `corePoolSize` | **상시 창구** 수 | 놀아도 유지하는 기본 스레드 수 |
| `queueCapacity` | **대기 의자** 수 | 처리 못 한 작업을 쌓아두는 공간 |
| `maxPoolSize` | **최대 창구** 수 | 바쁠 때 열 수 있는 상한 |
| `keepAliveTime` | 임시 창구 유지 시간 | core 초과 스레드가 이 시간 놀면 정리됨 |

---

## 3. ⭐ 작업이 오면 이 순서 (핵심)

```
작업 도착
   │
   ├─(1) 스레드 수 < corePoolSize ?   → YES: 새 스레드 생성해 처리
   │
   ├─(2) 큐에 자리 있나 ?             → YES: 큐에 적재하고 끝     ★
   │
   ├─(3) 스레드 수 < maxPoolSize ?    → YES: 새 스레드 생성해 처리
   │
   └─(4) 다 안 되면                   → RejectedExecutionHandler
```

### ⚠️ 함정 1: (2)가 (3)보다 먼저다 — "바쁘면 스레드가 늘겠지"는 틀렸다

```java
core=10, max=20, queue=1000

작업 500개 도착
  → 10개는 스레드가 처리
  → 490개는 큐로 직행
  → 스레드는 계속 10개.  max=20은 한 번도 안 쓰임 ❌
```

**큐가 가득 차야만** `max`까지 늘린다. 큐가 넉넉하면 `maxPoolSize`는 **죽은 설정**이 된다.

**왜 이 순서인가**: 스레드 생성이 비싸기 때문이다.

```
큐에 넣기     = 리스트에 참조 하나 추가        (거의 공짜)
스레드 만들기 = OS 요청 + 스택 ~1MB 할당       (비쌈 → threads.md §3)
```

싼 수단을 먼저 소진하는 설계다 — **"일단 쌓아 버텨보고, 그래도 안 되면 사람을 늘린다."**

---

## 4. 사용 예시

### 4-1. `Executors` 팩토리 — 편하지만 함정

```java
ExecutorService pool = Executors.newFixedThreadPool(10);
```

팩토리가 실제로 만드는 설정은 이렇다.

| 팩토리 | core | max | queue | 숨은 위험 |
|---|---|---|---|---|
| `newFixedThreadPool(n)` | n | n | **`LinkedBlockingQueue` 무한** | 작업이 밀리면 **큐가 무한정 쌓여 OOM** |
| `newSingleThreadExecutor()` | 1 | 1 | **무한** | 위와 동일 |
| `newCachedThreadPool()` | 0 | **`Integer.MAX_VALUE`** | `SynchronousQueue` | **스레드가 무한정 생성**돼 OOM |
| `newScheduledThreadPool(n)` | n | `Integer.MAX_VALUE` | `DelayedWorkQueue`(무한) | 큐 무한 |

> ⚠️ **그래서 `Executors` 팩토리를 쓰지 말라는 권고가 널리 통용된다** (알리바바 Java 코딩 규약, 다수 사내 가이드). 이유가 위 표의 "무한" — **한계가 안 보이는 설정**이라 장애가 나야 발견된다. 대신 `ThreadPoolExecutor`를 직접 만들어 **경계를 명시**한다.

### 4-2. `ThreadPoolExecutor` 직접 생성

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
        10,                                   // corePoolSize
        20,                                   // maximumPoolSize
        60L, TimeUnit.SECONDS,                // keepAliveTime
        new SynchronousQueue<>(),             // workQueue
        new ThreadPoolExecutor.CallerRunsPolicy());  // 거부 정책
```

### 4-3. Spring `ThreadPoolTaskExecutor` (실무 기본)

`ThreadPoolExecutor`를 감싼 Spring 래퍼. 빈으로 등록해 쓴다.

```java
@Bean(name = "externalApiTaskExecutor")
public Executor externalApiTaskExecutor() {
    ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
    taskExecutor.setCorePoolSize(10);
    taskExecutor.setMaxPoolSize(20);
    taskExecutor.setQueueCapacity(0);                    // ← SynchronousQueue (§5)
    taskExecutor.setThreadNamePrefix("external-api-");   // ← 스레드 덤프에서 식별
    taskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    taskExecutor.setWaitForTasksToCompleteOnShutdown(true);
    taskExecutor.setAwaitTerminationSeconds(30);
    taskExecutor.initialize();                           // ← 빠뜨리면 미초기화 상태
    return taskExecutor;
}
```

> ⚠️ **Spring 기본값이 곧 §3 함정이다**: `corePoolSize=1`, `maxPoolSize=Integer.MAX_VALUE`, `queueCapacity=Integer.MAX_VALUE`. 큐가 무한이므로 **스레드 1개로 전부 큐에 쌓는다.** 반드시 명시할 것.

> 💡 `setThreadNamePrefix`는 사치가 아니다. 장애 시 스레드 덤프에서 **어느 풀이 막혔는지**를 이름으로 판별한다.

---

## 5. `queueCapacity = 0` 트릭 — 큐를 없애 max를 살린다

응답을 기다리는 조회 작업은 **쌓아두면 안 된다**(쌓인 만큼 사용자가 더 기다림). 그래서 큐를 0으로 둔다.

```java
// Spring ThreadPoolTaskExecutor 내부
protected BlockingQueue<Runnable> createQueue(int queueCapacity) {
    if (queueCapacity > 0) return new LinkedBlockingQueue<>(queueCapacity);
    else                   return new SynchronousQueue<>();
}
```

**`SynchronousQueue` = 저장 공간이 0인 큐.** 지금 받아갈 스레드가 대기 중일 때만 전달이 성공하고, 아니면 즉시 실패한다. 즉 §3의 **(2)단계가 항상 실패**하므로 곧바로 (3)으로 넘어가 스레드를 늘린다.

```
core=10, max=20, queue=0

작업 15개 도착
  → 10개는 core 스레드
  → 큐 없음 → 즉시 스레드 5개 추가
  → 15개 전부 동시 처리 ✅
```

> ⚠️ 공짜는 아니다. 스레드 수가 급격히 튀어 **컨텍스트 스위칭 비용**이 늘고, `max`를 넘는 순간 바로 거부 정책으로 간다. **버스트가 잦으면** 큐를 조금 두는 편이 나을 수 있다.

---

## 6. 거부 정책 (RejectedExecutionHandler)

스레드도 max, 큐도 만석일 때의 처리다.

| 정책 | 동작 | 결과 |
|---|---|---|
| `AbortPolicy` **(기본값)** | `RejectedExecutionException` 발생 | **요청 실패**. 빠르게 드러남 |
| `CallerRunsPolicy` | **작업을 제출한 스레드가 직접 실행** | 느려지지만 실패 없음 + **백프레셔** |
| `DiscardPolicy` | 조용히 버림 | **무음 유실** — 원인 추적 불가 (지양) |
| `DiscardOldestPolicy` | 큐의 가장 오래된 작업을 버리고 재시도 | 먼저 온 작업이 밀려남 |

### `CallerRunsPolicy`가 백프레셔가 되는 원리

`Caller` = 작업을 던진 스레드(웹이면 보통 톰캣 워커). 얘가 직접 처리한다.

- **요청이 실패하지 않는다** — 최악이라도 병렬화 이전처럼 순차로 돌 뿐
- **자동 브레이크** — 워커가 직접 일하는 동안은 새 요청을 못 받으니 유입이 저절로 억제된다

### ⚠️ 함정 2: `CallerRunsPolicy`는 shutdown 중이면 조용히 버린다

```java
// ThreadPoolExecutor.CallerRunsPolicy 실제 구현
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {   // ← shutdown 상태면 아무것도 안 하고 끝
        r.run();
    }
}
```

종료 절차가 시작된 뒤 들어온 작업은 **예외도 없이 사라진다.** 유실되면 안 되는 작업엔 이 정책을 쓰면 안 된다.

> 💡 또 하나의 부작용: 톰캣 워커가 블로킹 작업을 직접 수행하면 **그 워커가 요청을 못 받는다.** 백프레셔는 의도된 효과지만, 과하면 서버 전체 처리량이 떨어진다.

---

## 7. 종료(shutdown) 옵션

```java
taskExecutor.setWaitForTasksToCompleteOnShutdown(true);  // 남은 작업 기다림
taskExecutor.setAwaitTerminationSeconds(30);             // 단, 최대 30초
```

- 둘은 **세트**다. 앞엣것만 켜면 작업이 안 끝날 때 **무한정 기다릴** 수 있다.
- ⚠️ **데몬 스레드로 만들면 위 설정이 무의미해진다.** 데몬은 유저 스레드가 끝나면 JVM과 함께 소멸하므로 종료 훅을 기다리지 않는다. (→ [스레드 기초 §6](./threads.md))
- Spring 래퍼(`ThreadPoolTaskExecutor`)는 `DisposableBean`이라 컨텍스트 종료 시 자동으로 정리된다. **다른 구현으로 감싸 반환하면 이 훅을 잃는다** — 빈 반환 타입만 `Executor`로 두고 객체는 그대로 넘길 것.

---

## 8. `@Async` — 풀을 쓰는 또 다른 방법

메서드에 붙이면 Spring이 **그 메서드를 다른 스레드에서 실행**하고 호출자는 즉시 리턴한다.

```java
@EnableAsync                    // ← 없으면 @Async가 조용히 무시된다
@Configuration
public class AsyncConfig { }

@Async
public void 메일보내기(String to) { /* 3초 걸림 */ }

// 호출부
메일보내기("a@b.com");
System.out.println("끝!");      // 3초 안 기다리고 바로 출력
```

### 어느 풀에서 도나 — 해석 순서

1. `@Async("beanName")`으로 **직접 지정**한 빈
2. `AsyncConfigurer.getAsyncExecutor()` 구현체
3. 타입이 `TaskExecutor`인 **유일한** 빈, 또는 이름이 정확히 **`taskExecutor`**인 빈
4. 못 찾으면 → **`SimpleAsyncTaskExecutor`**

### ⚠️ 함정 3: 폴백인 `SimpleAsyncTaskExecutor`는 풀이 아니다

이름과 달리 **스레드를 재사용하지 않고 호출마다 새로 만든다.** 풀의 개수 제한이 전혀 없어서, 트래픽이 몰리면 스레드가 무한정 늘어난다. "`@Async` 달았으니 풀에서 돌겠지"라고 믿는 순간 사고가 난다.

> 💡 Spring Boot는 `TaskExecutionAutoConfiguration`이 `applicationTaskExecutor`를 만들어주지만, **`@ConditionalOnMissingBean(Executor.class)`** 조건이라 **내가 `Executor` 빈을 하나라도 등록하면 자동 구성이 꺼진다.** 커스텀 풀을 만들면서 `@Async`도 쓴다면 3번 조건(유일 빈 or 이름 `taskExecutor`)을 만족하는지 확인하거나, `@Async("이름")`으로 못 박는 게 안전하다.

### ⚠️ 함정 4: 같은 클래스 내부 호출이면 안 먹힌다

```java
public void 주문처리() {
    메일보내기();      // ❌ 비동기 아님. 그냥 순차 실행
}

@Async
public void 메일보내기() { ... }
```

Spring이 **프록시(대리인)**로 가로채는 방식이라 **밖에서 들어오는 호출만** 잡을 수 있다. `@Transactional`이 자기 호출에서 안 먹히는 것과 **완전히 같은 메커니즘**. (→ [@Transactional 프록시 함정](../spring/transactional.md))

---

## 9. `@Async` vs `CompletableFuture` + executor

| | `@Async` | `CompletableFuture.supplyAsync(task, executor)` |
|---|---|---|
| 방식 | 메서드에 붙이는 **선언** | 코드 안에서 **직접 지정** |
| 적합 | 던지고 잊기 (메일·알림·로그) | **결과를 받아 변환·합성** |
| 단위 | 메서드 전체 | 메서드 **안의 일부 구간** |
| 실패 처리 | 호출자가 모름(void면 예외 유실) | `exceptionally`/`handle`로 처리 |

> ⚠️ `supplyAsync(task)`처럼 **executor를 생략하면 `ForkJoinPool.commonPool`**에서 돈다. 크기가 `CPU 코어 수 - 1`로 아주 작고 앱 전체가 공유하므로, 블로킹 I/O를 넣으면 무관한 작업까지 굶는다. (→ [동시성 도구 가이드 §2](./concurrency-tool-guide.md)) **전용 풀을 반드시 넘길 것.**

### ⚠️ `join()`은 예외를 한 겹 감싼다 — 벗기지 않으면 예외 처리가 통째로 바뀐다

작업이 예외로 끝나면 `join()`은 원래 예외를 **`CompletionException`으로 감싸서** 던진다(`get()`은 checked `ExecutionException`). 그대로 올리면 상위의 **타입별 예외 핸들러를 못 탄다.**

```java
try {
    return futures.stream().map(CompletableFuture::join).flatMap(List::stream).toList();
} catch (CompletionException e) {
    throw e.getCause() instanceof RuntimeException cause ? cause : e;   // 원래 예외로 복원
}
```

`CompletionException`은 `RuntimeException`의 하위라 컴파일도 되고 동작도 하지만, Spring `@RestControllerAdvice` 기준으로 이만큼 달라진다.

| | 벗김 | 안 벗김 |
|---|---|---|
| 타는 핸들러 | `@ExceptionHandler(BizException.class)` | `@ExceptionHandler(RuntimeException.class)` 폴백 |
| 응답 | 도메인 에러코드 + 다국어 메시지 | 일반 실패 코드 |
| 로그 레벨 | 원인 유무로 WARN/ERROR 구분 | 무조건 ERROR |
| 운영 알림 | 지정한 코드만 발송 | **전부 발송** (의도된 거부까지 알림) |

> 💡 **병렬화는 "예외가 지나가는 길"까지 바꾼다.** 순차 코드를 `CompletableFuture`로 옮길 때 성능만 보고 예외 경로를 안 보면, 배포 후 운영 알림이 갑자기 시끄러워지는 식으로 드러난다. (스트림 lazy 때문에 `try` 블록 안에서 `toList()`로 소비해야 catch가 유효한 것도 함께 → [Stream API 함정](../functional/stream-api.md))

---

## 10. 풀 크기는 어떻게 정하나

Brian Goetz, *Java Concurrency in Practice*의 고전 공식.

```
CPU 바운드 (계산):    스레드 수 ≒ 코어 수 + 1
I/O 바운드 (대기):    스레드 수 ≒ 코어 수 × (1 + 대기시간 / 실제작업시간)
```

외부 API 호출처럼 **대기가 대부분**이면(예: 대기 200ms / 계산 5ms) 계수가 40배까지 뛴다 — 코어 수보다 훨씬 많은 스레드가 정당해진다. 반대로 CPU 계산은 코어 수를 넘겨봐야 컨텍스트 스위칭 손해만 난다.

> ⚠️ 공식은 **출발점**이지 정답이 아니다. 실제로는 부하 테스트로 조정한다. 외부 API에는 **상대 서버의 동시 처리 한계**라는 별개 제약도 있다(내 풀을 키워도 상대가 못 받으면 무의미).

> 💡 **블로킹 I/O를 아주 많이 동시에** 다뤄야 한다면 풀 크기를 키우는 대신 [가상 스레드](./virtual-threads.md)가 답일 수 있다. 단 CPU 바운드엔 이득이 0이다.

---

## 11. 풀은 "성격"으로 나눈다 — 같은 풀에 섞지 말 것

실무에서 풀을 여러 개 두는 이유는 크기 때문이 아니라 **작업 성격이 반대**이기 때문이다.

| | 유실 방지형 | 지연 방지형 |
|---|---|---|
| 예 | 도메인 이벤트, 알림 발송 | 사용자 조회, 화면 렌더링용 외부 호출 |
| 설정 | `max = core`, **큐 크게** | `max > core`, **큐 0** |
| 의도 | "느려도 되니 다 받아라" | "지금 처리, 안 되면 호출자가 직접" |
| 거부 정책 | 기본(예외)로 유입 차단 | `CallerRuns`로 백프레셔 |

> ⚠️ **수백 초짜리 작업(스크래핑·대용량 파일)을 짧은 호출 전용 풀에 넣지 말 것.** 슬롯을 오래 점유해 수백 ms짜리 호출들을 순차로 떨어뜨린다. 성격이 다르면 풀을 분리한다.

---

## 12. ⭐ 반복 호출을 병렬화하기 전에 — 태스크 수를 입력 크기에서 떼어내라

풀 설정을 손대기 전에 **던지는 태스크 수 자체**를 먼저 본다.

외부 API를 `N`건 호출하는 목록 조회가 느릴 때, 반사적으로 "루프를 병렬로 돌리자"가 나온다. 하지만 **API가 목록 요청을 지원한다면** 태스크를 `N`개 만들 이유가 없다.

```
① 순차           : 태스크 0개, 호출 N회 순차          → 가장 느림
② 항목별 병렬     : 태스크 N개, 호출 N회              → 풀·상대 서버에 N배 부하
③ 묶어서 일괄     : 태스크 k개, 호출 k회 (k = 분류 수) → 호출 자체가 사라짐
```

②는 **동시 사용자 수만큼 곱해진다.** N=40인 화면을 10명이 열면 태스크 400개 → `max=20`을 훌쩍 넘겨 `CallerRuns`로 떨어지고, 같은 풀을 쓰는 **무관한 기능까지 굶는다.**

③은 사용자가 몇 명이든 요청당 태스크가 `k`개로 고정된다. **입력 크기와 태스크 수가 분리**되므로 풀 설정을 안 건드려도 규모에 안전하다.

> 💡 **판단 기준: "이 병렬화는 태스크 수가 입력 크기에 비례하는가?"** 비례한다면 병렬화 이전에 **호출을 묶을 수 없는지** 먼저 본다. 병렬화는 호출 수를 줄이지 못하고 **동시에 터뜨릴 뿐**이라, 상대 서버 부하와 풀 고갈은 그대로 남는다. 묶기가 불가능할 때(항목마다 다른 엔드포인트 등)만 항목별 병렬로 간다. — DB의 N+1을 `IN` 절로 접는 것과 완전히 같은 발상이다. (→ [배치의 세 층위](../spring/batch-three-meanings.md))

---

## 13. 💡 정리 — 어느 쪽을 고르나

| 상황 | 선택 |
|---|---|
| 풀을 새로 만든다 | `Executors` 팩토리 ❌ → **`ThreadPoolExecutor` 직접 생성**(경계 명시) |
| 사용자가 기다리는 조회 | `queue=0`, `max > core`, `CallerRuns` |
| 유실되면 안 되는 작업 | 큐 크게, `max=core`, 기본 거부 정책(예외로 드러내기) |
| 던지고 잊기 | `@Async` (단, 어느 풀인지 확인 — 함정 3) |
| 결과를 받아 합쳐야 함 | `CompletableFuture` + **전용 executor** |
| 반복 호출이 느림 | 병렬화보다 **일괄 호출을 먼저** 검토 (§12) |

> **가장 오래 사는 한 줄**: 풀 튜닝은 `core`·`max`·`queue` 숫자 놀음이 아니라 **"이 작업은 유실이 문제인가, 지연이 문제인가"**를 정하는 일이다. 그 답이 큐 크기와 거부 정책을 자동으로 결정한다.

---

## 14. 참고

- [ThreadPoolExecutor (Java SE Javadoc)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html) — 처리 순서·거부 정책 원문
- [SynchronousQueue (Javadoc)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/SynchronousQueue.html)
- [Spring ThreadPoolTaskExecutor (Javadoc)](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/concurrent/ThreadPoolTaskExecutor.html)
- [Spring - Task Execution and Scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)
- Brian Goetz, *Java Concurrency in Practice* §8.2 — 풀 크기 산정
- 관련 노트: [스레드 기초](./threads.md) · [동시성 도구 선택 가이드](./concurrency-tool-guide.md) · [가상 스레드](./virtual-threads.md) · [@Transactional 프록시 함정](../spring/transactional.md)

---

**학습 날짜**: 2026-08-11
**계기**: 목록 조회에서 외부 API를 항목마다 호출하던 코드를 **분류별 일괄 호출 + 병렬**로 바꾸다가, 주입할 `Executor` 빈이 두 개라 `@Qualifier`가 왜 필요한지 → 그 풀들의 `core 10/max 20/queue 0/CallerRuns` 설정이 무슨 뜻인지로 번져 정리. 특히 **큐가 max보다 먼저 채워진다**는 순서와, **병렬화보다 호출을 묶는 게 먼저**라는 판단 기준을 얻음.

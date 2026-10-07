# 동시성 테스트 작성법 — 락이 진짜 일하는지 검증하기

> **한 줄 요약**: 락(낙관/비관)이 실제로 동작하는지는 **여러 스레드를 동시에 띄워** 검증한다. 핵심 규칙은 **테스트에 `@Transactional`을 절대 붙이지 않는 것**(테스트 트랜잭션은 테스트 스레드에만 묶여 워커 스레드는 미commit 셋업을 못 본다), 그리고 `CountDownLatch`로 스레드를 **동시에 출발**시키는 것. 검증은 락 종류마다 다르다 — **비관 = 최종 합계, 낙관 = 1성공/N충돌(타이밍 의존)** — 그리고 **"락을 빼면 이 테스트가 실패하는가"**를 한 번 확인해야 테스트가 락을 증명한다.

관련 노트: [락 개념(이론)](../concurrency/locks.md) · [JVM 동시성 도구](../concurrency/jvm-concurrency-tools.md) · [읽기-수정-쓰기](../jpa/read-modify-write.md)

---

## 1. 큰 그림 — 왜 평범한 테스트로는 안 되나

락은 "여러 트랜잭션이 동시에 같은 행을 건드릴 때"만 의미가 있다. 단일 스레드 테스트는 그 상황을 못 만든다. → **스레드 풀로 N개를 동시에 실행**해야 한다.

```java
@SpringBootTest               // 진짜 빈·DB·트랜잭션이 필요 (Mockito ❌)
@DisplayName("User 동시성 테스트")
class UserConcurrencyTest {
    @Autowired UserCommandService userCommandService;
    @Autowired UserQueryService  userQueryService;
    // ...
}
```

> ⚠️ **테스트 클래스/메서드에 `@Transactional`을 붙이지 마라.** 테스트 트랜잭션은 ThreadLocal로 **테스트 스레드에만** 묶인다 — 워커 스레드는 거기 참여하지 않고 서비스의 `@Transactional`로 **각자 새 트랜잭션을 열어 commit**한다. 그래서 붙이면: ① 테스트 트랜잭션 안에서 만든 셋업 데이터가 **commit 전이라 워커가 못 보거나**, 그 트랜잭션이 잡은 락에 워커가 막힌다 ② 검증 조회가 테스트 트랜잭션의 1차 캐시·스냅샷을 읽어 워커 결과가 안 보일 수 있다 ③ 워커가 commit한 데이터는 **롤백되지도 않아** 격리 효과도 없다. ("한 트랜잭션을 공유한다"가 아니라 "서로 다른 트랜잭션인데 테스트 쪽만 commit을 안 한다"가 문제다. 트랜잭션이 스레드에 붙는 원리 → [ThreadLocal §4](../concurrency/thread-local.md))

---

## 2. 등장하는 동시성 도구 (java.util.concurrent)

| 도구 | 한 줄 정의 | 이 테스트에서 역할 |
|---|---|---|
| `ExecutorService` | 스레드 풀. 작업(`submit`)을 받아 풀의 스레드가 실행 | N개 작업을 병렬 실행 |
| `Executors.newFixedThreadPool(n)` | 고정 크기 n짜리 풀 생성 | 동시에 n개가 진짜로 겹치게 |
| `CountDownLatch(n)` | n부터 세는 카운터. `countDown()`로 1 감소, `await()`는 **0이 될 때까지 블록** | 출발 신호·완료 대기 |
| `AtomicInteger` | 여러 스레드가 안전하게 증가시키는 정수 | 성공/실패 횟수 집계(race 없이) |
| `executor.submit(Runnable)` | 풀에 작업 1개 제출(비동기 시작) | 각 스레드가 서비스 호출 |

> 📌 **구조는 하나가 아니다 — "겹침을 얼마나 강제하나"의 스펙트럼.** 공통 뼈대는 **"풀로 N개 동시 실행 → 전원 완료 대기 → 결과 검증"**이고, 동시 출발 강제 수준만 고른다:
> | 방식 | 겹침 강제 |
> |---|---|
> | `submit` N개 + `awaitTermination` | 약 (먼저 뜬 게 먼저 끝남) |
> | `CompletableFuture.allOf(...).join()` | 약~중 (완료 대기 간결) |
> | start latch 1개 + done 대기 | 중~강 |
> | **latch 3개**(ready/start/done) | 강 (겹침 최대) |
> → 겹침이 약하면 **락을 빼도 충돌이 안 나서 통과**한다(거짓 양성). 락이 있을 때 결과가 같다는 건 증명이 아니다 — 테스트의 가치는 "락이 없으면 깨지는가"에 있고, 그건 겹침 강도에 달렸다. 그래서 **비관 합계 검증도 최소 start latch**, **낙관 충돌**처럼 *찰나에 겹쳐야* 재현되는 건 3-latch가 안정적. `AtomicInteger` 사용법·원리는 → [JVM 동시성 도구 §7-2](../concurrency/jvm-concurrency-tools.md), `CountDownLatch` → [§6](../concurrency/jvm-concurrency-tools.md).

### `CountDownLatch` 3개를 쓰는 이유 — "동시에 출발"시키려고
스레드를 `submit`만 하면 **먼저 시작된 게 먼저 끝나버려** 겹침이 약하다(충돌 재현 X). 그래서 **출발선에 세워두고 동시에 출발**시킨다.

| latch | 초기값 | 역할 |
|---|---|---|
| `ready` | n | 각 스레드가 "준비됨" 알림(`countDown`). 메인은 `ready.await()`로 **전원 준비될 때까지** 대기 |
| `start` | 1 | 출발 신호. 스레드들은 `start.await()`로 **묶여서 대기**, 메인이 `start.countDown()` 하면 **동시에 출발** |
| `done` | n | 각 스레드 종료 알림. 메인은 `done.await()`로 **전원 끝날 때까지** 대기 후 검증 |

```java
CountDownLatch ready = new CountDownLatch(n);  // 전원 준비 대기
CountDownLatch start = new CountDownLatch(1);  // 출발 총성 (1발)
CountDownLatch done  = new CountDownLatch(n);  // 전원 완료 대기

for (int i = 0; i < n; i++) {
    executor.submit(() -> {
        ready.countDown();          // "나 준비됨"
        try {
            start.await();          // 총성 기다리며 대기 (여기서 전원이 멈춤)
            서비스호출();             // ← start 풀리면 전원 동시에 이 줄로
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();   // ★ 없으면 컴파일 에러 (아래 ⚠️)
        } finally {
            done.countDown();       // "나 끝남"
        }
    });
}
ready.await();                      // 전원 준비될 때까지
start.countDown();                  // 탕! → 모두 동시 출발
assertThat(done.await(10, TimeUnit.SECONDS)).isTrue();   // ★ 반환값 검사 (아래 ⚠️)
executor.shutdown();
```
> ⚠️ **람다 안의 `start.await()`는 `catch (InterruptedException)`가 필수.** 값을 반환하지 않는 블록 람다는 `Runnable`로만 해석되고(`Callable`은 값 반환 필요), `Runnable.run()`은 checked 예외를 못 던진다 → 테스트 메서드에 `throws InterruptedException`을 붙여도 람다 안은 컴파일 에러. (JLS §15.27.3)
>
> ⚠️ **`await(timeout)`의 반환값(boolean)을 버리지 마라.** 시간 안에 0이 안 되면 `false`를 돌려줄 뿐 예외가 아니다 → 무시하면 **아직 일하는 스레드가 있는 상태로 단언**이 돌아 엉뚱한 숫자로 실패(또는 우연히 통과)한다. `isTrue()`로 "전원 완료"부터 확인.
> 💡 비유: `ready`=선수 전원 출발선 섰나 확인, `start`=총성(1발), `done`=전원 결승선 통과 대기. 총성을 1발(`start`=1)로 두고 전원이 그걸 기다려야 "동시 출발"이 된다.

### 시간순 흐름 (한눈에)
```
[일꾼 스레드 10개]                       [main 스레드]
각자 시작하며 ready.countDown() ───▶  ready.await() 에서 멈춤
  (10→…→0)                              (0 되면 통과)
start.await() 에서 ★전원 멈춤★    ◀───  start.countDown() (1→0) "땅!"
  │ start 0 → 전원 동시 깨어남
서비스 호출 → 성공:success++ / 충돌:fail++
finally: done.countDown() ───────▶  done.await(10s) 에서 멈춤
  (10→…→0)                              (0 되면 통과)
                                     executor.shutdown()
```
- `countDown()` = 카운터 -1(안 멈춤) / `await()` = 0 될 때까지 **그 스레드를 멈춤**(=기다림).
- 멈추는 줄은 `ready.await()`·`done.await()`(main이 멈춤)와 `start.await()`(일꾼들이 멈춤). 나머지(`countDown`/`shutdown`)는 안 멈춤.

> ⚠️ **`done.countDown()`은 반드시 `finally`에.** 작업이 성공하든 예외(낙관락 충돌)가 나든 *무조건* 깎여야 한다. `try` 안에만 두면 예외난 스레드가 done을 안 깎아 → main의 `done.await()`가 **0이 안 돼 타임아웃까지 멈춤**. `finally`라 안전.

> ⚠️ **`ready`가 "공정한 출발"을 보장.** ready 없이 바로 `start.countDown()` 하면 아직 `submit`되어 시작도 안 한 일꾼이 있을 수 있다 → ready로 **전원이 `start.await()` 직전(출발선)에 도착**한 걸 확인하고 총을 쏜다. (latch가 3개인 이유)

---

## 3. 검증은 락 종류마다 다르다

### (1) 비관적 락 — 최종 합계로 (락이 있으면 결정적)
N스레드가 동시에 100씩 충전 → 락이 직렬화하면 **정확히 N×100**. 락이 없으면 lost update로 **그보다 적게** 나온다 — 단 *겹쳤을 때만*. 그래서 (3)의 대조 실험이 필요하다.
```java
// addBalance가 findByIdForUpdate(SELECT ... FOR UPDATE) 경로일 때
User actual = userQueryService.getUser(user.getId());
assertThat(actual.getBalance()).isEqualByComparingTo(BigDecimal.valueOf(1000)); // 100*10
```

### (2) 낙관적 락 — 1성공 / 나머지 충돌 (타이밍 의존)
2스레드가 동시에 같은 행 수정 → 둘 다 version=N 읽고 `UPDATE ... WHERE version=N` → **하나만 성공**, 나머지는 0건 매칭 → 예외.
```java
AtomicInteger success = new AtomicInteger(), fail = new AtomicInteger();
// 스레드 안:
try { userCommandService.changeName(id, name); success.incrementAndGet(); }
catch (ObjectOptimisticLockingFailureException e) { fail.incrementAndGet(); }  // ★ 기대한 충돌만 센다
// 검증:
assertThat(success.get()).isEqualTo(1);
assertThat(fail.get()).isEqualTo(1);
```
- 예외 타입: JPA `OptimisticLockException`을 Spring이 **`ObjectOptimisticLockingFailureException`**(DataAccessException 계열)으로 래핑.
- ⚠️ **`catch (Exception e)`로 뭉뚱그리지 마라.** NPE·제약 위반 같은 무관한 버그까지 "충돌"로 집계돼 테스트가 초록으로 남는다. 기대한 예외 타입만 잡고, 나머지는 따로 모아(`List<Throwable>`) 비어 있는지 단언한다.
- 같은 골격의 **"선착순 1명" 변형**(좌석·쿠폰): N명이 동시에 같은 대상을 요청 → `success == 1`, `fail == N-1`(업무 예외만 카운트). 짝 테스트로 **서로 다른 대상이면 전원 성공**하는지도 본다(락이 과하게 걸리지 않는지).

> ⚠️ **낙관 테스트는 타이밍 의존이라 드물게 흔들릴 수 있다.** 두 트랜잭션이 *안 겹치면*(하나가 commit 후 다른 게 read) 충돌이 안 난다 → `success == 2`. `start` latch로 겹침을 강제해 안정화하지만 100% 결정적은 아니다. 흔들림이 문제면 ① 정확한 개수 대신 **불변식**(`success + fail == n`, `success >= 1`, 최종 version = 시작 + success)으로 단언하거나 ② `TransactionTemplate` 두 개로 "T1 읽기 → T2 읽기 → T1 commit → T2 commit" 순서를 **직접 조종**해 결정적으로 만든다.

### (3) 대조 실험 — "락을 빼면 이 테스트가 실패하는가"
락 테스트가 초록이라는 건 "락이 있을 때 맞다"일 뿐, **"락 덕분에 맞다"는 아니다.** 겹침이 약하거나 스레드 수가 적으면 락이 없어도 통과한다. 처음 한 번은 락을 빼고(`findByIdForUpdate` → `findById`, `@Version` 제거 등) 돌려 **빨개지는지** 확인한다 — 안 빨개지면 그 테스트는 락을 증명하지 못하는 것이니 스레드 수·겹침(latch)을 늘린다.

---

## 4. 셋업 주의 — 테스트 밖에서 commit돼야 한다

스레드들이 같은 행을 보려면, 그 행이 **스레드 시작 전에 이미 DB에 commit**돼 있어야 한다. 테스트가 `@Transactional`이 아니므로, 서비스로 만든 데이터(`userCommandService.create(...)`)는 그 서비스의 트랜잭션이 **즉시 commit**한다 → OK.
```java
private User createUser() {
    return userCommandService.create(CreateUserCommand.builder()
            .email("test" + System.nanoTime() + "@test.com")  // 유니크 보장
            .password("password123").name("origin").build());
}
```
> ⚠️ 동시성 테스트는 롤백이 없어 **데이터가 쌓인다**(테스트 격리가 트랜잭션 롤백에 의존 X). 이메일을 `nanoTime`으로 유니크하게 만들어 충돌을 피하고, 다른 테스트를 오염시키지 않게 `@AfterEach`에서 정리한다(→ [JUnit 라이프사이클 §4](./junit-lifecycle.md)).

> ⚠️ **DB는 실제 DB(Testcontainers)로.** `FOR UPDATE`·갭 락·데드락 감지는 H2와 운영 DB의 동작이 다르다 — 락 테스트를 H2로 통과시키면 운영 DB에서의 보장은 아니다. (사용법 → [JPA Repository 테스트 §7](./jpa-repository-test.md))

---

## 5. 💡 판단 기준

> **락 테스트의 본질 = "동시에 겹치게 만들고 → 결과로 락 동작을 역추적".** ① `@Transactional` 금지(워커는 다른 트랜잭션 — 셋업이 commit돼 있어야 함) ② `CountDownLatch`로 동시 출발(겹침 강제) + `await` 반환값 확인 ③ 검증은 **비관=합계, 낙관=1성공/N충돌(타이밍 의존)**, 기대한 예외만 카운트 ④ **락을 뺐을 때 빨개지는지 한 번 확인**(대조 실험). 단위가 아니라 통합(`@SpringBootTest`) 테스트다 — Mockito로는 락을 검증할 수 없다(가짜 repo엔 DB 락이 없으니).

---

## 6. 참고
- [Spring Framework - Test-managed Transactions](https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/tx.html) — 트랜잭션 상태가 ThreadLocal로 현재 스레드에 묶임
- [CountDownLatch (Javadoc)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CountDownLatch.html) — `await(timeout)`은 boolean 반환
- [JLS §15.27.3 - Type of a Lambda Expression](https://docs.oracle.com/javase/specs/jls/se21/html/jls-15.html#jls-15.27.3)
- 관련 노트: [락 개념(이론)](../concurrency/locks.md) · [JVM 동시성 도구](../concurrency/jvm-concurrency-tools.md) · [읽기-수정-쓰기](../jpa/read-modify-write.md) · [@Lock 실무 패턴](../jpa/lock-practical.md)

---

**학습 날짜**: 2026-06-15
**계기**: User 잔액(비관)·이름(낙관) 동시성 테스트를 짜며 — `ExecutorService`/`CountDownLatch` 3개 패턴, `@Transactional` 금지 이유, 비관(합계 결정적)/낙관(1성공·타이밍 의존) 검증 차이를 정리.

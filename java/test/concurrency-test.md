# 동시성 테스트 작성법 — 락이 진짜 일하는지 검증하기

> **한 줄 요약**: 락(낙관/비관)이 실제로 동작하는지는 **여러 스레드를 동시에 띄워** 검증한다. 핵심 규칙은 **테스트에 `@Transactional`을 절대 붙이지 않는 것**(붙이면 한 트랜잭션에 묶여 동시성이 사라짐), 그리고 `CountDownLatch`로 스레드를 **동시에 출발**시키는 것. 검증은 락 종류마다 다르다 — **비관 = 최종 합계(결정적), 낙관 = 1성공/N충돌(타이밍 의존)**.

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

> ⚠️ **테스트 클래스/메서드에 `@Transactional`을 붙이지 마라.** 붙이면 ① 모든 스레드가 **한 트랜잭션**을 공유해 동시성이 사라지고 ② 끝나면 **롤백**돼서 결과를 못 본다. 동시성 테스트는 각 스레드가 **자기 트랜잭션을 commit**해야 한다(서비스의 `@Transactional`이 스레드별로 따로 열림).

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
> → **비관 합계 검증**은 결국 직렬화돼 결과가 같으니 느슨해도 OK(1개·`allOf`로 충분). **낙관 충돌**처럼 *찰나에 겹쳐야* 재현되는 건 빡센 버전(3-latch)이 안정적. `AtomicInteger` 사용법·원리는 → [JVM 동시성 도구 §7-2](../concurrency/jvm-concurrency-tools.md), `CountDownLatch` → [§6](../concurrency/jvm-concurrency-tools.md).

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
        } finally {
            done.countDown();       // "나 끝남"
        }
    });
}
ready.await();                      // 전원 준비될 때까지
start.countDown();                  // 탕! → 모두 동시 출발
done.await(10, TimeUnit.SECONDS);   // 전원 끝날 때까지(타임아웃 안전장치)
executor.shutdown();
```
> 💡 비유: `ready`=선수 전원 출발선 섰나 확인, `start`=총성(1발), `done`=전원 결승선 통과 대기. 총성을 1발(`start`=1)로 두고 전원이 그걸 기다려야 "동시 출발"이 된다.

---

## 3. 검증은 락 종류마다 다르다

### (1) 비관적 락 — 최종 합계로 (결정적, 안 흔들림)
N스레드가 동시에 100씩 충전 → 락이 직렬화하면 **정확히 N×100**. 락이 없으면 lost update로 **그보다 적게** 나온다.
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
catch (Exception e) { fail.incrementAndGet(); }  // ObjectOptimisticLockingFailureException
// 검증:
assertThat(success.get()).isEqualTo(1);
assertThat(fail.get()).isEqualTo(1);
```
- 예외 타입: JPA `OptimisticLockException`을 Spring이 **`ObjectOptimisticLockingFailureException`**(DataAccessException 계열)으로 래핑.

> ⚠️ **낙관 테스트는 타이밍 의존이라 드물게 흔들릴 수 있다.** 두 트랜잭션이 *안 겹치면*(하나가 commit 후 다른 게 read) 충돌이 안 난다. `start` latch로 겹침을 강제해 안정화하지만 100% 결정적은 아니다. 반면 **비관 테스트(최종 합계)는 결정적** — 락이 직렬화를 보장하므로 안 흔들린다.

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
> ⚠️ 동시성 테스트는 롤백이 없어 **데이터가 쌓인다**(테스트 격리가 트랜잭션 롤백에 의존 X). 이메일을 `nanoTime`으로 유니크하게 만들어 충돌을 피한다.

---

## 5. 💡 판단 기준

> **락 테스트의 본질 = "동시에 겹치게 만들고 → 결과로 락 동작을 역추적".** ① `@Transactional` 금지(동시성·commit 위해) ② `CountDownLatch`로 동시 출발(겹침 강제) ③ 검증은 **비관=합계(결정적), 낙관=1성공/N충돌(타이밍 의존)**. 단위가 아니라 통합(`@SpringBootTest`) 테스트다 — Mockito로는 락을 검증할 수 없다(가짜 repo엔 DB 락이 없으니).

---

## 6. 참고
- 관련 노트: [락 개념(이론)](../concurrency/locks.md) · [JVM 동시성 도구](../concurrency/jvm-concurrency-tools.md) · [읽기-수정-쓰기](../jpa/read-modify-write.md)

---

**학습 날짜**: 2026-06-15
**계기**: User 잔액(비관)·이름(낙관) 동시성 테스트를 짜며 — `ExecutorService`/`CountDownLatch` 3개 패턴, `@Transactional` 금지 이유, 비관(합계 결정적)/낙관(1성공·타이밍 의존) 검증 차이를 정리.

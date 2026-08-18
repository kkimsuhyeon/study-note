# Spring 이벤트 — @EventListener vs @TransactionalEventListener

> **한 줄 요약**: 한 JVM 안의 발행-구독. `@EventListener`는 **발행 즉시·같은 스레드**에서 돌고, `@TransactionalEventListener`는 실행을 **트랜잭션 phase에 예약**한다. 전자의 실행 시점은 발행 코드의 위치가 정하고, 후자는 트랜잭션 경계가 정한다.

## 언제 쓰나

- 같은 앱 안에서 모듈 간 결합을 끊고 싶을 때 — "주문 생성"이 알림·이력·통계에 직접 의존하지 않게
- 부가 작업(알림, 이력 적재, 후속 처리)을 핵심 업무 로직에서 분리할 때
- Kafka 같은 외부 브로커를 붙이기 전, 한 프로세스 안에서 이벤트 기반으로 설계할 때

## 기본 동작

```java
// 발행 — 컨테이너가 중개
applicationEventPublisher.publishEvent(new OrderCreatedEvent(orderId));

// 구독 — 파라미터 타입으로 매칭 (상속 관계도 매칭됨)
@Component
public class OrderListener {
    @EventListener
    public void handle(OrderCreatedEvent event) { ... }
}
```

- 임의의 객체를 이벤트로 발행 가능 (Spring 4.2+, `ApplicationEvent` 상속 불필요)
- **기본은 동기 + 같은 스레드 + 같은 콜스택.** `publishEvent()`가 리턴하기 전에 리스너가 다 실행된다. 사실상 "인터페이스 없는 메서드 호출"
- 같은 타입을 받는 리스너가 여러 개면 전부 호출된다. 순서는 `@Order`로 제어
- 리스너에서 던진 예외는 **발행자에게 그대로 전파**된다 (동기라서)
- 비동기로 만들려면 리스너에 `@Async` — 어느 풀에서 도는지는 [스레드 풀 내부](../concurrency/thread-pool.md) 참조

## @EventListener — 발행 즉시 실행

트랜잭션이 있든 없든 `publishEvent()` 자리에서 즉시 실행된다. 발행자가 트랜잭션 안이라면:

- 리스너의 `@Transactional`(기본 REQUIRED)은 발행자의 트랜잭션에 **합류**한다 → **DB 작업은 발행자와 같이 커밋/롤백된다** (전파 메커니즘은 [@Transactional](./transactional.md))
- 반대로 **DB 밖 부수효과(메일, HTTP, `@Async` 제출)는 롤백이 못 되돌린다** — "아직 확정 안 된 데이터를 근거로 이미 행동함"이 이 리스너의 본질적 위험

```java
@Transactional
public void createOrder() {
    orderRepository.save(order);
    publisher.publishEvent(new OrderCreatedEvent(...)); // 리스너가 지금 실행됨
    someValidation();                                   // 여기서 예외 → 롤백
}   // 리스너가 보낸 알림은 이미 나갔다. 주문은 존재하지 않는다
```

## @TransactionalEventListener — 트랜잭션 phase에 예약

`publishEvent()` 시점에는 트랜잭션에 **등록만** 해두고, phase가 왔을 때 실행한다.

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT) // AFTER_COMMIT이 기본값
public void handle(OrderCreatedEvent event) { ... }
```

| phase | 발동 시점 | 용도 |
|---|---|---|
| `BEFORE_COMMIT` | 커밋 직전 (아직 트랜잭션 안) | 같은 트랜잭션에 끼워 넣을 DB 작업 |
| `AFTER_COMMIT` (기본) | 커밋 성공 직후 | **가장 흔함** — 확정된 사실에 반응 (알림, 외부 호출) |
| `AFTER_ROLLBACK` | 롤백 직후 | 실패 보상 |
| `AFTER_COMPLETION` | 커밋/롤백 무관하게 종료 후 | 정리 작업 |

같은 이벤트 타입을 phase가 다른 리스너 여러 개가 받을 수 있다 — "커밋 전 이력 적재 + 커밋 후 실행"을 한 번의 발행으로 처리하는 패턴이 이 조합이다 ([이벤트 유실 방지 — outbox](./event-outbox-pattern.md)).

### 타임라인

```
@Transactional 메서드 시작
  ├─ save(order)
  ├─ publishEvent(event)        ← 등록만. 아무 리스너도 안 돎
  ├─ ... 나머지 로직 ...
  ├─ [BEFORE_COMMIT 리스너]
  ├─ COMMIT
  └─ [AFTER_COMMIT 리스너]       ← 롤백이었으면 실행 자체가 안 됨
```

롤백 시: `BEFORE_COMMIT` 리스너가 한 DB 작업은 같이 롤백되고, `AFTER_COMMIT` 리스너는 **아예 실행되지 않는다.** "되돌린다"가 아니라 "확정 전엔 시작하지 않는다" 방식의 일관성.

## phase별 안전한 작업 — 서로 반대다

| | DB 쓰기 | 외부 부수효과 (HTTP·메일·비동기 제출) |
|---|---|---|
| `BEFORE_COMMIT` | ✅ 같은 트랜잭션 합류, 실패 시 같이 롤백 | ❌ **커밋이 아직 실패할 수 있다** |
| `AFTER_COMMIT` | ❌ 증발 함정 (아래 ⚠️) | ✅ 성공 확정 후 |

`BEFORE_COMMIT`은 "트랜잭션과 운명을 같이할 일"용, `AFTER_COMMIT`은 "확정된 사실에 반응할 일"용. 커밋 시도는 리스너 실행 후에도 실패할 수 있다 — JPA는 커밋 시점에 flush하므로 그때 unique 제약 위반이 터질 수 있고(flush 시점은 [영속성 컨텍스트](../jpa/persistence-context.md)), `BEFORE_COMMIT` 리스너 자신의 예외도 커밋 경로 위라 전체를 롤백시킨다.

## ⚠️ 함정

1. **트랜잭션 없이 발행하면 `@TransactionalEventListener`는 조용히 침묵한다.** 예외도 로그도 없이 이벤트가 버려진다(정확히는 DEBUG 로그뿐). 트랜잭션 없이도 실행하려면 `fallbackExecution = true`. "트랜잭션 있는 발행"과 "없는 발행"이 섞이는 코드베이스면 이벤트 타입 자체를 둘로 나누고 리스너를 분리하는 게 명시적이다.
2. **AFTER_COMMIT에서 DB 쓰기 증발.** 커밋 "후"지만 여전히 이미 끝난 트랜잭션의 리소스 위에서 돈다. 여기서 `@Transactional(REQUIRED)` 메서드를 부르면 이미 커밋된 트랜잭션에 참여한 걸로 처리돼 **새 변경이 커밋되지 않고 조용히 사라진다.** 이 phase에서 DB에 써야 하면 `REQUIRES_NEW` 필수.
3. **"내 메서드의 끝" ≠ "트랜잭션의 끝".** REQUIRED 전파로 바깥 트랜잭션에 합류하면, 안쪽 메서드 마지막 줄의 `publishEvent()`도 물리 트랜잭션 한가운데다. `@EventListener`였다면 이후 바깥 로직이 실패해도 이미 실행됐다. `BEFORE_COMMIT`은 발행 위치와 무관하게 항상 전체 트랜잭션의 커밋 직전에 돈다 — **실행 시점을 코드 배치(관례)가 아니라 트랜잭션 경계(계약)에 묶는 것**이 이 어노테이션의 존재 이유.
4. **프록시 계열 공통 함정 그대로.** `@Async` 조합 시 `@EnableAsync` 없으면 조용히 동기로 돌고, 자기호출은 프록시를 안 탄다. ([@Transactional](./transactional.md)·[Spring Cache](./spring-cache.md)와 같은 메커니즘)
5. **`@EventListener`의 예외는 발행자를 무너뜨린다.** 같은 콜스택이라 리스너 예외가 발행자의 트랜잭션을 롤백시킬 수 있다. "부가 기능의 실패가 핵심 업무를 죽여도 되는가"를 리스너마다 물어야 한다.

## 💡 판단 기준

- **리스너가 하는 일의 성격으로 phase를 고른다**: 트랜잭션과 운명을 같이할 DB 작업 → `BEFORE_COMMIT`(또는 합류를 의도한 `@EventListener`), 확정된 사실에 반응하는 외부 행동 → `AFTER_COMMIT`. 한 리스너에 둘이 섞여 있으면 리스너를 쪼갤 신호다.
- **"이벤트가 롤백되나?"는 잘못된 질문.** 올바른 질문은 "리스너가 한 일이 발행자의 트랜잭션에 묶이나"다. DB 작업은 묶을 수 있지만, 밖으로 나간 행동은 어떤 phase도 되돌려주지 않는다 — 그래서 외부 행동은 확정 후(`AFTER_COMMIT`)로 미루는 것.
- 실무에서 겪은 케이스: 발행을 메서드 마지막 줄에 두면 `@EventListener`도 `BEFORE_COMMIT`과 같아 보인다 → 전파로 합류하는 순간 갈라진다는 걸 확인하고 나서야 차이가 잡혔다. "마지막 줄" 같은 배치 규칙에 기대지 말고 phase로 못 박을 것.

## 참고

- [Spring Framework — Transaction-bound Events](https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html)
- [Spring Framework — Standard and Custom Events](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-events)
- 관련 노트: [@Transactional](./transactional.md) · [트랜잭션 전파·롤백 예제](./transaction-rollback-example.md) · [스레드 풀 내부(@Async)](../concurrency/thread-pool.md) · [이벤트 유실 방지 — outbox](./event-outbox-pattern.md)

---
*학습: 2026-08-18 — 회사 프로젝트의 도메인 이벤트 신뢰성 개선 MR을 읽으며. 한 이벤트를 BEFORE_COMMIT(이력 적재)과 AFTER_COMMIT(비동기 실행) 리스너 둘이 나눠 받는 구조에서 phase의 의미를 파고듦.*

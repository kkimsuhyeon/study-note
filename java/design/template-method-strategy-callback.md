# 템플릿 메서드 / 전략 / 템플릿 콜백 — 변하는 코드와 변하지 않는 코드의 분리

> **한 줄 요약**: 로그 추적 같은 부가 기능(변하지 않는 코드)과 비즈니스 로직(변하는 코드)을 분리하는 세 가지 패턴의 발전사 — **상속(템플릿 메서드) → 위임(전략) → 위임+람다(템플릿 콜백)**. 스프링의 `JdbcTemplate`·`RestTemplate`·`TransactionTemplate`이 전부 마지막 형태다. ⚠️ 단, 아무리 발전시켜도 **원본 코드에 `template.execute(...)` 한 줄은 남는다** — 이 한계가 다음 챕터(프록시)를 부른다.

관련 노트: [ThreadLocal](../concurrency/thread-local.md) (트랙 10 이전 챕터 — 이 노트의 `LogTrace`가 거기서 만든 `ThreadLocalLogTrace`) · [람다 실행 타이밍](../functional/lambda-execution-timing.md) (콜백=코드 값, 실행 타이밍은 받는 쪽이 정함) · [커스텀 어노테이션](../annotation/custom-annotation.md) (프록시/AOP 처리기 — 이 트랙이 향하는 곳)

---

## 0. 문제 상황 — 로그 추적기를 넣었더니 비즈니스 로직이 파묻혔다

`TraceId`/`TraceStatus`/`LogTrace`의 정의와 동작은 [ThreadLocal 노트 §0](../concurrency/thread-local.md)에 있다. 여기서는 "그 로그 추적기를 모든 계층에 적용했더니 생긴 문제"에서 출발한다.

```java
// V0: 도입 전 — 순수한 비즈니스 로직
public class OrderServiceV0 {
    private final OrderRepositoryV0 orderRepository;

    public void orderItem(String itemId) {
        orderRepository.save(itemId);   // 이게 전부였다
    }
}
```

```java
// V3: 도입 후 — 핵심 로직 한 줄에 로그 코드 아홉 줄
public class OrderServiceV3 {
    private final OrderRepositoryV3 orderRepository;
    private final LogTrace trace;

    public void orderItem(String itemId) {
        TraceStatus status = null;
        try {
            status = trace.begin("OrderService.orderItem()"); // 변하지 않는 부분
            orderRepository.save(itemId);                      // ⭐ 변하는 부분 (핵심)
            trace.end(status);                                 // 변하지 않는 부분
        } catch (Exception e) {
            trace.exception(status, e);                        // 변하지 않는 부분
            throw e;                                           // ⚠️ 예외 되던지기 필수
        }
    }
}
```

- **변하지 않는 부분**: try-catch, `begin()`, `end()`, `exception()` — Controller/Service/Repository 어디든 똑같다.
- **변하는 부분**: `orderRepository.save(itemId)` — 계층마다 다르다.
- 이 구조가 모든 클래스에 복붙된다. **둘을 분리하는 게 이 챕터의 전부.**

---

## 1. 템플릿 메서드 패턴 — 상속으로 분리

> GOF: "알고리즘의 골격을 정의하고 일부 단계를 하위 클래스로 연기한다. 하위 클래스는 알고리즘의 구조를 변경하지 않고 특정 단계만 재정의할 수 있다."

### 1-1. 추상 클래스: 변하지 않는 부분을 부모에

```java
public abstract class AbstractTemplate<T> {

    private final LogTrace trace;

    public AbstractTemplate(LogTrace trace) {
        this.trace = trace;
    }

    // 변하지 않는 부분 — 이 메서드가 "템플릿"
    public T execute(String message) {
        TraceStatus status = null;
        try {
            status = trace.begin(message);
            T result = call();          // ⭐ 변하는 부분 호출 (자식이 구현)
            trace.end(status);
            return result;
        } catch (Exception e) {
            trace.exception(status, e);
            throw e;
        }
    }

    // 변하는 부분 — 자식은 이것만 구현
    protected abstract T call();
}
```

### 1-2. 방법 1: 구체 자식 클래스

```java
@Slf4j
public class SubClassLogic1 extends AbstractTemplate<Void> {
    public SubClassLogic1(LogTrace trace) { super(trace); }

    @Override
    protected Void call() {
        log.info("비즈니스 로직1 실행");
        return null;
    }
}

// 사용
AbstractTemplate<Void> template1 = new SubClassLogic1(trace);
template1.execute("비즈니스 로직1");
// 흐름: begin() → SubClassLogic1.call() → end()
```

⚠️ 로직이 하나 늘 때마다 SubClassLogic3, 4, ... **클래스 파일이 계속 늘어난다** → 익명 내부 클래스로.

### 1-3. 방법 2: 익명 내부 클래스 (V4 실전 적용)

```java
@Service
@RequiredArgsConstructor
public class OrderServiceV4 {
    private final OrderRepositoryV4 orderRepository;
    private final LogTrace trace;

    public void orderItem(String itemId) {
        // 객체 생성과 동시에 그 자리에서 상속+구현. 클래스 이름은 OrderServiceV4$1 형태
        AbstractTemplate<Void> template = new AbstractTemplate<>(trace) {
            @Override
            protected Void call() {
                orderRepository.save(itemId);
                return null;   // ⚠️ 제네릭엔 기본 타입 void 불가 → 래퍼 Void + return null
            }
        };
        template.execute("OrderService.orderItem()");
    }
}
```

```java
// Controller는 반환값이 있으므로 <String>
@GetMapping("/v4/request")
public String request(String itemId) {
    AbstractTemplate<String> template = new AbstractTemplate<>(trace) {
        @Override
        protected String call() {
            orderService.orderItem(itemId);
            return "ok";
        }
    };
    return template.execute("OrderController.request()");
}
```

### 1-4. 🔴 상속의 문제 — 이 패턴을 버리는 이유

1. **부모의 기능을 하나도 안 쓰는데 `extends`로 강결합.** 자식이 부모의 메서드를 호출하는 곳이 없는데도 부모가 바뀌면 자식 전부 영향.
2. 자바는 **단일 상속** — 이미 다른 클래스를 상속 중이면 이 패턴 자체가 불가.
3. 컴파일 시점에 부모가 코드에 박제됨 (의존관계 문제).

→ "상속 말고 **위임**으로 풀자" = 전략 패턴.

---

## 2. 전략 패턴 — 인터페이스 위임으로 분리

> GOF: "알고리즘 제품군을 정의하고 각각을 캡슐화하여 상호 교환 가능하게 만들자. 전략을 사용하면 알고리즘을 사용하는 클라이언트와 독립적으로 알고리즘을 변경할 수 있다."

- 변하지 않는 부분 = **Context** (템플릿 역할)
- 변하는 부분 = **Strategy 인터페이스** (알고리즘 역할)
- 템플릿 메서드와 결정적 차이: Context는 `Strategy` **인터페이스에만 의존** — 구현체가 바뀌어도 Context 코드는 그대로 (상속의 강결합 문제 해소)

```java
public interface Strategy {
    void call();
}
```

```java
@Slf4j
public class StrategyLogic1 implements Strategy {
    @Override
    public void call() { log.info("비즈니스 로직1 실행"); }
}

@Slf4j
public class StrategyLogic2 implements Strategy {
    @Override
    public void call() { log.info("비즈니스 로직2 실행"); }
}
```

### 2-1. ContextV1 — 필드 보관 = "선 조립, 후 실행"

```java
@Slf4j
public class ContextV1 {

    private Strategy strategy;          // 전략을 필드에 보관

    public ContextV1(Strategy strategy) {
        this.strategy = strategy;       // ⭐ 생성(조립) 시점에 전략 고정
    }

    public void execute() {
        long startTime = System.currentTimeMillis();
        strategy.call();                // 위임 (상속 아님!)
        long endTime = System.currentTimeMillis();
        log.info("resultTime={}", endTime - startTime);
    }
}
```

```java
// 사용 — 세 가지 방법 (구체 클래스 → 익명 → 람다)
ContextV1 c1 = new ContextV1(new StrategyLogic1());
c1.execute();

ContextV1 c2 = new ContextV1(new Strategy() {
    @Override public void call() { log.info("비즈니스 로직 실행"); }
});
c2.execute();

ContextV1 c3 = new ContextV1(() -> log.info("비즈니스 로직 실행"));  // 람다
c3.execute();

// ⚠️ 다른 전략을 쓰려면? → Context를 새로 만들어야 한다
```

- **스프링 DI가 정확히 이 방식**: 애플리케이션 로딩 시점에 의존관계(전략) 조립 → 이후에는 실행만 반복.
- 전략을 바꾸고 싶으면 setter로 갈아끼우기보다 **Context를 새로 생성**하는 쪽 — 필드 교체는 동시성 문제를 부른다 (싱글톤 필드 상태의 위험은 [ThreadLocal 노트 §1](../concurrency/thread-local.md) 그대로).

### 2-2. ContextV2 — 파라미터 전달 = 실행 시점마다 전략 선택

```java
@Slf4j
public class ContextV2 {

    // 필드가 없다!

    public void execute(Strategy strategy) {   // ⭐ 호출할 때마다 전략을 받음
        long startTime = System.currentTimeMillis();
        strategy.call();
        long endTime = System.currentTimeMillis();
        log.info("resultTime={}", endTime - startTime);
    }
}
```

```java
ContextV2 context = new ContextV2();               // 하나만 만들고
context.execute(() -> log.info("비즈니스 로직1")); // 매번 다른 전략을
context.execute(() -> log.info("비즈니스 로직2")); // 넘기면 된다
```

### 2-3. V1 vs V2 비교

| | ContextV1 (필드 보관) | ContextV2 (파라미터 전달) |
|---|---|---|
| 전략 결정 시점 | 조립(생성) 시 | 실행(`execute`) 시 |
| 전략 변경 | Context 새로 생성 | 호출마다 다른 전략 전달 |
| 객체 수 | 전략 조합마다 Context 1개 | **Context 1개 재사용** |
| 실제 예 | 스프링 DI | JdbcTemplate 등 xxxTemplate |

---

## 3. 템플릿 콜백 패턴 — ContextV2 + 함수형 인터페이스

> ⚠️ GOF 패턴이 아니라 **스프링이 부르는 이름**이다. 구조는 ContextV2와 완전히 동일 — Context→Template, Strategy→Callback으로 이름만 바뀐 것.

**콜백(Callback)** = 다른 코드의 인수로 넘기는 실행 가능한 코드 조각. 넘겨받은 쪽(template)이 필요한 시점에 실행한다(뒤에서(back) 호출(call)한다는 뜻 — 실행 타이밍의 주도권이 받는 쪽에 있다는 게 핵심, [람다 실행 타이밍](../functional/lambda-execution-timing.md)과 같은 이야기).

### 3-1. 함수형 인터페이스 — 람다가 가능한 조건

```java
/**
 * 추상 메서드가 "딱 1개" = 함수형 인터페이스 → 람다로 구현 가능.
 * 컴파일러가 "이 람다 = call()의 구현"이라고 추론할 수 있는 유일한 경우이기 때문.
 * @FunctionalInterface: 실수로 추상 메서드를 2개 만들면 컴파일 에러 (안전망)
 */
@FunctionalInterface
public interface Callback {
    void call();
}
```

```java
@Slf4j
public class TimeLogTemplate {
    public void execute(Callback callback) {
        long startTime = System.currentTimeMillis();
        callback.call();                // 콜백 실행 (위임)
        long endTime = System.currentTimeMillis();
        log.info("resultTime={}", endTime - startTime);
    }
}
```

```java
// 익명 내부 클래스 → 람다로 축약되는 과정
template.execute(new Callback() {
    @Override public void call() { log.info("비즈니스 로직1 실행"); }
});

template.execute(() -> log.info("비즈니스 로직1 실행"));   // 같은 코드
```

### 3-2. 실전 적용 (V5) — 반환 타입까지 제네릭으로

```java
@FunctionalInterface
public interface TraceCallback<T> {
    T call();
}
```

```java
public class TraceTemplate {

    private final LogTrace trace;

    public TraceTemplate(LogTrace trace) { this.trace = trace; }

    public <T> T execute(String message, TraceCallback<T> callback) {
        TraceStatus status = null;
        try {
            status = trace.begin(message);
            T result = callback.call();     // ⭐ 콜백 = 변하는 비즈니스 로직
            trace.end(status);
            return result;
        } catch (Exception e) {
            trace.exception(status, e);
            throw e;
        }
    }
}
```

```java
@RestController
public class OrderControllerV5 {
    private final OrderServiceV5 orderService;
    private final TraceTemplate template;      // ⭐ template을 필드에 (한 번만 생성)

    public OrderControllerV5(OrderServiceV5 orderService, LogTrace trace) {
        this.orderService = orderService;
        this.template = new TraceTemplate(trace);
    }

    @GetMapping("/v5/request")
    public String request(String itemId) {
        return template.execute("OrderController.request()", () -> {
            orderService.orderItem(itemId);
            return "ok";
        });
    }
}

@Service
public class OrderServiceV5 {
    private final OrderRepositoryV5 orderRepository;
    private final TraceTemplate template;

    public OrderServiceV5(OrderRepositoryV5 orderRepository, LogTrace trace) {
        this.orderRepository = orderRepository;
        this.template = new TraceTemplate(trace);
    }

    public void orderItem(String itemId) {
        template.execute("OrderService.orderItem()", () -> {
            orderRepository.save(itemId);
            return null;
        });
    }
}
```

```
[aaaaaaaa] OrderController.request()
[aaaaaaaa] |-->OrderService.orderItem()
[aaaaaaaa] |    |-->OrderRepository.save()
[aaaaaaaa] |    |<--OrderRepository.save() time=1001ms
[aaaaaaaa] |<--OrderService.orderItem() time=1003ms
[aaaaaaaa] OrderController.request() time=1004ms
```

스프링에서 이 패턴인 것들: **JdbcTemplate**(SQL+RowMapper 콜백) · **RestTemplate**(`execute()`가 RequestCallback·ResponseExtractor를 받음) · **TransactionTemplate**(트랜잭션 안에서 실행할 로직을 콜백으로) · RedisTemplate — 이름에 `xxxTemplate`이 붙어 있으면 거의 다 이 구조.

```java
// JdbcTemplate — 커넥션 획득/해제·예외 변환(변하지 않는 부분)은 템플릿이,
// "한 행을 객체로 어떻게 바꾸는가"(변하는 부분)만 RowMapper 콜백으로 전달
List<Member> members = jdbcTemplate.query(
        "select id, name from member where age > ?",
        (rs, rowNum) -> new Member(rs.getString("id"), rs.getString("name")),  // 콜백
        20);

// TransactionTemplate — begin/commit/rollback은 템플릿이, 비즈니스 로직만 콜백으로
transactionTemplate.executeWithoutResult(status -> memberRepository.save(member));
```

---

## 4. ⚠️ 함정 / 헷갈렸던 것

### "V2가 재사용된다"의 진짜 의미 (❓ 세션에서 막혔던 지점)

"하나의 template으로 orderService도 orderRepository도 실행"이라는 예시를 보면 이상하다 — **Service와 Repository는 애초에 사용처(계층)가 다른데?** 맞는 지적. 재사용의 실제 의미는 계층을 넘나드는 게 아니라, **한 클래스 안의 여러 메서드가 template 하나를 공유**한다는 것:

```java
@Service
public class OrderService {
    private final TimeLogTemplate template = new TimeLogTemplate(); // 한 번만

    public void orderItem(String id)  { template.execute(() -> repo.save(id)); }
    public void cancelItem(String id) { template.execute(() -> repo.delete(id)); }
    public void updateItem(String id) { template.execute(() -> repo.update(id)); }
    // V1이었다면 메서드마다 new ContextV1(다른전략) 을 만들어야 했다
}
```

### 그 외 함정들

- **캡처되는 지역변수는 effectively final이어야 한다**: V4·V5 코드가 익명 클래스/람다 안에서 바깥의 `itemId`를 쓸 수 있는 건, 자바가 그 값을 **복사(캡처)**해 넣기 때문 — 그래서 캡처된 변수는 재할당 불가(사실상 final)여야 한다. `itemId = itemId.trim();` 같은 재할당을 한 줄이라도 넣으면 람다 쪽에서 컴파일 에러. 원본이 아닌 복사본이라는 것은 [람다 실행 타이밍](../functional/lambda-execution-timing.md)의 "람다=코드 값"과 같은 맥락.
- **제네릭 + void**: 반환값 없는 로직은 `Void`(래퍼) + `return null` — 제네릭은 기본 타입(`void`, `int`)을 못 받는다.
- **함수형 인터페이스라는 용어**: "추상 메서드 1개면 람다 가능"까지 알아도 이름을 모르면 검색·면접에서 막힌다. `@FunctionalInterface`는 선택이지만 붙이는 게 안전망. (Effective Java Item 44가 이 주제 — 도달하면 여기 링크 추가)
- **예외 되던지기(`throw e`)**: 템플릿의 catch에서 로그만 남기고 삼키면 상위 계층이 실패를 모른다. `exception()` 후 반드시 다시 던질 것.
- **템플릿 콜백 "패턴"이라는 말**: GOF에 없는 스프링 용어. 면접에서 "전략 패턴의 변형(Strategy를 실행 시점에 파라미터로)"이라고 말할 수 있어야 한다.

### 🔴 이 챕터의 최종 한계 — 다음 챕터가 존재하는 이유

```java
public void orderItem(String itemId) {
    template.execute("OrderService.orderItem()", () -> {   // ← 이 감싸는 코드 자체가
        orderRepository.save(itemId);                      //    원본에 "추가"된 것
        return null;
    });
}
```

아무리 줄여도 **원본 클래스를 열어서 수정해야 한다**. 클래스 수백 개면 수백 번 — "더 편하게 수정하느냐"의 차이일 뿐 본질은 그대로.

**해결 방향(4장 예고): 프록시.** 원본과 **같은 인터페이스를 구현한 대리인**을 중간에 세우고, 대리인이 부가 기능을 수행한 뒤 진짜 객체에 위임한다. 클라이언트는 인터페이스만 보므로 대리인인지 진짜인지 모른다 → 원본도 클라이언트도 수정 0.

```java
public class OrderServiceProxy implements OrderService {
    private final OrderService target;   // 진짜 객체
    private final LogTrace trace;

    @Override
    public void orderItem(String itemId) {
        TraceStatus status = null;
        try {
            status = trace.begin("OrderService.orderItem()");
            target.orderItem(itemId);    // ⭐ 진짜 객체에 위임
            trace.end(status);
        } catch (Exception e) {
            trace.exception(status, e);
            throw e;
        }
    }
}
```

⚠️ 단, 프록시도 **클래스마다 일일이 만들어야 한다**는 같은 종류의 문제를 갖는다 → 5장 동적 프록시(런타임 자동 생성)로 이어진다. 이 "챕터의 한계가 다음 챕터를 부르는" 서사가 트랙 10의 뼈대.

---

## 5. 💡 판단 기준

- **"변하는 것과 변하지 않는 것을 분리하라"는 말이 추상적이면, try-catch로 감싸는 부가 기능(로그·시간측정·트랜잭션)이 비즈니스 한 줄을 파묻는 순간을 떠올려라** — 그게 분리 신호다. 그리고 2026년에 이걸 직접 구현할 일은 거의 없다: 스프링이 이미 `xxxTemplate`으로 만들어뒀고, 그마저도 원본 수정이 필요해서 결국 AOP로 간다. **이 챕터의 가치는 "왜 AOP가 그렇게 생겼는지"의 족보를 아는 것.**
- **상속이냐 위임이냐 고민되면 위임.** 템플릿 메서드→전략으로 넘어온 이유가 전부다: 부모 기능을 안 쓰는데도 강결합, 단일 상속 제약. 실제로 스프링 생태계에서 상속 기반 확장(추상 클래스 상속)보다 인터페이스+조합이 표준이 된 것도 같은 이유.
- **전략을 언제 정하느냐가 V1/V2 갈림길**: 앱 구동 시 한 번 정해지고 안 바뀌면 V1(=DI로 주입), 호출마다 로직이 달라지면 V2(=콜백 파라미터). "메서드마다 감싸는 내용이 다르다" = V2 신호.

---

## 참고

- 김영한, 스프링 핵심 원리 고급편 — Ch.3 템플릿 메서드 패턴과 콜백 패턴
- GOF, Design Patterns — Template Method / Strategy
- [Spring TransactionTemplate docs](https://docs.spring.io/spring-framework/reference/data-access/transaction/programmatic.html) — 템플릿 콜백 실전 예
- Effective Java Item 44 (함수형 인터페이스) — 도달 시 상호 링크

**학습 날짜**: 2026-08-12 · **계기**: 김영한 고급편 Ch.3 수강 후 Claude 소크라테스 복습 세션 — V1/V2 차이와 "재사용"의 의미를 질문으로 파고들었고, 프록시의 동작 원리를 스스로 추론해냄 ("로그 쪽이 service를 받아서 감싸고, 외부에서는 service 쓰듯 쓰면 되지 않나" → 정확히 프록시)

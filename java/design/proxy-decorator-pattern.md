# 프록시 패턴 / 데코레이터 패턴 — 원본 코드를 열지 않고 기능을 끼워넣기

> **한 줄 요약**: 원본과 **같은 인터페이스를 구현한 대리인(프록시)** 을 중간에 세우면, 클라이언트는 진짜인지 대리인인지 모른 채 호출하고 부가 기능이 끼어든다 — 원본 수정 0, 클라이언트 수정 0. 구조가 똑같은 두 패턴을 GOF는 **의도(intent)** 로 가른다: **접근 제어면 프록시, 기능 추가면 데코레이터.** ⚠️ 단, 프록시 클래스를 대상마다 하나씩 만들어야 하고(→5장 동적 프록시), 컴포넌트 스캔으로 등록된 빈에는 이 방식으로 못 끼운다(→7장 빈 후처리기).

관련 노트: [템플릿 메서드/전략/콜백](./template-method-strategy-callback.md) (트랙 10 직전 챕터 — 거기서 "원본에 한 줄은 남는다"로 끝난 자리가 이 노트의 출발점) · [동적 프록시](./dynamic-proxy.md) (다음 챕터 — 이 노트가 남긴 "프록시 클래스 폭발"과 "final 제약 3종"을 그대로 받아 간다) · [ThreadLocal](../concurrency/thread-local.md) (`LogTrace`의 출처) · [JPA 프록시와 지연 로딩](../jpa/proxy.md) (같은 개념이 ORM에서 쓰인 사례 — `getReference()`) · [@Transactional](../spring/transactional.md) · [Spring Cache](../spring/spring-cache.md) · [메서드 보안](../security/method-security.md) · [커스텀 어노테이션](../annotation/custom-annotation.md) — **뒤 네 개가 공유하는 "프록시라서 자기호출은 안 먹는다" 함정의 메커니즘이 바로 이 노트**

---

## 0. 출발점 — 템플릿 콜백으로도 안 되는 것

[직전 챕터](./template-method-strategy-callback.md)에서 템플릿 콜백까지 갔지만 한계가 남았다.

```java
public void orderItem(String itemId) {
    template.execute("OrderService.orderItem()", () -> {   // ← 이 감싸는 코드 자체가
        orderRepository.save(itemId);                      //    원본에 "추가"된 것
        return null;
    });
}
```

아무리 줄여도 **원본 클래스를 열어서 수정**해야 한다. 그래서 요구사항이 이렇게 바뀐다.

- 원본 코드를 **전혀 수정하지 않고** 로그 추적기를 적용할 것
- v1(인터페이스 있음) · v2(구체 클래스만) · v3(컴포넌트 스캔) **모든 케이스**에 적용 가능할 것

---

## 1. 프록시란 — 직접 호출과 간접 호출

```
직접 호출:  Client ──────────────────→ Server
간접 호출:  Client ────→ Proxy ────→ Server
```

마트에 내가 직접 갈 수도 있지만, 누군가에게 대신 장을 봐달라고 부탁할 수도 있다. 이 **대리자가 프록시(Proxy)**. 대리자를 끼우면 중간에서 여러 일이 가능해진다 — 라면이 이미 집에 있다고 알려주거나(캐싱), 주유를 부탁했는데 세차까지 해오거나(부가 기능), 또 다른 대리자에게 다시 부탁하거나(프록시 체인).

### 1-1. ⚠️ 프록시가 되기 위한 조건 — 대체 가능성

> 아무 객체나 프록시가 될 수 있는 게 아니다. **클라이언트는 서버에게 요청한 것인지 프록시에게 요청한 것인지조차 몰라야 한다.**

그래서 **서버와 프록시가 같은 인터페이스를 사용**해야 한다. 그래야 런타임에 DI로 `client → server`를 `client → proxy → server`로 바꿔도 **클라이언트 코드가 한 줄도 안 바뀐다**.

```
클래스 의존 관계                         런타임 객체 의존 관계
                                        (프록시 도입 전)  client ──→ server
Client ──→ ServerInterface              (프록시 도입 후)  client ──→ proxy ──→ server
              ▲        ▲
           Proxy    Server              → Client는 인터페이스에만 의존하므로 교체 사실을 모른다
```

프록시 체인도 가능하다 — `Client → Proxy1 → Proxy2 → Server`. 클라이언트는 그 뒤 과정을 모른다.

> 📌 이 "같은 인터페이스" 조건이 **Adapter와 갈리는 지점**이기도 하다. 어댑터는 *다른* 인터페이스로 바꿔 끼우는 게 목적이고, 프록시는 인터페이스를 *그대로* 유지한다.

### 1-2. 프록시로 할 수 있는 두 가지

| 범주 | 정의 | 하위 종류 / 예 |
|---|---|---|
| **접근 제어** | real subject에 **도달하기 전에 가로채서, 접근 여부 자체를 결정** | 권한에 따른 차단 · **캐싱** · 지연 로딩 |
| **부가 기능 추가** | real subject를 **반드시 호출하고**, 그 앞뒤에 기능을 더함 | 요청/응답 값 변형 · 실행 시간 측정 · 로그 |

**갈림길은 "real subject를 호출하느냐 마느냐"**다. 접근 제어는 안 할 수도 있고, 부가 기능은 반드시 한다.

#### ❓ "접근 제어가 정확히 뭔데?" (세션에서 막혔던 지점)

"안에서 service를 한 번 호출 안 하게 만드는 게 접근 제어인가?" → **맞다.** 세 가지로 나눠 보면 선명해진다.

```java
// ① 권한에 따른 접근 차단 — 권한 없으면 target을 아예 호출하지 않는다
public class AuthProxy implements AdminService {
    private final AdminService target;

    @Override
    public void deleteReservation(Long id) {
        if (!SecurityContext.isAdmin()) {
            throw new AccessDeniedException("관리자만 가능");   // target 호출 없음
        }
        target.deleteReservation(id);
    }
}
```

- **② 캐싱** — 이미 조회한 값이 있으면 target에 접근하지 않는다. 아래 `CacheProxy`가 이것. 실무에서 이걸 어노테이션으로 만든 게 [`@Cacheable`](../spring/spring-cache.md).
- **③ 지연 로딩(Lazy Loading)** — 무거운 객체 생성을 실제로 필요한 시점까지 미룬다. JPA의 `FetchType.LAZY`·`getReference()`가 정확히 이것 → [JPA 프록시 노트](../jpa/proxy.md).

①은 [메서드 보안](../security/method-security.md)의 `@PreAuthorize`, ②는 [Spring Cache](../spring/spring-cache.md), ③은 [JPA 프록시](../jpa/proxy.md) — **이미 볼트에 있는 세 노트가 전부 "접근 제어 프록시"의 실사용례**다.

---

## 2. 프록시 패턴 — 접근 제어 (전체 코드)

> **GOF Proxy 의도**: "다른 객체에 대한 접근을 제어하기 위해 그 객체의 대리인(surrogate)이나 자리표시자(placeholder)를 제공한다."

```java
// ① 인터페이스 — Client, Proxy, RealSubject가 모두 여기에 의존
public interface Subject {
    String operation();
}

// ② 실제 객체 — 호출마다 1초 걸리는 무거운 조회
@Slf4j
public class RealSubject implements Subject {
    @Override
    public String operation() {
        log.info("실제 객체 호출");
        sleep(1000);
        return "data";
    }
}

// ③ 프록시 — 같은 Subject를 구현하고, 안에 target을 들고 있다
@Slf4j
public class CacheProxy implements Subject {
    private Subject target;        // 실제 객체
    private String cacheValue;     // 캐시 저장소

    public CacheProxy(Subject target) {
        this.target = target;
    }

    @Override
    public String operation() {
        log.info("프록시 호출");
        if (cacheValue == null) {
            cacheValue = target.operation();   // ⭐ 최초 1회만 실제 객체 호출
        }
        return cacheValue;                     // 이후엔 캐시 반환 = 접근 제어
    }
}

// ④ 클라이언트 — Subject에만 의존. 프록시인지 진짜인지 모른다
@Slf4j
public class ProxyPatternClient {
    private Subject subject;

    public ProxyPatternClient(Subject subject) {
        this.subject = subject;
    }

    public void execute() {
        String result = subject.operation();
        log.info("result={}", result);
    }
}

// ⑤ 사용 — "조립하는 곳"에서만 프록시를 끼운다
@Test
void cacheProxyTest() {
    RealSubject realSubject = new RealSubject();
    CacheProxy cacheProxy = new CacheProxy(realSubject);
    ProxyPatternClient client = new ProxyPatternClient(cacheProxy);  // ⭐ 여기만 바뀐다

    client.execute();   // 캐시 없음 → realSubject 호출 (1초)
    client.execute();   // 캐시 있음 → realSubject 호출 X (0초)
    client.execute();   // 캐시 있음 → realSubject 호출 X (0초)
    // 총 1초. 프록시가 없었다면 3초
}
```

**핵심**: `RealSubject`와 `ProxyPatternClient` 어느 쪽도 수정하지 않았다. 조립 코드 한 줄만 바꿨다.

---

## 3. 데코레이터 패턴 — 부가 기능 추가 (전체 코드)

> **GOF Decorator 의도**: "객체에 추가 책임을 **동적으로** 부여한다. 데코레이터는 기능 확장을 위해 서브클래싱의 유연한 대안을 제공한다."

```java
// ① 인터페이스
public interface Component {
    String operation();
}

// ② 실제 객체
@Slf4j
public class RealComponent implements Component {
    @Override
    public String operation() {
        log.info("RealComponent 실행");
        return "data";
    }
}

// ③ 데코레이터 1 — 결과 메시지를 꾸민다
@Slf4j
public class MessageDecorator implements Component {
    private Component component;          // 꾸밀 대상

    public MessageDecorator(Component component) {
        this.component = component;
    }

    @Override
    public String operation() {
        log.info("MessageDecorator 실행");
        String result = component.operation();       // ⭐ 대상 반드시 호출
        String decoResult = "*****" + result + "*****";
        log.info("꾸미기 적용 전={}, 적용 후={}", result, decoResult);
        return decoResult;                           // 변형된 결과 반환
    }
}

// ④ 데코레이터 2 — 실행 시간을 측정한다
@Slf4j
public class TimeDecorator implements Component {
    private Component component;

    public TimeDecorator(Component component) {
        this.component = component;
    }

    @Override
    public String operation() {
        log.info("TimeDecorator 실행");
        long startTime = System.currentTimeMillis();

        String result = component.operation();       // ⭐ 대상 반드시 호출

        long endTime = System.currentTimeMillis();
        log.info("TimeDecorator 종료 resultTime={}ms", endTime - startTime);
        return result;                               // 결과는 그대로, 측정만 부가
    }
}

// ⑤ 클라이언트
@Slf4j
public class DecoratorPatternClient {
    private Component component;

    public DecoratorPatternClient(Component component) {
        this.component = component;
    }

    public void execute() {
        String result = component.operation();
        log.info("result={}", result);
    }
}

// ⑥ 사용 — 데코레이터 체이닝
@Test
void decorator2() {
    Component realComponent = new RealComponent();
    Component messageDecorator = new MessageDecorator(realComponent);
    Component timeDecorator = new TimeDecorator(messageDecorator);
    DecoratorPatternClient client = new DecoratorPatternClient(timeDecorator);
    client.execute();
}
```

```
호출:  client → timeDecorator → messageDecorator → realComponent
반환:  client ← timeDecorator ← messageDecorator ← realComponent

TimeDecorator 실행
MessageDecorator 실행
RealComponent 실행
MessageDecorator 꾸미기 적용 전=data, 적용 후=*****data*****
TimeDecorator 종료 resultTime=7ms
result=*****data*****
```

### 3-1. GOF가 말하는 Decorator 추상 클래스

데코레이터들끼리 중복이 있다 — 꾸며주는 역할이라 **스스로 존재할 수 없고**, 항상 내부에 `component`를 들고 **반드시 호출**해야 한다. 이 중복을 뽑아 추상 클래스로 만든 게 GOF의 기본 예제.

```
Component (인터페이스)
├── RealComponent
└── Decorator (추상 클래스 — component 필드 보유)
    ├── TimeDecorator
    └── MessageDecorator
```

부수 효과로 클래스 다이어그램에서 **무엇이 진짜 컴포넌트고 무엇이 데코레이터인지 명확히 구분**된다. ⚠️ 다만 "Decorator 추상 클래스를 만들어야만 데코레이터 패턴"인 건 아니다 — 패턴을 가르는 건 구조가 아니라 의도.

> 📌 **자바 표준 라이브러리의 데코레이터**: `java.io`가 통째로 이 패턴이다. `new BufferedInputStream(new FileInputStream(f))` — `InputStream`이라는 같은 타입을 유지하면서 버퍼링이라는 책임을 덧입힌다. 위 체이닝 코드와 완전히 같은 모양.

> 📘 **Effective Java Item 18과 같은 이야기**: "상속보다는 컴포지션을 사용하라"에서 권하는 **래퍼 클래스(wrapper class)** 가 바로 데코레이터다 — Bloch도 책에서 명시적으로 그렇게 부른다. 상속으로 기능을 확장하면 조합 폭발(기능 n개 → 클래스 2ⁿ)과 깨지기 쉬운 결합이 생기지만, 래퍼를 쌓으면 **런타임에 재귀적으로 조합**할 수 있다. 위 `new TimeDecorator(new MessageDecorator(real))`이 그 예. ⚠️ 단 Item 18은 래퍼의 약점도 같이 짚는다 — **콜백 프레임워크와는 안 어울린다(SELF 문제)**. 이건 §9의 자기호출 함정과 완전히 같은 메커니즘이다. (이펙티브 자바 노트 작성 시 상호 링크)

---

## 4. 프록시 vs 데코레이터 — 구조가 같은데 왜 두 패턴인가

의문이 두 개 든다. ① Decorator 추상 클래스를 만들어야만 데코레이터인가? ② 둘의 모양이 거의 같은데?

**디자인 패턴에서 중요한 건 겉모양이 아니라 그 패턴을 만든 의도**다. 실제로 GOF는 구조가 겹치는 패턴들(Proxy·Decorator·Adapter·Facade)을 전부 의도로 구분한다.

| | **프록시 패턴** | **데코레이터 패턴** |
|---|---|---|
| **의도** | 접근을 **제어**하려고 대리자를 제공 | 객체에 추가 책임을 **동적으로 추가**, 서브클래싱의 유연한 대안 |
| 핵심 질문 | "target에 접근할까 말까?" | "target 결과에 뭘 더할까?" |
| target 호출 | 조건에 따라 **안 할 수도** 있음 | **반드시** 호출 |
| 강의 예제 | `CacheProxy` | `MessageDecorator` · `TimeDecorator` |
| 클라이언트 인식 | 대체로 **모른다** (몰라야 이득) | 대체로 **안다** (직접 조립해서 씀) |
| 체이닝 | 가능하지만 주 용도 아님 | **재귀적 조립이 본령** (`java.io`) |
| 대상 확보 | 프록시가 **스스로 생성**하기도 함 | 항상 **밖에서 주입**받음 |

> 🔎 커뮤니티에서 자주 인용되는 표현: **"데코레이터는 클라이언트에게 알리고 힘을 주고, 프록시는 클라이언트를 제한한다."** 그리고 "누가 인스턴스를 만드는가"도 실전 감별 포인트다 — 데코레이터는 클라이언트가 조립하고, 프록시는 클라이언트 모르게 끼워진다. (스프링 AOP가 정확히 후자)

**정리**: 프록시를 쓰는데 목적이 접근 제어면 프록시 패턴, 새 기능 추가면 데코레이터 패턴. "프록시 패턴"이라는 이름 때문에 이 패턴만 프록시를 쓰는 것으로 오해하기 쉬운데, **데코레이터 패턴도 프록시를 쓴다.**

---

## 5. 실전 적용 ①: 인터페이스 기반 프록시 (V1)

```
클래스 의존 관계                              런타임 객체 의존 관계
Client → OrderRepositoryV1                    client → orderRepositoryProxy → orderRepositoryV1Impl
            ▲              ▲                            (LogTrace 부가 기능)     (실제 로직)
  OrderRepositoryProxy   OrderRepositoryV1Impl
```

```java
// ① 인터페이스 (Controller / Service / Repository 각각 존재)
public interface OrderRepositoryV1 {
    void save(String itemId);
}

// ② 실제 구현체 — 손대지 않는다
public class OrderRepositoryV1Impl implements OrderRepositoryV1 {
    @Override
    public void save(String itemId) {
        if (itemId.equals("ex")) {
            throw new IllegalStateException("예외 발생!");
        }
        sleep(1000);
    }
}

// ③ 프록시 — 같은 인터페이스 구현 + target 위임
@RequiredArgsConstructor
public class OrderRepositoryInterfaceProxy implements OrderRepositoryV1 {
    private final OrderRepositoryV1 target;   // 실제 객체
    private final LogTrace logTrace;          // 부가 기능

    @Override
    public void save(String itemId) {
        TraceStatus status = null;
        try {
            status = logTrace.begin("OrderRepository.save()");
            target.save(itemId);              // ⭐ 실제 객체에 위임
            logTrace.end(status);
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;                          // ⚠️ 되던지기 필수
        }
    }
}

// ④ 설정 — 프록시를 빈으로 등록하고 실제 객체를 target으로 주입
@Configuration
public class InterfaceProxyConfig {

    @Bean
    public OrderRepositoryV1 orderRepository(LogTrace logTrace) {
        OrderRepositoryV1Impl realRepository = new OrderRepositoryV1Impl();
        return new OrderRepositoryInterfaceProxy(realRepository, logTrace);
    }
    // Service, Controller도 동일한 구조로 등록
}
```

### ⚠️ 스프링 컨테이너에 등록되는 건 "프록시"다

```
스프링 빈 저장소                                  실제 객체 (자바 힙에만 존재)
orderController → OrderControllerInterfaceProxy ──→ OrderControllerV1Impl
orderService    → OrderServiceInterfaceProxy    ──→ OrderServiceV1Impl
orderRepository → OrderRepositoryInterfaceProxy ──→ OrderRepositoryV1Impl
```

실제 객체는 **프록시를 통해서만 참조**될 뿐, 스프링 컨테이너가 관리하지 않는다. → 이후 `@Transactional`·`@Cacheable`을 붙였을 때 "내가 주입받은 그 빈은 사실 프록시"라는 감각이 여기서 생긴다.

---

## 6. 실전 적용 ②: 클래스 기반 프록시 (V2)

인터페이스가 없어도 된다. **자바 다형성은 인터페이스 구현이든 클래스 상속이든 상위 타입만 맞으면 적용**되기 때문.

```java
// ① 구체 클래스만 존재 (인터페이스 없음)
public class OrderRepositoryV2 {
    public void save(String itemId) {
        if (itemId.equals("ex")) throw new IllegalStateException("예외 발생!");
        sleep(1000);
    }
}

// ② 프록시 — implements가 아니라 extends
public class OrderRepositoryConcreteProxy extends OrderRepositoryV2 {
    private final OrderRepositoryV2 target;
    private final LogTrace logTrace;

    public OrderRepositoryConcreteProxy(OrderRepositoryV2 target, LogTrace logTrace) {
        this.target = target;
        this.logTrace = logTrace;
    }

    @Override
    public void save(String itemId) {
        TraceStatus status = null;
        try {
            status = logTrace.begin("OrderRepository.save()");
            target.save(itemId);      // ⭐ target.save()지 super.save()가 아니다!
            logTrace.end(status);
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }
}

// ③ 설정 — 상속했으므로 부모 타입으로 빈 등록 가능 (다형성)
@Configuration
public class ConcreteProxyConfig {
    @Bean
    public OrderRepositoryV2 orderRepositoryV2(LogTrace logTrace) {
        return new OrderRepositoryConcreteProxy(new OrderRepositoryV2(), logTrace);
    }
}
```

⚠️ **`target.save()` vs `super.save()`**: `super.save()`를 부르면 "상속받은 껍데기 자신의 로직"이 돌아 프록시 의미가 없어진다. target은 **별도로 주입받은 실제 인스턴스**다. 상속은 타입을 맞추기 위한 수단일 뿐, 동작은 여전히 **위임**.

### 6-1. ❓ `super(null)`은 왜 하는 건가 (세션에서 막혔던 지점)

자바 규칙: **자식 생성자가 실행될 때 부모 생성자가 무조건 먼저 호출되어야 한다.** 컴파일러가 강제한다. 부모에 기본 생성자가 있으면 컴파일러가 `super()`를 몰래 넣어주므로 **아무 일도 안 일어난 것처럼 보인다.** 문제는 **부모에 파라미터 있는 생성자만 있을 때** — 이때는 직접 써야 하고, 넘길 값이 없다.

위 `OrderRepositoryV2`는 생성자가 없어서(=기본 생성자) 프록시도 `super()`를 안 써도 됐다. 그런데 `OrderServiceV2`는 다르다.

```java
// 부모 — 생성자가 OrderRepositoryV2를 요구한다
public class OrderServiceV2 {
    private final OrderRepositoryV2 orderRepository;

    public OrderServiceV2(OrderRepositoryV2 orderRepository) {   // 기본 생성자 없음!
        this.orderRepository = orderRepository;
    }

    public void orderItem(String itemId) {
        orderRepository.save(itemId);
    }
}

// 프록시 — 상속했으니 부모 생성자를 불러야 한다
public class OrderServiceConcreteProxy extends OrderServiceV2 {
    private final OrderServiceV2 target;
    private final LogTrace logTrace;

    public OrderServiceConcreteProxy(OrderServiceV2 target, LogTrace logTrace) {
        super(null);              // ⚠️ 부모가 OrderRepositoryV2를 요구 → null이라도 넘겨야 컴파일 통과
        this.target = target;
        this.logTrace = logTrace;
    }

    @Override
    public void orderItem(String itemId) {
        TraceStatus status = null;
        try {
            status = logTrace.begin("OrderService.orderItem()");
            target.orderItem(itemId);      // 부모의 orderRepository가 아니라 target에 위임
            logTrace.end(status);
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }
}
```

**프록시는 부모의 필드를 하나도 안 쓴다.** `orderItem()`은 오버라이딩했고, 내부에선 `target.orderItem()`을 부른다. 부모의 `orderRepository`는 `null`인 채로 영영 방치되고 **그래도 아무 문제가 없다** — 애초에 쓰이지 않으니까. 즉 `super(null)`은 의미 있는 초기화가 아니라 **문법을 통과시키려는 요식 행위**다.

⚠️ 위험한 지점: 만약 오버라이딩을 빠뜨린 메서드가 있으면, 그 메서드는 **부모 구현이 실행되면서 `null`인 필드를 건드려 NPE**가 난다. 클래스 기반 프록시는 "모든 public 메서드를 빠짐없이 오버라이딩했는가"가 암묵적 전제다.

인터페이스 기반이면 `implements`만 하면 되니 이 문제가 통째로 없다.

### 6-2. ⚠️ 클래스 기반 프록시의 제약 3가지

| 제약 | 왜 | 실무에서 터지는 지점 |
|---|---|---|
| **부모 생성자 호출 필수** | 자바 상속 규칙 | 파라미터 많으면 `super(null, null, null, ...)` |
| **`final` 클래스 → 상속 불가** | `extends` 자체가 안 됨 | 프록시를 아예 만들 수 없음 |
| **`final` 메서드 → 오버라이딩 불가** | 재정의가 안 되니 가로챌 수 없음 | 그 메서드만 부가 기능이 안 붙음 |

> 🔴 **이 세 개는 5장 CGLIB, 나아가 스프링 AOP에 그대로 상속된다.** 스프링 부트는 기본이 CGLIB(클래스 기반) 프록시라서 남 얘기가 아니다.
>
> **실패하는 방식이 다르다는 점이 중요**하다 — `final` **클래스**는 프록시 생성 자체가 실패해서 **기동 시점에 시끄럽게** 터진다. 반면 `final` **메서드**는 CGLIB가 그 메서드만 **조용히 건너뛴다**(`Enhancer`가 final 메서드를 필터링해서 제외). 예외도, 눈에 띄는 경고도 없이 `@Transactional`이 그냥 안 먹는다. Kotlin은 `final`이 기본값이라 이 사고가 더 잦다.

---

## 7. 인터페이스 기반 vs 클래스 기반

| | 인터페이스 기반 | 클래스 기반 |
|---|---|---|
| 적용 범위 | 인터페이스만 같으면 **모든 구현체** | **그 클래스에만** |
| 상속 제약 | 없음 | `final` 클래스/메서드 제약 |
| 생성자 | 문제 없음 | `super(null)` 필요 |
| 전제 조건 | **인터페이스가 있어야 함** | 없어도 됨 |
| 타입 캐스팅 | ⚠️ **구체 클래스로 캐스팅/주입 불가** | 자식 타입이라 구체 클래스 주입 가능 |

⚠️ 마지막 줄이 실무에서 덧난다. 인터페이스 기반 프록시는 **인터페이스 타입으로만 존재**하므로 `OrderServiceImpl`처럼 구현체 타입으로 주입받으려 하면 터진다(`BeanNotOfRequiredTypeException`). 강의에서도 "인터페이스 기반 프록시는 캐스팅 관련 단점이 있다"고 뒷부분 예고를 달아둔다. 스프링 부트가 기본값을 CGLIB로 잡은 이유 중 하나.

### ❓ "그럼 왜 처음부터 인터페이스로 안 하나?" (세션에서 나온 질문 — 타당한 의문)

인터페이스 기반이 제약도 적고 역할/구현이 나뉘어 더 낫다. **맞다.** 다만 강의의 결론은 "항상 인터페이스를 쓰자"가 아니다.

> 인터페이스 도입은 **구현을 변경할 가능성이 있을 때** 효과적이다. 바뀔 일이 거의 없는 코드에 무작정 인터페이스를 넣는 건 번거롭고 실용적이지 않다. 실무에는 V1 같은 구조도 V2 같은 구조도 있으니 **둘 다 대응할 수 있어야 한다.**

그리고 이 의문이 그대로 [5장의 구조](./dynamic-proxy.md)가 된다:

- **JDK 동적 프록시** → 인터페이스 필수
- **CGLIB** → 클래스 상속 방식, 인터페이스 없어도 됨

**두 기술이 공존하는 이유가 여기에 있다.** (스프링 부트는 두 상황을 다 커버하려고 기본을 CGLIB로 잡았다)

---

## 8. ❓ V3(컴포넌트 스캔)는 왜 안 했나 — 빠뜨린 게 아니라 숙제

V1·V2는 `@Configuration`에서 **수동으로** 빈을 등록했으니, 설정 파일에서 실제 객체 대신 프록시를 리턴하면 끝이었다. 그런데 V3는 `@Controller`·`@Service`·`@Repository`로 **컴포넌트 스캔에 의해 자동 등록**된다. 스프링이 알아서 실제 객체를 빈으로 만들어버리니, **개발자가 "이 빈 대신 프록시를 넣어줘"라고 끼어들 자리가 없다.**

"프록시 구현체 쪽으로 어노테이션을 옮기면?" → 그건 **원본 코드 수정**이라 요구사항 위반.

**→ 7장 빈 후처리기(`BeanPostProcessor`)가 이 문제를 푼다.** 스프링이 빈을 등록하기 **직전에** 가로채서 원본 객체를 프록시로 **바꿔치기**한다. 등록 방식(수동/컴포넌트 스캔)과 무관하게 적용된다.

---

## 9. ⚠️ 그 외 함정 / 헷갈렸던 것

### 🔴 자기호출(self-invocation)은 왜 프록시를 못 거치는가 — 볼트 전체가 공유하는 함정의 정체

[@Transactional](../spring/transactional.md) · [Spring Cache](../spring/spring-cache.md) · [메서드 보안](../security/method-security.md) · [커스텀 어노테이션](../annotation/custom-annotation.md) 네 노트가 전부 "자기호출은 안 먹는다"를 경고한다. **이유는 손으로 짠 프록시에서 이미 드러난다.**

```java
public class OrderServiceV1Impl implements OrderServiceV1 {

    @Override
    public void orderItem(String itemId) {
        this.cancelItem(itemId);   // ⚠️ 내부 호출
    }

    @Override
    public void cancelItem(String itemId) { ... }
}
```

```
client → proxy.orderItem()  → [로그 남김] → target.orderItem()
                                                    └─→ this.cancelItem()   ← 🔴 proxy를 안 거침 → 로그 없음
```

`target` 입장에서 `this`는 **자기 자신(실제 객체)이지 프록시가 아니다.** 프록시는 밖에서 들어오는 입구만 감싸고 있을 뿐, 객체 내부의 호출까지 가로채는 수단이 없다. 이건 스프링의 버그가 아니라 **프록시라는 구조의 필연적 결과**다. 그래서 `@Transactional`을 붙인 메서드를 같은 빈 안에서 부르면 트랜잭션이 안 열리고, 해결책도 전부 똑같다 — **호출을 객체 밖으로 내보내기**(별도 빈으로 분리하거나 자기 자신을 `@Lazy`로 주입받아 그 참조로 호출).

> 💡 이건 *Effective Java* Item 18의 **SELF 문제**와 정확히 같은 메커니즘이다. 래퍼(데코레이터)로 감싼 객체가 콜백을 위해 `this`를 넘기면, 넘어가는 건 래퍼가 아니라 **감싸진 안쪽 객체**라 이후 콜백은 래퍼를 우회한다. "감싸기는 밖에서 들어오는 호출만 잡는다"는 한 문장이 자바 레벨에서도, 스프링 레벨에서도 그대로 적용된다.

### 스프링 시큐리티는 프록시인가? (세션 질문)

"`beforeFilter` 같은 걸 보니 요청과 Controller 사이에 뭔가 해주는 것 같던데?" → 맞는 관찰이지만 **레벨이 다르다.**

| 시큐리티 기능 | 동작 방식 | 가로채는 레벨 |
|---|---|---|
| URL 기반 보안 (`SecurityFilterChain`) | **서블릿 필터 체인** (`FilterChainProxy`) | **HTTP 요청** — Controller에 닿기도 전 |
| 메서드 보안 (`@PreAuthorize`·`@Secured`) | **스프링 AOP 프록시** | **메서드 호출** |

```
필터:   HTTP 요청 → [SecurityFilter] → ... → DispatcherServlet → Controller
프록시: Controller → ServiceProxy(권한 체크) → ServiceImpl
```

**필터는 HTTP 요청 레벨, 프록시는 메서드 호출 레벨.** 자세한 건 [메서드 보안 노트](../security/method-security.md).

### 프록시의 비용 — 공짜가 아니다

프록시는 호출 스택을 한 겹 늘린다. 잘못 만든 프록시(무거운 로직, 동기 I/O)는 그대로 응답 시간에 더해진다. "부가 기능이니 가벼울 것"이라는 전제가 항상 맞진 않는다.

### 프록시 객체의 상태는 공유된다

`CacheProxy`의 `cacheValue`는 **싱글톤 빈이라면 모든 요청이 공유하는 필드**다. 사용자별로 달라야 하는 값을 프록시 필드에 넣으면 [ThreadLocal 노트](../concurrency/thread-local.md)에서 본 것과 똑같은 동시성 사고가 난다.

### 🔴 이 챕터의 한계 — 다음 챕터가 존재하는 이유

프록시 클래스가 하는 일은 전부 똑같다(`LogTrace` 호출). **대상 클래스만 다를 뿐인데 대상이 100개면 프록시도 100개**를 만들어야 한다.

→ **[5장 동적 프록시](./dynamic-proxy.md)**: 프록시 클래스를 직접 만들지 않고 **런타임에 자동 생성**한다. 부가 로직을 `InvocationHandler`·`MethodInterceptor` **한 군데**에만 적고, 대상별 클래스는 JDK/CGLIB이 찍어낸다.

---

## 10. 💡 판단 기준

- **"프록시냐 데코레이터냐"로 30분 고민하고 있다면 그 고민 자체가 실익이 없다.** 구조는 같고 이름만 다르니 코드는 어느 쪽으로 불러도 동작한다. 실익은 **클래스 이름을 지을 때** 나온다 — `XxxCacheProxy`/`XxxAuthProxy`처럼 "막는" 이름인지 `XxxLoggingDecorator`처럼 "더하는" 이름인지가 다음 사람에게 **target을 호출 안 할 수도 있는지**를 알려준다. 이름이 곧 계약.
- **"원본을 수정하지 않고 기능을 넣고 싶다"가 신호.** 원본을 열어도 되는 상황이면 프록시는 과한 도구다 — 그냥 메서드에 코드를 넣는 게 낫다. 프록시의 값은 **남의 코드/공용 코드/수백 개 클래스**처럼 열 수 없거나 열기 싫을 때 나온다.
- **인터페이스가 이미 있으면 인터페이스 기반, 없으면 굳이 만들지 말고 클래스 기반.** 인터페이스는 구현이 바뀔 가능성이 있을 때 값을 하지, 프록시를 붙이려고 만드는 건 본말전도다. (스프링도 결국 두 방식을 다 지원하는 쪽으로 갔다)
- **`final`을 붙이기 전에 "여기 프록시가 붙을 수 있나"를 한 번 묻는다.** 불변성을 위해 `final class`를 붙였는데 나중에 `@Transactional`·`@Cacheable`을 못 붙이는 상황이 온다. 특히 **`final` 메서드는 조용히 실패**하므로, 트랜잭션이 안 먹는데 원인을 못 찾겠다면 이걸 의심할 것.
- **이 챕터의 진짜 가치는 "스프링 AOP가 왜 그렇게 생겼는지"의 족보.** 2026년에 프록시를 손으로 짤 일은 거의 없다. 하지만 `@Transactional`이 자기호출에서 안 먹는 이유, `@Cacheable`이 `@EnableCaching` 없으면 조용히 무시되는 이유, JPA `getReference()`가 `==` 비교에서 배신하는 이유가 **전부 "그건 프록시다"라는 한 문장에서 나온다.**

---

## 참고

- 김영한, 스프링 핵심 원리 고급편 — Ch.4 프록시 패턴과 데코레이터 패턴
- GOF, *Design Patterns* — Proxy / Decorator (구조가 겹치는 패턴은 의도로 구분)
- **Effective Java Item 18** — "상속보다는 컴포지션을 사용하라" (래퍼 클래스 = 데코레이터, SELF 문제) — 도달 시 상호 링크
- [Baeldung — Proxy, Decorator, Adapter and Bridge Patterns](https://www.baeldung.com/java-structural-design-patterns)
- [Stack Overflow — Differences between Proxy and Decorator Pattern](https://stackoverflow.com/questions/18618779/differences-between-proxy-and-decorator-pattern) ("데코레이터는 클라이언트에게 힘을 주고 프록시는 제한한다")
- [spring-framework #26729 — Fail explicitly if a final method is invoked on a CGLIB proxy](https://github.com/spring-projects/spring-framework/issues/26729) (final 메서드가 **조용히** 제외되는 근거)

**학습 날짜**: 2026-08-14 · **계기**: 김영한 고급편 Ch.4 수강 후 Claude 소크라테스 복습 세션 — 진단 4문항 중 "프록시의 대체 가능성 조건"과 "`final` 제약 2가지"를 못 짚었고, **접근 제어의 정의**(target을 호출 안 하는 것도 제어인가) · **`super(null)`의 이유** · **V3를 왜 안 했는지** · **스프링 시큐리티도 프록시인지**를 질문으로 파고들어 채움

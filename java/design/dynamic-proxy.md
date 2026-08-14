# 동적 프록시 — JDK 동적 프록시 / CGLIB, 프록시 클래스를 런타임에 찍어내기

> **한 줄 요약**: 4장에서 프록시를 손으로 만들었더니 **대상마다 프록시 클래스가 하나씩** 필요했다. 동적 프록시는 그 클래스를 **런타임에 자동 생성**해서, 부가 로직 **핸들러 하나로 대상 전부**를 커버한다 — 인터페이스가 있으면 **JDK 동적 프록시**(`InvocationHandler` + `Proxy.newProxyInstance`), 없으면 **CGLIB**(`MethodInterceptor` + `Enhancer`). ⚠️ 이 API를 직접 칠 일은 거의 없다. **`@Transactional`·`@Cacheable`·`@Async`가 바로 이 기술로 동작한다**는 것, 그래서 **`this.method()` 자기호출이 왜 안 먹는지**가 이 챕터의 진짜 소득.

관련 노트: [프록시/데코레이터 패턴](./proxy-decorator-pattern.md) (직전 챕터 — "프록시 클래스 폭발"로 끝난 자리가 이 노트의 출발점) · **ProxyFactory (6장, 아직 미작성)** — 이 노트가 남긴 "JDK/CGLIB 이중 관리"를 Advice 추상화로 해결 · [템플릿 메서드/전략/콜백](./template-method-strategy-callback.md) · [ThreadLocal](../concurrency/thread-local.md) (`LogTrace`의 출처) · [@Transactional](../spring/transactional.md) · [Spring Cache](../spring/spring-cache.md) · [메서드 보안](../security/method-security.md) · [커스텀 어노테이션](../annotation/custom-annotation.md) — **이 네 노트가 공유하는 "자기호출은 안 먹는다" 함정의 실행 엔진이 이 노트** · [JPA 프록시](../jpa/proxy.md) (하이버네이트도 같은 원리의 프록시를 쓴다)

---

## 0. 출발점 — 프록시 클래스가 대상 수만큼 필요하다

[직전 챕터](./proxy-decorator-pattern.md)에서 원본 수정 0으로 로그를 넣는 데는 성공했다. 그런데 프록시 클래스를 **대상마다 하나씩** 만들어야 했다.

```java
// Controller / Service / Repository 3개에 로그를 넣으려면 프록시도 3개
public class OrderControllerInterfaceProxy implements OrderControllerV1 {
    private final OrderControllerV1 target;
    private final LogTrace logTrace;

    @Override
    public String request(String itemId) {
        TraceStatus status = null;
        try {
            status = logTrace.begin("OrderController.request()");
            String result = target.request(itemId);   // ← 메서드 이름이 하드코딩
            logTrace.end(status);
            return result;
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }
}

public class OrderServiceInterfaceProxy implements OrderServiceV1 { /* 똑같은 코드 */ }
public class OrderRepositoryInterfaceProxy implements OrderRepositoryV1 { /* 똑같은 코드 */ }
```

**부가 기능 로직은 세 개가 완전히 동일하다.** 다른 건 `target`의 타입과 호출하는 메서드 이름뿐. 대상이 100개면 프록시도 100개 — 4장의 템플릿 콜백이 "원본 수정"을 못 없앤 것처럼, 4장 프록시는 "클래스 수"를 못 없앴다.

> 💭 **이 챕터가 푸는 질문**: "부가 로직은 하나인데, 왜 클래스를 대상 수만큼 만들어야 하지?"

**해결**: 프록시 클래스를 **런타임에 동적으로 만든다**. 개발자는 `.java` 파일을 쓰지 않고, 부가 로직만 한 곳에 적는다.

---

## 1. 예열 — 리플렉션 (동적 프록시의 재료)

동적 프록시가 가능한 이유는 자바가 **클래스 메타정보를 런타임에 다룰 수 있기** 때문이다.

```java
// 리플렉션 없이 — 호출할 메서드가 코드에 박혀 있다
Hello target = new Hello();
target.callA();   // 이 이름을 바꾸려면 코드를 고쳐야 함
target.callB();
```

```java
// 리플렉션 — 메서드를 "값"으로 다룬다
Class<?> classHello = Class.forName("hello.proxy.jdkdynamic.ReflectionTest$Hello");
Object target = new Hello();

Method methodCallA = classHello.getMethod("callA");   // 메서드를 객체로 획득
Object result1 = methodCallA.invoke(target);          // 그 객체로 호출

Method methodCallB = classHello.getMethod("callB");
Object result2 = methodCallB.invoke(target);
```

핵심은 **`Method`가 "메서드 그 자체"가 아니라 "어떤 메서드인지 가리키는 메타정보 객체"**라는 것. 메서드를 변수에 담고 파라미터로 넘길 수 있으니, **"호출할 메서드"를 런타임에 결정**할 수 있게 된다.

```java
// 공통 로직을 메서드로 뽑을 수 있다 — 호출 대상이 파라미터가 되었으므로
private void dynamicCall(Method method, Object target) throws Exception {
    log.info("start");
    Object result = method.invoke(target);   // 어떤 메서드든 호출 가능
    log.info("result={}", result);
}
```

> ⚠️ **리플렉션은 주의해서 쓸 것.** 컴파일러가 잡아주던 오류가 **런타임 오류로 미뤄진다** — `getMethod("callAA")`처럼 오타를 내도 컴파일은 통과하고 실행 시점에 `NoSuchMethodException`이 터진다. 그래서 **일반 애플리케이션 로직에는 쓰지 않고, 프레임워크/공통 처리 같은 곳에 제한적으로** 쓴다. 우리가 리플렉션을 직접 칠 일이 없는 이유이기도 하다 — 스프링이 대신 쳐준다.
>
> 📚 **Effective Java Item 65 "리플렉션보다는 인터페이스를 사용하라"가 정확히 이 주제**다. 그래서 책이 제시하는 해법도 "**객체 생성만 리플렉션으로 하고, 사용은 인터페이스로**" — JDK 동적 프록시가 정확히 그 모양이다(프록시는 리플렉션으로 만들고, 클라이언트는 `AInterface`로 쓴다). *(EJ 노트 작성 시 여기 링크 추가)*

---

## 2. JDK 동적 프록시 — 인터페이스 기반

자바가 기본 제공. **인터페이스가 반드시 있어야 한다.**

### 2-1. 구성 요소는 딱 둘

| 역할 | 무엇 |
|---|---|
| **부가 로직을 적는 곳** | `InvocationHandler` 구현체 (개발자가 작성) |
| **프록시 객체를 찍는 곳** | `Proxy.newProxyInstance(...)` (JDK가 제공) |

```java
package java.lang.reflect;

public interface InvocationHandler {
    Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
}
```

| 파라미터 | 의미 |
|---|---|
| `proxy` | **프록시 자신** (⚠️ target이 아니다) |
| `method` | **호출된 메서드의 메타정보** |
| `args` | 메서드에 넘어온 인수 배열 |

### 2-2. 전체 코드 — 시간 측정 프록시

```java
// ① 인터페이스
public interface AInterface {
    String call();
}

public interface BInterface {
    String call();
}

// ② 구현체 — 손대지 않는다
@Slf4j
public class AImpl implements AInterface {
    @Override
    public String call() {
        log.info("A 호출");
        return "a";
    }
}

@Slf4j
public class BImpl implements BInterface {
    @Override
    public String call() {
        log.info("B 호출");
        return "b";
    }
}

// ③ 부가 로직 — ⭐ 이 클래스 하나가 A도 B도 전부 커버한다
@Slf4j
public class TimeInvocationHandler implements InvocationHandler {

    private final Object target;   // 타입이 Object — 어떤 구현체든 받는다

    public TimeInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        log.info("TimeProxy 실행");
        long startTime = System.currentTimeMillis();

        Object result = method.invoke(target, args);   // ⭐ 리플렉션으로 실제 객체 호출

        long endTime = System.currentTimeMillis();
        log.info("TimeProxy 종료 resultTime={}", endTime - startTime);
        return result;
    }
}

// ④ 사용 — 프록시 클래스를 한 개도 만들지 않았다
@Test
void dynamicA() {
    AInterface target = new AImpl();
    TimeInvocationHandler handler = new TimeInvocationHandler(target);

    AInterface proxy = (AInterface) Proxy.newProxyInstance(
            AInterface.class.getClassLoader(),   // 프록시 클래스를 로딩할 클래스로더
            new Class[]{AInterface.class},       // ⭐ 구현할 인터페이스 (배열 = 여러 개 가능)
            handler                              // 호출을 받을 핸들러
    );

    proxy.call();
    log.info("targetClass={}", target.getClass());  // class ...AImpl
    log.info("proxyClass={}", proxy.getClass());    // class com.sun.proxy.$Proxy1
}

@Test
void dynamicB() {
    BInterface target = new BImpl();
    TimeInvocationHandler handler = new TimeInvocationHandler(target);  // ⭐ 핸들러 재사용

    BInterface proxy = (BInterface) Proxy.newProxyInstance(
            BInterface.class.getClassLoader(), new Class[]{BInterface.class}, handler);

    proxy.call();
}
```

```
TimeProxy 실행
A 호출
TimeProxy 종료 resultTime=0
proxyClass=class com.sun.proxy.$Proxy1     ← 우리가 만들지 않은 클래스
```

> 📌 프록시 클래스 이름은 JDK 버전에 따라 `com.sun.proxy.$Proxy1` 또는 `jdk.proxy1.$Proxy1` 형태. **`$Proxy`가 보이면 JDK 동적 프록시**라고 읽으면 된다.

### 2-3. ⚠️ `Method method`의 정체 — 세션에서 틀렸던 지점

> ❓ **내 답**: "실제로 실행될 구현체의 메서드라서 중요한 것 아닌가?"
> ✅ **정확히는**: `method`는 **메서드 그 자체가 아니라, "클라이언트가 프록시에게 호출한 메서드"의 메타정보 객체**다. 이름·파라미터 타입·선언 클래스 같은 정보를 들고 있고, `invoke(target, args)`를 통해 **리플렉션으로 실행**시킬 수 있는 핸들.

이 구분이 왜 중요하냐면, **"동적 프록시가 왜 클래스 폭발을 해결하는가"의 답이 정확히 여기**에 있기 때문이다.

```java
// 4장: 호출할 메서드가 코드에 박혀 있다 → 메서드마다·클래스마다 프록시 필요
target.request(itemId);

// 5장: 호출할 메서드가 파라미터로 들어온다 → 핸들러 하나로 전부 처리
method.invoke(target, args);
```

`method`가 **파라미터**이기 때문에 `call()`이든 `save()`든 `findAll()`이든 같은 코드가 처리한다. **"메서드를 값으로 다룰 수 있게 되었다"가 클래스 폭발을 해결한 메커니즘.**

### 2-4. 실전 적용 — LogTrace를 동적 프록시로

```java
public class LogTraceBasicHandler implements InvocationHandler {

    private final Object target;
    private final LogTrace logTrace;

    public LogTraceBasicHandler(Object target, LogTrace logTrace) {
        this.target = target;
        this.logTrace = logTrace;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        TraceStatus status = null;
        try {
            // ⭐ 메시지도 method 메타정보에서 뽑는다 — 하드코딩 0
            String message = method.getDeclaringClass().getSimpleName()
                           + "." + method.getName() + "()";
            status = logTrace.begin(message);

            Object result = method.invoke(target, args);   // 실제 로직 호출

            logTrace.end(status);
            return result;
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;                                       // ⚠️ 되던지기 필수
        }
    }
}
```

```java
@Configuration
public class DynamicProxyBasicConfig {

    @Bean
    public OrderControllerV1 orderControllerV1(LogTrace logTrace) {
        OrderControllerV1 orderController =
                new OrderControllerV1Impl(orderServiceV1(logTrace));

        return (OrderControllerV1) Proxy.newProxyInstance(
                OrderControllerV1.class.getClassLoader(),
                new Class[]{OrderControllerV1.class},
                new LogTraceBasicHandler(orderController, logTrace));
    }

    @Bean
    public OrderServiceV1 orderServiceV1(LogTrace logTrace) {
        OrderServiceV1 orderService = new OrderServiceV1Impl(orderRepositoryV1(logTrace));
        return (OrderServiceV1) Proxy.newProxyInstance(
                OrderServiceV1.class.getClassLoader(),
                new Class[]{OrderServiceV1.class},
                new LogTraceBasicHandler(orderService, logTrace));
    }

    @Bean
    public OrderRepositoryV1 orderRepositoryV1(LogTrace logTrace) {
        OrderRepositoryV1 orderRepository = new OrderRepositoryV1Impl();
        return (OrderRepositoryV1) Proxy.newProxyInstance(
                OrderRepositoryV1.class.getClassLoader(),
                new Class[]{OrderRepositoryV1.class},
                new LogTraceBasicHandler(orderRepository, logTrace));
    }
}
```

**프록시 클래스 파일이 0개.** 4장에서 3개(`OrderControllerInterfaceProxy` 등)를 만들었던 자리가 통째로 사라졌다.

### 2-5. 메서드 필터링 — 모든 호출이 핸들러로 들어오는 문제

`invoke()`에는 **인터페이스의 모든 메서드 호출**이 들어온다. `no-log`처럼 로그를 남기면 안 되는 메서드도 마찬가지.

```java
public class LogTraceFilterHandler implements InvocationHandler {

    private final Object target;
    private final LogTrace logTrace;
    private final String[] patterns;    // 예: {"request*", "order*", "save*"}

    public LogTraceFilterHandler(Object target, LogTrace logTrace, String[] patterns) {
        this.target = target;
        this.logTrace = logTrace;
        this.patterns = patterns;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        // ⭐ 패턴에 안 맞으면 부가 기능 없이 그대로 통과
        String methodName = method.getName();
        if (!PatternMatchUtils.simpleMatch(patterns, methodName)) {
            return method.invoke(target, args);
        }

        TraceStatus status = null;
        try {
            String message = method.getDeclaringClass().getSimpleName()
                           + "." + method.getName() + "()";
            status = logTrace.begin(message);
            Object result = method.invoke(target, args);
            logTrace.end(status);
            return result;
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }
}
```

`PatternMatchUtils.simpleMatch()`는 스프링이 주는 단순 매칭 유틸 — `xxx`(완전일치) · `xxx*`(접두) · `*xxx`(접미) · `*xxx*`(포함).

> 🔴 **여기에 냄새가 난다.** 지금 핸들러 하나가 **두 가지 관심사**를 들고 있다.
> - **어디에 적용할지** (`patterns` 매칭) → 나중에 **Pointcut**이 될 것
> - **무슨 부가 기능인지** (`logTrace` 호출) → 나중에 **Advice**가 될 것
>
> 보안 체크·트랜잭션을 더 붙이면 **핸들러마다 필터링 코드를 복붙**하게 된다. 이 분리를 스프링이 대신 해주는 게 6장 `ProxyFactory`(Advice/Pointcut/Advisor)다.

### 2-6. ⚠️ 왜 인터페이스가 "필수"인가 — 세션에서 부족했던 지점

> ❓ **내 답**: "동적 프록시가 인터페이스 기반이라서 필요한 것 같은데, 핸들러에 인터페이스를 전달해야 해서 필수인가?"
> ✅ **정확히는**: JDK는 런타임에 **그 인터페이스를 `implements`한 새 클래스**를 바이트코드로 정의한다. 인터페이스가 있어야 **"어떤 메서드들을 오버라이드해야 하는지" 목록**을 알 수 있다. 그 목록의 출처가 인터페이스뿐이라서 필수인 것.

JDK가 메모리에 만들어내는 클래스는 개념적으로 이런 모양이다.

```java
// 실제로는 바이트코드로 정의되며 소스 파일은 없다
public final class $Proxy1 extends java.lang.reflect.Proxy implements AInterface {

    private static Method m3;   // AInterface.call()의 Method 객체 (static 초기화 시 채움)

    public $Proxy1(InvocationHandler h) { super(h); }

    @Override
    public String call() {
        // 자기가 하는 일은 없다. 전부 핸들러로 넘긴다.
        return (String) super.h.invoke(this, m3, null);
    }
}
```

- 인터페이스를 `implements`하므로 **`AInterface`로 캐스팅 가능** → 클라이언트는 진짜인지 프록시인지 모른다 ([대체 가능성](./proxy-decorator-pattern.md#1-1-프록시가-되기-위한-조건--대체-가능성))
- 이미 `java.lang.reflect.Proxy`를 **`extends`하고 있다** → 자바는 단일 상속이므로 **구체 클래스를 상속할 자리가 없다**. 이것도 "인터페이스만 가능"한 구조적 이유.

```
클라이언트 → proxy($Proxy1).call()
              └─ handler.invoke(proxy, method, args)   ← 우리가 짠 부가 로직
                   └─ method.invoke(target, args)      ← 리플렉션으로 실제 객체
                        └─ AImpl.call()
```

---

## 3. CGLIB — 구체 클래스 기반

**C**ode **G**enerator **Lib**rary. 바이트코드를 조작해 **대상 클래스를 `extends`한 자식 클래스**를 런타임에 만든다. 인터페이스가 없어도 된다.

> 📌 **CGLIB은 따로 받을 필요가 없다.** 스프링이 `org.springframework.cglib`로 **repackage해서 `spring-core`에 내장**해 두었다. 우리 코드에서도 `org.springframework.cglib.proxy.*`를 그대로 import한다. (원본 CGLIB 프로젝트 자체는 사실상 유지보수가 멈춰서, 스프링이 자체 패치 버전을 안고 가는 상태)

### 3-1. 전체 코드

```java
// ① 구체 클래스 — 인터페이스 없음
@Slf4j
public class ConcreteService {
    public void call() {
        log.info("ConcreteService 호출");
    }
}

// ② 부가 로직 — JDK의 InvocationHandler에 대응하는 것이 MethodInterceptor
@Slf4j
public class TimeMethodInterceptor implements MethodInterceptor {

    private final Object target;

    public TimeMethodInterceptor(Object target) {
        this.target = target;
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args,
                            MethodProxy methodProxy) throws Throwable {
        log.info("TimeProxy 실행");
        long startTime = System.currentTimeMillis();

        Object result = methodProxy.invoke(target, args);   // ⭐ MethodProxy 권장

        long endTime = System.currentTimeMillis();
        log.info("TimeProxy 종료 resultTime={}", endTime - startTime);
        return result;
    }
}

// ③ 사용 — Proxy.newProxyInstance 대신 Enhancer
@Test
void cglib() {
    ConcreteService target = new ConcreteService();

    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(ConcreteService.class);              // ⭐ 상속할 구체 클래스
    enhancer.setCallback(new TimeMethodInterceptor(target));    // 부가 로직
    ConcreteService proxy = (ConcreteService) enhancer.create(); // 프록시 생성

    proxy.call();

    log.info("targetClass={}", target.getClass()); // class ...ConcreteService
    log.info("proxyClass={}", proxy.getClass());
    // class ...ConcreteService$$EnhancerByCGLIB$$25d6b0e3
}
```

`MethodInterceptor` 시그니처:

| 파라미터 | 의미 |
|---|---|
| `obj` | **프록시 자신** |
| `method` | 호출된 메서드 메타정보 |
| `args` | 인수 |
| `methodProxy` | ⭐ **메서드 호출용 프록시** — 리플렉션보다 빠른 경로 |

### 3-2. ⚠️ `methodProxy.invoke` vs `invokeSuper` — 헷갈리면 무한 루프

CGLIB은 호출 방법이 셋이라 헷갈린다.

| 호출 | 무엇을 부르나 | 언제 |
|---|---|---|
| `method.invoke(target, args)` | 리플렉션으로 target 호출 | 동작하지만 **가장 느림** |
| `methodProxy.invoke(target, args)` | **다른 인스턴스(target)** 의 원본 메서드 | ⭐ **target을 따로 주입받는 구조**(강의 방식) |
| `methodProxy.invokeSuper(obj, args)` | **프록시 자신**의 부모(super) 메서드 | target 없이 `Enhancer`만으로 만들 때 |

`MethodProxy`가 빠른 이유는 **FastClass** — 메서드마다 인덱스를 부여해 `switch`로 분기하는 클래스를 미리 생성해 두고, 리플렉션 대신 그 인덱스로 직접 호출한다.

```java
// ⚠️ 무한 루프 — 부가 로직이 자기 자신을 다시 부른다 (StackOverflowError)
methodProxy.invoke(obj, args);   // obj = 프록시 자신
method.invoke(obj, args);        // 같은 이유

// ⚠️ 무한 루프가 아니라 "잘못된 짝" — 즉시 예외
methodProxy.invokeSuper(target, args);
// invokeSuper는 **프록시 클래스용 FastClass**로 호출한다.
// target은 프록시가 아니라 CGLIB이 만든 synthetic 메서드가 없음
// → 타입 불일치로 ClassCastException / IllegalArgumentException
```

**규칙**: `target`을 주입받았으면 `invoke(target, ...)`, 안 받았으면 `invokeSuper(obj, ...)`. **첫 인수와 메서드 이름이 짝**이라고 외우면 안전하다.

> 💡 **둘을 구분하는 말**: `invoke`는 "**남**의 원본 메서드", `invokeSuper`는 "**내** 부모 메서드". 전자는 위임(target 보유), 후자는 상속(target 없음) 구조다. — 4장의 [`target.save()` vs `super.save()`](./proxy-decorator-pattern.md) 구분과 정확히 같은 질문이 API로 나타난 것.

### 3-3. 실전 적용 — LogTrace를 CGLIB으로

```java
public class LogTraceBasicHandler implements MethodInterceptor {

    private final Object target;
    private final LogTrace logTrace;

    public LogTraceBasicHandler(Object target, LogTrace logTrace) {
        this.target = target;
        this.logTrace = logTrace;
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args,
                            MethodProxy methodProxy) throws Throwable {
        TraceStatus status = null;
        try {
            String message = method.getDeclaringClass().getSimpleName()
                           + "." + method.getName() + "()";
            status = logTrace.begin(message);

            Object result = methodProxy.invoke(target, args);

            logTrace.end(status);
            return result;
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }
}
```

```java
@Configuration
public class CglibConfig {

    @Bean
    public OrderControllerV2 orderControllerV2(LogTrace logTrace) {
        OrderControllerV2 target = new OrderControllerV2(orderServiceV2(logTrace));

        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(OrderControllerV2.class);
        enhancer.setCallback(new LogTraceBasicHandler(target, logTrace));
        return (OrderControllerV2) enhancer.create(
                new Class[]{OrderServiceV2.class},   // 부모 생성자 시그니처
                new Object[]{orderServiceV2(logTrace)});  // 부모 생성자 인수
    }
    // ...
}
```

⚠️ 부모에 기본 생성자가 없으면 `enhancer.create(argTypes, args)`로 **부모 생성자 인수를 직접 넘겨야** 한다. 4장의 `super(null)` 문제가 API 형태로 다시 나온 것 — [프록시 패턴 노트 §6-1](./proxy-decorator-pattern.md#6-1--supernull은-왜-하는-건가-세션에서-막혔던-지점).

### 3-4. ⚠️ CGLIB 제약 3가지 (그리고 스프링이 둘을 없앤 이야기)

| 제약 | 왜 | 순수 CGLIB | **스프링의 CGLIB** |
|---|---|---|---|
| **기본 생성자 필요** | 자식 생성 시 부모 생성자 호출 | 필요 (`Enhancer.create()`) | ❌ **불필요** — Objenesis로 생성자를 아예 건너뜀 (Spring 4.0+) |
| **`final` 클래스 → 상속 불가** | `extends` 자체가 안 됨 | 프록시 생성 실패 | 동일 — **기동 시 시끄럽게 실패** |
| **`final` 메서드 → 오버라이드 불가** | 재정의가 안 되니 가로챌 수 없음 | 그 메서드만 스킵 | 동일 — ⚠️ **조용히 스킵** |

> 🔎 **강의는 "기본 생성자가 필요하다"고 말하지만, 실무에서 만나는 스프링의 CGLIB에는 이 제약이 없다.** Spring 4.0부터 Objenesis(스프링에 인라인 repackage됨)로 **생성자를 호출하지 않고** 인스턴스를 만들기 때문. 그래서 생성자 주입(`@RequiredArgsConstructor`)만 쓰는 요즘 스타일에서도 `@Transactional`이 잘 붙는다.
>
> 부수 효과가 하나 있다 — **프록시 인스턴스는 생성자가 실행되지 않았으므로 필드가 전부 null**이다. 프록시는 어차피 target에 위임만 하니 문제가 없지만, "프록시 객체 자체의 필드를 들여다보면 비어 있다"는 사실은 디버깅할 때 알고 있어야 한다.

> 🔴 **`final` 메서드가 조용히 실패하는 것**이 실무에서 제일 위험하다. `@Transactional`을 붙였는데 예외도 경고도 없이 트랜잭션이 안 걸린다. Kotlin은 `final`이 기본값이라 이 사고가 더 잦다 (그래서 `kotlin-spring` 플러그인이 `open`을 자동으로 붙여준다). 근거: [spring-framework #26729](https://github.com/spring-projects/spring-framework/issues/26729)

---

## 4. JDK 동적 프록시 vs CGLIB 비교

| | **JDK 동적 프록시** | **CGLIB** |
|---|---|---|
| 출처 | 자바 표준 (`java.lang.reflect`) | 외부 라이브러리 (스프링에 repackage 내장) |
| 프록시 생성 방식 | 인터페이스를 **`implements`** | 구체 클래스를 **`extends`** |
| 부가 로직 인터페이스 | `InvocationHandler` | `MethodInterceptor` |
| 실행 메서드 | `invoke(proxy, method, args)` | `intercept(obj, method, args, methodProxy)` |
| 생성 API | `Proxy.newProxyInstance()` | `Enhancer` |
| 원본 호출 | `method.invoke(target, args)` | `methodProxy.invoke(target, args)` |
| 인터페이스 필요? | **필수** | 불필요 |
| 제약 | 인터페이스 없으면 불가 | `final` 클래스/메서드 |
| 프록시 클래스명 | `$Proxy1` / `jdk.proxy1.$Proxy1` | `Xxx$$EnhancerByCGLIB$$1a2b3c` (순수)<br>`Xxx$$SpringCGLIB$$0` (Spring 6+) |
| 호출 성능 | 리플렉션 | FastClass로 리플렉션 회피 |

### ⚠️ 스프링이 실제로 무엇을 고르는가 — 강의와 현실의 차이

강의는 "인터페이스가 있으면 JDK, 없으면 CGLIB"으로 정리한다. **Spring Framework 단독은 그렇지만, Spring Boot는 다르다.**

| 환경 | 기본 동작 |
|---|---|
| Spring Framework (순수) | 인터페이스 **있으면 JDK**, 없으면 CGLIB |
| **Spring Boot 2.0+** | `spring.aop.proxy-target-class=true`가 기본 → **인터페이스가 있어도 CGLIB** |

즉 **실무에서 `getClass()`를 찍으면 대부분 CGLIB 프록시가 나온다.**

| 버전 | 프록시 클래스명 |
|---|---|
| Spring 5 이하 (Boot 2.x) | `OrderService$$EnhancerBySpringCGLIB$$1a2b3c4d` |
| **Spring 6+ (Boot 3.x)** | `OrderService$$SpringCGLIB$$0` |

부트가 CGLIB을 기본으로 돌린 이유는 JDK 프록시가 만드는 사고 때문 — 인터페이스로만 캐스팅되므로 **구체 클래스 타입으로 주입받으면 실패**한다.

```
Bean named 'orderService' is expected to be of type 'OrderServiceImpl'
but was actually of type 'jdk.proxy2.$Proxy61'
```

JDK 프록시로 강제하고 싶으면 `spring.aop.proxy-target-class=false` 또는 `@EnableAspectJAutoProxy(proxyTargetClass = false)`.

> 💭 **여기가 6장으로 넘어가는 지점.** 두 기술은 인터페이스(`InvocationHandler` / `MethodInterceptor`)부터 다르다. 대상에 인터페이스가 있냐 없냐에 따라 **부가 로직 클래스를 두 벌 만들어야** 하고, 설정 코드도 `if`로 갈라진다. → **6장 `ProxyFactory`가 이 둘을 하나의 `Advice`로 추상화**한다. (그리고 `Advice`의 표준 인터페이스 이름이 하필 `MethodInterceptor` — CGLIB 것과 이름만 같은 다른 인터페이스라 헷갈리기 쉬우니 6장에서 주의)

---

## 5. ⚠️ 함정 / 헷갈렸던 것

### 5-1. 🔴 `@Transactional`이 바로 이걸로 동작한다 — 세션에서 몰랐던 것

> ❓ **내 질문**: "이걸 실제로 내가 쓸 일이 있나? 사용법까지 익숙해져야 하나?"
> ✅ **답**: `Proxy.newProxyInstance`나 `Enhancer`를 직접 칠 일은 **거의 없다.** 하지만 **`@Transactional`·`@Cacheable`·`@Async`·`@PreAuthorize`가 전부 이 기술로 동작**한다. 어노테이션이 마법이 아니라, **스프링이 CGLIB으로 프록시를 만들어 빈으로 등록**하는 것.

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;

    @Transactional
    public void createOrder(OrderRequest request) {
        orderRepository.save(request);
    }
}
```

스프링이 만드는 프록시를 개념적으로 풀면:

```java
class OrderService$$SpringCGLIB$$0 extends OrderService {

    private OrderService target;                        // 진짜 빈
    private PlatformTransactionManager txManager;

    @Override
    public void createOrder(OrderRequest request) {
        TransactionStatus status = txManager.getTransaction(new DefaultTransactionDefinition());
        try {
            target.createOrder(request);                // 진짜 메서드
            txManager.commit(status);                   // 성공 → 커밋
        } catch (RuntimeException | Error e) {          // ⚠️ 기본 롤백 규칙은 이 둘뿐
            txManager.rollback(status);
            throw e;
        } catch (Exception e) {                         // checked 예외는 기본적으로
            txManager.commit(status);                   // ⚠️ 롤백하지 않고 커밋한다!
            throw e;                                    // → rollbackFor = Exception.class 필요
        }
    }
}
```

직접 확인하는 법:

```java
@Autowired OrderService orderService;

@PostConstruct
void check() {
    log.info("{}", orderService.getClass());
    // class com.example.OrderService$$SpringCGLIB$$0   (Spring 6+)
    // Spring 5라면 OrderService$$EnhancerBySpringCGLIB$$1a2b3c4d
    //   → @Autowired로 주입받은 건 원본이 아니라 프록시다
    log.info("{}", AopUtils.isAopProxy(orderService));     // true
    log.info("{}", AopUtils.isCglibProxy(orderService));   // true
    log.info("{}", AopUtils.isJdkDynamicProxy(orderService)); // false
}
```

### 5-2. 🔴 `this.method()` 자기호출이 안 먹는 이유 — `this`는 프록시가 아니다

> ❓ **내 추측**: "this가 가리키는 건 프록시겠지?"
> ✅ **정답**: `this`는 **진짜 객체(target)** 다. 프록시가 target의 메서드를 호출한 순간부터 코드는 **target 내부에서** 실행되고, 그 안의 `this`는 당연히 target 자신. **프록시를 다시 거치지 않으므로 AOP가 적용되지 않는다.**

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder(OrderRequest request) { ... }

    public void process(OrderRequest request) {
        this.createOrder(request);   // ❌ 트랜잭션 안 걸림 (this 생략해도 동일)
    }
}
```

```
[외부 호출 — 정상 ✅]
Controller → 프록시.createOrder() → [트랜잭션 시작] → target.createOrder()
                 ↑ 프록시를 거침

[자기호출 — 무음 실패 ❌]
Controller → 프록시.process() → target.process() → this.createOrder()
                                                       ↑
                                              여기는 이미 target 내부.
                                              this = target, 프록시는 관여 안 함
```

**핵심은 "프록시는 진입점에서만 한 번 개입한다"**는 것. 한 번 target 안으로 들어가면 그 안의 호출은 프록시가 볼 수 없다.

이 함정이 [@Transactional](../spring/transactional.md) · [Spring Cache](../spring/spring-cache.md) · [메서드 보안](../security/method-security.md) · [커스텀 어노테이션](../annotation/custom-annotation.md) 노트에 **전부 따로 적혀 있는데, 원인은 이 한 가지**다. 해결책(자기 주입 · `AopContext.currentProxy()` · **별도 빈으로 분리**)은 13장에서 다룬다.

> 💡 실무 감별법: **"어노테이션을 붙였는데 조용히 아무 일도 안 일어난다"** → ① `@Enable~`을 안 했거나 ② 자기호출이거나 ③ `final` 메서드거나 ④ `private` 메서드. 넷 다 **프록시가 개입할 자리가 없는** 경우다.

### 5-3. ⚠️ `equals`·`hashCode`·`toString`도 핸들러로 들어온다

JDK 동적 프록시는 `Object`가 선언한 **`equals`·`hashCode`·`toString` 세 개도 `invoke()`로 디스패치**한다 (`getClass()`·`wait()`·`notify()`는 아님).

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    log.info("[LOG] {} 시작", method.getName());  // toString()에도 로그가 찍힌다!
    return method.invoke(target, args);
}
```

- 디버거가 변수를 표시하려고 `toString()`을 부르는 순간에도 부가 로직이 돈다 → **로그가 오염**된다.
- `HashMap`의 키로 넣으면 `hashCode()`가 target에 위임되어 **프록시와 target이 같은 버킷**에 들어간다.
- 핸들러가 `target`을 null로 두고 있는 상태에서 `toString()`이 불리면 **NPE가 엉뚱한 곳에서** 터진다.

```java
// 방어: Object 메서드는 부가 기능 없이 통과시킨다
if (method.getDeclaringClass() == Object.class) {
    return method.invoke(target, args);
}
```

CGLIB도 `Object` 메서드가 `intercept()`로 들어온다. 스프링 AOP는 내부적으로 이런 처리를 이미 해 두었다.

### 5-4. ⚠️ 프록시는 공짜가 아니다

- **클래스가 늘어난다**: 대상 1개당 프록시 클래스 1개 + (CGLIB이면) FastClass 2개(target용 f1 / 프록시용 f2)가 메타스페이스에 올라간다. 빈이 수천 개면 기동 시간과 메모리에 영향. (네이티브 이미지에서 CGLIB이 문제가 되는 것도 이 런타임 생성 때문 — AOT로 미리 찍어두어야 한다)
- **호출 스택이 깊어진다**: 스택 트레이스에 `$Proxy`·`CglibAopProxy`·`ReflectiveMethodInvocation` 같은 프레임이 잔뜩 낀다. **예외를 읽을 때 이 프레임들은 건너뛰고 읽는다**는 감각이 필요.
- **리플렉션 호출 비용**: `method.invoke()`는 직접 호출보다 느리다. CGLIB의 FastClass, 스프링의 캐싱이 이를 줄이지만 0은 아니다.

### 5-5. ⚠️ JDK 프록시는 인터페이스에 없는 메서드를 잃는다

`Proxy.newProxyInstance`에 넘긴 인터페이스의 메서드만 프록시에 존재한다. 구현체에만 있는 public 메서드는 **호출할 방법이 없다**(캐스팅 시 `ClassCastException`). 인터페이스를 여러 개 구현했는데 배열에 하나만 넘겨도 마찬가지. 부트가 CGLIB을 기본으로 잡은 이유 중 하나.

### 5-6. 💡 `@Configuration`도 CGLIB 프록시다 — 이 노트의 설정 코드가 동작하는 이유

앞의 `DynamicProxyBasicConfig`를 다시 보면 이상한 게 있다.

```java
@Bean
public OrderControllerV1 orderControllerV1(LogTrace logTrace) {
    // orderServiceV1()을 직접 호출한다 — 새 인스턴스가 만들어지는 거 아닌가?
    new OrderControllerV1Impl(orderServiceV1(logTrace));
}
```

일반 자바라면 `orderServiceV1()`을 두 번 부르면 객체가 두 개 생긴다. 그런데 싱글톤이 깨지지 않는다. **`@Configuration` 클래스 자체가 CGLIB 프록시**로 바꿔치기 되어, `@Bean` 메서드 호출을 가로채고 "이미 컨테이너에 있으면 그걸 리턴"하기 때문이다.

```java
@Configuration
public class AppConfig {
    @Bean public A a() { return new A(b()); }
    @Bean public B b() { return new B(); }   // 두 번 불려도 인스턴스는 하나
}

// 확인
System.out.println(appConfig.getClass());
// class AppConfig$$SpringCGLIB$$0   ← 설정 클래스도 프록시!
```

⚠️ **`@Configuration(proxyBeanMethods = false)`를 주면 이 프록시를 안 만든다** — 기동은 빨라지지만 `@Bean` 메서드를 직접 호출하는 순간 **새 인스턴스가 생긴다**. 스프링 부트의 자동구성 클래스들이 이 옵션을 쓰는데, 그래서 거기서는 메서드 직호출 대신 **파라미터 주입**을 쓴다.

> 📌 `final` 클래스에 `@Configuration`을 못 붙이는 것도 같은 이유. 상속을 못 하니 프록시를 못 만든다.

---

## 6. 💡 판단 기준

- **API 사용법은 잊어도 된다. 남길 것은 "스프링의 어노테이션 = 프록시"라는 한 문장.** 2026년에 `Proxy.newProxyInstance`를 손으로 칠 일은 프레임워크를 만들 때뿐이다. 하지만 `@Transactional`이 안 먹을 때 **"프록시가 개입할 자리가 있나?"**를 5초 안에 떠올릴 수 있느냐가 실무 디버깅 속도를 가른다 — 자기호출·`final`·`private`·`@Enable~` 누락이 전부 같은 질문의 변주.
- **"인터페이스가 있으면 JDK"는 강의용 정리다. 실무 기본값은 CGLIB.** Spring Boot 2.0부터 `proxy-target-class=true`가 기본이라, 인터페이스가 있어도 CGLIB 프록시가 만들어진다. 그래서 **`final`을 붙일 때 "여기 `@Transactional`이 붙을 수 있나"를 한 번 물어야** 한다 — JDK 프록시라면 상관없을 `final`이 CGLIB에서는 무음 실패를 만든다.
- **`getClass()`를 찍어보는 습관.** "이 빈이 프록시인가"는 로그 한 줄로 끝난다(`AopUtils.isAopProxy()`). 트랜잭션·캐시·보안이 안 먹는 상황에서 **원인을 코드에서 찾기 전에 객체 정체부터 확인**하는 게 훨씬 빠르다.
- **디버깅할 때 `$Proxy`·`$$SpringCGLIB`·`$$EnhancerBySpringCGLIB`·`ReflectiveMethodInvocation`이 보이면 "여긴 AOP 구간"으로 읽고 넘긴다.** 이 프레임들을 자기 코드로 착각해 파고들면 시간만 버린다. 진짜 원인은 그 아래(target) 또는 위(호출자)에 있다.
- **적용 대상 판단(Pointcut)과 부가 기능(Advice)이 한 클래스에 섞이면 확장할 때 복붙이 시작된다.** `LogTraceFilterHandler`가 `patterns`와 `logTrace`를 같이 들고 있는 게 그 신호였다. 직접 프록시 인프라를 짤 일은 없지만, **"판단 로직과 실행 로직이 한 덩어리인가"는 일반적인 설계 냄새**다 — 같은 냄새가 [계산 파이프라인](./calculation-pipeline.md)의 `supports`/`execute` 분리에도 나온다.

---

## 참고

- 김영한, 스프링 핵심 원리 고급편 — Ch.5 동적 프록시 기술
- [Java SE Docs — Dynamic Proxy Classes](https://docs.oracle.com/javase/8/docs/technotes/guides/reflection/proxy.html) (`equals`·`hashCode`·`toString`이 핸들러로 디스패치되는 근거)
- [Spring Framework Reference — Proxying Mechanisms](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html) (JDK/CGLIB 선택 규칙, Objenesis로 생성자 미호출)
- [Spring 4: CGLIB-based proxy classes with no default constructor](https://blog.codeleak.pl/2014/07/spring-4-cglib-based-proxy-classes-with-no-default-ctor.html) (기본 생성자 제약이 사라진 경위)
- [spring-boot #12194](https://github.com/spring-projects/spring-boot/issues/12194) (Spring Boot 2.0부터 `spring.aop.proxy-target-class` 기본값이 true)
- [spring-framework #26729](https://github.com/spring-projects/spring-framework/issues/26729) (`final` 메서드가 **조용히** 제외되는 근거)
- [cglib: The missing manual](http://mydailyjava.blogspot.com/2013/11/cglib-missing-manual.html) (`MethodProxy`·FastClass·`invokeSuper` 동작)
- [spring-framework #31272](https://github.com/spring-projects/spring-framework/issues/31272) (Spring 6 프록시 클래스명이 `$$SpringCGLIB$$N` 형태임을 보여주는 이슈)
- Effective Java Item 65 — 리플렉션보다는 인터페이스를 사용하라 (§1 리플렉션 경고의 출처)

**학습 날짜**: 2026-08-14 · **계기**: 김영한 고급편 Ch.5 수강 후 "거의 이해를 못 한 것 같다"며 Claude와 처음부터 재구성한 세션. 틀리거나 몰랐던 것 4가지 — ① `Method method`를 "실행될 구현체의 메서드"로 이해(→ 메타정보 객체) ② 인터페이스 필수인 이유를 "인터페이스 기반이라서"로만 답함(→ `implements`할 클래스를 런타임 생성하므로 메서드 목록의 출처가 필요) ③ **`@Transactional`이 이 기술로 동작한다는 사실 자체를 몰랐음**("이걸 실제로 쓸 일이 있나?"라는 질문에서 출발) ④ 자기호출 시 `this`를 프록시로 착각(→ target). ②④는 질문을 이어가며 스스로 도달함

# ProxyFactory — 동적 프록시 추상화 + Pointcut / Advice / Advisor

> **한 줄 요약**: 5장에서 JDK 동적 프록시와 CGLIB를 직접 다뤘더니 **① 기술 선택을 개발자가 판단** ② **같은 로직을 핸들러 두 벌로 작성** ③ **부가 기능과 적용 판단이 한 클래스에 혼재**하는 문제가 있었다. `ProxyFactory`는 이 세 문제를 각각 **자동 기술 선택 / `Advice` 추상화 / `Pointcut` 분리**로 해결한다. ⚠️ ProxyFactory로도 해결 안 되는 **설정 지옥(빈마다 프록시 코드 작성)과 컴포넌트 스캔 미지원** 문제가 남고, 이것이 빈 후처리기(Ch.7)로 이어진다.

관련 노트: [동적 프록시](./dynamic-proxy.md) (직전 챕터 — "InvocationHandler/MethodInterceptor를 각각 만들어야 한다"로 끝난 자리가 이 노트의 출발점) · [프록시/데코레이터 패턴](./proxy-decorator-pattern.md) (챕터 4 — 데코레이터 체이닝이 "여러 프록시" 방식) · [ThreadLocal](../concurrency/thread-local.md) (`LogTrace`의 출처) · [템플릿 메서드/전략/콜백](./template-method-strategy-callback.md)

---

## 0. 출발점 — 동적 프록시의 3가지 한계

[직전 챕터](./dynamic-proxy.md)에서 동적 프록시로 "프록시 클래스 폭발"은 해결했다. 그런데 **실무에 적용하려니** 세 가지가 불편했다:

| # | 문제 | 핵심 |
|---|------|------|
| ① | 인터페이스가 있으면 JDK, 없으면 CGLIB — **개발자가 판단해서 골라야** | 기술 선택 |
| ② | 시간 측정 같은 같은 로직인데 `InvocationHandler`용 + `MethodInterceptor`용 **두 벌** 필요 | 중복 |
| ③ | `InvocationHandler.invoke()` 안에 `PatternMatchUtils`로 **적용 여부까지 판단** — 부가 기능과 필터링이 한 곳에 혼재 | 책임 미분리 |

스프링은 이 세 문제를 각각 **ProxyFactory** / **Advice** / **Pointcut**이라는 개념으로 풀었다.

---

## 1. ProxyFactory — 기술 선택 자동화

```java
// target만 넘기면 ProxyFactory가 알아서 JDK or CGLIB 선택
ServiceInterface target = new ServiceImpl();
ProxyFactory proxyFactory = new ProxyFactory(target);
proxyFactory.addAdvice(new TimeAdvice());
ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy();
```

### 기술 선택 규칙

| 조건 | 선택 |
|------|------|
| 대상에 **인터페이스가 있으면** | JDK 동적 프록시 |
| **인터페이스가 없으면** (구체 클래스만) | CGLIB |
| `proxyTargetClass=true` 설정 시 | **무조건 CGLIB** (인터페이스 여부 무관) |

```java
// 인터페이스가 있어도 CGLIB 강제
ProxyFactory proxyFactory = new ProxyFactory(target);
proxyFactory.setProxyTargetClass(true);   // ← 이 한 줄
proxyFactory.addAdvice(new TimeAdvice());
```

### 생성된 프록시 확인 — `AopUtils`

어떤 기술이 선택됐는지 테스트로 검증할 수 있다:

```java
assertThat(AopUtils.isAopProxy(proxy)).isTrue();
assertThat(AopUtils.isJdkDynamicProxy(proxy)).isTrue();
assertThat(AopUtils.isCglibProxy(proxy)).isFalse();
```

실행 결과로도 구분된다:

```
# 인터페이스 있음 → JDK
proxyClass=class com.sun.proxy.$Proxy13

# 구체 클래스만 → CGLIB
proxyClass=class hello.proxy.common.service.ConcreteService$$EnhancerBySpringCGLIB$$2bbf51ab
```

> 참고: JDK 프록시 클래스명은 Java 8까지 `com.sun.proxy.$ProxyN`, Java 9+ 모듈 환경에서는 `jdk.proxy2.$ProxyN` 형태로 나올 수 있다.

> **주의**: Spring Boot는 기본적으로 `proxyTargetClass=true`로 설정되어 있어서, 인터페이스가 있어도 **항상 CGLIB**를 사용한다. 이유는 바로 아래.

### ⚠️ 왜 스프링 부트는 CGLIB를 기본으로 바꿨나 — 캐스팅 문제

JDK 동적 프록시는 **인터페이스만 구현**하기 때문에 구체 클래스로 캐스팅하면 터진다:

```java
// target: ServiceImpl implements ServiceInterface
ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy();   // ✅ OK
ServiceImpl impl = (ServiceImpl) proxy;   // ❌ ClassCastException!
```

프록시는 `ServiceInterface`를 구현한 별도의 클래스일 뿐, `ServiceImpl`을 상속한 게 아니다. 이것이 의존관계 주입에서도 동일하게 문제가 된다 — 구체 클래스 타입으로 주입받으려면 실패한다.

CGLIB는 **구체 클래스를 상속**해서 프록시를 만들기 때문에 둘 다 된다. 그래서 스프링 부트는 일관성을 위해 CGLIB를 기본으로 선택했다. (자세한 내용은 Ch.13 실무 주의사항)

### ⚠️ 내부 구조 — Advice가 호출되는 원리

개발자는 `Advice` 하나만 만들면 되지만, 실제로 JDK 동적 프록시는 `InvocationHandler`를, CGLIB는 `MethodInterceptor`를 필요로 한다. ProxyFactory가 이걸 어떻게 해결하나?

→ **내부에서 Advice를 호출하는 전용 InvocationHandler / MethodInterceptor를 자동 생성**해준다.

```
[JDK 동적 프록시 선택 시]
클라이언트 → JDK 프록시 → InvocationHandler(자동생성) → Advice → target

[CGLIB 선택 시]
클라이언트 → CGLIB 프록시 → MethodInterceptor(자동생성) → Advice → target
```

이것이 스프링의 전형적인 **서비스 추상화** 패턴이다 — 구체 기술(JDK/CGLIB)에 의존하지 않고, 추상화된 개념(`Advice`)만 사용하면 내부에서 알아서 연결한다.

---

## 2. Advice — 부가 기능 로직 추상화

`Advice`를 만들려면 `org.aopalliance.intercept.MethodInterceptor`를 구현한다.

> ⚠️ CGLIB의 `org.springframework.cglib.proxy.MethodInterceptor`와 **이름이 같다**. 패키지에 주의.

```java
package hello.proxy.common.advice;

import lombok.extern.slf4j.Slf4j;
import org.aopalliance.intercept.MethodInterceptor;  // ← aopalliance 패키지!
import org.aopalliance.intercept.MethodInvocation;

@Slf4j
public class TimeAdvice implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        log.info("TimeProxy 실행");
        long startTime = System.currentTimeMillis();

        Object result = invocation.proceed();  // target 호출

        long endTime = System.currentTimeMillis();
        long resultTime = endTime - startTime;
        log.info("TimeProxy 종료 resultTime={}ms", resultTime);
        return result;
    }
}
```

### 이전과 달라진 점

- **target 정보가 코드에 없다.** `MethodInvocation invocation` 안에 target, args, 메서드 정보가 모두 들어 있다. ProxyFactory를 생성할 때 target을 넘기기 때문에 Advice는 target을 몰라도 된다.
- **`invocation.proceed()`** 한 줄로 target을 호출한다. JDK 동적 프록시의 `method.invoke(target, args)`나 CGLIB의 `methodProxy.invoke(target, args)`를 직접 쓸 필요 없다.

### 상속 구조

```
Advice (최상위)
  └── Interceptor
        └── MethodInterceptor (org.aopalliance.intercept)
```

`MethodInterceptor`는 `Interceptor`를 상속하고, `Interceptor`는 `Advice` 인터페이스를 상속한다.

### Advice의 다른 구현체들

`MethodInterceptor`가 가장 많이 쓰이지만, 특정 시점만 필요하면 더 좁은 인터페이스도 있다:

| 인터페이스 | 시점 |
|---|---|
| `MethodInterceptor` | **전후 전부** (`proceed()` 앞뒤) — 가장 강력 |
| `MethodBeforeAdvice` | 대상 호출 **전**만 |
| `AfterReturningAdvice` | 정상 반환 **후**만 |
| `ThrowsAdvice` | 예외 발생 시만 |

실무에서는 뒤 챕터의 `@Around`, `@Before`, `@AfterReturning` 등 애노테이션 방식을 쓰게 되지만, 그 아래에서 동작하는 것이 이 인터페이스들이다.

### ⚠️ `proceed()`를 안 부르면 target은 실행되지 않는다

```java
@Override
public Object invoke(MethodInvocation invocation) throws Throwable {
    if (!hasPermission()) {
        throw new AccessDeniedException();   // proceed() 호출 안 함 → target 실행 안 됨
    }
    return invocation.proceed();
}
```

이것이 [챕터 4](./proxy-decorator-pattern.md)에서 본 **접근 제어 프록시**의 정체다. `@PreAuthorize`가 권한을 막는 방식이고, 캐시 히트 시 `@Cacheable`이 메서드를 건너뛰는 방식도 같다 — 캐시된 값을 반환하고 `proceed()`를 호출하지 않는다.

---

## 3. Pointcut, Advice, Advisor — 역할 분리

### 정의

| 개념 | 역할 | 비유 |
|------|------|------|
| **Advice** | 프록시가 호출하는 **부가 기능 로직** | 조언(Advice) — "무엇을" |
| **Pointcut** | 어디에 적용할지 판단하는 **필터** | 어디(Pointcut) — "어디에" |
| **Advisor** | Pointcut 1개 + Advice 1개 묶음 | 조언자(Advisor) — "어디에 무엇을" |

> 쉽게 외우기: **조언(Advice)을 어디(Pointcut)에 할 것인가? 조언자(Advisor)는 어디에 조언을 해야할지 알고 있다.**

### Advisor가 왜 필요한가

여러 부가 기능을 적용할 때, 어떤 Advice가 어떤 Pointcut과 짝인지를 보장하는 단위가 필요하다.

```
Advisor1 = 모든 메서드 Pointcut + 로그 Advice
Advisor2 = save()만 Pointcut   + 보안 Advice
```

Pointcut과 Advice가 따로 놀면 매칭이 불가능하다. **Advisor는 짝을 보장하는 단위**.

### 기본 사용

```java
ServiceInterface target = new ServiceImpl();
ProxyFactory proxyFactory = new ProxyFactory(target);

// Advisor = Pointcut + Advice
DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(Pointcut.TRUE, new TimeAdvice());
proxyFactory.addAdvisor(advisor);

ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy();
proxy.save();   // TimeAdvice 적용
proxy.find();   // TimeAdvice 적용 (Pointcut.TRUE이므로 전부)
```

### addAdvice() 편의 메서드

```java
// 이렇게 Advice만 넘겨도
proxyFactory.addAdvice(new TimeAdvice());

// 내부에서 이렇게 Advisor를 자동 생성
new DefaultPointcutAdvisor(Pointcut.TRUE, new TimeAdvice());
```

**Advisor가 필수**이지만, 편의 메서드가 내부에서 `Pointcut.TRUE`를 끼워서 만들어준다.

---

## 4. Pointcut 인터페이스 구조

```java
public interface Pointcut {
    ClassFilter getClassFilter();      // 클래스가 맞는지
    MethodMatcher getMethodMatcher();  // 메서드가 맞는지
}

public interface ClassFilter {
    boolean matches(Class<?> clazz);
}

public interface MethodMatcher {
    boolean matches(Method method, Class<?> targetClass);          // 정적 매칭
    boolean isRuntime();                                          // 동적 매칭 여부
    boolean matches(Method method, Class<?> targetClass, Object... args);  // 동적 매칭
}
```

**ClassFilter와 MethodMatcher 둘 다 true**를 반환해야 Advice가 적용된다.

### ⚠️ 정적 매칭 vs 동적 매칭 (`isRuntime()`)

| | `isRuntime() == false` (정적) | `isRuntime() == true` (동적) |
|---|---|---|
| 호출되는 메서드 | 2인자 `matches(method, targetClass)` | 3인자 `matches(method, targetClass, args)` |
| 판단 재료 | 클래스/메서드 **메타정보만** | 메타정보 + **실제 인자 값** |
| 평가 시점 | 프록시 생성 시 한 번 → **캐싱** | 매 호출마다 |
| 성능 | 빠름 | 느림 |

인자 값을 보지 않고도 판단되는 포인트컷은 `isRuntime()`을 `false`로 둔다. 그러면 스프링이 결과를 캐싱해서 매 호출마다 다시 평가하지 않는다. 대부분의 포인트컷은 여기 해당한다.

> 💡 뒤 챕터의 AspectJ 포인트컷 지시자 중 `args`, `@args`, `this`, `target`이 **동적 매칭**에 해당한다. 그것들이 단독으로 쓰이기 어려운 이유가 여기서 출발한다 — 매 호출마다 모든 빈의 모든 메서드를 평가해야 하므로.

### 직접 만든 Pointcut 예시

```java
static class MyPointcut implements Pointcut {
    @Override
    public ClassFilter getClassFilter() {
        return ClassFilter.TRUE;   // 클래스 필터는 항상 통과
    }

    @Override
    public MethodMatcher getMethodMatcher() {
        return new MyMethodMatcher();
    }
}

static class MyMethodMatcher implements MethodMatcher {
    private String matchName = "save";

    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        boolean result = method.getName().equals(matchName);
        log.info("포인트컷 호출 method={} targetClass={}", method.getName(), targetClass);
        log.info("포인트컷 결과 result={}", result);
        return result;
    }

    @Override
    public boolean isRuntime() {
        return false;   // false면 정적 매칭 (캐싱 가능)
    }

    @Override
    public boolean matches(Method method, Class<?> targetClass, Object... args) {
        throw new UnsupportedOperationException();
    }
}
```

### 스프링이 제공하는 포인트컷

| Pointcut | 설명 |
|----------|------|
| `NameMatchMethodPointcut` | 메서드 이름 기반 (`*xxx*` 패턴 지원) |
| `JdkRegexpMethodPointcut` | JDK 정규 표현식 기반 |
| `AnnotationMatchingPointcut` | 애노테이션 기반 |
| **`AspectJExpressionPointcut`** | **실무에서 가장 많이 사용** — AspectJ 표현식 |
| `TruePointcut` | 항상 true |

> 실무에서는 거의 `AspectJExpressionPointcut`만 쓴다. 나머지는 "이런 게 있다" 수준.

### NameMatchMethodPointcut 사용 예시

```java
NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
pointcut.setMappedNames("save");   // save 메서드만 매칭

DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, new TimeAdvice());
proxyFactory.addAdvisor(advisor);

ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy();
proxy.save();   // TimeAdvice 적용됨
proxy.find();   // TimeAdvice 적용 안됨 — Pointcut이 false
```

---

## 5. 여러 Advisor 적용 — 프록시는 몇 개?

하나의 target에 여러 부가 기능을 적용하려면? 떠오르는 방법은 **프록시를 여러 개 만드는 것**이다.

### 방법 1 — 여러 프록시 (비효율적)

```java
@Test
@DisplayName("여러 프록시")
void multiAdvisorTest1() {
    //client -> proxy2(advisor2) -> proxy1(advisor1) -> target

    //프록시1 생성
    ServiceInterface target = new ServiceImpl();
    ProxyFactory proxyFactory1 = new ProxyFactory(target);
    DefaultPointcutAdvisor advisor1 = new DefaultPointcutAdvisor(Pointcut.TRUE, new Advice1());
    proxyFactory1.addAdvisor(advisor1);
    ServiceInterface proxy1 = (ServiceInterface) proxyFactory1.getProxy();

    //프록시2 생성, target -> proxy1 입력
    ProxyFactory proxyFactory2 = new ProxyFactory(proxy1);   // ← target 자리에 proxy1
    DefaultPointcutAdvisor advisor2 = new DefaultPointcutAdvisor(Pointcut.TRUE, new Advice2());
    proxyFactory2.addAdvisor(advisor2);
    ServiceInterface proxy2 = (ServiceInterface) proxyFactory2.getProxy();

    proxy2.save();
}
```

이것이 잘못된 것은 아니지만 **어드바이저가 10개면 프록시도 10개**를 생성해야 한다.

> 📝 **[챕터 4](./proxy-decorator-pattern.md)의 데코레이터 체이닝과 같은 구조**다. 거기서는 프록시 클래스를 손으로 만들어 `target` 자리에 다른 프록시를 넣었다:
> ```java
> RealComponent real = new RealComponent();
> TimeDecorator timeDecorator = new TimeDecorator(real);
> MessageDecorator messageDecorator = new MessageDecorator(timeDecorator);
> // messageDecorator → timeDecorator → real
> ```
> 만드는 방법(수작업 vs ProxyFactory)만 다를 뿐 **"프록시를 겹겹으로 쌓는다"**는 발상은 동일하다.

### 방법 2 — ⚠️ 하나의 프록시, 여러 어드바이저 (스프링 방식)

스프링은 **하나의 프록시에 여러 어드바이저를 리스트로** 담을 수 있게 만들어두었다.

```java
@Test
@DisplayName("하나의 프록시, 여러 어드바이저")
void multiAdvisorTest2() {
    //proxy -> advisor2 -> advisor1 -> target
    DefaultPointcutAdvisor advisor2 = new DefaultPointcutAdvisor(Pointcut.TRUE, new Advice2());
    DefaultPointcutAdvisor advisor1 = new DefaultPointcutAdvisor(Pointcut.TRUE, new Advice1());

    ServiceInterface target = new ServiceImpl();
    ProxyFactory proxyFactory1 = new ProxyFactory(target);
    proxyFactory1.addAdvisor(advisor2);   // 먼저 등록 → 먼저 실행
    proxyFactory1.addAdvisor(advisor1);
    ServiceInterface proxy = (ServiceInterface) proxyFactory1.getProxy();

    proxy.save();
}

@Slf4j
static class Advice1 implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        log.info("advice1 호출");
        return invocation.proceed();
    }
}

@Slf4j
static class Advice2 implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        log.info("advice2 호출");
        return invocation.proceed();
    }
}
```

실행 결과:
```
MultiAdvisorTest$Advice2 - advice2 호출
MultiAdvisorTest$Advice1 - advice1 호출
ServiceImpl - save 호출
```

**결과는 같고 성능은 더 좋다.** `addAdvisor()` **등록 순서대로** advisor가 호출된다.

> ⚠️ **순서 제어 방식이 상황마다 다르다.** 직접 `ProxyFactory`를 쓸 땐 **등록 순서**가 실행 순서지만, 뒤 챕터의 자동 프록시 생성기를 쓰면 Advisor 빈의 등록 순서를 신뢰할 수 없어서 `@Order` / `Ordered` 인터페이스로 명시해야 한다.

> 💡 이 "프록시 1개 + Advisor N개" 구조가 다음 챕터의 자동 프록시 생성기에서 그대로 쓰인다. 빈 하나가 advisor1, advisor2의 포인트컷을 모두 만족해도 **프록시는 1개**만 생성되고 그 안에 둘 다 들어간다.

---

## 6. 실전 적용 — V1(인터페이스), V2(구체 클래스)

### LogTraceAdvice

```java
@Slf4j
public class LogTraceAdvice implements MethodInterceptor {
    private final LogTrace logTrace;

    public LogTraceAdvice(LogTrace logTrace) {
        this.logTrace = logTrace;
    }

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        TraceStatus status = null;
        try {
            Method method = invocation.getMethod();
            String message = method.getDeclaringClass().getSimpleName() + "."
                    + method.getName() + "()";
            status = logTrace.begin(message);

            Object result = invocation.proceed();  // target 호출

            logTrace.end(status);
            return result;
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }
}
```

### V1 Config (인터페이스 → JDK 동적 프록시)

```java
@Slf4j
@Configuration
public class ProxyFactoryConfigV1 {

    @Bean
    public OrderControllerV1 orderControllerV1(LogTrace logTrace) {
        OrderControllerV1Impl orderController = new OrderControllerV1Impl(orderServiceV1(logTrace));
        ProxyFactory factory = new ProxyFactory(orderController);
        factory.addAdvisor(getAdvisor(logTrace));
        OrderControllerV1 proxy = (OrderControllerV1) factory.getProxy();
        log.info("ProxyFactory proxy={}, target={}", proxy.getClass(), orderController.getClass());
        return proxy;
    }

    // orderServiceV1, orderRepositoryV1도 동일 패턴
    @Bean
    public OrderServiceV1 orderServiceV1(LogTrace logTrace) {
        OrderServiceV1Impl orderService = new OrderServiceV1Impl(orderRepositoryV1(logTrace));
        ProxyFactory factory = new ProxyFactory(orderService);
        factory.addAdvisor(getAdvisor(logTrace));
        OrderServiceV1 proxy = (OrderServiceV1) factory.getProxy();
        log.info("ProxyFactory proxy={}, target={}", proxy.getClass(), orderService.getClass());
        return proxy;
    }

    @Bean
    public OrderRepositoryV1 orderRepositoryV1(LogTrace logTrace) {
        OrderRepositoryV1Impl orderRepository = new OrderRepositoryV1Impl();
        ProxyFactory factory = new ProxyFactory(orderRepository);
        factory.addAdvisor(getAdvisor(logTrace));
        OrderRepositoryV1 proxy = (OrderRepositoryV1) factory.getProxy();
        log.info("ProxyFactory proxy={}, target={}", proxy.getClass(), orderRepository.getClass());
        return proxy;
    }

    private Advisor getAdvisor(LogTrace logTrace) {
        // pointcut
        NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
        pointcut.setMappedNames("request*", "order*", "save*");   // noLog()를 제외하기 위함
        // advice
        LogTraceAdvice advice = new LogTraceAdvice(logTrace);
        // advisor = pointcut + advice
        return new DefaultPointcutAdvisor(pointcut, advice);
    }
}
```

### 실행 결과 — V1 (JDK 동적 프록시)

```
ProxyFactory proxy=class com.sun.proxy.$Proxy50, target=class ...OrderRepositoryV1Impl
ProxyFactory proxy=class com.sun.proxy.$Proxy52, target=class ...OrderServiceV1Impl
ProxyFactory proxy=class com.sun.proxy.$Proxy53, target=class ...OrderControllerV1Impl
```

### 실행 결과 — V2 (CGLIB)

V2는 인터페이스 없이 구체 클래스만 있으므로 CGLIB이 선택된다:

```
ProxyFactory proxy=class ...OrderRepositoryV2$$EnhancerBySpringCGLIB$$594e4e8
ProxyFactory proxy=class ...OrderServiceV2$$EnhancerBySpringCGLIB$$59e5130b
ProxyFactory proxy=class ...OrderControllerV2$$EnhancerBySpringCGLIB$$79c0b9e
```

---

## 7. 아직 남은 문제 → 빈 후처리기(Ch.7)로 연결

ProxyFactory + Advisor로 많은 문제가 해결됐지만 **두 가지가 남았다**:

| 문제 | 설명 | 다음 챕터 |
|------|------|----------|
| **설정 지옥** | 빈 100개면 프록시 생성 코드도 100개. `ProxyFactoryConfigV1`, `V2` 같은 설정 파일이 끝없이 늘어남 | 빈 후처리기 |
| **컴포넌트 스캔 미지원** | `@Controller`, `@Service`로 자동 등록된 빈은 **프록시로 바꿔치기할 타이밍이 없음**. `@Bean`으로 직접 등록할 때만 프록시를 끼울 수 있다 | 빈 후처리기 |

→ **빈 후처리기(BeanPostProcessor)**: 스프링이 빈을 저장소에 넣기 **직전에** 가로채서 프록시로 교체한다.

### 챕터 흐름 전체 지도

```
Ch.3  템플릿 메서드 → 변하는/변하지 않는 분리, 하지만 원본 수정 필요
Ch.4  프록시/데코레이터 → 원본 수정 0, 하지만 프록시 클래스 폭발
Ch.5  동적 프록시 → 클래스 폭발 해결, 하지만 JDK/CGLIB 이중 관리 + 책임 혼재
Ch.6  ProxyFactory → 추상화로 해결, 하지만 설정 지옥 + 컴포넌트 스캔
Ch.7  빈 후처리기 → 설정 지옥과 컴포넌트 스캔 한방 해결
```

---

## ⚠️ 함정 / 주의

### 1. `MethodInterceptor` 이름 충돌

- `org.aopalliance.intercept.MethodInterceptor` — **Advice 만들 때 사용** (스프링 AOP)
- `org.springframework.cglib.proxy.MethodInterceptor` — CGLIB용 (직접 쓸 일 거의 없음)

import할 때 패키지를 확인하자.

### 2. Advisor 등록 순서 = 실행 순서

`addAdvisor()`를 호출한 순서대로 Advice가 실행된다. 순서가 중요한 부가 기능(예: 트랜잭션 → 로그)이라면 등록 순서에 주의.

### 3. `addAdvice()`는 편의 메서드

`addAdvice()`를 써도 내부에서 `DefaultPointcutAdvisor(Pointcut.TRUE, advice)`로 감싸진다. **ProxyFactory는 항상 Advisor 단위로 동작**한다는 점을 기억.

### 4. 프록시 팩토리를 쓸 때 Advisor는 필수

Advisor 없이 `getProxy()`를 호출하면 부가 기능이 없는 프록시가 만들어진다. 무의미하지만 예외가 나지는 않으므로, "프록시는 만들어졌는데 로그가 안 찍힐 때" Advisor 등록을 먼저 확인할 것.

### 5. 포인트컷은 메서드마다 물어본다

`MyMethodMatcher`에 로그를 찍어보면 `save()`, `find()` 뿐 아니라 `toString()` 같은 메서드에도 포인트컷이 호출된다. 프록시가 가로채는 **모든 메서드**가 포인트컷 판단 대상이다.

---

## 💡 판단 기준

**"스프링의 추상화는 구체 기술을 감추고 개발자에게 하나의 인터페이스를 제공한다"** — 이 챕터가 전형적인 사례다. JDK/CGLIB라는 구체 기술을 `ProxyFactory` + `Advice`로 감췄다. 이 패턴은 스프링 곳곳에서 반복된다 (`JdbcTemplate`, `TransactionManager` 등). "구체 기술이 2개 이상이고, 사용법이 다르면, 스프링은 추상화 계층을 제공했을 가능성이 높다"는 관점으로 보면 스프링의 설계 철학이 보인다.

**Advisor가 "짝"의 단위라는 것.** Pointcut과 Advice를 따로 관리하면 N:M 매핑이 되어 조합이 터진다. Advisor로 1:1 묶음을 보장하는 것은 단순해 보이지만, 여러 부가 기능이 한 프록시에 공존하는 순간 필수가 된다.

**추상화는 공짜가 아니다 — 그래서 내부를 보는 게 의미가 있다.** ProxyFactory가 편해진 대가로 "지금 JDK인가 CGLIB인가"가 코드에서 안 보이게 됐다. 평소엔 상관없지만 **캐스팅 예외나 프록시 관련 문제가 터질 때는 그 감춰진 계층을 열어봐야** 원인이 보인다. `AopUtils.isJdkDynamicProxy()`로 확인하는 습관이 그럴 때 쓰인다.

---

## 참고

- 김영한, 스프링 핵심 원리 고급편 — Ch.6 스프링이 지원하는 프록시
- [Spring Framework Reference — ProxyFactory](https://docs.spring.io/spring-framework/reference/core/aop-api/pfb.html)
- [Spring AOP API — Pointcut](https://docs.spring.io/spring-framework/reference/core/aop-api/pointcuts.html)
- [AOP Alliance — `org.aopalliance.intercept`](http://aopalliance.sourceforge.net/) (Advice/MethodInterceptor 인터페이스 출처)

**학습 날짜**: 2026-08-14 · **계기**: 김영한 고급편 Ch.6 수강 후 Claude 소크라테스 복습 세션 — 5문항 중 ① 3가지 문제점 중 "핸들러 중복 작성" 문제는 힌트 후 도달 ② **ProxyFactory 내부에서 Advice를 호출하는 전용 InvocationHandler/MethodInterceptor가 자동 생성된다는 구조를 전혀 기억 못 함** (이 세션에서 가장 약했던 부분) ③ Advisor의 필요 이유를 "addAdvice가 내부에서 만들어주니까 왜 필요한지 모르겠다"고 답함 → 여러 부가 기능의 Pointcut-Advice 짝 보장 역할로 이해 ④ "프록시 1개"는 기억했으나 동작을 "어드바이저를 갈아끼우는 식"으로 오해 → 리스트로 들고 있다가 순회하는 구조로 정정 ⑤ 세션 끝에 "챕터 4 체이닝이 어떤 모양이었는지" 직접 질문해 두 방식(여러 프록시 vs 프록시 1개)의 연결을 스스로 확인함

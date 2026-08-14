# @Aspect AOP — 애노테이션으로 Advisor 만들기

> **한 줄 요약**: `@Aspect`는 **Pointcut + Advice = Advisor**를 애노테이션으로 선언하는 문법이고, 이걸 실제로 동작시키는 주체는 **`AnnotationAwareAspectJAutoProxyCreator`** 하나다. 이 빈 후처리기가 ① `@Aspect` 빈을 찾아 **Advisor로 변환·캐싱**하고 ② 그 Advisor들로 **프록시를 생성**한다 — 두 가지 일을 한다. ⚠️ `@Aspect`만 붙이고 **스프링 빈으로 등록하지 않으면 조용히 아무 일도 일어나지 않는다**(예외조차 없음). ⚠️ `ProceedingJoinPoint`는 "어디에 적용할지"를 정하는 Pointcut이 **아니라** [Ch.6의 `MethodInvocation`](./proxy-factory.md)에 대응하는 **호출 핸들**이다.

관련 노트: [AOP 개념](./aop-concepts.md) (다음 챕터 — 이 메커니즘이 해결하려는 문제와 용어 체계) · [빈 후처리기](./bean-post-processor.md) (직전 챕터 — 자동 프록시 생성기의 나머지 절반이 여기서 완성된다) · [ProxyFactory](./proxy-factory.md) (`Advisor = Pointcut + Advice`의 출처) · [동적 프록시](./dynamic-proxy.md) · [프록시/데코레이터 패턴](./proxy-decorator-pattern.md) · [ThreadLocal](../concurrency/thread-local.md) (`LogTrace`의 출처) · [@Transactional](../spring/transactional.md)·[Spring Cache](../spring/spring-cache.md)·[메서드 보안](../security/method-security.md) (전부 이 메커니즘 위에서 동작) · [커스텀 어노테이션](../annotation/custom-annotation.md) (`@annotation` 포인트컷으로 이어짐)

---

## 0. 출발점 — Ch.7이 남긴 마지막 불편함

[직전 챕터](./bean-post-processor.md)에서 자동 프록시 생성기 덕에 **개발자가 할 일은 `Advisor`를 빈으로 등록하는 것뿐**이 되었다. 그런데 그 Advisor를 만드는 코드가 여전히 이렇게 생겼다:

```java
@Bean
public Advisor advisor3(LogTrace logTrace) {
    AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();   // ① 포인트컷 객체 생성
    pointcut.setExpression("execution(* hello.proxy.app..*(..))");          // ② 표현식 주입
    LogTraceAdvice advice = new LogTraceAdvice(logTrace);                   // ③ 어드바이스 객체 생성
    return new DefaultPointcutAdvisor(pointcut, advice);                    // ④ 둘을 조립
}
```

부가 기능 하나 추가할 때마다 **객체 3개를 손으로 조립**해야 한다. Advice는 별도 클래스(`LogTraceAdvice`)로 빠져 있어서 "이 포인트컷이 어떤 로직을 부르는지"가 두 파일에 흩어진다.

> **발상**: 포인트컷은 어차피 문자열이고, 어드바이스는 어차피 메서드 하나다. **메서드 위에 애노테이션으로 표현식을 붙이면** 둘을 한 곳에 쓸 수 있지 않나?
> → 그게 `@Aspect`다.

---

## 1. @Aspect 기본 — 전체 코드

### 의존성 (Ch.7에서 이미 추가)

```gradle
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

### LogTraceAspect — Aspect 클래스

```java
package hello.proxy.config.v6_aop.aspect;

import hello.proxy.trace.TraceStatus;
import hello.proxy.trace.logtrace.LogTrace;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;

@Slf4j
@Aspect                                    // ← 이 클래스를 Advisor 소스로 인식시킨다
public class LogTraceAspect {

    private final LogTrace logTrace;

    public LogTraceAspect(LogTrace logTrace) {
        this.logTrace = logTrace;
    }

    @Around("execution(* hello.proxy.app..*(..))")   // ← 표현식 = Pointcut
    public Object execute(ProceedingJoinPoint joinPoint) throws Throwable {
        //                ↑ 메서드 본문 전체 = Advice
        TraceStatus status = null;

        // log.info("target={}", joinPoint.getTarget());        // 실제 호출 대상
        // log.info("getArgs={}", joinPoint.getArgs());         // 전달 인자
        // log.info("getSignature={}", joinPoint.getSignature()); // join point 시그니처

        try {
            String message = joinPoint.getSignature().toShortString();
            status = logTrace.begin(message);

            Object result = joinPoint.proceed();   // ← 실제 target 호출
            logTrace.end(status);
            return result;
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }
}
```

### AopConfig — 빈 등록

```java
package hello.proxy.config.v6_aop;

import hello.proxy.config.AppV1Config;
import hello.proxy.config.AppV2Config;
import hello.proxy.config.v6_aop.aspect.LogTraceAspect;
import hello.proxy.trace.logtrace.LogTrace;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;

@Configuration
@Import({AppV1Config.class, AppV2Config.class})   // V1, V2는 수동 등록 필요
public class AopConfig {

    @Bean
    public LogTraceAspect logTraceAspect(LogTrace logTrace) {
        return new LogTraceAspect(logTrace);      // ★ @Aspect가 있어도 빈 등록은 별개
    }
}
```

### ProxyApplication

```java
//@Import({AppV1Config.class, AppV2Config.class})
//@Import(InterfaceProxyConfig.class)
//@Import(ConcreteProxyConfig.class)
//@Import(DynamicProxyBasicConfig.class)
//@Import(DynamicProxyFilterConfig.class)
//@Import(ProxyFactoryConfigV1.class)
//@Import(ProxyFactoryConfigV2.class)
//@Import(BeanPostProcessorConfig.class)
//@Import(AutoProxyConfig.class)
@Import(AopConfig.class)
@SpringBootApplication(scanBasePackages = "hello.proxy.app")
public class ProxyApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProxyApplication.class, args);
    }

    @Bean
    public LogTrace logTrace() {
        return new ThreadLocalLogTrace();
    }
}
```

실행하면 V1·V2·V3 전부 프록시가 적용된다.
```
http://localhost:8080/v1/request?itemId=hello
http://localhost:8080/v2/request?itemId=hello
http://localhost:8080/v3/request?itemId=hello
```

> 📌 `@Aspect`는 스프링이 만든 애노테이션이 아니라 **AspectJ 프로젝트의 것**이다. 스프링은 이 *문법만* 차용해서 프록시 기반 AOP를 구현한다 — AspectJ의 바이트코드 위빙을 쓰는 게 아니다. (자세한 건 다음 챕터 "스프링 AOP 개념")

---

## 2. Ch.6 수동 방식과 1:1 매핑

**이 표가 이 챕터의 절반이다.** 새 개념이 아니라 **이미 아는 것의 다른 표기법**이라는 점이 핵심.

| 역할 | 수동 방식 (Ch.6 ProxyFactory) | @Aspect 방식 (Ch.8) |
|---|---|---|
| **어디에 적용할지** (Pointcut) | `AspectJExpressionPointcut` 객체 + `setExpression()` | `@Around("execution(…)")` 의 **문자열 인자** |
| **무엇을 할지** (Advice) | `MethodInterceptor` 구현체 (`LogTraceAdvice`) | `@Around` 가 붙은 **메서드 본문** |
| **둘의 결합** (Advisor) | `new DefaultPointcutAdvisor(pointcut, advice)` | **자동 생성** (개발자가 안 만듦) |
| **호출 핸들** | `MethodInvocation invocation` | `ProceedingJoinPoint joinPoint` |
| **target 호출** | `invocation.proceed()` | `joinPoint.proceed()` |

```java
// Ch.6 — 수동
public class LogTraceAdvice implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Object result = invocation.proceed();
        return result;
    }
}

// Ch.8 — @Aspect
@Around("execution(* hello.proxy.app..*(..))")
public Object execute(ProceedingJoinPoint joinPoint) throws Throwable {
    Object result = joinPoint.proceed();
    return result;
}
```

> 💡 `@Aspect`는 **새로운 실행 엔진이 아니라 Advisor를 만드는 축약 문법**이다. 변환이 끝나고 나면 런타임에는 Ch.6·Ch.7과 완전히 같은 것(프록시 + Advisor 리스트)이 돌아간다. "@Aspect를 쓰면 뭔가 다른 방식으로 동작한다"고 생각하면 디버깅이 막힌다.

---

## 3. ⚠️ ProceedingJoinPoint — Pointcut이 아니다

**세션에서 헷갈렸던 지점.** 이름에 `Point`가 들어가서 "프록시를 적용할지 결정하는 조건"으로 읽기 쉽지만 **완전히 다른 층의 물건**이다.

| | 정체 | 언제 쓰이나 |
|---|---|---|
| **Pointcut** | 어디에 적용할지 **판단하는 필터** | 로딩 시점(프록시 생성 여부) + 호출 시점(어드바이스 적용 여부) |
| **JoinPoint** | 실제로 적용된 **그 지점의 정보 객체** | Advice **실행 중** |
| **ProceedingJoinPoint** | `JoinPoint` + **`proceed()` 실행 능력** | `@Around` Advice 실행 중 |

### 용어 정리

- **Join Point**: 어드바이스가 적용될 수 **있는** 모든 위치 (스프링 AOP에서는 **항상 메서드 실행 지점**)
- **Pointcut**: 그 후보들 중 **실제로 고르는 조건식**
- **ProceedingJoinPoint**: 골라져서 실행되는 **바로 그 순간의 컨텍스트 객체**

> 📐 비유: Join Point = 전국의 모든 정류장 / Pointcut = "강남구에 있는 정류장만" 이라는 규칙 / ProceedingJoinPoint = 지금 서 있는 그 정류장의 표지판(+ 다음 정류장으로 출발시키는 버튼)

### ProceedingJoinPoint가 제공하는 것

```java
@Around("execution(* hello.proxy.app..*(..))")
public Object execute(ProceedingJoinPoint joinPoint) throws Throwable {
    joinPoint.getTarget();      // 실제 호출 대상 객체 (프록시 아님!)
    joinPoint.getThis();        // 프록시 객체
    joinPoint.getArgs();        // Object[] 전달 인자
    joinPoint.getSignature();   // 메서드 시그니처 (이름·반환타입·파라미터)
    joinPoint.getSignature().toShortString();   // "OrderServiceV1.orderItem(..)"

    return joinPoint.proceed(); // ★ 실제 target 호출 — 안 부르면 target이 실행되지 않는다
    // joinPoint.proceed(new Object[]{...})  — 오버로드: 인자를 바꿔치기해서 target 호출 가능
}
```

> ⚠️ **`ProceedingJoinPoint`는 `@Around` 전용이며, 반드시 첫 번째 파라미터여야 한다.** 공식 문서: "around advice is required to declare a **first** parameter of type ProceedingJoinPoint". `@Before`·`@After`·`@AfterReturning`·`@AfterThrowing`은 `proceed()` 권한이 없으므로 부모 타입인 `JoinPoint`만 받을 수 있고, 이것도 선언한다면 첫 파라미터여야 한다. (다음 챕터 내용이지만 여기서 같이 잡아두면 편함)

---

## 4. 🔴 AnnotationAwareAspectJAutoProxyCreator — 이 챕터의 핵심

Ch.7에서 이미 등장했던 그 자동 프록시 생성기다. 다만 그때는 "Advisor 빈을 찾아 프록시를 만든다"까지만 봤고, **이름 앞의 `AnnotationAware`가 무슨 뜻인지는 미뤄뒀다.** 그 나머지 절반이 이 챕터다.

### 하는 일은 정확히 2가지

```
① @Aspect 를 보고 Advisor 로 변환해서 저장한다      ← AnnotationAware 가 담당하는 부분
② 저장된 Advisor 를 기반으로 프록시를 생성한다        ← Ch.7에서 배운 부분
```

### ① @Aspect → Advisor 변환·저장

```
1. 실행        : 스프링 애플리케이션 로딩 시점에 자동 프록시 생성기가 호출된다
2. 조회        : 컨테이너에서 @Aspect 애노테이션이 붙은 스프링 빈을 **모두 조회**한다
3. 생성        : @Aspect 어드바이저 빌더가 애노테이션 정보를 기반으로 Advisor를 생성한다
4. 저장(캐싱)  : 생성한 Advisor를 빌더 내부 저장소에 캐시한다
```

**`BeanFactoryAspectJAdvisorsBuilder`** 가 이 일을 맡는다. `@Aspect` 정보를 기반으로 포인트컷·어드바이스·어드바이저를 만들고 보관한다. 이미 만들어져 있으면 **캐시에 저장된 것을 반환**하므로 매 빈마다 다시 파싱하지 않는다.

실제 스프링 소스에서 ①과 ②가 합류하는 지점:

```java
public class AnnotationAwareAspectJAutoProxyCreator extends AspectJAwareAdvisorAutoProxyCreator {

    private BeanFactoryAspectJAdvisorsBuilder aspectJAdvisorsBuilder;

    @Override
    protected List<Advisor> findCandidateAdvisors() {
        List<Advisor> advisors = super.findCandidateAdvisors();          // 3-1. 컨테이너의 Advisor 빈
        if (this.aspectJAdvisorsBuilder != null) {
            advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());  // 3-2. @Aspect 변환분
        }
        return advisors;
    }
}
```

> 💡 **`@Bean Advisor`(Ch.7 방식)와 `@Aspect`(Ch.8 방식)는 경쟁 관계가 아니라 같은 리스트에 합쳐진다.** 한 프로젝트에서 둘을 섞어 써도 되고, 섞으면 둘 다 적용된다.

### ② Advisor 기반 프록시 생성 (Ch.7과 동일)

```
1. 생성            : 스프링 빈 대상 객체 생성 (@Bean, 컴포넌트 스캔 모두)
2. 전달            : 빈 저장소 등록 직전에 빈 후처리기에 전달
3-1. Advisor 빈 조회 : 컨테이너에서 Advisor 빈을 모두 조회
3-2. @Aspect Advisor 조회 : @Aspect 어드바이저 빌더 내부에 저장된 Advisor를 모두 조회   ← ★ 추가된 단계
4. 프록시 적용 대상 체크 : Advisor의 Pointcut으로 판단. 클래스 정보는 물론이고
                        **해당 객체의 모든 메서드를 하나하나 매칭** → 하나라도 만족하면 대상
5. 프록시 생성      : 대상이면 프록시 생성해서 반환 / 아니면 원본 반환
6. 빈 등록          : 반환된 객체가 빈 저장소에 등록
```

**3-2가 이 챕터에서 새로 끼어든 단 하나의 단계다.** 나머지는 Ch.7 그대로.

### 이름 해부

| 조각 | 의미 |
|---|---|
| `Annotation` **`Aware`** | `@Aspect` **애노테이션을 인식**해서 Advisor로 바꿔준다 ← 이 챕터 |
| `AspectJ` | 포인트컷 표현식이 AspectJ 문법 |
| `Advisor` | Advisor를 찾아서 |
| `AutoProxyCreator` | 프록시를 자동 생성한다 ← Ch.7 |

### Spring Boot에서의 활성화

`spring-boot-starter-aop`를 추가하면 **`AopAutoConfiguration`**이 동작해 `@EnableAspectJAutoProxy`와 동일한 효과를 낸다. 즉 **부트에서는 `@EnableAspectJAutoProxy`를 직접 붙일 필요가 없다.**

| 프로퍼티 | 기본값 | 의미 |
|---|---|---|
| `spring.aop.auto` | `true` | `false`면 AOP 자동 설정 전체가 비활성화 |
| `spring.aop.proxy-target-class` | `true` | 기본이 **CGLIB** — 인터페이스가 있어도 구체 클래스 프록시 |

> `@EnableAspectJAutoProxy`는 클래스패스에 **`aspectjweaver`가 있어야** 동작한다. starter-aop가 이걸 같이 끌어온다.

---

## 5. ⚠️ @Aspect만 붙이면 아무 일도 안 일어난다

**세션에서 "왜 빈으로 등록해야 하나"를 정확히 답하지 못한 지점.**

이유는 §4의 **①-2단계**에 그대로 적혀 있다: 자동 프록시 생성기는 **"컨테이너에서 `@Aspect` 빈을 조회"** 한다. 빈이 아니면 조회 대상 자체가 아니므로 → 변환 안 됨 → Advisor 없음 → 프록시 없음. **그냥 평범한 자바 클래스로 남는다.**

```java
// 방법 1: @Bean으로 직접 등록
@Configuration
public class AopConfig {
    @Bean
    public LogTraceAspect logTraceAspect(LogTrace logTrace) {
        return new LogTraceAspect(logTrace);
    }
}

// 방법 2: @Component로 컴포넌트 스캔 (실무에서 더 흔함)
@Aspect
@Component
public class LogTraceAspect { ... }
```

> ⚠️ **가장 나쁜 점은 "조용히" 실패한다는 것이다.** 컴파일 에러도, 기동 실패도, 경고 로그도 없다. 그냥 부가 기능이 안 걸린다. [Ch.7 노트의 함정 정리](./bean-post-processor.md#️-함정-정리)와 [Spring Cache](../spring/spring-cache.md)·[메서드 보안](../security/method-security.md)에서 반복되는 **"활성화 안 하면 예외가 아니라 무시"** 패턴과 정확히 같은 종류다.

---

## 6. ⚠️ 함정 정리

### 1. `proceed()`를 안 부르면 target이 실행되지 않는다 (조용히)

```java
@Around("execution(* hello.proxy.app..*(..))")
public Object execute(ProceedingJoinPoint joinPoint) throws Throwable {
    log.info("start");
    return null;              // ⚠️ proceed() 없음 → 원본 메서드가 아예 실행 안 됨
}
```

`@Around`는 **target 호출 권한을 통째로 위임받는다.** 안 부르면 원본이 실행되지 않고 반환값도 `null`이 된다. 예외가 안 나므로 "왜 서비스 로직이 안 도는지" 추적이 어렵다. `@Before`·`@After`는 이 실수가 구조적으로 불가능하다(프레임워크가 알아서 호출).

단, 공식 문서가 명시하듯 `proceed()`는 **0번·1번·여러 번 호출 모두 합법**이다 — 캐싱 aspect가 캐시 히트 시 `proceed()` 없이 캐시값을 반환하는 게 대표 예(이게 `@Cacheable`의 원리다). 문제는 "의도 없이" 빠뜨렸을 때다.

⚠️ **target 메서드가 primitive(int 등)를 반환하는데 advice가 `null`을 반환하면** 런타임에 `AopInvocationException: Null return value from advice does not match primitive return type`이 터진다. 또한 advice 반환 타입을 `void`로 선언하면 `proceed()` 결과가 무엇이든 호출자는 **항상 null**을 받는다 — 그래서 공식 권장이 `Object` 반환이다.

### 2. 반환 타입은 `Object`, 예외는 `throws Throwable`

`@Around` 메서드가 `void`거나 구체 타입이면 target의 반환값을 흘려보낼 수 없다. `proceed()`가 `Throwable`을 던지므로 시그니처에 선언해야 한다.

### 3. ⚠️ `@Aspect` 클래스 자신은 프록시 대상이 되지 않는다

`AnnotationAwareAspectJAutoProxyCreator`는 `isInfrastructureClass()`를 오버라이드해서 **aspect 클래스를 인프라 클래스로 취급**한다:

```java
@Override
protected boolean isInfrastructureClass(Class<?> beanClass) {
    return (super.isInfrastructureClass(beanClass) || this.aspectJAdvisorFactory.isAspect(beanClass));
}
```

→ **`@Aspect` 클래스 안에 `@Transactional`·`@Cacheable`을 붙여도 안 먹는다.** (`Advice`·`Advisor`·`AopInfrastructureBean`도 같은 이유로 제외된다.) 아스펙트가 자기 자신을 감싸는 무한 재귀를 막기 위한 설계.

### 4. ⚠️ 자기호출(self-invocation)은 여전히 안 먹는다

`@Aspect`도 결국 **프록시 기반**이다. [동적 프록시 노트](./dynamic-proxy.md)·[@Transactional 노트](../spring/transactional.md)에서 다룬 함정이 그대로 살아 있다.

```java
@Service
public class OrderService {
    public void outer() {
        this.inner();   // ⚠️ this = target(원본) → 프록시를 안 거침 → 어드바이스 미적용
    }
    public void inner() { ... }
}
```

`@Aspect`로 바꿨다고 이 문제가 해결되지 않는다. **문법이 편해졌을 뿐 실행 모델은 그대로.**

### 5. ⚠️ `@Order`는 **Aspect(클래스) 단위**로만 동작한다

Spring 공식 문서: 서로 다른 aspect의 advice가 같은 조인 포인트에 걸리면 **명시하지 않는 한 실행 순서는 정의되지 않는다.** 순서는 `@Order` 애노테이션이나 `Ordered` 인터페이스로 지정한다.

```java
@Aspect
@Order(1)          // ✅ 클래스에 붙여야 의미 있음
@Component
public class LogAspect { ... }

@Aspect
@Order(2)
@Component
public class TxAspect { ... }
```

**같은 `@Aspect` 클래스 안**에서는 규칙이 두 층으로 갈린다:

| 상황 | 순서 |
|---|---|
| 같은 aspect, **다른 타입**의 advice | ✅ 타입 우선순위로 정의됨: `@Around` > `@Before` > `@After` > `@AfterReturning` > `@AfterThrowing` (단 `@After`는 finally 의미라 실제 호출은 `@AfterReturning`/`@AfterThrowing` **뒤**) |
| 같은 aspect, **같은 타입**의 advice 2개 (예: `@After` 둘) | ⚠️ **미정의** — javac 컴파일 클래스에서 소스 선언 순서를 리플렉션으로 알 수 없음. 메서드에 `@Order` 붙여도 소용없다 |

같은 타입끼리 순서가 중요하면 공식 권장은 둘 중 하나: **한 메서드로 합치거나, aspect 클래스를 분리**해 클래스 단위 `@Order`로 정렬한다.

> 📌 [Ch.7 노트](./bean-post-processor.md#️-함정-1--order-애노테이션은-beanpostprocessor에-안-먹는다)와 대비해서 외울 것: **BeanPostProcessor에는 `@Order`가 안 먹고(`Ordered` 인터페이스만), `@Aspect`에는 `@Order`가 먹는다.** 같은 AOP 계열인데 규칙이 반대라 헷갈리기 쉽다.

### 6. 포인트컷은 여전히 두 번 사용된다

[Ch.7의 핵심](./bean-post-processor.md#6-️-포인트컷은-두-번-사용된다--이-챕터의-핵심)이 그대로 적용된다. ① 클래스 단위 = 프록시를 만들지 / ② 메서드 단위 = 어드바이스를 적용할지. `@Aspect`로 표기법만 바뀌었을 뿐 판정 구조는 동일하다.

### 7. Advisor 중복 등록

Ch.7의 `advisor1~3`을 주석 처리하지 않은 채 `AopConfig`를 추가하면 어드바이저가 중복 적용되어 로그가 두 번 찍힌다. `ProxyApplication`의 `@Import`를 하나만 남길 것.

---

## 7. 인접 개념 — Advice 5종 (다음 챕터 예고)

이 챕터는 `@Around`만 쓰지만, 나머지도 같은 자리에 들어간다. **`@Around` 하나로 전부 표현 가능**하고 나머지는 그 부분집합이다.

| 애노테이션 | 시점 | 파라미터 | target 호출 차단 |
|---|---|---|---|
| `@Around` | 전/후 전부 | `ProceedingJoinPoint` | ✅ 가능 |
| `@Before` | 호출 전 | `JoinPoint` | ❌ (예외 던지면 가능) |
| `@AfterReturning` | 정상 반환 후 | `JoinPoint` + `returning` | ❌ |
| `@AfterThrowing` | 예외 발생 시 | `JoinPoint` + `throwing` | ❌ |
| `@After` | 정상·예외 무관 (finally) | `JoinPoint` | ❌ |

> 💡 **강력함이 아니라 제약이 선택 기준이다.** `@Around`는 `proceed()`를 빠뜨릴 수 있고 반환값을 조작할 수 있어서 실수 여지가 크다. 로그만 남기면 되는데 `@Around`를 쓰는 건, 읽는 사람에게 "이 메서드가 흐름을 바꿀 수도 있다"는 잘못된 신호를 준다. **부가 기능이 흐름에 개입하지 않는다면 `@Before`/`@After`가 의도를 정확히 드러낸다.**

---

## 8. 횡단 관심사 (Cross-Cutting Concerns) — Ch.4~8의 도착점

로그 추적 기능은 **특정 기능 하나에 관심이 있는 기능이 아니다.** 애플리케이션의 여러 기능들 **사이에 걸쳐서** 들어가는 관심사다. 이것을 **횡단 관심사**라고 한다.

```
              OrderController  OrderService  OrderRepository
                    │               │               │
  로그 추적  ───────┼───────────────┼───────────────┼──────▶  횡단(cross-cutting)
  트랜잭션  ───────┼───────────────┼───────────────┼──────▶
  보안      ───────┼───────────────┼───────────────┼──────▶
                    │               │               │
                  (핵심 관심사 = 각자의 고유 로직)
```

**세로**가 각 클래스의 핵심 관심사, **가로**가 횡단 관심사. 객체지향의 상속·위임만으로는 이 가로줄을 깔끔하게 뽑아내기 어렵다 — 그래서 프록시가 필요했다.

### Ch.4 → Ch.8 발전사

| 챕터 | 도구 | 해결한 것 | 남은 문제 |
|---|---|---|---|
| Ch.4 | 프록시/데코레이터 패턴 | 원본·클라이언트 코드 수정 0 | 대상 클래스 수만큼 프록시 클래스 폭발 |
| Ch.5 | 동적 프록시 | 프록시 클래스를 런타임 생성 | JDK/CGLIB 이원화, 기술마다 코드 다름 |
| Ch.6 | ProxyFactory + Advisor | 기술 추상화, Pointcut+Advice 개념화 | 설정 지옥, 컴포넌트 스캔 미적용 |
| Ch.7 | 빈 후처리기 | 자동 프록시 생성, V3까지 적용 | Advisor 조립이 여전히 수작업 |
| **Ch.8** | **@Aspect** | **애노테이션으로 Advisor 선언** | (프록시 기반의 본질적 한계 → Ch.13) |

**실무에서 프록시를 적용할 때는 대부분 이 방식(@Aspect)을 사용한다.**

---

## 💡 판단 기준

**"편해진 문법"과 "달라진 동작"을 구분해야 디버깅이 된다.** `@Aspect`는 Advisor를 만드는 축약 표기이지, 새로운 실행 엔진이 아니다. 변환이 끝나면 런타임에는 Ch.6·Ch.7과 똑같은 것(프록시 1개 + Advisor 리스트)이 돈다. 그래서 `@Aspect`로 바꿨다고 자기호출 함정이 사라지거나 `final` 메서드가 프록시되기 시작하지 않는다. **새 애노테이션을 배울 때 "이게 무엇으로 변환되는가"를 먼저 물으면, 그 애노테이션이 상속하는 제약까지 함께 온다.**

**"조용히 안 되는" 실패는 활성화·등록 조건부터 의심한다.** `@Aspect`를 빈으로 안 올리면 예외 없이 무시된다. 이건 `@EnableCaching` 없는 `@Cacheable`, `@EnableMethodSecurity` 없는 `@PreAuthorize`, `@Enable~` 없는 대부분과 **똑같은 실패 모양**이다. 세션에서 배운 실용적 순서는 이것 — **① 활성화됐나(빈 등록·`@Enable~`) → ② 프록시가 생겼나(`AopUtils.isAopProxy`) → ③ 포인트컷이 맞나 → ④ 자기호출인가.** 네 단계를 순서대로 짚으면 "AOP가 안 먹어요"의 대부분이 잡힌다.

**같은 계열이라도 규칙이 같을 거라고 가정하지 않는다.** `@Order`는 `@Aspect`에는 먹고 `BeanPostProcessor`에는 안 먹는다. 둘 다 AOP·프록시 계열이라 직관적으로는 같아야 할 것 같지만 다르다. 게다가 `@Aspect`에서도 **클래스 단위로만** 유효하고 같은 클래스 내부 메서드끼리는 순서가 보장되지 않는다. **"이 애노테이션이 어느 단위에 붙어서 무엇을 정렬하는가"를 확인하지 않으면, 순서가 어긋나도 에러가 없어서 오래 모른 채 지나간다.**

---

## 참고

- 김영한, 스프링 핵심 원리 고급편 — Ch.8 @Aspect AOP
- [Spring Framework Javadoc — `AnnotationAwareAspectJAutoProxyCreator`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/aspectj/annotation/AnnotationAwareAspectJAutoProxyCreator.html) (`AspectJAwareAdvisorAutoProxyCreator` 상속, `isInfrastructureClass` 오버라이드로 aspect 자신은 프록시 제외)
- [`AnnotationAwareAspectJAutoProxyCreator` 소스](https://github.com/spring-projects/spring-framework/blob/main/spring-aop/src/main/java/org/springframework/aop/aspectj/annotation/AnnotationAwareAspectJAutoProxyCreator.java) (`findCandidateAdvisors()`에서 `super` + `buildAspectJAdvisors()` 합류)
- [Spring Framework Javadoc — `BeanFactoryAspectJAdvisorsBuilder`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/aspectj/annotation/BeanFactoryAspectJAdvisorsBuilder.html) ("@AspectJ 빈을 BeanFactory에서 가져와 Advisor로 만드는 헬퍼")
- [Spring Framework Reference — Declaring Advice](https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/advice.html) (advice 5종, PJP는 첫 파라미터 필수, 서로 다른 aspect 간 순서는 미정의 → `@Order`/`Ordered`로 지정 / 같은 aspect 내는 타입 우선순위 정의·같은 타입끼리만 미정의 / "가장 덜 강력한 advice를 쓰라" 공식 권장)
- [Spring Framework Reference — Proxying Mechanisms](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html) (JDK vs CGLIB 선택 규칙, 자기호출 미인터셉트)
- [Spring Boot Javadoc — `AopAutoConfiguration`](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/autoconfigure/aop/AopAutoConfiguration.html) (`spring.aop.auto` 기본 true, `proxyTargetClass` 기본 true)
- [Spring Framework Javadoc — `@EnableAspectJAutoProxy`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/EnableAspectJAutoProxy.html) (`aspectjweaver` 필요)

**학습 날짜**: 2026-08-14 · **계기**: 김영한 고급편 Ch.8 수강 후 Claude 소크라테스 복습 세션 — "어디까지 알아야 하고 외워야 하는지 감이 안 잡힌다"는 질문으로 시작. ① `AnnotationAwareAspectJAutoProxyCreator`의 2가지 역할을 **"@Aspect를 Advisor로 만든다"와 "프록시를 만든다" 중 어느 쪽인지 헷갈려 함** — 실제로는 둘 다이며, 이게 이 챕터의 전부라는 걸 확인 ② `@Around` 표현식=Pointcut / 메서드 본문=Advice 매핑은 감으로 맞혔으나 **"Advisor로 변환된다"는 결론까지는 도달 못 함** ③ ⚠️ **`ProceedingJoinPoint`를 "프록시 생성 여부를 결정하는 조건"(=Pointcut)으로 오해** — 실제로는 Ch.6 `MethodInvocation`에 대응하는 호출 핸들. 이름의 `Point` 때문에 생긴 혼동으로 보여 §3에 Join Point/Pointcut/ProceedingJoinPoint 3층 구분을 별도 정리 ④ `@Aspect` 빈 등록 필요성은 **방법(@Bean/@Component)은 정확히 답했으나 이유("스프링이 관리해야 하니까")가 막연** → "조회 대상이 아니면 변환 자체가 안 일어난다"로 구체화 ⑤ 횡단 관심사는 그림으로는 이해하나 **말로 설명하기 어려워함** → "여러 기능을 가로질러 걸쳐 있는 관심사" 정의 정리. 📌 세션 후 조사에서 추가 확인한 것: `@Aspect` 클래스 자신은 `isInfrastructureClass`로 프록시 제외됨 / `@Order`가 `@Aspect`에는 먹지만 클래스 단위만 — 같은 aspect 내에서는 advice 타입 우선순위는 정의되고(5.2.7+) **같은 타입끼리만 미보장** / Boot의 `spring.aop.proxy-target-class` 기본 true

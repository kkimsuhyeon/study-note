# 스프링 AOP 구현 — @Pointcut 분리 · 조합 · 순서 · Advice 5종

> **한 줄 요약**: [`@Aspect`](./aspect-aop.md)로 Advisor를 선언하는 법을 알았다면, 이 챕터는 그걸 **실무에서 쓸 수 있게 정리하는 문법**이다 — `@Pointcut`으로 표현식을 분리하고, `&&`로 조합하고, 외부 클래스로 공용화하고, `@Order`로 순서를 주고, `@Around` 말고도 4종의 Advice를 골라 쓴다. ⚠️ **`@Order`는 클래스 단위로만 먹는다** — 같은 `@Aspect` 안의 메서드끼리는 순서를 줄 수 없어서, 순서가 필요하면 **aspect 클래스를 쪼개야 한다**(강의의 `static class` 분리가 그것). ⚠️ `@AfterReturning`은 반환값을 **읽을 수만 있고 바꿀 수 없다** — 바꾸려면 `@Around`뿐.

관련 노트: [@Aspect AOP](./aspect-aop.md) (직전 챕터 — `@Aspect`가 Advisor로 변환되는 메커니즘) · [AOP 개념](./aop-concepts.md) (용어 체계·위빙) · [빈 후처리기](./bean-post-processor.md) (자동 프록시 생성기) · [ProxyFactory](./proxy-factory.md) (`Advisor = Pointcut + Advice`의 출처) · [동적 프록시](./dynamic-proxy.md) · [프록시/데코레이터 패턴](./proxy-decorator-pattern.md) · [ThreadLocal](../concurrency/thread-local.md) (트랙 10 출발점) · [@Transactional](../spring/transactional.md) (이 챕터의 `doTransaction()` 예제가 흉내 내는 실물) · [Spring Cache](../spring/spring-cache.md) · [커스텀 어노테이션](../annotation/custom-annotation.md)

---

## 1. 예제 프로젝트 — 이 챕터 전체가 이 두 클래스 위에서 돈다

강의는 새 프로젝트(`hello.aop`)로 시작한다. 의존성은 Lombok + AOP만 있으면 된다.

```gradle
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

### OrderRepository

```java
package hello.aop.order;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Repository;

@Slf4j
@Repository
public class OrderRepository {

    public String save(String itemId) {
        log.info("[orderRepository] 실행");
        // 저장 로직
        if (itemId.equals("ex")) {
            throw new IllegalStateException("예외 발생!");
        }
        return "ok";
    }
}
```

### OrderService

```java
package hello.aop.order;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public void orderItem(String itemId) {
        log.info("[orderService] 실행");
        orderRepository.save(itemId);
    }
}
```

### AopTest — 학습 테스트

```java
package hello.aop;

import hello.aop.order.OrderRepository;
import hello.aop.order.OrderService;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import org.springframework.aop.support.AopUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.assertj.core.api.Assertions.assertThatThrownBy;

@Slf4j
@SpringBootTest
public class AopTest {

    @Autowired OrderService orderService;
    @Autowired OrderRepository orderRepository;

    @Test
    void aopInfo() {
        log.info("isAopProxy, orderService={}", AopUtils.isAopProxy(orderService));
        log.info("isAopProxy, orderRepository={}", AopUtils.isAopProxy(orderRepository));
    }

    @Test
    void success() {
        orderService.orderItem("itemA");
    }

    @Test
    void exception() {
        assertThatThrownBy(() -> orderService.orderItem("ex"))
                .isInstanceOf(IllegalStateException.class);
    }
}
```

> 📌 **`AopUtils.isAopProxy(obj)`** — 프록시가 적용됐는지 확인하는 진단 도구. Aspect를 아직 안 만들었으면 `false`, 등록하면 `true`. "AOP가 안 먹어요"를 디버깅할 때 **제일 먼저 찍어볼 것**. ([@Aspect 노트의 4단계 진단](./aspect-aop.md#-판단-기준) 중 2단계)

---

## 2. V1 — 가장 단순한 형태 & ⚠️ 빈 등록이 필수

```java
package hello.aop.order.aop;

import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;

@Slf4j
@Aspect
public class AspectV1 {

    // hello.aop.order 패키지와 그 하위 패키지
    @Around("execution(* hello.aop.order..*(..))")
    public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("[log] {}", joinPoint.getSignature());  // join point 시그니처
        return joinPoint.proceed();
    }
}
```

### ⚠️ `@Aspect`는 "표식"이지 컴포넌트 스캔 대상이 아니다

`@Aspect`만 붙이면 **아무 일도 일어나지 않는다.** 예외도, 경고 로그도 없다. 이유는 [@Aspect 노트](./aspect-aop.md#5-️-aspect만-붙이면-아무-일도-안-일어난다)에 있는 그대로 — 자동 프록시 생성기는 **"컨테이너에서 `@Aspect` 빈을 조회"** 하기 때문에, 빈이 아니면 조회 대상 자체가 아니다.

**빈으로 등록하는 3가지 방법:**

| 방법 | 코드 | 언제 |
|---|---|---|
| `@Bean` | `@Bean public AspectV1 aspectV1() {...}` | 설정 클래스에서 명시적으로 |
| `@Component` | `@Aspect @Component class ...` | **실무에서 가장 흔함** |
| `@Import` | `@Import(AspectV1.class)` | 원래는 설정 파일 추가용이지만 빈 등록도 됨 |

강의 테스트에서는 V1→V6로 갈아끼워야 해서 `@Import`를 쓴다:

```java
@Slf4j
@Import(AspectV1.class)   // ★ 이게 없으면 프록시가 안 만들어진다
@SpringBootTest
public class AopTest { ... }
```

**실행 결과 (`success()`)**
```
[log] void hello.aop.order.OrderService.orderItem(String)
[orderService] 실행
[log] String hello.aop.order.OrderRepository.save(String)
[orderRepository] 실행
```

`orderService`와 `orderRepository` **둘 다** 패턴에 걸리므로 로그가 두 번 찍힌다. 호출 흐름은 이렇게 감싸진다:

```
클라이언트 → [doLog] → orderService.orderItem() → [doLog] → orderRepository.save()
              ①                                      ②
              ④ ←─────────────────────────────────── ③
```

---

## 3. V2 — @Pointcut으로 표현식 분리

`@Around`에 표현식을 직접 쓰면 여러 어드바이스에서 같은 문자열을 복붙하게 된다. `@Pointcut`으로 이름을 붙여 재사용한다.

```java
@Slf4j
@Aspect
public class AspectV2 {

    @Pointcut("execution(* hello.aop.order..*(..))")   // ← pointcut expression
    private void allOrder() {}                          // ← pointcut signature

    @Around("allOrder()")                               // ← 시그니처로 참조
    public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("[log] {}", joinPoint.getSignature());
        return joinPoint.proceed();
    }
}
```

### @Pointcut 작성 규칙

| 항목 | 규칙 |
|---|---|
| 반환 타입 | **`void`** 여야 한다 |
| 메서드 바디 | **비워둔다** `{}` |
| 메서드 이름 + 파라미터 | 합쳐서 **포인트컷 시그니처**라 부른다 |
| 접근 제어자 | 같은 클래스 안에서만 쓰면 `private`, **다른 애스펙트에서 참조하려면 `public`** |

### ⚠️ 정정 — 파라미터는 "없어야" 하는 게 아니다

**세션에서 "파라미터는 있어도 되나?"라고 물었고 내가 "없어야 한다"고 답했는데, 그건 정확하지 않다.** 파라미터가 없는 게 *기본형*일 뿐, **파라미터 바인딩**을 하려면 선언할 수 있다. Spring 공식 문서의 예시:

```java
// 포인트컷이 Account 값을 "제공"하고, advice가 그걸 받는다
@Pointcut("execution(* com.xyz.dao.*.*(..)) && args(account,..)")
private void accountDataAccessOperation(Account account) {}

@Before("accountDataAccessOperation(account)")
public void validateAccount(Account account) {
    // account를 직접 사용
}
```

즉 **"파라미터 = 포인트컷이 advice에게 넘겨줄 값의 선언"** 이다. 값을 넘길 게 없으면 비우는 것뿐. 이게 다음 챕터(Ch.11)의 **매개변수 전달** 주제로 이어진다.

> 💡 규칙을 "금지"로 외우면 이런 오해가 생긴다. **`void`와 빈 바디는 진짜 제약이지만(포인트컷은 실행되는 메서드가 아니라 이름표니까), 파라미터는 제약이 아니라 기능이다.** 규칙을 외울 때 "왜 그런가"를 붙이면 어느 쪽인지 구분된다.

---

## 4. V3 — 포인트컷 조합 (`&&`, `||`, `!`)

트랜잭션 어드바이스를 추가한다. 단, 로그는 order 패키지 전체에, 트랜잭션은 **`*Service` 클래스에만** 걸고 싶다.

```java
@Slf4j
@Aspect
public class AspectV3 {

    // hello.aop.order 패키지와 하위 패키지
    @Pointcut("execution(* hello.aop.order..*(..))")
    public void allOrder() {}

    // 타입 이름 패턴이 *Service
    @Pointcut("execution(* *..*Service.*(..))")
    private void allService() {}

    @Around("allOrder()")
    public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("[log] {}", joinPoint.getSignature());
        return joinPoint.proceed();
    }

    // ★ 조합: order 패키지이면서 && 클래스 이름이 *Service
    @Around("allOrder() && allService()")
    public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        try {
            log.info("[트랜잭션 시작] {}", joinPoint.getSignature());
            Object result = joinPoint.proceed();
            log.info("[트랜잭션 커밋] {}", joinPoint.getSignature());
            return result;
        } catch (Exception e) {
            log.info("[트랜잭션 롤백] {}", joinPoint.getSignature());
            throw e;
        } finally {
            log.info("[리소스 릴리즈] {}", joinPoint.getSignature());
        }
    }
}
```

### 적용 결과

| 대상 | 적용되는 어드바이스 |
|---|---|
| `orderService` | `doLog()` + `doTransaction()` |
| `orderRepository` | `doLog()` 만 (`*Service` 패턴에 안 걸림) |

**실행 결과 (`success()`)**
```
[log] void hello.aop.order.OrderService.orderItem(String)
[트랜잭션 시작] void hello.aop.order.OrderService.orderItem(String)
[orderService] 실행
[log] String hello.aop.order.OrderRepository.save(String)
[orderRepository] 실행
[트랜잭션 커밋] void hello.aop.order.OrderService.orderItem(String)
[리소스 릴리즈] void hello.aop.order.OrderService.orderItem(String)
```

**실행 결과 (`exception()`)** — 커밋 대신 롤백, 릴리즈는 항상
```
...
[트랜잭션 롤백] void hello.aop.order.OrderService.orderItem(String)
[리소스 릴리즈] void hello.aop.order.OrderService.orderItem(String)
```

> 📌 이 `doTransaction()`이 흉내 내는 게 바로 [`@Transactional`](../spring/transactional.md)의 실제 동작이다. "선언적 트랜잭션"이 마법이 아니라 **이 모양의 `@Around` 어드바이스**라는 걸 여기서 확인할 수 있다.

> ⚠️ 그리고 위 실행 결과의 `[doLog] → [doTransaction]` 순서는 **보장된 게 아니라 우연이다.** 같은 조인 포인트에 걸린 advice들의 순서는 명시하지 않으면 **미정의**(공식 문서) — 지금은 이렇게 나왔지만 버전·환경이 바뀌면 뒤집힐 수 있다. 순서가 중요해지는 순간 V5(`@Order`)가 필요해지는 이유가 이것.

---

## 5. V4 — 포인트컷을 외부 클래스로 공용화

여러 `@Aspect`가 같은 포인트컷을 쓰면 별도 클래스로 모은다.

```java
package hello.aop.order.aop;

import org.aspectj.lang.annotation.Pointcut;

public class Pointcuts {

    // hello.aop.order 패키지와 하위 패키지
    @Pointcut("execution(* hello.aop.order..*(..))")
    public void allOrder() {}          // ★ 외부 참조하려면 public 필수

    // 타입 패턴이 *Service
    @Pointcut("execution(* *..*Service.*(..))")
    public void allService() {}

    // ★ 조합해서 새 포인트컷을 만들 수도 있다
    @Pointcut("allOrder() && allService()")
    public void orderAndService() {}
}
```

```java
@Slf4j
@Aspect
public class AspectV4Pointcut {

    // 패키지명을 포함한 FQCN + 포인트컷 시그니처
    @Around("hello.aop.order.aop.Pointcuts.allOrder()")
    public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("[log] {}", joinPoint.getSignature());
        return joinPoint.proceed();
    }

    @Around("hello.aop.order.aop.Pointcuts.orderAndService()")
    public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        try {
            log.info("[트랜잭션 시작] {}", joinPoint.getSignature());
            Object result = joinPoint.proceed();
            log.info("[트랜잭션 커밋] {}", joinPoint.getSignature());
            return result;
        } catch (Exception e) {
            log.info("[트랜잭션 롤백] {}", joinPoint.getSignature());
            throw e;
        } finally {
            log.info("[리소스 릴리즈] {}", joinPoint.getSignature());
        }
    }
}
```

**참조 규칙**: `패키지.클래스이름.포인트컷시그니처()` — 같은 클래스 안이면 `allOrder()`처럼 짧게, 외부면 FQCN 전체.

> ⚠️ `Pointcuts` 클래스는 `@Aspect`도 아니고 빈 등록도 필요 없다. **표현식 문자열의 보관소일 뿐**이다. (advice가 없으니 Advisor로 변환될 것도 없음)

---

## 6. 🔴 V5 — 어드바이스 순서: @Order는 클래스 단위

현재 순서는 `[doLog] → [doTransaction]`이다. 그런데 **실행 시간을 재는데 트랜잭션 시간을 빼고 싶다면** 트랜잭션이 바깥에 와야 한다: `[doTransaction] → [doLog]`.

### ⚠️ 함정: `@Order`를 메서드에 붙여도 안 먹는다

```java
@Aspect
public class AspectV5 {
    @Around("...") @Order(2)   // ❌ 아무 효과 없음
    public Object doLog(...) {...}

    @Around("...") @Order(1)   // ❌ 아무 효과 없음
    public Object doTransaction(...) {...}
}
```

Spring 공식 문서가 명시한다 — 서로 다른 aspect의 advice가 같은 조인 포인트에 걸릴 때 순서는 **명시하지 않으면 미정의**이고, 순서 지정은 `@Order` 또는 `Ordered` 인터페이스로 하되 **aspect(클래스) 단위**다. 그리고 **같은 `@Aspect` 안의 같은 타입 advice 2개는 순서를 줄 방법이 아예 없다** — javac로 컴파일된 클래스에서는 리플렉션으로 소스 선언 순서를 알 수 없기 때문.

### 해결: aspect 클래스를 쪼갠다

강의는 내부 `static class`로 분리한다 (파일을 나눠도 된다 — 학습 편의상 한 파일에 모은 것).

```java
package hello.aop.order.aop;

import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.core.annotation.Order;

@Slf4j
public class AspectV5Order {

    @Aspect
    @Order(2)                      // ★ 클래스에 붙어야 의미가 있다
    public static class LogAspect {

        @Around("hello.aop.order.aop.Pointcuts.allOrder()")
        public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[log] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }
    }

    @Aspect
    @Order(1)                      // ★ 값이 작을수록 우선순위가 높다 = 바깥쪽
    public static class TxAspect {

        @Around("hello.aop.order.aop.Pointcuts.orderAndService()")
        public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
            try {
                log.info("[트랜잭션 시작] {}", joinPoint.getSignature());
                Object result = joinPoint.proceed();
                log.info("[트랜잭션 커밋] {}", joinPoint.getSignature());
                return result;
            } catch (Exception e) {
                log.info("[트랜잭션 롤백] {}", joinPoint.getSignature());
                throw e;
            } finally {
                log.info("[리소스 릴리즈] {}", joinPoint.getSignature());
            }
        }
    }
}
```

```java
// ⚠️ @Import는 @Repeatable이 아니다 — 두 번 붙이면 컴파일 에러. 반드시 배열로:
@Import({AspectV5Order.LogAspect.class, AspectV5Order.TxAspect.class})
```

**실행 결과 — 트랜잭션이 바깥으로 나왔다**
```
[트랜잭션 시작] void hello.aop.order.OrderService.orderItem(String)
[log] void hello.aop.order.OrderService.orderItem(String)
[orderService] 실행
[log] String hello.aop.order.OrderRepository.save(String)
[orderRepository] 실행
[트랜잭션 커밋] void hello.aop.order.OrderService.orderItem(String)
[리소스 릴리즈] void hello.aop.order.OrderService.orderItem(String)
```

### 우선순위 방향 정리

| | 값 | 들어갈 때(on the way in) | 나올 때(on the way out) |
|---|---|---|---|
| 높은 우선순위 | `@Order(1)` | **먼저** 실행 | **나중에** 실행 |
| 낮은 우선순위 | `@Order(2)` | 나중에 실행 | 먼저 실행 |

즉 **우선순위가 높을수록 바깥쪽 껍질**이다. 양파 구조를 떠올리면 된다.

> 📌 [Ch.7 빈 후처리기 노트](./bean-post-processor.md)와 대비: **`BeanPostProcessor`에는 `@Order`가 안 먹고(`Ordered` 인터페이스만), `@Aspect`에는 먹는다.** 같은 AOP 계열인데 규칙이 반대라 헷갈리기 쉽다.

---

## 7. V6 — 어드바이스 5종

### 전체 코드

```java
package hello.aop.order.aop;

import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;

@Slf4j
@Aspect
public class AspectV6Advice {

    @Around("hello.aop.order.aop.Pointcuts.orderAndService()")
    public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        try {
            // @Before 자리
            log.info("[트랜잭션 시작] {}", joinPoint.getSignature());
            Object result = joinPoint.proceed();
            // @AfterReturning 자리
            log.info("[트랜잭션 커밋] {}", joinPoint.getSignature());
            return result;
        } catch (Exception e) {
            // @AfterThrowing 자리
            log.info("[트랜잭션 롤백] {}", joinPoint.getSignature());
            throw e;
        } finally {
            // @After 자리
            log.info("[리소스 릴리즈] {}", joinPoint.getSignature());
        }
    }

    // ① 대상 호출 전 — proceed()를 부를 필요가 없다(프레임워크가 알아서 호출)
    @Before("hello.aop.order.aop.Pointcuts.orderAndService()")
    public void doBefore(JoinPoint joinPoint) {
        log.info("[before] {}", joinPoint.getSignature());
    }

    // ② 정상 반환 후 — returning 이름 == 파라미터 이름
    @AfterReturning(value = "hello.aop.order.aop.Pointcuts.orderAndService()",
                    returning = "result")
    public void doReturn(JoinPoint joinPoint, Object result) {
        log.info("[return] {} return={}", joinPoint.getSignature(), result);
    }

    // ③ 예외 발생 시 — throwing 이름 == 파라미터 이름
    @AfterThrowing(value = "hello.aop.order.aop.Pointcuts.orderAndService()",
                   throwing = "ex")
    public void doThrowing(JoinPoint joinPoint, Exception ex) {
        log.info("[ex] {} message={}", joinPoint.getSignature(), ex.getMessage());
    }

    // ④ finally — 정상/예외 무관하게 항상
    @After(value = "hello.aop.order.aop.Pointcuts.orderAndService()")
    public void doAfter(JoinPoint joinPoint) {
        log.info("[after] {}", joinPoint.getSignature());
    }
}
```

### 비교표

| 애노테이션 | 시점 | 첫 파라미터 | `proceed()` | 반환값 조작 | 예외 변환 |
|---|---|---|---|---|---|
| `@Around` | 전·후 전부 | **`ProceedingJoinPoint`** | **직접 호출 필수** | ✅ 가능 | ✅ 가능 |
| `@Before` | 호출 전 | `JoinPoint` | 자동 | ❌ | ❌ (예외 던져 중단은 가능) |
| `@AfterReturning` | 정상 반환 후 | `JoinPoint` (+ `returning`) | 자동 | ❌ **읽기만** | ❌ |
| `@AfterThrowing` | 예외 발생 시 | `JoinPoint` (+ `throwing`) | 자동 | ❌ | ❌ |
| `@After` | finally (항상) | `JoinPoint` | 자동 | ❌ | ❌ |

> **`JoinPoint`는 `ProceedingJoinPoint`의 부모**다. `proceed()`는 `ProceedingJoinPoint`에만 있고, 그래서 `@Around`만 흐름을 통제할 수 있다. ([Join Point / Pointcut / PJP 3층 구분은 @Aspect 노트 §3](./aspect-aop.md#3-️-proceedingjoinpoint--pointcut이-아니다))

### 실행 순서

`@Around`가 가장 바깥, `@After`가 `@AfterReturning`보다 뒤에 온다(finally 의미):

```
[around][트랜잭션 시작]
    [before]
        --- 실제 target 실행 ---
    [return]        (정상일 때)  /  [ex]  (예외일 때)
    [after]                        ← finally 니까 항상, 그리고 return/ex 뒤
[around][트랜잭션 커밋]           (예외면 [트랜잭션 롤백])
[around][리소스 릴리즈]           ← @Around 안의 finally도 마지막에 실행된다
```

Spring 공식 문서 기준, **같은 `@Aspect` 안**에서 advice 타입 우선순위는 `@Around` > `@Before` > `@After` > `@AfterReturning` > `@AfterThrowing` 이지만, `@After`는 "after finally" 의미라 **실제 호출은 `@AfterReturning`/`@AfterThrowing`보다 뒤**다.

---

## 8. ⚠️ 함정 / 판단 기준

### 함정 1 — `returning` / `throwing`은 **이름 매칭 + 타입 필터** 두 가지 일을 한다

```java
// "result"라는 이름으로 연결 — 이 문자열과 파라미터 변수명이 반드시 같아야 한다
@AfterReturning(value = "allOrder()", returning = "result")
public void doReturn(JoinPoint joinPoint, Object result) { ... }

// 이름을 바꾸면 파라미터도 함께 바꿔야 한다
@AfterReturning(value = "allOrder()", returning = "abc")
public void doReturn(JoinPoint joinPoint, Object abc) { ... }
```

여기서 덜 알려진 게 **타입이 필터로도 작동**한다는 점이다. 공식 문서: `returning` 절은 **지정한 타입을 반환하는 메서드 실행만 매칭하도록 제한**한다.

| 선언 | 결과 |
|---|---|
| `Object result` | 모든 반환값 수용 |
| `String result` | **`String`을 반환하는 메서드에만 advice가 적용됨** |

`@AfterThrowing`의 `throwing`도 동일하다 — `Exception ex`면 대부분, `IllegalStateException ex`면 그 타입만.

> 📌 여담으로 `@AfterReturning(value = ..., returning = ...)`의 `value`와 `pointcut`은 **별칭(alias)**이다. 강의는 `value`, 공식 문서 예제는 `pointcut`을 쓴다 — 둘 다 같은 것.

> ⚠️ **세션에서 "타입이 안 맞으면 타겟이 실행이 안 되더라"고 했는데, 정확히는 그렇지 않다.** 타겟은 정상 실행되고 **advice만 조용히 건너뛴다.** 이게 더 나쁜 종류의 함정이다 — 예외가 안 나니까 "로그가 왜 안 찍히지?"로만 보인다. `@AfterReturning`이 안 도는 것 같으면 **파라미터 타입부터 `Object`로 넓혀보는 게 1차 진단**이다.

### 함정 2 — `@AfterReturning`으로는 반환값을 바꿀 수 없다

Spring 공식 문서가 못을 박는다: after returning advice에서는 **완전히 다른 참조를 반환하는 것이 불가능**하다.

```java
// ❌ 이렇게 해도 호출자가 받는 값은 안 바뀐다
@AfterReturning(value = "allOrder()", returning = "result")
public void doReturn(Object result) {
    result = "바꿔치기";   // 지역 변수만 바뀜. 의미 없음
}

// ✅ 바꾸려면 @Around
@Around("allOrder()")
public Object change(ProceedingJoinPoint joinPoint) throws Throwable {
    Object result = joinPoint.proceed();
    return "바꿔치기";     // 호출자가 이 값을 받는다
}
```

**단, 참조가 가리키는 객체의 내부 상태는 바꿀 수 있다** — `result`가 가변 객체면 `result.setName(...)`은 먹는다. "참조를 교체할 수 없다"와 "객체를 못 건드린다"는 다른 얘기다.

### 함정 3 — `@Around`에서 `proceed()`를 빼먹으면 조용히 죽는다

```java
@Around("allOrder()")
public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
    log.info("[log] {}", joinPoint.getSignature());
    return null;    // ⚠️ proceed() 없음 → 원본 메서드가 아예 실행 안 됨
}
```

`@Around`는 **target 호출 권한을 통째로 위임받는다.** 안 부르면 원본이 실행되지 않고 반환값도 `null`. 예외가 안 나서 추적이 어렵다.

- 반환 타입은 **`Object`** 로 선언할 것. `void`로 선언하면 `proceed()` 결과가 무엇이든 **호출자는 항상 `null`을 받는다**(공식 문서 명시).
- target이 primitive(`int` 등)를 반환하는데 advice가 `null`을 반환하면 `AopInvocationException: Null return value from advice does not match primitive return type`.
- 참고로 `proceed()`를 **0번·1번·여러 번 호출하는 건 전부 합법**이다. 캐시 히트 시 `proceed()` 없이 캐시값을 반환하는 게 [`@Cacheable`](../spring/spring-cache.md)의 원리. 문제는 "의도 없이" 빠뜨렸을 때.

### 함정 4 — `@Order`를 메서드에 붙이는 실수 (§6)

한 번 더: **클래스 단위.** 순서가 필요하면 aspect를 쪼갠다.

### 함정 5 — `@AfterThrowing`은 범용 예외 핸들러가 아니다

공식 문서 주의사항: `@AfterThrowing` advice는 **조인 포인트(target 메서드) 자신이 던진 예외만** 받도록 되어 있고, 같이 붙은 `@After`/`@AfterReturning` 메서드에서 발생한 예외는 받지 못한다. 예외를 **삼켜서 흐름을 바꾸려는 목적**이면 `@AfterThrowing`이 아니라 `@Around`의 `catch`를 써야 한다.

### 함정 6 — 자기호출은 여전히 안 먹는다

`@Aspect`로 문법이 편해졌을 뿐 **실행 모델은 프록시 그대로**다.

```java
@Service
public class OrderService {
    public void outer() {
        this.inner();   // ⚠️ this = target(원본) → 프록시 미경유 → advice 미적용
    }
    public void inner() { ... }
}
```

[동적 프록시 노트](./dynamic-proxy.md) · [@Transactional 노트](../spring/transactional.md)에서 다룬 그 함정.

---

## 9. 이 챕터의 발전 흐름 요약

| 버전 | 도구 | 해결한 것 | 남은 불편 |
|---|---|---|---|
| V1 | `@Around` + 인라인 표현식 | 가장 단순한 AOP | 표현식이 어드바이스마다 복붙 |
| V2 | `@Pointcut` 분리 | 표현식 재사용 | 클래스 밖에서는 못 씀 |
| V3 | `&&` 조합 | 정밀한 대상 선별 | 여러 aspect가 공유 불가 |
| V4 | `Pointcuts` 외부 클래스 | 전역 공용화 | 순서를 못 정함 |
| V5 | `@Order` + aspect 분리 | 실행 순서 제어 | `@Around`밖에 모름 |
| V6 | Advice 5종 | 의도에 맞는 advice 선택 | (포인트컷 표현식 자체 → **Ch.11**) |

**다음 챕터(Ch.11 포인트컷)**: 지금까지 `execution`만 썼지만 지시자는 10종이 있다 — `execution` · `within` · `args` · `this` · `target` · `@target` · `@within` · `@annotation` · `@args` · `bean`. 그리고 §3에서 미뤄둔 **매개변수 전달**도 거기서 다룬다.

---

## 💡 판단 기준

**"가장 강력한 것"이 아니라 "가장 덜 강력한 것"을 고른다.** Spring 공식 문서의 표현이 그대로 원칙이다 — *요구사항을 충족하는 가장 덜 강력한 형태의 advice를 항상 사용하라. before advice로 충분하면 around advice를 쓰지 말 것.* 이유는 두 가지다. ① `@Around`는 `proceed()`를 빠뜨릴 수 있고 반환값을 바꿀 수 있어서 **실수의 여지 자체가 크다.** ② 읽는 사람에게 잘못된 신호를 준다 — 로그만 남기는데 `@Around`를 쓰면 "이 메서드가 흐름을 바꿀 수도 있다"고 읽힌다. **제약이 곧 문서다.** 세션에서 "@Around가 컨트롤할 수 있는 게 많다"까지는 답했지만, 그게 **선택 이유가 아니라 회피 이유**라는 게 이 챕터의 관점.

**규칙을 "금지"로 외우지 말고 "왜"를 붙인다.** `@Pointcut`의 `void`·빈 바디는 진짜 제약이지만(실행되는 메서드가 아니라 이름표니까), 파라미터는 제약이 아니라 **바인딩 기능**이다. 세션에서 이 둘을 같은 줄에 놓고 "전부 없어야 한다"고 외웠다가 틀렸다. 규칙 하나하나에 "이게 왜 필요한가"를 붙여두면 **어디까지가 문법 제약이고 어디부터가 기능인지** 구분된다 — 그리고 그 구분이 안 되면 쓸 수 있는 기능을 못 쓴 채 지나간다.

**"조용히 안 되는" 실패는 계층을 나눠 진단한다.** 이 챕터에서만 무음 실패가 네 종류 나왔다 — 빈 등록 누락 / `@Order`를 메서드에 붙임 / `returning` 타입이 좁아 advice 스킵 / `proceed()` 누락. 전부 예외 없이 "그냥 안 된다". 실무 진단 순서는 **① `AopUtils.isAopProxy()`로 프록시 여부 → ② 포인트컷 표현식이 맞나 → ③ advice 파라미터 타입이 좁지 않나 → ④ 자기호출인가.** ①이 `false`면 등록 문제, `true`인데 안 걸리면 ②~④. **에러 메시지가 없는 영역에서는 "확인 순서"를 미리 정해두는 것 자체가 도구다.**

---

## 참고

- 김영한, 스프링 핵심 원리 고급편 — Ch.10 스프링 AOP 구현
- [Spring Framework Reference — Declaring Advice](https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/advice.html) (advice 5종 / `returning`·`throwing`은 이름 매칭 + 타입 제한 / after returning으로는 다른 참조 반환 불가 / `@After`는 "after finally" 의미 / `@AfterThrowing`은 동반 advice의 예외를 받지 않음 / around는 `Object` 반환·PJP 첫 파라미터·`void`면 항상 null / "가장 덜 강력한 advice를 쓰라" / advice ordering — 다른 aspect 간은 미정의, `@Order`/`Ordered`로 지정, 같은 aspect 내 같은 타입끼리는 순서 지정 불가)
- [Spring Framework Reference — Declaring a Pointcut](https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/pointcuts.html) (named pointcut, 포인트컷 조합, 파라미터 바인딩)
- [Spring Framework Reference — Proxying Mechanisms](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html) (자기호출 미인터셉트)

**학습 날짜**: 2026-08-14 · **계기**: 김영한 고급편 Ch.10 수강 후 Claude 소크라테스 복습 세션(5문항). ① `@Aspect` 빈 등록 필요성·`@Import` 방법 **정확히 답함** ② `@Pointcut` 분리 방식과 `@Around`에서의 참조 **정확히 답함** — 단 "파라미터는 있어도 되나?"라는 본인 질문에 내가 "없어야 한다"고 잘못 답했고, 세션 후 공식 문서 확인 결과 **파라미터 바인딩용으로 선언 가능**이 맞아 §3에 정정 수록 ③ `@Order`가 클래스 단위 + `static class` 분리 전략 **정확히 답함** ④ Advice 5종 중 **`@After` 이름을 떠올리지 못함**("final처럼 작동하는 게 있었는데") → §7에 finally 의미로 정리 / `proceed()`를 "process"로 부름 → 용어 정정 ⑤ `@AfterReturning`이 반환값을 **바꿀 수 없다는 판단은 정확했음**(근거도 "리턴 이후에 실행되니까"로 맞음) — 다만 `returning` 속성 문법은 기억 못 함, 그리고 "타입이 안 맞으면 타겟이 실행 안 된다"고 했는데 실제로는 **타겟은 정상 실행되고 advice만 스킵**이라 §8 함정 1에 정정. 📌 세션 후 조사에서 추가로 확인: `returning`/`throwing`의 **타입 필터 역할** / `@AfterThrowing`은 동반 advice의 예외를 못 받음 / `@After`가 `@AfterReturning`보다 뒤에 호출되는 이유 / around advice를 `void`로 선언하면 항상 null 반환

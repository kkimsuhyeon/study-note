# 스프링 AOP 포인트컷 — 지시자 10종 · 정적/동적 매칭 · 파라미터 바인딩

> **한 줄 요약**: [AOP 구현](./aop-implementation.md)까지 `execution` 하나로 버텼다면, 이 챕터는 **나머지 9개 지시자가 각각 "무엇을 보고 판단하는가"** 를 정리한다. 핵심은 지시자를 외우는 게 아니라 **판단 기준의 축 3개**를 잡는 것이다 — ① **정적(선언된 정보) vs 동적(런타임 실제 객체)** ② **프록시 vs 타깃** ③ **인스턴스의 클래스 vs 메서드가 선언된 클래스**. ⚠️ **`args`·`@args`·`@target`은 단독으로 쓰면 안 된다** — 런타임에만 판단 가능해서 스프링이 **모든 빈에 프록시를 만들려고 시도**하고, 그 과정에서 `final` 내부 빈에 걸려 기동이 깨질 수 있다. ⚠️ 그리고 **`this`는 프록시 종류에 따라 결과가 달라지는 유일한 지시자**다 — JDK 동적 프록시에서 `this(구체클래스)`만 매칭에 실패한다.

관련 노트: [AOP 구현](./aop-implementation.md) (직전 챕터 — `@Pointcut` 분리·조합·`@Order`·Advice 5종) · [@Aspect AOP](./aspect-aop.md) (`@Aspect` → Advisor 변환) · [AOP 개념](./aop-concepts.md) (조인 포인트·위빙 용어 체계) · [빈 후처리기](./bean-post-processor.md) (자동 프록시 생성기 — 이 챕터의 "모든 빈에 프록시 시도" 문제의 무대) · [ProxyFactory](./proxy-factory.md) (`Pointcut = ClassFilter + MethodMatcher`) · [동적 프록시](./dynamic-proxy.md) (**JDK vs CGLIB 구조 — `this`/`target` 이해의 전제**) · [프록시/데코레이터 패턴](./proxy-decorator-pattern.md) · [ThreadLocal](../concurrency/thread-local.md) (트랙 10 출발점) · [커스텀 어노테이션](../annotation/custom-annotation.md) (`@annotation` 지시자의 실무 용도) · [@Transactional](../spring/transactional.md) · [Spring Cache](../spring/spring-cache.md) · [메서드 보안](../security/method-security.md)

---

## 1. 포인트컷 지시자(PCD)란

포인트컷 표현식은 **포인트컷 지시자**(Pointcut Designator, 줄여서 **PCD**)로 시작한다. `execution(...)`의 `execution`이 바로 지시자다.

이 표현식 문법은 스프링이 만든 게 아니라 **AspectJ의 포인트컷 표현식**을 빌려 쓰는 것이다. ([AOP 개념 노트](./aop-concepts.md)에서 정리한 "AspectJ 문법 차용 ≠ AspectJ 직접 사용"이 이것)

### 지시자 10종 한눈에 보기

| 지시자 | 무엇을 보는가 | 판단 시점 | 실무 빈도 |
|---|---|---|---|
| **`execution`** | **메서드 시그니처**(선언된 정보) | 정적 | ⭐⭐⭐ 압도적 1위 |
| `within` | 타입(클래스/패키지) — 정확히 일치 | 정적 | ⭐⭐ 범위 좁히기 |
| `args` | **실제 넘어온 인자 객체** | **동적** | ⭐ 바인딩용 |
| `this` | **프록시 객체**의 타입 | 정적* | ⭐ 바인딩용 |
| `target` | **타깃 객체**의 타입 | 정적* | ⭐ 바인딩용 |
| `@target` | **인스턴스 클래스**의 애노테이션 | **동적** | ⭐ |
| `@within` | **메서드 선언 클래스**의 애노테이션 | 정적 | ⭐ |
| `@annotation` | **메서드**의 애노테이션 | 정적 | ⭐⭐⭐ 커스텀 AOP 단골 |
| `@args` | 실제 인자의 **런타임 타입**의 애노테이션 | **동적** | 거의 안 씀 |
| `bean` | **스프링 빈 이름** | 정적 | ⭐ 스프링 전용 |

> \* `this`/`target`은 타입 판단이지만 프록시가 만들어진 뒤에야 확정되므로 강의에서는 "실제 객체를 만들어야 테스트 가능"하다고 설명한다.

### ⚠️ 스프링 AOP가 지원하지 않는 AspectJ 지시자

AspectJ 원본 언어에는 훨씬 많은 지시자가 있지만 스프링 AOP는 **메서드 실행 조인 포인트만** 지원하므로 다음은 못 쓴다:

```
call, get, set, preinitialization, staticinitialization, initialization,
handler, adviceexecution, withincode, cflow, cflowbelow, if, @this, @withincode
```

이걸 스프링 AOP 표현식에 쓰면 **`IllegalArgumentException`** 이 발생한다. (필드 접근 `get`/`set`이나 생성자 `initialization`을 가로채고 싶다면 프록시 방식으로는 불가능 — AspectJ 로드 타임 위빙으로 가야 한다. [AOP 개념 노트](./aop-concepts.md)의 "위빙 3방식" 참고)

> 📌 `bean`은 반대로 **스프링에만 있고 AspectJ에는 없는** 지시자다. AspectJ는 타입 레벨에서만 동작하는데, `bean`은 스프링 빈 팩토리와 밀접하게 통합되어 **인스턴스 레벨**로 동작한다 — 이건 프록시 기반 AOP만이 가진 능력이다.

---

## 2. 예제 프로젝트 전체 코드

이 챕터의 모든 예제가 이 코드 위에서 돈다. ([AOP 구현 노트](./aop-implementation.md)의 `hello.aop` 프로젝트에 `member` 패키지를 추가한 것)

### 애노테이션 2종

```java
package hello.aop.member.annotation;

import java.lang.annotation.*;

@Target(ElementType.TYPE)          // ★ 클래스에 붙는 애노테이션
@Retention(RetentionPolicy.RUNTIME) // ★ RUNTIME 필수 (기본값 CLASS면 AOP가 못 읽는다)
public @interface ClassAop {
}
```

```java
package hello.aop.member.annotation;

import java.lang.annotation.*;

@Target(ElementType.METHOD)        // ★ 메서드에 붙는 애노테이션
@Retention(RetentionPolicy.RUNTIME)
public @interface MethodAop {
    String value();
}
```

> ⚠️ **`@Retention(RUNTIME)`을 빼면 AOP가 조용히 안 걸린다.** 애노테이션의 기본 `@Retention`은 `CLASS`라서 런타임에 리플렉션으로 읽을 수 없다. 자세한 건 [커스텀 어노테이션 노트](../annotation/custom-annotation.md).

### 인터페이스 + 구현체

```java
package hello.aop.member;

public interface MemberService {
    String hello(String param);
}
```

```java
package hello.aop.member;

import hello.aop.member.annotation.ClassAop;
import hello.aop.member.annotation.MethodAop;
import org.springframework.stereotype.Component;

@ClassAop                              // ★ 클래스 애노테이션 → @target, @within 실험용
@Component
public class MemberServiceImpl implements MemberService {

    @Override
    @MethodAop("test value")           // ★ 메서드 애노테이션 → @annotation 실험용
    public String hello(String param) {
        return "ok";
    }

    // ★ 인터페이스에 없는 메서드 → execution의 부모 타입 매칭 실험용
    public String internal(String param) {
        return "ok";
    }
}
```

이 구조가 중요한 이유:

| 구성 요소 | 무엇을 실험하기 위한 장치인가 |
|---|---|
| `MemberService` 인터페이스 | `execution`의 **부모 타입 허용** / `within`의 **부모 타입 불가** |
| `internal(String)` (인터페이스에 없음) | 부모 타입 선언 시 **부모에 없는 메서드는 매칭 실패** |
| `@ClassAop` (클래스) | `@target` / `@within` |
| `@MethodAop` (메서드) | `@annotation` |
| `hello(String param)` | `execution(* *(Object))` **실패** vs `args(Object)` **성공** |

### 테스트 뼈대

```java
package hello.aop.pointcut;

import hello.aop.member.MemberServiceImpl;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.aop.aspectj.AspectJExpressionPointcut;

import java.lang.reflect.Method;

import static org.assertj.core.api.Assertions.assertThat;

public class ExecutionTest {

    AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
    Method helloMethod;

    @BeforeEach
    public void init() throws NoSuchMethodException {
        helloMethod = MemberServiceImpl.class.getMethod("hello", String.class);
    }

    @Test
    void printMethod() {
        // public java.lang.String hello.aop.member.MemberServiceImpl.hello(java.lang.String)
        System.out.println("helloMethod = " + helloMethod);
    }
}
```

> 📌 **`AspectJExpressionPointcut`** — 스프링 컨테이너 없이 포인트컷 표현식만 단독 테스트할 수 있는 클래스. `setExpression(표현식)` 후 `matches(메서드, 대상클래스)`로 `true`/`false`를 받는다. **표현식이 헷갈릴 때 학습 테스트로 찍어보는 게 제일 빠르다.**
>
> ⚠️ 단, `AspectJExpressionPointcut`에는 **표현식을 한 번만** 지정할 수 있다. 여러 표현식을 한 테스트에서 비교하려면 강의처럼 생성 메서드를 만든다:
> ```java
> private AspectJExpressionPointcut pointcut(String expression) {
>     AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
>     pointcut.setExpression(expression);
>     return pointcut;
> }
> ```

---

## 3. execution — 문법

```
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern) throws-pattern?)
execution(접근제어자?        반환타입          선언타입?               메서드이름(파라미터)        예외?)
```

- `?`가 붙은 부분은 **생략 가능** (접근제어자·선언타입·예외)
- **생략 불가**: 반환타입 · 메서드이름 · 파라미터
- `*` 패턴 사용 가능

### 가장 정확한 표현식 → 가장 느슨한 표현식

```java
// 모든 내용이 정확히 매칭
@Test
void exactMatch() {
    pointcut.setExpression(
        "execution(public String hello.aop.member.MemberServiceImpl.hello(String))");
    assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

// 가장 느슨한 표현식 — 접근제어자·선언타입·예외 전부 생략
@Test
void allMatch() {
    pointcut.setExpression("execution(* *(..))");
    assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}
```

### 메서드 이름 패턴

```java
@Test
void nameMatchStar1() {
    pointcut.setExpression("execution(* hel*(..))");     // hello → 매칭
    assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void nameMatchStar2() {
    pointcut.setExpression("execution(* *el*(..))");     // 중간 포함도 가능
    assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void nameMatchFalse() {
    pointcut.setExpression("execution(* nono(..))");
    assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isFalse();
}
```

### 패키지 매칭 — `.` 과 `..` 의 차이 (⚠️ 자주 틀림)

```java
// 정확한 패키지 — 해당 패키지에 "직접" 있는 클래스만
@Test
void packageExactMatch1() {
    pointcut.setExpression("execution(* hello.aop.member.MemberServiceImpl.hello(..))");
    assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

// hello.aop.member 패키지의 모든 클래스, 모든 메서드
@Test
void packageExactMatch2() {
    pointcut.setExpression("execution(* hello.aop.member.*.*(..))");
    assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

// ⚠️ hello.aop 패키지에 "직접" 있는 클래스만 → member 하위 패키지는 제외 → 실패
@Test
void packageExactFalse() {
    pointcut.setExpression("execution(* hello.aop.*.*(..))");
    assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isFalse();
}

// ★ `..` = 해당 패키지 + 그 하위 패키지 전부 → 성공
@Test
void packageMatchSubPackage1() {
    pointcut.setExpression("execution(* hello.aop.member..*.*(..))");
    assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void packageMatchSubPackage2() {
    pointcut.setExpression("execution(* hello.aop..*.*(..))");
    assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}
```

| 패턴 | 의미 |
|---|---|
| `.` | **정확히 그 위치**의 패키지. `hello.aop.*`는 `hello.aop`에 직접 있는 클래스만 |
| `..` | **그 패키지 + 모든 하위 패키지** |

> 💡 실무 표현식이 거의 항상 `hello.aop..*(..)` 모양인 이유가 이것 — `..`을 써야 하위 패키지까지 걸린다. **`.` 하나 차이로 AOP가 조용히 안 걸리는** 대표적인 오타 지점.

### 파라미터 매칭 규칙

```java
pointcut.setExpression("execution(* *(String))");       // 정확히 String 1개
pointcut.setExpression("execution(* *())");             // 파라미터 없음
pointcut.setExpression("execution(* *(*))");            // 정확히 1개, 타입 무관
pointcut.setExpression("execution(* *(*, *))");         // 정확히 2개, 타입 무관
pointcut.setExpression("execution(* *(..))");           // 0개 이상, 타입 무관
pointcut.setExpression("execution(* *(String, ..))");   // String으로 시작, 이후 자유
```

| 패턴 | 의미 |
|---|---|
| `(String)` | 정확히 `String` 타입 파라미터 1개 |
| `()` | 파라미터 **없어야** 함 |
| `(*)` | 정확히 1개, 모든 타입 허용 |
| `(*, *)` | 정확히 2개, 모든 타입 허용 |
| `(..)` | 개수 무관(0개 포함), 모든 타입 허용 — `0..*` 로 이해 |
| `(String, ..)` | `String`으로 시작, 이후 개수·타입 무관 |

---

## 4. execution — 타입 매칭과 상속 (⚠️ 함정 지대)

### 부모 타입을 선언해도 자식이 매칭된다

```java
@Test
void typeExactMatch() {
    pointcut.setExpression("execution(* hello.aop.member.MemberServiceImpl.*(..))");
    assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void typeMatchSuperType() {
    // ★ 부모(인터페이스)를 선언했는데 자식 구현체가 매칭된다
    pointcut.setExpression("execution(* hello.aop.member.MemberService.*(..))");
    assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
}
```

**다형성에서 `부모타입 = 자식타입` 할당이 가능**한 것과 같은 원리다.

### ⚠️ 단, 부모 타입에 **없는 메서드**는 매칭 실패

```java
@Test
void typeMatchInternal() throws NoSuchMethodException {
    // 구체 클래스를 선언 → internal()도 매칭 성공
    pointcut.setExpression("execution(* hello.aop.member.MemberServiceImpl.*(..))");
    Method internalMethod = MemberServiceImpl.class.getMethod("internal", String.class);
    assertThat(pointcut.matches(internalMethod, MemberServiceImpl.class)).isTrue();
}

@Test
void typeMatchNoSuperTypeMethodFalse() throws NoSuchMethodException {
    // ★ 부모(MemberService)를 선언 → 부모에 없는 internal()은 매칭 실패
    pointcut.setExpression("execution(* hello.aop.member.MemberService.*(..))");
    Method internalMethod = MemberServiceImpl.class.getMethod("internal", String.class);
    assertThat(pointcut.matches(internalMethod, MemberServiceImpl.class)).isFalse();
}
```

> **규칙**: 표현식에 부모 타입을 선언하면 **그 부모 타입에 선언된 메서드**만 매칭 대상이 된다. `hello(String)`은 인터페이스에 있으니 성공, `internal(String)`은 인터페이스에 없으니 실패.
>
> 💡 이게 실무에서 의미하는 것: **인터페이스 기준으로 포인트컷을 쓰면 "인터페이스에 노출된 메서드에만" AOP가 걸린다.** 구현체에만 있는 공개 메서드는 조용히 빠진다. AOP가 일부 메서드에만 안 걸린다면 **표현식의 타입이 인터페이스인지 구현체인지**부터 확인할 것.
>
> 📌 한 발 더 — **JDK 동적 프록시 환경에서는 `internal()`이 "매칭 실패" 이전에 프록시로 호출하는 것 자체가 불가능하다.** JDK 프록시는 인터페이스만 구현하므로 프록시 객체에 `internal()` 메서드가 아예 없다(공식 문서: JDK 프록시는 **public 인터페이스 메서드만** 인터셉트, CGLIB는 public·protected까지). 스프링 부트 기본값인 CGLIB에서는 구체 클래스를 상속하니 `internal()`도 프록시를 통해 호출·인터셉트된다 — **같은 코드가 프록시 기술에 따라 AOP 적용 범위가 달라지는** 또 하나의 사례고, §9의 `this` 문제와 뿌리가 같다. (Ch.13 실무 주의사항에서 본격적으로 다룬다)

---

## 5. within — 타입 범위 좁히기 (부모 타입 불가)

`within`은 **특정 타입 내부의 조인 포인트**를 매칭한다. 그 타입 안에 있는 메서드는 전부 대상.

```java
public class WithinTest {

    AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
    Method helloMethod;

    @BeforeEach
    public void init() throws NoSuchMethodException {
        helloMethod = MemberServiceImpl.class.getMethod("hello", String.class);
    }

    @Test
    void withinExact() {
        pointcut.setExpression("within(hello.aop.member.MemberServiceImpl)");
        assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
    }

    @Test
    void withinStar() {
        pointcut.setExpression("within(hello.aop.member.*Service*)");
        assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
    }

    @Test
    void withinSubPackage() {
        pointcut.setExpression("within(hello.aop..*)");
        assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();
    }

    // 🔴 여기가 execution과 갈리는 지점
    @Test
    @DisplayName("타깃의 타입에만 직접 적용, 인터페이스를 선정하면 안 된다.")
    void withinSuperTypeFalse() {
        pointcut.setExpression("within(hello.aop.member.MemberService)");
        assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isFalse();  // ❌
    }

    @Test
    @DisplayName("execution은 타입 기반, 인터페이스를 선정 가능.")
    void executionSuperTypeTrue() {
        pointcut.setExpression("execution(* hello.aop.member.MemberService.*(..))");
        assertThat(pointcut.matches(helloMethod, MemberServiceImpl.class)).isTrue();   // ✅
    }
}
```

### execution vs within

| | 부모 타입(인터페이스) 지정 | 메서드 시그니처 지정 | 성격 |
|---|---|---|---|
| `execution` | ✅ 매칭 (다형성 적용) | ✅ 반환타입·이름·파라미터 전부 | **kinded**(어떤 종류의 조인 포인트인가) |
| `within` | ❌ **실패** — 정확한 타입만 | ❌ 타입 단위만 | **scoping**(어느 범위인가) |

> 💡 **`within`은 "패키지·클래스 범위를 빠르게 자르는 도구"**로 이해하면 된다. 공식 문서도 `within(com.xyz.service..*)` 같은 **레이어 구분용 공용 포인트컷** 예시로 `within`을 쓴다. 메서드 조건은 `execution`이, 범위 자르기는 `within`이 담당하는 역할 분담. (→ §12 "좋은 포인트컷 쓰기")

---

## 6. 🔴 args — 정적 매칭 vs 동적 매칭

문법은 `execution`의 파라미터 부분과 똑같다. 그런데 **결과가 다르다.**

```java
public class ArgsTest {

    Method helloMethod;   // MemberServiceImpl.hello(String param)

    @BeforeEach
    public void init() throws NoSuchMethodException {
        helloMethod = MemberServiceImpl.class.getMethod("hello", String.class);
    }

    private AspectJExpressionPointcut pointcut(String expression) {
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
        pointcut.setExpression(expression);
        return pointcut;
    }

    @Test
    void args() {
        assertThat(pointcut("args(String)").matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("args(Object)").matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("args()").matches(helloMethod, MemberServiceImpl.class)).isFalse();
        assertThat(pointcut("args(..)").matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("args(*)").matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("args(String,..)").matches(helloMethod, MemberServiceImpl.class)).isTrue();
    }

    /**
     * execution(* *(java.io.Serializable)): 메서드의 시그니처로 판단 (정적)
     * args(java.io.Serializable):           런타임에 전달된 인수로 판단 (동적)
     */
    @Test
    void argsVsExecution() {
        // args — 부모 타입 전부 허용
        assertThat(pointcut("args(String)").matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("args(java.io.Serializable)").matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("args(Object)").matches(helloMethod, MemberServiceImpl.class)).isTrue();

        // execution — 시그니처와 정확히 일치해야 함
        assertThat(pointcut("execution(* *(String))").matches(helloMethod, MemberServiceImpl.class)).isTrue();
        assertThat(pointcut("execution(* *(java.io.Serializable))").matches(helloMethod, MemberServiceImpl.class)).isFalse();  // ❌
        assertThat(pointcut("execution(* *(Object))").matches(helloMethod, MemberServiceImpl.class)).isFalse();               // ❌
    }
}
```

### 왜 결과가 다른가 — **정적 vs 동적**

`String`은 `Object`와 `java.io.Serializable`의 **하위 타입**이다. 그런데:

```
메서드 선언:  public String hello(String param)
                                  ~~~~~~ ← 클래스 파일에 "String"이라고 적혀 있음

실제 호출:    memberService.hello("helloA")
                                  ~~~~~~~~ ← 런타임에 넘어온 건 String 인스턴스
```

| | 무엇을 보는가 | 판단 시점 | 부모 타입 |
|---|---|---|---|
| **`execution(* *(Object))`** | **메서드 시그니처**(클래스에 선언된 정보) | **컴파일된 정보만으로** = **정적** | ❌ 정확히 일치해야 함 → `String` ≠ `Object` |
| **`args(Object)`** | **실제로 넘어온 인자 객체 인스턴스** | **호출이 일어나야** = **동적** | ✅ `"helloA" instanceof Object` → true |

**한 문장 정리** (면접용):

> **`execution`은 메서드 시그니처(선언된 정보)를 보는 정적 매칭이고, `args`는 실제 넘어온 객체 인스턴스를 `instanceof`로 보는 동적 매칭이다. 그래서 `args`만 부모 타입을 허용한다.**

공식 문서도 같은 구분을 명시한다 — `args(java.io.Serializable)`은 **런타임에 전달된 인수가 `Serializable`이면** 매칭이고, `execution(* *(java.io.Serializable))`은 **메서드 시그니처가 `Serializable` 파라미터 하나를 선언했을 때** 매칭이라고.

> ⚠️ **"정적/동적"은 단순한 수식어가 아니라 §11(단독 사용 금지)의 원인이다.** 동적 판단은 프록시가 있어야만 가능하고, 프록시를 만들지 말지는 로딩 시점에 결정해야 한다 — 이 순환이 §11의 전부다.

---

## 7. 🔴 @target vs @within — 상속 관점 (this/target과 헷갈리기 쉬움)

```java
@target(hello.aop.member.annotation.ClassAop)
@within(hello.aop.member.annotation.ClassAop)
```

둘 다 **타입(클래스)에 붙은 애노테이션**으로 판단한다. 차이는 **상속 계층에서 어디까지 적용하느냐**다.

### ⚠️ 먼저 오해부터 정리 — 이건 프록시 이야기가 **아니다**

> **세션에서 "@target은 진짜 객체, @within은 프록시 객체 아닌가?"라고 답했는데, 그건 `this` vs `target`(§9) 이야기다.**
>
> 이름이 비슷해서 생기는 착각인데, **축이 완전히 다르다**:
> - `this` vs `target` → **프록시냐 타깃이냐** (§9)
> - `@target` vs `@within` → **인스턴스의 클래스냐, 메서드가 선언된 클래스냐** (여기)
>
> `@target`/`@within`은 프록시와 아무 상관 없다. **`@`가 붙으면 "애노테이션으로 판단한다"는 뜻일 뿐**, 앞의 단어 의미가 그대로 이어지지 않는다.

### 정의

| 지시자 | 판단 기준 | 결과 |
|---|---|---|
| **`@target`** | **실행되는 인스턴스의 클래스**에 애노테이션이 있는가 | 있으면 **부모에서 상속받은 메서드까지 전부** 적용 |
| **`@within`** | **그 메서드가 선언된 클래스**에 애노테이션이 있는가 | **그 클래스에 직접 선언된 메서드만** 적용 |

### 전체 예제 코드

```java
package hello.aop.pointcut;

import hello.aop.member.annotation.ClassAop;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;

@Slf4j
@Import(AtTargetAtWithinTest.Config.class)
@SpringBootTest
public class AtTargetAtWithinTest {

    @Autowired
    Child child;

    @Test
    void success() {
        log.info("child Proxy={}", child.getClass());
        child.childMethod();   // 부모, 자식 모두 있는 메서드
        child.parentMethod();  // 부모 클래스에만 있는 메서드
    }

    static class Config {
        @Bean public Parent parent() { return new Parent(); }
        @Bean public Child child() { return new Child(); }
        @Bean public AtTargetAtWithinAspect atTargetAtWithinAspect() {
            return new AtTargetAtWithinAspect();
        }
    }

    static class Parent {
        public void parentMethod() {}       // 부모에만 있는 메서드
    }

    @ClassAop                                // ★ 애노테이션은 Child에만!
    static class Child extends Parent {
        public void childMethod() {}
    }

    @Slf4j
    @Aspect
    static class AtTargetAtWithinAspect {

        // @target: 인스턴스 기준 → 모든 메서드의 조인 포인트를 선정, 부모 타입의 메서드도 적용
        @Around("execution(* hello.aop..*(..)) && @target(hello.aop.member.annotation.ClassAop)")
        public Object atTarget(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[@target] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }

        // @within: 선택된 클래스 내부에 있는 메서드만 조인 포인트로 선정, 부모 타입의 메서드는 적용 안 됨
        @Around("execution(* hello.aop..*(..)) && @within(hello.aop.member.annotation.ClassAop)")
        public Object atWithin(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[@within] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }
    }
}
```

### 실행 결과

```
[@target] void hello.aop.pointcut.AtTargetAtWithinTest$Child.childMethod()
[@within] void hello.aop.pointcut.AtTargetAtWithinTest$Child.childMethod()
[@target] void hello.aop.pointcut.AtTargetAtWithinTest$Parent.parentMethod()
                                                       ↑
                            [@within]이 없다! ← parentMethod()는 Parent에 선언되어 있고
                                                Parent에는 @ClassAop이 없으니까
```

### 판단 흐름

```
child.parentMethod() 호출

@target(ClassAop) → "이 인스턴스의 클래스(Child)에 @ClassAop 있나?" → 있다 → ✅ 적용
@within(ClassAop) → "parentMethod()가 선언된 클래스(Parent)에 @ClassAop 있나?" → 없다 → ❌ 미적용
```

| | `childMethod()` (Child 선언) | `parentMethod()` (Parent 선언) |
|---|---|---|
| `@target(ClassAop)` | ✅ | ✅ |
| `@within(ClassAop)` | ✅ | ❌ |

> 💡 **비유**: `@target`은 **"너 어느 팀 소속이지? Child 팀? 거기 인증 도장 있으니까 네가 하는 일 전부 통과"**, `@within`은 **"이 업무 매뉴얼이 실제로 작성된 부서가 어디인데? Parent? 거기엔 도장 없네, 반려"**.
>
> 공식 문서 표현으로는 `@target`은 *"the class of the **executing object**"*, `@within`은 *"the **declared type** of the target object"* — **executing(실행 중인 인스턴스) vs declared(선언된 타입)** 이 갈림길이다.

---

## 8. @annotation, @args

### @annotation — 실무에서 제일 많이 쓰는 애노테이션 지시자

**메서드**에 애노테이션이 붙어 있으면 매칭한다.

```java
public class MemberServiceImpl {
    @MethodAop("test value")
    public String hello(String param) {
        return "ok";
    }
}
```

```java
package hello.aop.pointcut;

import hello.aop.member.MemberService;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;

@Slf4j
@Import(AtAnnotationTest.AtAnnotationAspect.class)
@SpringBootTest
public class AtAnnotationTest {

    @Autowired
    MemberService memberService;

    @Test
    void success() {
        log.info("memberService Proxy={}", memberService.getClass());
        memberService.hello("helloA");
    }

    @Slf4j
    @Aspect
    static class AtAnnotationAspect {
        @Around("@annotation(hello.aop.member.annotation.MethodAop)")
        public Object doAtAnnotation(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[@annotation] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }
    }
}
```

**실행 결과**
```
[@annotation] String hello.aop.member.MemberService.hello(String)
```

> 📌 **`@annotation`이 바로 커스텀 AOP 애노테이션의 정체다.** `@Transactional`, `@Cacheable`, `@PreAuthorize`가 전부 이 방식으로 걸린다 — "이 애노테이션이 붙은 메서드만 가로채라". [커스텀 어노테이션 노트](../annotation/custom-annotation.md)의 "AOP 처리 방식"이 이것이고, 실무에서 직접 만드는 AOP는 대부분 `@annotation` + 파라미터 바인딩(§10) 조합이다.
>
> ⚠️ `@target`/`@within`(클래스 애노테이션)과 `@annotation`(메서드 애노테이션)의 대상이 다르다는 걸 놓치기 쉽다. 애노테이션 선언의 `@Target(ElementType.TYPE)` vs `@Target(ElementType.METHOD)`와 짝을 이룬다.

### @args — 실무에서 거의 안 씀

**전달된 실제 인수의 런타임 타입**에 주어진 애노테이션이 있으면 매칭한다.

```java
@args(test.Check)
```

즉 `someMethod(파라미터)`를 호출했을 때, 넘어온 그 객체의 클래스에 `@Check`가 붙어 있으면 적용. **런타임 판단이라 단독 사용 금지 대상**(§11)이다.

---

## 9. 🔴 this vs target — 프록시 종류에 따라 결과가 달라진다

이 챕터에서 **가장 헷갈리는 지점**이자, 앞의 [동적 프록시 노트](./dynamic-proxy.md)를 안 읽었으면 이해가 안 되는 부분이다.

### 정의

| 지시자 | 대상 | 스프링 빈으로 등록된 것 |
|---|---|---|
| **`this`** | **프록시 객체** (스프링 AOP 프록시) | 이게 스프링 빈 |
| **`target`** | **타깃 객체** (프록시가 감싸고 있는 실제 대상) | 프록시 내부에 숨어 있음 |

공통 규칙:
- **타입 하나를 정확히 지정**해야 한다. `*` 같은 패턴 **사용 불가**
- **부모 타입 허용** (다형성 적용)

> 📌 참고로 **AspectJ에서는 `this`와 `target`이 같은 객체**를 가리킨다 — AspectJ는 프록시를 안 만들고 바이트코드에 직접 위빙하기 때문에 "프록시 객체"라는 게 존재하지 않는다. **`this` ≠ `target`은 프록시 기반인 스프링 AOP만의 구분**이다. (공식 문서 명시)

### 왜 프록시 종류에 따라 달라지는가

```
[ JDK 동적 프록시 ]                        [ CGLIB ]

MemberService (인터페이스)                 MemberServiceImpl (구체 클래스)
      ↑ implements                              ↑ extends (상속!)
$Proxy53          ← this가 보는 것       MemberServiceImpl$$EnhancerBySpringCGLIB
      │                                          │
      ↓ 내부에 들고 있음                          ↓
MemberServiceImpl ← target이 보는 것      MemberServiceImpl
```

**핵심**: JDK 동적 프록시가 만든 `$Proxy53`은 `MemberService` 인터페이스를 구현한 **완전히 새로운 클래스**다. `MemberServiceImpl`과는 **아무 상속 관계가 없다.** 반면 CGLIB 프록시는 `MemberServiceImpl`을 **상속**하므로 `MemberServiceImpl` 타입이기도 하다.

### ✅ 8칸 비교표 (정답)

| 프록시 종류 | `this(MemberService)` | `this(MemberServiceImpl)` | `target(MemberService)` | `target(MemberServiceImpl)` |
|---|---|---|---|---|
| **JDK 동적 프록시** | ✅ | **❌** | ✅ | ✅ |
| **CGLIB** | ✅ | ✅ | ✅ | ✅ |

**X는 딱 한 칸.** `JDK 동적 프록시` × `this(구체 클래스)` 뿐이다.

이유를 한 줄씩:

| 케이스 | 판단 | 결과 |
|---|---|---|
| JDK + `this(MemberService)` | 프록시(`$Proxy53`)가 `MemberService`를 구현했나? → 그렇다 | ✅ |
| JDK + `this(MemberServiceImpl)` | 프록시(`$Proxy53`)가 `MemberServiceImpl` 타입인가? → **아니다. 인터페이스만 구현한 남남** | **❌** |
| JDK + `target(...)` | 타깃(`MemberServiceImpl`)이 그 타입인가? → 둘 다 그렇다 | ✅✅ |
| CGLIB + `this(MemberServiceImpl)` | 프록시가 `MemberServiceImpl`을 **상속**했나? → 그렇다 (부모 타입 허용) | ✅ |

> ⚠️ **세션에서 나온 오해 정정**: "`target(MemberServiceImpl)`로 하면 CGLIB가 사용될 것 같다"고 답했는데, **`this`/`target`은 프록시 생성 방식을 결정하지 않는다.** 이미 만들어진 프록시/타깃 중 **어느 쪽을 보고 매칭할지**만 정한다. 프록시 기술 선택은 전적으로 `spring.aop.proxy-target-class` 설정(과 인터페이스 유무)이 결정한다. **"지시자는 판단 대상을 고르는 것이지 프록시를 만드는 것이 아니다."**
>
> 📌 **영상 오류 정정**(강의 자료에 명시): 영상에서 "JDK Proxy는 `MemberService`를 알 수 없다"고 설명했는데, **`MemberServiceImpl`을 알 수 없다**가 맞다.

### 전체 예제 코드

```java
package hello.aop.pointcut;

import hello.aop.member.MemberService;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;

/**
 * application.properties에 넣는 대신 이 테스트에서만 임시 적용
 * proxy-target-class=false → JDK 동적 프록시 우선 (단, 인터페이스가 없으면 CGLIB)
 * proxy-target-class=true  → 항상 CGLIB (생략 시 스프링 부트 기본값)
 */
@Slf4j
@Import(ThisTargetTest.ThisTargetAspect.class)
@SpringBootTest(properties = "spring.aop.proxy-target-class=false")   // JDK 동적 프록시
//@SpringBootTest(properties = "spring.aop.proxy-target-class=true")  // CGLIB
public class ThisTargetTest {

    @Autowired
    MemberService memberService;

    @Test
    void success() {
        log.info("memberService Proxy={}", memberService.getClass());
        memberService.hello("helloA");
    }

    @Slf4j
    @Aspect
    static class ThisTargetAspect {

        // this: 스프링 AOP 프록시 객체 대상 / 부모 타입(인터페이스) 지정
        @Around("this(hello.aop.member.MemberService)")
        public Object doThisInterface(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[this-interface] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }

        // target: 실제 target 객체 대상 / 부모 타입(인터페이스) 지정
        @Around("target(hello.aop.member.MemberService)")
        public Object doTargetInterface(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[target-interface] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }

        // ★ this + 구체 클래스 → JDK 동적 프록시일 때만 매칭 실패
        // JDK 동적 프록시는 인터페이스 기반으로 생성되므로 구현 클래스를 알 수 없음
        // CGLIB 프록시는 구현 클래스를 상속해서 생성되므로 구현 클래스를 알 수 있음
        @Around("this(hello.aop.member.MemberServiceImpl)")
        public Object doThis(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[this-impl] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }

        // target + 구체 클래스 → 항상 매칭 (타깃은 언제나 MemberServiceImpl)
        @Around("target(hello.aop.member.MemberServiceImpl)")
        public Object doTarget(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[target-impl] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }
    }
}
```

**실행 결과 — `spring.aop.proxy-target-class=false` (JDK 동적 프록시)**
```
memberService Proxy=class com.sun.proxy.$Proxy53
[target-impl]      String hello.aop.member.MemberService.hello(String)
[target-interface] String hello.aop.member.MemberService.hello(String)
[this-interface]   String hello.aop.member.MemberService.hello(String)
                                   ↑
                     [this-impl]이 없다! ← 유일하게 매칭 실패한 칸
```

**실행 결과 — `spring.aop.proxy-target-class=true` (CGLIB, 스프링 부트 기본값)**
```
memberService Proxy=class hello.aop.member.MemberServiceImpl$$EnhancerBySpringCGLIB$$7df96bd3
[target-impl]      String hello.aop.member.MemberServiceImpl.hello(String)
[target-interface] String hello.aop.member.MemberServiceImpl.hello(String)
[this-impl]        String hello.aop.member.MemberServiceImpl.hello(String)
[this-interface]   String hello.aop.member.MemberServiceImpl.hello(String)
                                   ↑
                            4개 전부 출력된다
```

> 💡 **실무에서는 이 함정을 만날 일이 거의 없다.** 스프링 부트 2.0부터 `proxyTargetClass=true`가 기본이라 **항상 CGLIB**이고, CGLIB에서는 8칸이 전부 O이기 때문. 하지만 **"왜 CGLIB이 기본이 되었는가"** 를 이해하려면 이 차이를 알아야 한다 — JDK 동적 프록시는 **구체 클래스 타입으로 의존관계를 주입할 수 없다**는 문제가 있었고, 그게 정확히 여기서 `this(MemberServiceImpl)`가 실패하는 것과 **같은 원인**(프록시가 구체 클래스 타입이 아님)이다. ([동적 프록시 노트](./dynamic-proxy.md) 참고)

---

## 10. bean — 스프링 전용 지시자

**스프링 빈의 이름**으로 AOP 적용 여부를 지정한다. `*` 패턴 사용 가능.

```java
package hello.aop.pointcut;

import hello.aop.order.OrderService;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;

@Slf4j
@Import(BeanTest.BeanAspect.class)
@SpringBootTest
public class BeanTest {

    @Autowired
    OrderService orderService;

    @Test
    void success() {
        orderService.orderItem("itemA");
    }

    @Aspect
    static class BeanAspect {
        @Around("bean(orderService) || bean(*Repository)")
        public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
            log.info("[bean] {}", joinPoint.getSignature());
            return joinPoint.proceed();
        }
    }
}
```

**실행 결과**
```
[bean] void hello.aop.order.OrderService.orderItem(String)
[orderService] 실행
[bean] String hello.aop.order.OrderRepository.save(String)
[orderRepository] 실행
```

> 📌 **`bean`은 AspectJ에 없는 스프링만의 확장**이다. AspectJ는 타입 레벨에서만 동작하는데 `bean`은 **인스턴스 레벨**(스프링 빈 이름 기준)로 동작한다 — 스프링 빈 팩토리와 밀접하게 통합된 프록시 기반 AOP만이 할 수 있는 일. 반대로 말하면 **AspectJ 네이티브 위빙으로 갈아타면 `bean`은 못 쓴다.**
>
> ⚠️ 빈 이름 규칙에 의존하는 표현식이라, **네이밍 컨벤션이 흔들리면 조용히 안 걸린다.** [빈 후처리기 노트](./bean-post-processor.md)에서 "이름만 보는 포인트컷은 스프링 내부 빈에 오폭한다"고 정리했던 것의 다른 얼굴 — 이름 기반은 편하지만 안전하지 않다.

---

## 11. 매개변수 전달 (파라미터 바인딩)

포인트컷 표현식으로 **어드바이스에 값을 넘길 수 있다.** [AOP 구현 노트 §3](./aop-implementation.md)에서 "`@Pointcut` 파라미터는 금지가 아니라 바인딩 기능"이라고 정정했던 것의 본론이 여기다.

**바인딩 가능한 지시자**: `this`, `target`, `args`, `@target`, `@within`, `@annotation`, `@args`

```java
@Before("allMember() && args(arg,..)")
public void logArgs3(String arg) {          // ← 포인트컷의 arg와 파라미터 이름이 같아야 함
    log.info("[logArgs3] arg={}", arg);
}
```

### 규칙

1. **포인트컷 표현식의 이름과 어드바이스 메서드 파라미터 이름을 맞춰야 한다** (여기서는 `arg`)
2. **파라미터 타입이 곧 추가 필터가 된다** — 위 예제는 파라미터가 `String`이므로 `args(arg,..)`가 실질적으로 `args(String,..)`로 좁혀진다

> ⚠️ 2번이 중요하다. [AOP 구현 노트 §8 함정 1](./aop-implementation.md)의 `@AfterReturning(returning=...)` 타입 필터와 **완전히 같은 메커니즘**이다 — **타입을 좁게 쓰면 타깃은 정상 실행되고 advice만 조용히 스킵**된다.

### 전체 예제 코드

```java
package hello.aop.pointcut;

import hello.aop.member.MemberService;
import hello.aop.member.annotation.ClassAop;
import hello.aop.member.annotation.MethodAop;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;

@Slf4j
@Import(ParameterTest.ParameterAspect.class)
@SpringBootTest
public class ParameterTest {

    @Autowired
    MemberService memberService;

    @Test
    void success() {
        log.info("memberService Proxy={}", memberService.getClass());
        memberService.hello("helloA");
    }

    @Slf4j
    @Aspect
    static class ParameterAspect {

        @Pointcut("execution(* hello.aop.member..*.*(..))")
        private void allMember() {}

        // ① 바인딩 없이 — getArgs()로 직접 꺼내기
        @Around("allMember()")
        public Object logArgs1(ProceedingJoinPoint joinPoint) throws Throwable {
            Object arg1 = joinPoint.getArgs()[0];
            log.info("[logArgs1]{}, arg={}", joinPoint.getSignature(), arg1);
            return joinPoint.proceed();
        }

        // ② args로 바인딩 — 인덱스 대신 이름으로 받는다
        @Around("allMember() && args(arg,..)")
        public Object logArgs2(ProceedingJoinPoint joinPoint, Object arg) throws Throwable {
            log.info("[logArgs2]{}, arg={}", joinPoint.getSignature(), arg);
            return joinPoint.proceed();
        }

        // ③ @Before 축약 버전 + 타입을 String으로 제한
        @Before("allMember() && args(arg,..)")
        public void logArgs3(String arg) {
            log.info("[logArgs3] arg={}", arg);
        }

        // ④ 프록시 객체를 전달받는다
        @Before("allMember() && this(obj)")
        public void thisArgs(JoinPoint joinPoint, MemberService obj) {
            log.info("[this]{}, obj={}", joinPoint.getSignature(), obj.getClass());
        }

        // ⑤ 실제 타깃 객체를 전달받는다
        @Before("allMember() && target(obj)")
        public void targetArgs(JoinPoint joinPoint, MemberService obj) {
            log.info("[target]{}, obj={}", joinPoint.getSignature(), obj.getClass());
        }

        // ⑥⑦ 타입의 애노테이션을 전달받는다
        @Before("allMember() && @target(annotation)")
        public void atTarget(JoinPoint joinPoint, ClassAop annotation) {
            log.info("[@target]{}, obj={}", joinPoint.getSignature(), annotation);
        }

        @Before("allMember() && @within(annotation)")
        public void atWithin(JoinPoint joinPoint, ClassAop annotation) {
            log.info("[@within]{}, obj={}", joinPoint.getSignature(), annotation);
        }

        // ⑧ ★ 메서드의 애노테이션을 전달받는다 — 실무에서 제일 많이 쓰는 형태
        @Before("allMember() && @annotation(annotation)")
        public void atAnnotation(JoinPoint joinPoint, MethodAop annotation) {
            log.info("[@annotation]{}, annotationValue={}",
                     joinPoint.getSignature(), annotation.value());
        }
    }
}
```

**실행 결과** (순서는 보기 좋게 조정한 것)
```
memberService Proxy=class hello.aop.member.MemberServiceImpl$$EnhancerBySpringCGLIB$$82

[logArgs1]    String ...hello(String), arg=helloA
[logArgs2]    String ...hello(String), arg=helloA
[logArgs3]    arg=helloA
[this]        String ...hello(String), obj=class ...MemberServiceImpl$$EnhancerBySpringCGLIB$$8   ← 프록시!
[target]      String ...hello(String), obj=class hello.aop.member.MemberServiceImpl                ← 진짜 객체!
[@target]     String ...hello(String), obj=@hello.aop.member.annotation.ClassAop()
[@within]     String ...hello(String), obj=@hello.aop.member.annotation.ClassAop()
[@annotation] String ...hello(String), annotationValue=test value
```

> 💡 **`[this]`와 `[target]`의 출력 클래스명을 비교해보면 §9가 눈으로 보인다** — `this`는 `$$EnhancerBySpringCGLIB`가 붙은 프록시, `target`은 순수한 `MemberServiceImpl`. **개념이 안 잡히면 이 두 줄을 직접 찍어보는 게 가장 빠른 확인법.**
>
> 📌 `@annotation(annotation)` + `annotation.value()` 조합이 실무 커스텀 AOP의 표준 패턴이다. 예: `@RetryCount(3)` 애노테이션을 만들고 어드바이스에서 `annotation.value()`로 3을 읽어 재시도 횟수로 쓰는 식. (다음 챕터 Ch.12 실전 예제의 재시도 AOP가 정확히 이 패턴)

---

## 12. ⚠️ args, @args, @target은 단독 사용 금지

강의가 명시적으로 경고하는 부분이다.

```java
// ❌ 위험 — 단독 사용
@Around("@target(hello.aop.member.annotation.ClassAop)")

// ✅ 안전 — execution으로 범위를 먼저 좁힘
@Around("execution(* hello.aop..*(..)) && @target(hello.aop.member.annotation.ClassAop)")
```

### 왜 위험한가 — 순환 구조

```
① args / @args / @target 은 실제 객체 인스턴스가 생성되고 "실행될 때"라야 판단 가능 (동적, §6)
                    ↓
② 실행 시점에 가로채서 판단하려면 → 프록시가 있어야 한다
                    ↓
③ 그런데 프록시를 만드는 시점은 → 애플리케이션 로딩 시점 (스프링 컨테이너 생성 시)
                    ↓
④ 로딩 시점에는 "어떤 빈이 조건에 맞을지" 알 수 없다
                    ↓
⑤ → 스프링은 "일단 모든 빈에 프록시를 만들어 두자"고 시도한다
                    ↓
⑥ 💥 스프링 내부 빈 중에는 final 클래스가 있고, CGLIB는 상속 기반이라 final을 못 감싼다 → 기동 실패
```

> ⚠️ **세션 정정**: "final class 때문인가?"라고 답했는데 **결론은 맞다.** 다만 연결 고리가 빠져 있었다 — `final`이 직접적 원인이 아니라, **"동적 판단 → 프록시 필요 → 로딩 시점엔 판단 불가 → 전체 빈에 프록시 시도"** 라는 연쇄의 **마지막 증상**이 `final` 오류다. 원인을 `final`로 기억하면 "final이 없으면 괜찮겠네"로 잘못 이어진다. **실제 원인은 "판단 시점과 프록시 생성 시점의 불일치"** 고, 그래서 **범위를 좁히는 표현식(`execution`/`within`)을 반드시 함께 써야 한다.**

### 왜 `this`/`target`/`@within`은 괜찮은가

| 지시자 | 판단 대상 | 로딩 시점에 알 수 있나 |
|---|---|---|
| `args`, `@args` | 런타임에 넘어온 **인자 객체** | ❌ 호출해봐야 안다 |
| `@target` | 실행 중인 **인스턴스의 클래스** | ❌ (엄밀히는 인스턴스 기준이라 런타임 판단) |
| `this`, `target` | 프록시/타깃의 **타입** | ⭕ 빈 생성 시점에 타입은 정해짐 |
| `@within`, `@annotation` | **선언된 클래스/메서드**의 애노테이션 | ⭕ 클래스 파일에 이미 있음 |
| `execution`, `within`, `bean` | 시그니처 / 타입 / 빈 이름 | ⭕ |

> 💡 `@target`과 `@within`이 **또 한 번 갈리는 지점**이다 — `@target`은 단독 금지, `@within`은 괜찮다. §7의 "실행 중인 인스턴스 vs 선언된 타입" 구분이 여기서 **성능·안정성 문제로 이어진다.** 개념 차이가 실무 제약으로 직결되는 좋은 예.

---

## 13. 💡 판단 기준

### ① 지시자를 외우지 말고 **"무엇을 보는가"의 축 3개**로 정리한다

세션에서 5문항 중 3개가 헷갈렸는데, 틀린 방식이 전부 같았다 — **비슷한 이름의 두 지시자를 놓고 엉뚱한 축으로 구분**한 것. `@target` vs `@within`을 "프록시 vs 진짜 객체"로 답한 게 대표적이다(그건 `this` vs `target`의 축). 이름이 비슷하다는 이유로 기준축까지 같을 거라고 가정하면 반드시 틀린다. 세 축은 이렇게 독립적이다:

| 축 | 대립쌍 | 질문 |
|---|---|---|
| **정적 vs 동적** | `execution` ↔ `args` | 선언된 시그니처를 보나, 실제 넘어온 객체를 보나? |
| **프록시 vs 타깃** | `this` ↔ `target` | 스프링 빈(프록시)을 보나, 그 안의 진짜 객체를 보나? |
| **인스턴스 vs 선언 위치** | `@target` ↔ `@within` | 실행 중인 인스턴스의 클래스를 보나, 메서드가 선언된 클래스를 보나? |

**새 지시자를 만나면 "이건 어느 축인가"부터 묻는다.** `@annotation`은? → 어느 축도 아니고 "메서드 자신"을 본다. `bean`은? → 타입이 아니라 이름을 본다. 축을 먼저 세우면 표가 저절로 채워진다.

### ② 좋은 포인트컷은 **kinded + scoping** 을 함께 쓴다 (공식 문서 원칙)

Spring/AspectJ 공식 문서는 지시자를 세 그룹으로 분류한다:

| 그룹 | 지시자 | 역할 |
|---|---|---|
| **kinded** | `execution` (+ AspectJ의 `get`, `set`, `call`, `handler`) | **어떤 종류의** 조인 포인트인가 |
| **scoping** | `within` (+ `withincode`) | **어느 범위**인가 |
| **contextual** | `this`, `target`, `@annotation` | **문맥으로 매칭**하고 값을 바인딩 |

그리고 원칙은 이것: **잘 쓰인 포인트컷은 최소한 kinded와 scoping 두 종류를 포함해야 한다.** contextual만 쓰거나 kinded만 쓰면 동작은 하지만 **위빙 성능(시간·메모리)에 악영향**을 준다. scoping 지시자는 매칭이 매우 빨라서, 처리할 필요 없는 조인 포인트 그룹을 **초기에 통째로 걷어낸다.**

> 이게 §12의 "단독 사용 금지"와 같은 이야기의 **일반화 버전**이다. `@target` 단독이 위험한 건 극단적 사례일 뿐, **범위를 안 좁힌 표현식은 전부 정도의 차이만 있다.** 실무 표현식이 `execution(* com.xyz..service..*(..)) && @annotation(...)` 모양인 이유.
>
> 📌 참고로 **표현식을 쓰는 순서는 신경 쓸 필요 없다.** AspectJ가 포인트컷을 DNF(선언 정규형)로 재작성하면서 **평가 비용이 싼 조건을 앞으로 정렬**한다. 순서 최적화는 프레임워크가 알아서 하니, 개발자가 할 일은 **범위를 좁히는 조건을 "포함시키는 것"** 뿐이다.

### ③ 실무에서 실제로 쓰는 건 사실상 3개다

강의에서도 "`execution` 말고는 자주 안 쓴다"고 하는데, 정확히는 이렇게 갈린다:

| | 지시자 | 용도 |
|---|---|---|
| **주력** | `execution` | 90% 이상. 패키지·레이어·메서드 패턴 |
| **주력** | `@annotation` | 커스텀 AOP 애노테이션. `@Transactional`·`@Cacheable`이 이 방식 |
| **보조** | `within` | 범위 좁히기(scoping). 공용 포인트컷 클래스에서 레이어 정의용 |
| 바인딩용 | `args`, `this`, `target`, `@target`, `@within`, `@args` | 값을 **어드바이스로 넘길 때**. 매칭 조건으로 단독 사용하는 경우는 거의 없음 |
| 특수 | `bean` | 빈 이름 컨벤션이 잘 잡힌 프로젝트에서 |

**그러면 나머지 6개는 왜 배우나?** 두 가지 이유다. ① **파라미터 바인딩(§11)에서 다시 만난다** — `@annotation(annotation)`으로 애노테이션 값을 꺼내 쓰는 건 실무 필수 패턴이다. ② **`this`/`target`의 차이를 이해하는 과정이 곧 프록시 구조를 이해하는 것**이다. §9의 8칸 표를 채울 수 있으면 [동적 프록시](./dynamic-proxy.md)와 "왜 스프링 부트는 CGLIB을 기본으로 골랐는가"를 함께 설명할 수 있게 된다. **지시자 자체는 안 써도, 그걸 이해하면서 잡히는 프록시 모델이 남는다.**

### ④ 표현식이 헷갈리면 **학습 테스트로 찍어본다**

`AspectJExpressionPointcut`은 스프링 컨테이너 없이 `matches()` 한 줄로 true/false를 준다. `.`과 `..`을 헷갈렸는지, 인터페이스를 지정해서 일부 메서드가 빠진 건지 — **머리로 추론하는 것보다 세 줄 짜리 테스트가 빠르다.** [AOP 구현 노트](./aop-implementation.md)에서 정리한 **무음 실패 진단 순서**(`AopUtils.isAopProxy()` → 표현식 → 파라미터 타입 → 자기호출)의 2단계가 정확히 이 도구를 쓰는 자리다.

---

## 14. 다음 챕터로

| 챕터 | 내용 | 이 챕터와의 연결 |
|---|---|---|
| Ch.12 실전 예제 | 로그 출력 AOP, 재시도 AOP | **`@annotation` + 파라미터 바인딩**의 실전 적용 |
| Ch.13 실무 주의사항 | 프록시 기술과 한계, 자기호출 | §9의 `this`/`target` 차이가 여기서 **실무 문제**로 돌아온다 |

> 📌 강의 자료도 명시한다 — *"혹시 `this`/`target` 내용이 잘 이해되지 않으면 스프링 AOP 실무 주의사항에서 프록시 기술과 한계를 듣고 다시 들어보면 더 이해가 쉽다."* §9가 안 잡히면 Ch.13을 먼저 듣고 돌아오는 게 낫다.

---

## 참고

- 김영한, 스프링 핵심 원리 고급편 — Ch.11 스프링 AOP - 포인트컷
- [Spring Framework Reference — Declaring a Pointcut](https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/pointcuts.html) (지원 PCD 9종 + `bean` / 미지원 지시자는 `IllegalArgumentException` / AspectJ에서는 `this`와 `target`이 같은 객체지만 프록시 기반인 스프링은 구분 / `bean`은 스프링 전용·인스턴스 레벨 / `args(Serializable)`과 `execution(* *(Serializable))`의 차이 / Writing Good Pointcuts — kinded·scoping·contextual 분류와 "최소 kinded+scoping" 원칙 / DNF 재작성으로 순서는 신경 쓸 필요 없음 / JDK 프록시는 public 인터페이스 메서드만, CGLIB는 public·protected까지 인터셉트)
- [Spring Framework Reference — Declaring Advice](https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/advice.html) (파라미터 바인딩 형태)
- [Spring Framework Reference — Proxying Mechanisms](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)
- [AspectJ Programming Guide — Language Semantics](https://www.eclipse.org/aspectj/doc/released/progguide/semantics-pointcuts.html) (원본 포인트컷 언어 명세)

**학습 날짜**: 2026-08-14 · **계기**: 김영한 고급편 Ch.11 수강 후 Claude 소크라테스 복습 세션(5문항). 강의를 듣고 "전반적으로 다 이해 못 한 것 같다"는 느낌으로 시작. 결과는 이렇게 갈렸다 —
① **`execution` vs `within`(부모 타입 허용 여부) 정확히 답함** ✅
② `execution` vs `args`의 O/X는 맞혔으나 **"정적/동적"이라는 용어로 설명하지 못함** ⚠️ → §6에 "무엇을 보는가 / 판단 시점 / 부모 타입" 3열 표와 한 문장 정리 수록
③ **`@target` vs `@within`을 `this` vs `target`과 혼동** 🔴 ("@target은 진짜 객체, @within은 프록시 객체 아닌가?") → §7 첫머리에 **축이 다르다**는 경고 박스를 따로 세움. `@`가 붙으면 "애노테이션으로 판단"이라는 뜻일 뿐 앞 단어 의미가 이어지지 않는다는 게 핵심
④ `this`/`target` 8칸 표에서 **X 위치를 반대로 찍음**(`O O O X / X X X O` → 정답 `O X O O / O O O O`) 🔴. 다만 "JDK 동적 프록시는 인터페이스 기반"이라는 **원리는 맞게 짚었고**, 다음 문장에서 "그러면 CGLIB가 사용될 것 같다"로 이어진 게 오류 → §9에 **"지시자는 판단 대상을 고르는 것이지 프록시를 만드는 것이 아니다"** 정정 수록
⑤ 단독 사용 금지 이유를 **`final` 클래스 + "런타임에 판단되고 그런 건가"로 방향은 맞게 답함** ⚠️ 다만 연결 고리가 비어 있었음 → §12에 **6단계 연쇄 다이어그램**으로 정리하고, `final`은 원인이 아니라 마지막 증상이라는 점을 명시.
📌 세션 후 조사에서 추가로 확인해 수록: 스프링 AOP가 **지원하지 않는** AspectJ 지시자 목록과 `IllegalArgumentException` / `bean`이 **인스턴스 레벨**이라 AspectJ 네이티브 위빙에서는 못 쓴다는 점 / 공식 문서의 **kinded·scoping·contextual 3분류와 "최소 kinded+scoping" 원칙**(§13-②) — 강의의 "단독 사용 금지"가 이 원칙의 극단적 사례라는 연결 / DNF 재작성 덕에 **표현식 순서는 최적화할 필요 없다**는 점 / `@within`은 단독 사용해도 안전한 이유(선언 정보라 로딩 시점에 판단 가능)

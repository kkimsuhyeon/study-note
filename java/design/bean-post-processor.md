# 빈 후처리기 (BeanPostProcessor) — 빈 등록 직전에 가로채 프록시로 바꿔치기

> **한 줄 요약**: 빈 후처리기는 스프링이 생성한 객체를 **빈 저장소에 등록하기 직전에** 가로채서 조작하거나 **다른 객체로 바꿔치기**하는 후킹 포인트다. 이걸로 ProxyFactory가 못 풀었던 **설정 지옥**과 **컴포넌트 스캔 프록시 미적용**이 한 번에 해결된다. ⚠️ `Before/After`의 기준은 "빈 등록"이 아니라 **`@PostConstruct` 초기화**이며(둘 다 등록 *전*에 실행), ⚠️ 하나의 프록시에 여러 Advisor가 들어갈 때 **각 Advisor의 포인트컷은 독립적으로 판단**된다 — 전부 적용되거나 전부 안 되는 게 아니다.

관련 노트: [ProxyFactory](./proxy-factory.md) (직전 챕터 — "설정 지옥 + 컴포넌트 스캔"으로 끝난 자리가 이 노트의 출발점) · [동적 프록시](./dynamic-proxy.md) · [프록시/데코레이터 패턴](./proxy-decorator-pattern.md) (V3 컴포넌트 스캔에 못 끼운다는 문제를 처음 제기한 챕터) · [ThreadLocal](../concurrency/thread-local.md) (`LogTrace`의 출처) · [@Transactional](../spring/transactional.md)·[Spring Cache](../spring/spring-cache.md)·[메서드 보안](../security/method-security.md) (전부 이 자동 프록시 생성기 위에서 동작한다)

---

## 0. 출발점 — ProxyFactory로도 안 풀린 두 가지

[직전 챕터](./proxy-factory.md)에서 `ProxyFactory` + `Advisor`로 기술 선택·중복·책임 분리를 해결했다. 그런데 **실무 적용에서 두 가지가 남았다**:

| # | 문제 | 구체적으로 |
|---|------|-----------|
| ① | **설정 지옥** | 빈이 100개면 프록시 생성 코드도 100개. `ProxyFactoryConfigV1`, `V2` 같은 설정 클래스가 끝없이 늘어남 |
| ② | **컴포넌트 스캔 미지원** | `@Component`·`@Service`로 자동 등록되는 빈은 **개발자가 프록시를 끼워넣을 타이밍이 없음**. `@Bean`으로 직접 등록할 때만 프록시를 반환할 수 있었다 |

②가 특히 치명적이다. 실무 코드는 거의 전부 컴포넌트 스캔인데, 거기에 프록시를 못 넣으면 이제까지 배운 게 무용지물이다.

> **발상의 전환**: "내가 프록시를 만들어서 반환할 수 없다면, **스프링이 만든 걸 뺏어서 바꿔치기**하면 되지 않나?"
> → 그 틈이 바로 **빈 저장소에 등록되기 직전**이고, 그 틈에 끼어드는 장치가 **빈 후처리기**다.

---

## 1. 빈 후처리기 기본

### 일반적인 빈 등록 과정 (후처리기 없음)

```java
package hello.proxy.postprocessor;

public class BasicTest {

    @Test
    void basicConfig() {
        ApplicationContext applicationContext =
                new AnnotationConfigApplicationContext(BasicConfig.class);

        // A는 "beanA"라는 이름으로 스프링 빈에 등록된다.
        A a = applicationContext.getBean("beanA", A.class);
        a.helloA();

        // B는 빈으로 등록되지 않는다.
        Assertions.assertThrows(NoSuchBeanDefinitionException.class,
                () -> applicationContext.getBean(B.class));
    }

    @Slf4j
    @Configuration
    static class BasicConfig {
        @Bean(name = "beanA")
        public A a() {
            return new A();
        }
    }

    @Slf4j
    static class A {
        public void helloA() { log.info("hello A"); }
    }

    @Slf4j
    static class B {
        public void helloB() { log.info("hello B"); }
    }
}
```

`@Bean`이나 컴포넌트 스캔으로 빈을 등록하면, 스프링은 대상 객체를 생성하고 **컨테이너 내부의 빈 저장소에 등록**한다. 이후에는 컨테이너를 통해 조회해서 쓴다.

### BeanPostProcessor 인터페이스 — 스프링 제공

```java
public interface BeanPostProcessor {
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

- `postProcessBeforeInitialization` : 객체 생성 이후 **`@PostConstruct` 같은 초기화가 발생하기 전**에 호출
- `postProcessAfterInitialization` : 객체 생성 이후 **초기화가 발생한 다음**에 호출

이 인터페이스를 구현하고 **스프링 빈으로 등록만 하면** 컨테이너가 알아서 빈 후처리기로 인식한다. (별도 활성화 애노테이션이 필요 없다)

### 빈 후처리기 4단계 과정

```
1. 생성    : 스프링 빈 대상이 되는 객체를 생성한다 (@Bean, 컴포넌트 스캔 모두 포함)
2. 전달    : 생성된 객체를 빈 저장소에 등록하기 직전에 빈 후처리기에 전달한다
3. 후처리  : 빈 후처리기가 객체를 조작하거나 다른 객체로 바꿔치기한다
4. 등록    : 반환된 객체가 빈 저장소에 등록된다
             (그대로 반환 → 원본 등록 / 바꿔치기 → 다른 객체 등록)
```

> 📦 **비유**: 공장(스프링)에서 상품(빈 객체)을 만들어 창고(빈 저장소)에 넣으려는데, 중간에 **검수센터(빈 후처리기)**가 있다. 검수센터는 상품을 손볼 수도 있고 **아예 다른 상품으로 교체**해서 창고에 넣을 수도 있다.

### 바꿔치기 예제 — A를 B로

```java
package hello.proxy.postprocessor;

public class BeanPostProcessorTest {

    @Test
    void postProcessor() {
        ApplicationContext applicationContext =
                new AnnotationConfigApplicationContext(BeanPostProcessorConfig.class);

        // beanA 이름으로 B 객체가 빈으로 등록된다.
        B b = applicationContext.getBean("beanA", B.class);
        b.helloB();

        // A는 빈으로 등록조차 되지 않는다.
        Assertions.assertThrows(NoSuchBeanDefinitionException.class,
                () -> applicationContext.getBean(A.class));
    }

    @Slf4j
    @Configuration
    static class BeanPostProcessorConfig {
        @Bean(name = "beanA")
        public A a() {
            return new A();
        }

        @Bean
        public AToBPostProcessor helloPostProcessor() {   // 빈으로 등록만 하면 동작
            return new AToBPostProcessor();
        }
    }

    @Slf4j
    static class A {
        public void helloA() { log.info("hello A"); }
    }

    @Slf4j
    static class B {
        public void helloB() { log.info("hello B"); }
    }

    @Slf4j
    static class AToBPostProcessor implements BeanPostProcessor {
        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName)
                throws BeansException {
            log.info("beanName={} bean={}", beanName, bean);
            if (bean instanceof A) {
                return new B();      // ← A 대신 B가 빈 저장소에 등록된다
            }
            return bean;
        }
    }
}
```

실행 결과:
```
..AToBPostProcessor - beanName=beanA bean=hello.proxy.postprocessor...A@21362712
..B - hello B
```

**`"beanA"`라는 이름에 A가 아니라 B가 등록됐다.** A는 빈으로 등록조차 되지 않는다.

> 💡 이 예제가 사소해 보이지만, "**빈 이름은 그대로인데 내용물이 바뀐다**"가 프록시 적용의 전부다. 원본 자리에 프록시를 넣어도 이름·타입이 유지되므로 **주입받는 쪽 코드는 전혀 바뀌지 않는다**.

---

## 2. ⚠️ Before vs After — 기준은 "빈 등록"이 아니라 "@PostConstruct"

**가장 헷갈리기 쉬운 지점.** 이름만 보면 "빈 등록 전 / 후"로 읽히지만 틀렸다.

**두 메서드 모두 빈이 저장소에 등록되기 *전*에 실행된다.** Before/After를 가르는 기준은 그 사이에 끼어 있는 **초기화(Initialization)** 콜백이다.

```
객체 생성 + 의존관계 주입(DI) 완료
        ↓
  ▶ postProcessBeforeInitialization    ← DI는 끝남, 초기화는 아직
        ↓
    초기화 콜백 실행                    ← @PostConstruct 등
        ↓
  ▶ postProcessAfterInitialization     ← DI도 끝남, 초기화도 끝남
        ↓
    빈 저장소에 등록
```

### 여기서 "초기화(Initialization)"의 범위

메서드 이름의 `Initialization`은 다음을 뜻한다:

| 방식 | 설명 |
|---|---|
| `@PostConstruct` | 애노테이션 기반 (가장 흔함) |
| `InitializingBean.afterPropertiesSet()` | 인터페이스 기반 (스프링 종속적이라 요즘 잘 안 씀) |
| `@Bean(initMethod = "init")` | 설정 메타데이터 기반 |

### @PostConstruct란 — 빈 생명주기 안에서의 위치

객체 생성 후, **의존관계 주입이 모두 완료된 시점**에 추가 초기화 작업을 하기 위해 호출되는 메서드.

```java
@Component
public class CacheService {

    private final DataRepository repository;
    private Map<String, Object> cache;

    // 1단계: 생성자 → 객체 생성 + 의존관계 주입
    public CacheService(DataRepository repository) {
        this.repository = repository;
    }

    // 2단계: DI가 모두 끝난 뒤 자동 호출
    @PostConstruct
    public void init() {
        this.cache = repository.loadAll();   // repository가 null이 아님이 보장됨
    }
}
```

**왜 생성자에서 안 하고 분리하나?** 생성자가 호출되는 시점에는 다른 의존성이 아직 안 들어와 있을 수 있다. `@PostConstruct`는 **모든 의존관계 주입이 끝난 뒤**가 보장되므로 주입받은 협력 객체를 안전하게 쓸 수 있다.

빈 생명주기 전체:
```
객체 생성 → 의존관계 주입 → @PostConstruct(초기화) → 사용 → @PreDestroy(소멸)
```

> 📌 **패키지 주의**: Spring Framework 6 / Spring Boot 3부터 `javax.annotation.PostConstruct`가 아니라 **`jakarta.annotation.PostConstruct`**다. Jakarta EE 네임스페이스 이전의 결과.

### ⚠️ 프록시 바꿔치기를 After에서 하는 이유

```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    // 초기화까지 완전히 끝난 "완성품"을 프록시로 감싼다
    ProxyFactory proxyFactory = new ProxyFactory(bean);
    proxyFactory.addAdvisor(advisor);
    return proxyFactory.getProxy();
}
```

Before에서 바꿔치기하면 이후에 실행될 **`@PostConstruct`가 원본이 아니라 프록시 객체를 향해 호출**된다. 프록시가 target에 위임하는 구조라 대개는 동작하지만, 프록시 종류·구현에 따라 예측하기 어려운 문제가 생길 수 있다. **초기화까지 끝난 완성품을 감싸는 After가 안전하고 예측 가능**하다.

실제로 스프링이 제공하는 자동 프록시 생성기도 `postProcessAfterInitialization`에서 프록시를 만든다.

### @PostConstruct 자체도 빈 후처리기로 동작한다

`@PostConstruct`는 마법이 아니다. 생각해보면 "초기화"라는 게 결국 **애노테이션 붙은 메서드를 한 번 호출하는 것**, 즉 생성된 빈을 한 번 조작하는 것이다. 그러니 그 일을 하는 빈 후처리기가 있으면 된다.

스프링은 **`CommonAnnotationBeanPostProcessor`**를 자동 등록하고, 이 후처리기가 `postProcessBeforeInitialization` 단계에서 `@PostConstruct` 붙은 메서드를 찾아 호출한다.

```
CommonAnnotationBeanPostProcessor
  └─ extends InitDestroyAnnotationBeanPostProcessor   ← 실제 init 메서드 호출 담당
       └─ implements BeanPostProcessor, PriorityOrdered
```

→ **스프링 스스로도 내부 기능 확장을 위해 빈 후처리기를 쓴다.** 지금 배우는 이 메커니즘이 프레임워크의 뼈대라는 증거.

---

## 3. ⚠️ 빈 후처리기가 여러 개일 때 — 순서는 인터페이스로만 정해진다

빈 후처리기는 여러 개 등록할 수 있다. 그러면 실행 순서가 문제가 된다.

### 스프링의 3그룹 정렬

`PostProcessorRegistrationDelegate.registerBeanPostProcessors()`는 BPP를 **세 그룹으로 나눠 순서대로** 등록한다:

```
1. PriorityOrdered 구현체   (그룹 내에서 order 값으로 정렬)
2. Ordered 구현체            (그룹 내에서 order 값으로 정렬)
3. 나머지 (정렬 안 함)
4. MergedBeanDefinitionPostProcessor는 마지막에 재등록
```

```java
public class MyPostProcessor implements BeanPostProcessor, Ordered {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    @Override
    public int getOrder() {
        return 0;   // 숫자가 작을수록 먼저 — 단, 같은 그룹 안에서만 의미 있음
    }
}
```

### ⚠️ 함정 1 — `@Order` 애노테이션은 BeanPostProcessor에 안 먹는다

Spring 공식 Javadoc이 명시한다: **BeanPostProcessor 빈에 대해서는 `@Order` 애노테이션이 고려되지 않는다.** 반드시 `Ordered` / `PriorityOrdered` **인터페이스를 구현**해야 한다.

다른 곳(`@Aspect`, `Advisor` 등)에서는 `@Order`가 잘 먹기 때문에 **여기서만 안 먹는다는 걸 모르면 조용히 순서가 어긋난다.**

### ⚠️ 함정 2 — order 값보다 그룹이 먼저다

`CommonAnnotationBeanPostProcessor`는 **`PriorityOrdered`**다. 그래서 내가 만든 후처리기가 `Ordered`만 구현했다면, **order 값을 아무리 작게(=우선순위 높게) 줘도 `PriorityOrdered` 그룹 뒤로 밀린다.**

즉 `Ordered`만 구현한 커스텀 BPP의 `postProcessBeforeInitialization`은 **`@PostConstruct`보다 항상 늦게 실행된다.** "order를 작게 주면 @PostConstruct보다 먼저 끼어들 수 있다"는 직관은 틀렸다 — 그러려면 `PriorityOrdered`를 구현하고 그 그룹 안에서 경쟁해야 한다.

> 📌 **§2 그림과 모순이 아니다.** §2의 "Before → 초기화 → After" 그림은 `afterPropertiesSet`·`init-method` 기준의 **계약**이다 — 이들은 컨테이너가 *모든* Before 콜백이 끝난 뒤 호출한다. 반면 `@PostConstruct`는 자체가 CABPP라는 빈 후처리기의 **Before 안에서** 실행되므로, 다른 BPP의 Before와의 순서는 그림이 아니라 **BPP 등록 순서**가 결정한다.

### ⚠️ 함정 3 — 프로그래밍 방식 등록은 정렬을 무시한다

`beanFactory.addBeanPostProcessor()`로 직접 등록하면 **등록한 순서 그대로** 적용되고, `Ordered`/`PriorityOrdered`로 표현한 순서 정보는 **무시**된다. 자동 감지(빈 등록)와 프로그래밍 등록이 규칙이 다르다.

> 실무에서 빈 후처리기를 직접 만들 일은 거의 없다. 다만 **"Before/After라는 이름은 하나의 후처리기 내부 기준일 뿐, 여러 후처리기 사이의 순서는 완전히 별개 규칙"**이라는 점은 알아둘 값어치가 있다.

---

## 4. 실전 적용 — 직접 만든 빈 후처리기

### PackageLogTraceProxyPostProcessor

```java
package hello.proxy.config.v4_postprocessor.postprocessor;

import lombok.extern.slf4j.Slf4j;
import org.springframework.aop.Advisor;
import org.springframework.aop.framework.ProxyFactory;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;

@Slf4j
public class PackageLogTraceProxyPostProcessor implements BeanPostProcessor {

    private final String basePackage;
    private final Advisor advisor;

    public PackageLogTraceProxyPostProcessor(String basePackage, Advisor advisor) {
        this.basePackage = basePackage;
        this.advisor = advisor;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        log.info("param beanName={} bean={}", beanName, bean.getClass());

        // 프록시 적용 대상 여부 체크 → 대상이 아니면 원본을 그대로 반환
        String packageName = bean.getClass().getPackageName();
        if (!packageName.startsWith(basePackage)) {
            return bean;
        }

        // 프록시 대상이면 프록시를 만들어서 반환
        ProxyFactory proxyFactory = new ProxyFactory(bean);
        proxyFactory.addAdvisor(advisor);
        Object proxy = proxyFactory.getProxy();
        log.info("create proxy: target={} proxy={}", bean.getClass(), proxy.getClass());
        return proxy;
    }
}
```

**왜 `basePackage`로 거르나?** 스프링 부트가 기본 등록하는 수많은 빈들이 전부 이 후처리기를 통과한다. 거기에 모두 프록시를 적용하면 오류가 나거나 비용만 낭비된다. 그래서 "우리 애플리케이션 패키지만" 대상으로 좁혔다.

### BeanPostProcessorConfig

```java
package hello.proxy.config.v4_postprocessor;

@Slf4j
@Configuration
@Import({AppV1Config.class, AppV2Config.class})
public class BeanPostProcessorConfig {

    @Bean
    public PackageLogTraceProxyPostProcessor logTraceProxyPostProcessor(LogTrace logTrace) {
        return new PackageLogTraceProxyPostProcessor("hello.proxy.app", getAdvisor(logTrace));
    }

    private Advisor getAdvisor(LogTrace logTrace) {
        // pointcut
        NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
        pointcut.setMappedNames("request*", "order*", "save*");
        // advice
        LogTraceAdvice advice = new LogTraceAdvice(logTrace);
        // advisor = pointcut + advice
        return new DefaultPointcutAdvisor(pointcut, advice);
    }
}
```

> `@Import({AppV1Config.class, AppV2Config.class})` : V3는 컴포넌트 스캔으로 자동 등록되지만, V1·V2는 수동 등록이 필요해서 여기서 import한다.

**핵심**: 설정 파일에 **프록시 생성 코드가 사라졌다.** 순수한 빈 등록만 고민하면 되고, 프록시 생성·등록은 빈 후처리기가 전부 처리한다.

### ProxyApplication

```java
//@Import({AppV1Config.class, AppV2Config.class})
//@Import(InterfaceProxyConfig.class)
//@Import(ConcreteProxyConfig.class)
//@Import(DynamicProxyBasicConfig.class)
//@Import(DynamicProxyFilterConfig.class)
//@Import(ProxyFactoryConfigV1.class)
//@Import(ProxyFactoryConfigV2.class)
@Import(BeanPostProcessorConfig.class)
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

### 실행 로그 — V1·V2·V3 전부 적용

```
#v1 수동 등록 + 인터페이스 있음 → JDK 동적 프록시
create proxy: target=v1.OrderRepositoryV1Impl proxy=class com.sun.proxy.$Proxy50
create proxy: target=v1.OrderServiceV1Impl    proxy=class com.sun.proxy.$Proxy51
create proxy: target=v1.OrderControllerV1Impl proxy=class com.sun.proxy.$Proxy52

#v2 수동 등록 + 구체 클래스만 → CGLIB
create proxy: target=v2.OrderRepositoryV2 proxy=v2.OrderRepositoryV2$$EnhancerBySpringCGLIB$$x4
create proxy: target=v2.OrderServiceV2    proxy=v2.OrderServiceV2$$EnhancerBySpringCGLIB$$x5
create proxy: target=v2.OrderControllerV2 proxy=v2.OrderControllerV2$$EnhancerBySpringCGLIB$$x6

#v3 컴포넌트 스캔 + 구체 클래스만 → CGLIB  ★ 이게 핵심
create proxy: target=v3.OrderRepositoryV3 proxy=v3.OrderRepositoryV3$$EnhancerBySpringCGLIB$$x1
create proxy: target=v3.OrderServiceV3    proxy=v3.OrderServiceV3$$EnhancerBySpringCGLIB$$x2
create proxy: target=v3.OrderControllerV3 proxy=v3.OrderControllerV3$$EnhancerBySpringCGLIB$$x3
```

**V3가 적용된 것이 이 챕터의 성과다.** [챕터 4](./proxy-decorator-pattern.md)부터 계속 "컴포넌트 스캔에는 못 끼운다"고 미뤄왔던 문제가 여기서 해결됐다.

### 남은 한계 — 패키지 기준은 너무 투박하다

`packageName.startsWith(basePackage)`는 단순 문자열 비교다. 그런데 우리는 이미 **"어디에 적용할지"를 정밀하게 판단하는 도구**를 갖고 있다 — [Advisor 안의 **Pointcut**](./proxy-factory.md#4-pointcut-인터페이스-구조)이다. 그걸 놔두고 문자열 비교를 하고 있는 셈.

---

## 5. 스프링이 제공하는 빈 후처리기 — 자동 프록시 생성기

### 의존성 추가

```gradle
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

이걸 추가하면 `aspectjweaver` 라이브러리가 등록되고, 스프링 부트가 AOP 관련 클래스를 자동으로 빈에 등록한다. 부트가 없던 시절에는 `@EnableAspectJAutoProxy`를 직접 붙여야 했다. (부트가 활성화하는 빈은 `AopAutoConfiguration` 참고)

### AnnotationAwareAspectJAutoProxyCreator

이름이 길어서 어려워 보이지만, **하는 일은 우리가 직접 만든 것과 같다.** 딱 하나가 다르다:

| | 필터링 기준 |
|---|---|
| 직접 만든 `PackageLogTraceProxyPostProcessor` | `packageName.startsWith(basePackage)` — 문자열 비교 |
| 스프링의 `AnnotationAwareAspectJAutoProxyCreator` | **Advisor 안의 Pointcut** — 정밀하고 유연 |

이 빈 후처리기는 **스프링 빈으로 등록된 `Advisor`들을 자동으로 찾아서** 프록시가 필요한 곳에 프록시를 적용한다. `Advisor` 안에 Pointcut과 Advice가 다 들어 있으므로, **Advisor만 알면 "어디에 무엇을"이 전부 결정**된다.

> 이름 앞의 `AnnotationAware`(애노테이션을 인식하는)는 `@Aspect`도 자동 인식해서 Advisor로 변환해준다는 뜻이다. 이건 다음 챕터(@Aspect AOP)에서 다룬다.

### 작동 과정 6단계

```
1. 생성            : 스프링 빈 대상 객체 생성 (@Bean, 컴포넌트 스캔 모두)
2. 전달            : 빈 저장소 등록 직전에 빈 후처리기에 전달
3. 모든 Advisor 조회 : 컨테이너에서 Advisor 빈을 전부 조회
4. 프록시 적용 대상 체크 : Advisor의 Pointcut으로 이 객체가 대상인지 판단
                        (클래스 정보 + 모든 메서드를 하나하나 매칭)
                        → 조건이 하나라도 만족하면 프록시 대상
5. 프록시 생성      : 대상이면 프록시 생성해서 반환 / 아니면 원본 반환
6. 빈 등록          : 반환된 객체가 빈 저장소에 등록
```

**개발자가 할 일은 이제 `Advisor`를 빈으로 등록하는 것뿐이다.** 빈 후처리기는 등록조차 필요 없다.

### AutoProxyConfig — advisor1 (메서드 이름 기반)

```java
package hello.proxy.config.v5_autoproxy;

@Configuration
@Import({AppV1Config.class, AppV2Config.class})
public class AutoProxyConfig {

    @Bean
    public Advisor advisor1(LogTrace logTrace) {
        NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
        pointcut.setMappedNames("request*", "order*", "save*");
        LogTraceAdvice advice = new LogTraceAdvice(logTrace);
        return new DefaultPointcutAdvisor(pointcut, advice);
    }
}
```

**⚠️ 그런데 애플리케이션 로딩 시 이상한 로그가 올라온다:**

```
EnableWebMvcConfiguration.requestMappingHandlerAdapter()
EnableWebMvcConfiguration.requestMappingHandlerAdapter() time=63ms
```

포인트컷이 **메서드 이름만** 보기 때문이다. 스프링 내부 빈에도 `request`로 시작하는 메서드가 있으면 프록시가 만들어지고 어드바이스까지 적용된다.

→ **패키지 + 메서드 이름을 함께 지정할 수 있는 정밀한 포인트컷이 필요하다.**

### advisor2 (AspectJ 표현식)

```java
@Bean
public Advisor advisor2(LogTrace logTrace) {
    AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
    pointcut.setExpression("execution(* hello.proxy.app..*(..))");
    LogTraceAdvice advice = new LogTraceAdvice(logTrace);
    return new DefaultPointcutAdvisor(pointcut, advice);
}
```

표현식 해석:
| 조각 | 의미 |
|---|---|
| `*` | 모든 반환 타입 |
| `hello.proxy.app..` | 해당 패키지 **와 그 하위 패키지** |
| `*(..)` | 모든 메서드 이름, 파라미터 무관 |

> ⚠️ **주의**: advisor1의 `@Bean`은 반드시 주석 처리할 것. 안 그러면 어드바이저가 중복 등록된다.

### advisor3 (제외 조건 추가)

advisor2는 패키지만 보므로 `noLog()`에도 로그가 남는 문제가 있다.

```java
@Bean
public Advisor advisor3(LogTrace logTrace) {
    AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
    pointcut.setExpression(
        "execution(* hello.proxy.app..*(..)) && !execution(* hello.proxy.app..noLog(..))");
    LogTraceAdvice advice = new LogTraceAdvice(logTrace);
    return new DefaultPointcutAdvisor(pointcut, advice);
}
```

- `&&` : 두 조건 모두 만족
- `!` : 반대

→ `hello.proxy.app` 하위 전체를 매칭하되 `noLog()`는 제외.

---

## 6. ⚠️ 포인트컷은 두 번 사용된다 — 이 챕터의 핵심

같은 Pointcut이 **완전히 다른 두 시점에** 쓰인다. 이걸 구분 못 하면 "왜 프록시는 생겼는데 로그가 안 찍히지?"를 영원히 못 푼다.

### 1단계 — 프록시를 **생성할지** (빈 후처리기 시점 / 클래스 단위)

```
스프링 빈 생성 → 이 클래스(의 메서드들 중 하나라도) Pointcut에 매칭되나?
                ├── No  → 원본 그대로 빈 등록 (프록시 없음)
                └── Yes → 프록시 생성해서 빈 등록
```

### 2단계 — 어드바이스를 **적용할지** (프록시 호출 시점 / 메서드 단위)

```
실제 요청 도착 → 이 메서드가 Pointcut에 매칭되나?
                ├── No  → 어드바이스 없이 target 메서드 바로 호출
                └── Yes → 어드바이스 적용 후 target 호출
```

### 구체 예시

`execution(* hello.proxy.app..*(..))` 포인트컷일 때:

- `org.springframework.boot`의 빈들 → **1단계에서 탈락** → 프록시 자체가 안 만들어짐
- `hello.proxy.app.v1.OrderControllerV1Impl` → 1단계 통과 → 프록시 생성
  - `request()` 호출 → 2단계 통과 → 어드바이스 실행 후 target 호출
  - `noLog()` 호출 (advisor3 기준) → 2단계 탈락 → **어드바이스 없이 target만 호출**

> 💡 **프록시를 모든 곳에 만드는 것은 비용 낭비다.** 자동 프록시 생성기가 1단계에서 한 번 걸러내는 이유가 이것 — 어드바이스가 쓰일 가능성이 있는 곳에만 프록시를 만든다.

---

## 7. ⚠️ 하나의 프록시, 여러 Advisor — 각각 독립적으로 판단한다

### 프록시는 몇 개 생성되나?

어떤 빈이 advisor1과 advisor2의 포인트컷을 **모두** 만족하면?

→ **프록시는 하나만 생성된다.** 프록시 팩토리가 만드는 프록시는 내부에 **여러 Advisor를 리스트로** 담을 수 있기 때문이다. 프록시를 여러 개 만들어 비용을 낭비할 이유가 없다.

```
클라이언트 → [프록시 1개]
               ├── Advisor1
               ├── Advisor2
               └── target 호출
```

### 상황별 정리

| 상황 | 결과 |
|---|---|
| advisor1의 포인트컷만 만족 | 프록시 1개 생성, 프록시에 **advisor1만** 포함 |
| advisor1, advisor2 포인트컷 모두 만족 | 프록시 1개 생성, 프록시에 **advisor1, advisor2 모두** 포함 |
| 둘 다 만족하지 않음 | **프록시가 생성되지 않음** (원본 등록) |

> 📝 [챕터 4의 데코레이터 체이닝](./proxy-decorator-pattern.md)과 [챕터 6의 "여러 프록시" 방식](./proxy-factory.md#5-여러-advisor-적용--프록시는-몇-개)은 프록시를 겹겹이 쌓았다. 스프링 방식은 **프록시 1개 + Advisor 리스트**로 같은 결과를 더 싸게 낸다.

### ⚠️ 각 Advisor의 Pointcut은 독립적으로 평가된다

**전부 적용되거나 전부 안 되는 게 아니다.** 프록시 안의 Advisor마다 개별적으로 "이 메서드에 내 Advice를 적용할까?"를 판단한다.

`OrderService`에 Advisor 2개가 걸린 상황:
- **Advisor1 (로그)**: 포인트컷이 모든 메서드 매칭
- **Advisor2 (트랜잭션)**: 포인트컷이 `order()`만 매칭

| 호출 | Advisor1 (로그) | Advisor2 (트랜잭션) | 결과 |
|---|---|---|---|
| `order()` | ✅ 매칭 | ✅ 매칭 | 로그 O, 트랜잭션 O |
| `getStatus()` | ✅ 매칭 | ❌ 매칭 안 됨 → 건너뜀 | **로그 O, 트랜잭션 X** |

`getStatus()`는 프록시를 거치긴 하지만 트랜잭션은 안 걸리고 로그만 찍힌다.

> 💡 이 독립성이 실무 디버깅의 핵심이다. "`@Transactional`은 안 먹는데 로그는 찍힌다"는 상황은 **프록시가 없어서가 아니라 그 Advisor의 포인트컷만 안 맞아서**일 수 있다.

---

## 8. "빈 저장소 등록" vs "객체 생성 + DI" — 두 단계는 별개다

빈 후처리기가 끼어드는 위치를 정확히 이해하려면 이 둘의 구분이 필요하다.

### 객체 생성 + 의존관계 주입

```java
new OrderService(orderRepository, paymentService)
```

메모리에 객체가 존재하고 필드에 협력 객체가 다 들어간 상태. **하지만 아직 스프링 빈이 아니다.** 그냥 자바 객체다.

### 빈 저장소 등록

스프링 컨테이너 내부에 빈 이름 → 빈 객체 매핑을 저장하는 저장소가 있다(개념적으로 `Map`).

```java
// 개념적 표현
beanStore.put("orderService", orderServiceObject);
```

**여기 들어가야 비로소 스프링 빈**이다. `applicationContext.getBean()`으로 조회되고, 다른 빈에 주입될 수 있다.

### 빈 후처리기의 위치

```
new OrderService(repository, payment)         ← 객체 생성 + DI (아직 그냥 자바 객체)
        ↓
   빈 후처리기 통과                            ← 이 틈에서 프록시로 바꿔치기
        ↓
beanStore.put("orderService", 프록시객체)      ← 빈 저장소 등록
```

**"객체는 만들어졌지만 아직 저장소에는 안 들어간" 그 틈**이 빈 후처리기의 무대다.

### DI 순서는 스프링이 재귀적으로 해결한다

`OrderService`가 `OrderRepository`를 필요로 하면, 스프링은 `OrderRepository`를 먼저 완성한 뒤 그걸 넣어 `OrderService`를 만든다:

```
OrderService 생성 시작
  → OrderRepository 필요 → 생성 → DI → 초기화 → 등록 완료
  → PaymentService 필요  → 생성 → DI → 초기화 → 등록 완료
  → 둘 다 준비됨 → OrderService 생성 → DI → 초기화 → 등록 완료
```

개발자가 순서를 관리할 필요는 없다. 의존관계 그래프를 보고 스프링이 정한다.

### ⚠️ 단, 그래프에 사이클이 있으면 순서를 정할 수 없다 — 순환 참조

A가 B를 필요로 하고 B도 A를 필요로 하면 재귀가 끝나지 않는다.

**Spring Boot 2.6부터 순환 참조는 기본적으로 금지**되어 기동 시점에 실패한다:

```
***************************
APPLICATION FAILED TO START
***************************

Description:
The dependencies of some of the beans in the application context form a cycle:
   orderService → paymentService → orderService

Action:
Relying upon circular references is discouraged and they are prohibited by default.
Update your application to remove the dependency cycle between beans.
```

`spring.main.allow-circular-references=true`로 되돌릴 수는 있지만 **권장되지 않는다.** 근본 해결은 의존 방향을 한쪽으로 정리하는 것 — 이건 [포트와 어댑터](./ports-and-adapters.md), [애그리거트 소유권](./aggregate-ownership.md)에서 다루는 설계 문제로 이어진다.

> 💡 "빈 등록에 순서가 있다"는 감각이 순환 참조 에러를 이해하는 열쇠다. 에러 메시지가 화살표(`→`)로 사이클을 그려주는 이유가 바로 이것.

---

## ⚠️ 함정 정리

### 1. Before/After는 "빈 등록" 기준이 아니다
`@PostConstruct` 같은 **초기화 콜백** 기준이며, 둘 다 빈 등록 *전*에 실행된다. 이름만 보고 "After = 등록 후"로 읽으면 전체 흐름이 어긋난다.

### 2. `@Order`는 BeanPostProcessor에 안 먹는다
`Ordered` / `PriorityOrdered` **인터페이스**만 인정된다. 다른 곳에서는 `@Order`가 잘 먹기 때문에 더 위험하다 — 예외 없이 **조용히** 순서가 어긋난다.

### 3. order 값보다 그룹(PriorityOrdered > Ordered > 나머지)이 우선
`Ordered`만 구현한 커스텀 BPP는 order 값과 무관하게 `PriorityOrdered`인 `CommonAnnotationBeanPostProcessor` 뒤에 실행된다.

### 4. 빈 후처리기는 스프링 부트 내부 빈까지 전부 통과시킨다
필터링(패키지든 포인트컷이든)이 없으면 `DataSource`, `RequestMappingHandlerAdapter` 같은 인프라 빈에도 프록시가 씌워져 오류가 나거나 기동 로그가 오염된다.

### 5. 메서드 이름만 보는 포인트컷은 스프링 내부 빈에 오폭한다
`NameMatchMethodPointcut("request*")`는 `EnableWebMvcConfiguration.requestMappingHandlerAdapter()`까지 잡는다. 실무 포인트컷은 **패키지 + 메서드**를 함께 지정해야 한다 (`AspectJExpressionPointcut`).

### 6. Advisor 중복 등록
advisor1, advisor2, advisor3를 실습하며 `@Bean`을 주석 처리하지 않으면 어드바이저가 중복 등록되어 로그가 두 번 찍힌다.

### 7. 프록시가 생겼다고 모든 메서드에 어드바이스가 걸린 건 아니다
1단계(클래스)와 2단계(메서드)는 별개 판정이다. "프록시는 CGLIB로 잘 만들어졌는데 특정 메서드만 부가 기능이 안 먹는다"면 2단계 포인트컷을 봐야 한다.

### 8. 빈 후처리기가 의존하는 빈은 후처리를 받지 못할 수 있다
BPP는 모든 빈에 적용되어야 하므로 다른 빈들보다 **먼저 생성**된다. 그래서 BPP 자신과 **BPP가 의존해서 덩달아 먼저 생성된 빈들**은 아직 등록되지 않은 다른 BPP(자동 프록시 생성기 포함)의 처리를 못 받는다. 스프링이 기동 로그로 경고한다:
```
Bean 'xxx' of type [...] is not eligible for getting processed by all
BeanPostProcessors (for example: not eligible for auto-proxying)
```
이 경고가 보이면 해당 빈에는 `@Transactional`·`@Cacheable` 같은 프록시 기반 기능이 **조용히 안 걸릴 수 있다.** 커스텀 BPP를 만들 때 일반 서비스 빈을 주입받으면 이 함정을 밟기 쉽다 — BPP의 의존성은 최소화하는 게 원칙이다.

---

## 💡 판단 기준

**"프레임워크가 자동으로 해주는 것"의 정체는 대부분 후킹 포인트다.** `@PostConstruct`가 마법처럼 호출되는 게 아니라 `CommonAnnotationBeanPostProcessor`가 Before 단계에서 호출하는 것이고, `@Transactional`이 마법처럼 걸리는 게 아니라 자동 프록시 생성기가 After 단계에서 프록시를 씌우는 것이다. **"이건 스프링이 알아서 해주는 거"라고 넘어가던 것에 이름이 붙는 순간, 디버깅 가능한 대상이 된다.** 실무에서 `@Transactional`이 안 먹을 때 이 계층을 알면 "프록시가 안 만들어졌나 / 포인트컷이 안 맞았나 / 자기호출인가"로 나눠 볼 수 있고, 모르면 그냥 미신이 된다.

**필터링 기준을 어디에 두느냐가 확장성을 결정한다.** 직접 만든 후처리기는 패키지 문자열로 걸렀고, 스프링은 Pointcut으로 거른다. 결과는 같아 보이지만 전자는 "이 패키지 전부/아무것도"밖에 표현 못 하고, 후자는 애노테이션·반환타입·파라미터·제외조건까지 표현한다. **판단 로직을 문자열 비교로 하드코딩하려는 순간, 이미 존재하는 표현력 있는 추상화가 있는지 먼저 찾아보는 게 낫다.**

**추상화 계층이 늘어날수록 "몇 단계에서 걸렀는지"를 물어야 한다.** 이 챕터에서 배운 가장 실무적인 감각은 포인트컷이 **클래스 단계와 메서드 단계 두 번** 쓰인다는 것이다. 증상이 "프록시가 아예 없음"인지 "프록시는 있는데 어드바이스만 안 걸림"인지에 따라 봐야 할 곳이 완전히 다르다. `AopUtils.isAopProxy(bean)`으로 1단계를, 로그로 2단계를 확인하는 순서가 몸에 붙어야 한다.

---

## 참고

- 김영한, 스프링 핵심 원리 고급편 — Ch.7 빈 후처리기
- [Spring Framework Javadoc — `BeanPostProcessor`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/config/BeanPostProcessor.html) (정렬 규칙: `@Order` 미적용, 프로그래밍 등록 시 정렬 무시)
- [Spring Framework Javadoc — `CommonAnnotationBeanPostProcessor`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/CommonAnnotationBeanPostProcessor.html) (`InitDestroyAnnotationBeanPostProcessor` 상속, `PriorityOrdered`)
- [`PostProcessorRegistrationDelegate` 소스](https://github.com/spring-projects/spring-framework/blob/main/spring-context/src/main/java/org/springframework/context/support/PostProcessorRegistrationDelegate.java) (PriorityOrdered → Ordered → 나머지 → internal 재등록)
- [Spring Boot 2.6 Release Notes — Circular References Prohibited by Default](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6-Release-Notes#circular-references-prohibited-by-default)
- [Spring Framework Reference — AOP APIs](https://docs.spring.io/spring-framework/reference/core/aop-api.html)

**학습 날짜**: 2026-08-14 · **계기**: 김영한 고급편 Ch.7 수강 후 Claude 소크라테스 복습 세션 — "자동화가 갑자기 많아지고 처음 보는 함수가 늘어서 이해가 안 된다"고 느껴 복습 시작. ① 4~6장의 **문제 인식(설정 지옥 + 컴포넌트 스캔)은 정확히 기억**했고 이게 전체 이해의 앵커가 됨 ② ⚠️ **Before/After 기준을 "빈 등록 전/후"로 오해** — 실제로는 `@PostConstruct` 기준이고 둘 다 등록 *전*임 ③ `@PostConstruct`가 무엇인지 몰라 세션 중 직접 질문 → 빈 생명주기 전체를 다시 정리 ④ `PackageLogTraceProxyPostProcessor`의 필터 기준을 "메서드 이름 매칭"으로 오답 (실제는 패키지) — 메서드 이름은 Pointcut의 역할이라 혼동한 것 ⑤ 포인트컷의 **이중 사용(클래스/메서드)은 정확히 답함** ⑥ ⚠️ **여러 Advisor가 있을 때 각각 독립 판단**한다는 점을 놓쳐, `getStatus()` 호출 시 "어드바이저가 적용 안 된다"고 답함 (실제로는 로그 Advisor는 적용됨) ⑦ 세션 후반에 "빈 저장소 등록이 무슨 뜻이냐" → "DI가 끝났다는 게 다 주입됐다는 거냐" → "그럼 등록 순서가 중요하겠네" → **순환 참조를 스스로 도출**, 회사에서 겪은 기동 실패 경험과 연결됨. 📌 세션 중 Claude가 "`@Order`로도 BPP 순서를 정할 수 있다"고 답한 것은 **오답** — 공식 Javadoc 확인 결과 BPP에는 `@Order`가 적용되지 않으며, 이 노트에서 정정함

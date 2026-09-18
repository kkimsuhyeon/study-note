# ApplicationContext — 스프링 컨테이너 그 자체 / 언제 직접 꺼내 쓰나

> **한 줄 요약**: `ApplicationContext`는 빈을 **만들고·보관하고·연결하는 스프링 IoC 컨테이너 객체**다. 빈 저장소(`BeanFactory`)에 환경설정(`Environment`)·이벤트·리소스·메시지 기능을 더한 것. 평소엔 컨테이너가 의존성을 **밀어넣어 주고(DI)** 우리는 컨테이너의 존재를 모른 채 코드를 쓴다. 컨테이너를 **직접 쥐고 `getBean`으로 꺼내는** 건 "빈 이름이 런타임에 정해진다" 같은 예외 상황에서만 정당하고, 습관이 되면 **서비스 로케이터 안티패턴**이 된다.

관련 노트: [빈 후처리기](../design/bean-post-processor.md) · [Spring 이벤트](./application-events.md) · [@Transactional(프록시 함정)](./transactional.md) · [동적 스케줄링](./dynamic-scheduling.md)

---

## 1. 정체 — "스프링이 해준다"의 실체는 객체 하나

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = SpringApplication.run(App.class, args);  // ← 이게 컨테이너
        OrderService svc = ctx.getBean(OrderService.class);                            // 안에 빈이 다 들어 있다
    }
}
```

기동 시 컨테이너가 하는 일을 순서대로 놓으면 다음과 같다. 우리가 쓰는 어노테이션 대부분이 이 중 한 단계에 끼어드는 훅이다.

| 단계 | 하는 일 | 끼어드는 것 |
|---|---|---|
| ① 빈 **정의** 수집 | 컴포넌트 스캔·`@Bean` 메서드를 읽어 "무엇을 만들지" 목록(BeanDefinition) 작성 | `@Component`, `@Configuration`, `@Bean` |
| ② 정의 후처리 | 정의 목록 자체를 수정·추가 | `BeanFactoryPostProcessor`, `@ConfigurationProperties` 등록 |
| ③ 인스턴스화 + 주입 | 생성자 호출 → 필드/세터 주입 | `@Autowired`, 생성자 주입 |
| ④ Aware 콜백 | 빈에게 컨테이너 정보를 알려줌 | `BeanNameAware`, **`ApplicationContextAware`** |
| ⑤ 초기화 | 초기화 메서드 호출 | `@PostConstruct`, `InitializingBean` |
| ⑥ 빈 후처리 | 만들어진 객체를 **다른 객체(프록시)로 바꿔치기** 가능 | `BeanPostProcessor` → [빈 후처리기](../design/bean-post-processor.md), `@Transactional`·AOP 프록시 |
| ⑦ 준비 완료 | 이벤트 발행 | `ContextRefreshedEvent`, `ApplicationReadyEvent` |

> ④가 ⑤보다 먼저다. 그래서 `ApplicationContextAware`로 받은 컨테이너를 같은 빈의 `@PostConstruct`에서 쓰는 건 안전하다. 반대로 **다른 빈**이 아직 ③~⑥ 중이면 `getBean`으로 받은 게 미완성일 수 있다 (§7).

### 어노테이션은 표식, 일하는 건 processor

위 표의 "끼어드는 것" 열을 정확히 말하면 — **어노테이션 자체는 아무 일도 안 한다.** 클래스·메서드·필드에 붙은 메타데이터(표식)일 뿐이다. 실제 일은 그 단계의 훅에 등록된 스프링 내장 processor가 "이 빈에 내 표식이 붙어 있나" 확인해서 **대신** 한다. 표식과 읽는 쪽이 항상 짝으로 있다 — `@Autowired`를 붙이면 값이 꽂히는 게 아니라, ③단계의 `AutowiredAnnotationBeanPostProcessor`가 표식을 보고 꽂아주는 것.

컨테이너는 **빈 하나를 ③→⑤→⑥ 끝까지 만든 뒤 다음 빈**으로 넘어간다. 어노테이션을 한 번에 훑는 게 아니라 **빈 단위·단계 단위**로 돈다. 각 단계에서 등록된 processor들을 차례로 호출하고, 각 processor는 자기 어노테이션이 있으면 처리하고 없으면 그냥 넘긴다. 그래서 "어느 단계냐"가 중요해진다.

| 어노테이션 | 읽는 processor | 단계 | 하는 일 |
|---|---|---|---|
| `@Component`, `@Bean`, `@ComponentScan` | `ConfigurationClassPostProcessor` (BeanFactoryPostProcessor) | ① | BeanDefinition 목록에 추가 |
| `@Autowired`, `@Value`, `@Inject` | `AutowiredAnnotationBeanPostProcessor` | ③ | 필드·세터·생성자에 값 주입 |
| `@PostConstruct`, `@PreDestroy`, `@Resource` | `CommonAnnotationBeanPostProcessor` (Before) | ⑤ | 초기화 메서드 호출 |
| `@ConfigurationProperties` | `ConfigurationPropertiesBindingPostProcessor` (Before, Boot) | ⑤ | 프로퍼티 바인딩 |
| `@Transactional`, `@Aspect` 대상 | `InfrastructureAdvisorAutoProxyCreator` / `AnnotationAwareAspectJAutoProxyCreator` (After) | ⑥ | 원본을 프록시로 바꿔치기 |
| `@Async` | `AsyncAnnotationBeanPostProcessor` (After) | ⑥ | 원본을 프록시로 바꿔치기 |
| `@Scheduled` | `ScheduledAnnotationBeanPostProcessor` (After) | ⑥ | 메서드 찾아 스케줄러에 등록 |
| `@EventListener` | `EventListenerMethodProcessor` (SmartInitializingSingleton) | ⑦ 직전 | 리스너로 등록 |

→ AOP(`@Transactional` 프록시)는 "이 메커니즘과 비슷한 것"이 아니라 **이 메커니즘의 사례 하나**다. ⑥단계에 등록된 processor 한 개가 하는 일. 프록시로 바꿔치기하는 원리 자체는 [빈 후처리기](../design/bean-post-processor.md).

⚠️ **⑤가 ⑥보다 앞이라서 생기는 함정**: `@PostConstruct` 메서드가 실행되는 ⑤ 시점엔 **이 빈의 프록시가 아직 만들어지지 않았다.** 그래서 초기화 메서드 안에서 같은 빈의 `@Transactional` 메서드를 `this.xxx()`로 불러도 트랜잭션이 안 걸린다. 다른 빈을 부르는 건 그 빈이 이미 프록시라 정상 → 자세히는 [@Transactional §6(1)](./transactional.md).

💡 **"이 어노테이션 왜 안 먹지?"는 항상 두 질문으로 쪼개진다.**
1. **읽는 processor가 등록돼 있나?** — `@EnableScheduling` 없이 `@Scheduled`, `@EnableAsync` 없이 `@Async`는 표식만 있고 읽는 쪽이 없어 조용히 무시된다. (`@Transactional`은 Boot가 `@EnableTransactionManagement`를 자동으로 켜줘서 잊기 쉽다.)
2. **내 코드가 그 processor의 단계보다 앞에서 도나?** — `@PostConstruct`(⑤) 안에서 `@Transactional`(⑥)이 그 예.

커스텀 어노테이션을 만들어 붙여도 아무 일이 안 일어나는 이유도 같다 — **읽는 processor를 내가 써야 한다** ([빈 후처리기 §4](../design/bean-post-processor.md)의 직접 만든 후처리기가 그 예).

### 인터페이스 계층 — "BeanFactory + 부가 기능"

```java
public interface ApplicationContext extends
        EnvironmentCapable,        // getEnvironment()          — 설정·프로필
        ListableBeanFactory,       // getBean, getBeansOfType   — 빈 저장소 (BeanFactory 확장)
        HierarchicalBeanFactory,   // getParentBeanFactory      — 부모 컨텍스트
        MessageSource,             // getMessage                — 다국어 메시지
        ApplicationEventPublisher, // publishEvent              — 이벤트
        ResourcePatternResolver    // getResource(s)            — classpath:/file: 리소스
```

`BeanFactory`가 "빈 저장소"라는 최소 계약이고, `ApplicationContext`는 그 위에 엔터프라이즈 기능을 얹은 것. 실무에서 `BeanFactory`를 직접 쓸 일은 없고 항상 `ApplicationContext`(또는 그 중 필요한 인터페이스 하나)를 쓴다.

---

## 2. 할 수 있는 것 다섯 가지

| 역할 | 대표 메서드 | 보통은 이걸로 대신한다 |
|---|---|---|
| 빈 조회 | `getBean(name)`, `getBean(Class)`, `getBean(name, Class)`, `getBeansOfType(Class)` | 생성자 주입, `List<T>`/`Map<String,T>` 주입, `ObjectProvider<T>` |
| 환경·설정 | `getEnvironment().getProperty(key)`, `acceptsProfiles(...)`, `getActiveProfiles()` | `@Value`, `@ConfigurationProperties`, `@Profile` |
| 이벤트 | `publishEvent(event)` | `ApplicationEventPublisher` 주입 |
| 리소스 | `getResource("classpath:...")`, `getResources("classpath*:**/*.xml")` | `ResourceLoader` 주입, `@Value("classpath:...") Resource` |
| 메시지 | `getMessage(code, args, locale)` | `MessageSource` 주입 |

```java
// 빈 조회
OrderService a = ctx.getBean(OrderService.class);                       // 타입으로 (유일해야 함)
AbstactCommand b = ctx.getBean("marketingAgreeNotifyCommand", AbstactCommand.class);  // 이름 + 타입
Map<String, AbstactCommand> all = ctx.getBeansOfType(AbstactCommand.class);  // 키 = 빈 이름
boolean exists = ctx.containsBean("someBean");

// 환경·설정 — "설정을 가져올 수 있나?" → 있다
Environment env = ctx.getEnvironment();
String url   = env.getProperty("spring.datasource.url");
int poolSize = env.getProperty("batch.pool-size", Integer.class, 4);      // 타입 변환 + 기본값
boolean demo = env.acceptsProfiles(Profiles.of("demo"));

// 이벤트 / 리소스 / 메시지
ctx.publishEvent(new OrgCreatedEvent(orgId));
Resource ftl = ctx.getResource("classpath:templates/welcome.ftl");
String msg   = ctx.getMessage("error.invalid", new Object[]{field}, Locale.KOREA);
```

> 표의 오른쪽 열이 핵심이다. **컨테이너가 할 수 있는 일 대부분은 컨테이너를 쥐지 않고도 "그 기능만 담당하는 좁은 인터페이스"를 주입받아 할 수 있다.** `ApplicationContext` 전체를 받는 건 다섯 기능을 한꺼번에 받는 것이라 의존성이 과하게 넓어진다.

---

## 3. 컨테이너 참조를 얻는 네 가지 방법

```java
// ① 생성자 주입 — 권장. ApplicationContext도 그냥 빈처럼 주입된다
@Component
@RequiredArgsConstructor
public class DynamicBatch {
    private final ApplicationContext applicationContext;
}

// ② ApplicationContextAware — 컨테이너가 ④단계에서 setApplicationContext()를 호출해 준다
@Component
public class SomeLegacyBean implements ApplicationContextAware {
    private ApplicationContext ctx;
    @Override public void setApplicationContext(ApplicationContext ctx) { this.ctx = ctx; }
}

// ③ static 홀더 — 빈이 아닌 코드에서 접근하기 위해 ②를 static 필드에 담아두는 변형
@Component
public class AppContextProvider implements ApplicationContextAware {
    private static ApplicationContext appContext;
    @Override public void setApplicationContext(ApplicationContext ctx) { appContext = ctx; }
    public static ApplicationContext get() { return appContext; }
}

// ④ SpringApplication.run() 반환값 — main 메서드·테스트 코드에서
ConfigurableApplicationContext ctx = SpringApplication.run(App.class, args);
```

| 방법 | 언제 | 주의 |
|---|---|---|
| ① 생성자 주입 | 빈 안에서 컨테이너가 필요할 때. **기본값** | 그래도 §6의 좁은 대안이 먼저 |
| ② Aware | 생성자 주입을 못 쓰는 레거시·추상 상위 클래스 | ①로 대체 가능하면 ① |
| ③ static 홀더 | **빈이 아닌 코드**(static 유틸, 역직렬화된 객체, 레거시 서블릿 필터)가 빈을 써야 할 때 | 초기화 순서·테스트 격리 함정(§7). **최후의 수단** |
| ④ run 반환값 | 부트스트랩 코드·통합 테스트 | 애플리케이션 코드에 흘려보내지 않기 |

> ⚠️ 위 pharos_batch의 `AppContextProvider`는 ③ 패턴인데, 실제 사용처는 `DynamicBatch` **한 곳이고 그 클래스는 이미 빈**이다. 빈이면 ①로 받으면 끝이라 ③을 거칠 이유가 없다. "static으로 감싸는 건 빈이 아닌 코드를 위한 것"이라는 용도를 모르고 관성으로 만든 우회.

---

## 4. 설정 읽기 — `Environment` vs `@Value` vs `@ConfigurationProperties`

세 가지가 결국 같은 `Environment`를 읽는다. `@Value`는 내부에서 `Environment`의 플레이스홀더 해석을 쓰고, `@ConfigurationProperties`는 `Binder`가 `Environment`를 바인딩한다. 차이는 **키가 언제 정해지느냐**와 **얼마나 묶어 받느냐**.

```java
// 키 하나, 컴파일 시점 고정 → @Value
@Value("${batch.pool-size:4}") private int poolSize;

// 키 여러 개가 한 묶음, 타입 안전 → @ConfigurationProperties
@ConfigurationProperties(prefix = "scrap")
public record ScrapProps(Duration timeout, int retry, String baseUrl) {}   // scrap.timeout, scrap.retry, ...

// 키가 런타임에 만들어짐 → Environment 직접
String key = "batch." + jobName + ".enabled";          // jobName은 DB에서 옴
boolean enabled = environment.getProperty(key, Boolean.class, true);
```

| | `@Value` | `@ConfigurationProperties` | `Environment.getProperty` |
|---|---|---|---|
| 키 결정 시점 | 컴파일 | 컴파일 (prefix) | **런타임** |
| 묶음 | 값 1개 | 객체 1개(여러 키) | 값 1개씩 |
| 완화 바인딩(`pool-size`↔`poolSize`) | ✕ 정확한 키 | ✓ | ✕ 정확한 키 |
| 검증(`@Validated`) | ✕ | ✓ | ✕ |
| 쓸 곳 | 단발 설정값 | 설정 묶음(권장 기본) | 동적 키, 조건 분기, 프레임워크성 코드 |

> ⚠️ **완화 바인딩은 `Binder`(=`@ConfigurationProperties`) 기능**이다. `environment.getProperty("batch.poolSize")`는 yml에 `pool-size`로 적혀 있으면 못 찾는다. 환경변수 `BATCH_POOL_SIZE`→`batch.pool-size` 매핑은 별도 프로퍼티 소스가 해주니 그건 된다.

---

## 5. 서비스 로케이터 안티패턴 — 이렇게 쓰면 잘못 쓰는 것

**정의**: 객체가 자기 의존성을 **주입받는 대신 컨테이너(로케이터)에게 요청해서 꺼내오는** 방식. Fowler가 DI와 나란히 놓고 비교한 패턴인데, 스프링처럼 DI가 기본인 환경에서 로케이터식으로 쓰면 DI의 장점을 스스로 버리는 꼴이 된다.

```java
// ❌ 1. 주입 대신 조회 — 생성자만 보면 PaymentService에 의존하는지 알 수 없다
@Service
@RequiredArgsConstructor
public class OrderService {
    private final ApplicationContext ctx;

    public void place(Order order) {
        PaymentService payment = ctx.getBean(PaymentService.class);   // 그냥 주입받으면 되는 걸 조회
        payment.pay(order);
    }
}

// ❌ 2. static 홀더 + 유틸 — 어디서든 꺼낼 수 있게 만들어 놓으면 어디서든 꺼낸다
public class Beans {
    public static <T> T get(Class<T> type) { return AppContextProvider.get().getBean(type); }
}
public class Order {                       // 도메인 객체
    public void confirm() {
        Beans.get(OrderRepository.class).save(this);   // 도메인이 컨테이너·저장소에 묶임 → new Order()로 단위 테스트 불가
    }
}

// ❌ 3. 순환 참조 우회용 — A↔B가 서로 필요해서 한쪽을 getBean으로 늦게 꺼냄
@Service
public class A {
    private final ApplicationContext ctx;
    void run() { ctx.getBean(B.class).doSomething(); }   // 순환은 설계 신호인데 신호만 지움
}
```

**왜 나쁜가** — 네 가지 비용이 동시에 발생한다.

| 비용 | 설명 |
|---|---|
| 의존성 은닉 | 생성자 시그니처가 거짓말을 한다. "이 클래스가 뭘 필요로 하나"를 알려면 본문 전체를 읽어야 한다 |
| 테스트 | `new OrderService(mockPayment)`가 불가능. 컨테이너를 통째로 띄우는 `@SpringBootTest`로 강제된다 |
| 컴파일 타임 검증 상실 | 주입은 빈이 없으면 **기동 시** 실패. `getBean`은 그 코드가 **실행되는 순간** `NoSuchBeanDefinitionException`. 이름 문자열 오타는 IDE가 못 잡는다 |
| 프레임워크 결합 | 도메인·유틸 코드가 `org.springframework` 패키지를 import 한다. 스프링 없이 재사용·테스트 불가 |

> 💡 **반대 의견도 있다.** Fowler는 "둘 다 유효하며, 로케이터는 단순하고 DI는 의존성이 드러난다"고 중립적으로 썼고, 프레임워크·플러그인 레지스트리처럼 "무엇이 등록될지 모르는" 인프라 코드에서는 로케이터가 자연스럽다. Seemann은 "애플리케이션 코드에서는 안티패턴"으로 못 박았다. 이 노트의 입장: **애플리케이션 코드에서는 Seemann, 프레임워크성 코드(§6-b)에서는 Fowler.** 그리고 스프링은 그 중간 지대를 위해 `ObjectProvider`·`Map<String,T>` 주입 같은 "절제된 조회" 도구를 이미 제공한다.

---

## 6. 실제로는 이렇게 쓴다 — 상황별 1순위 도구

| 상황 | 1순위 | 컨테이너 직접 조회는 |
|---|---|---|
| a. 같은 타입 빈 여러 개 중 고르기 (전략 패턴) | `List<T>` / `Map<String,T>` 주입 → 자체 Map 구성 | 불필요 |
| b. 빈 이름이 DB·설정에서 옴 | `Map<String,T>` 주입 + 기동 시 검증 | 2순위 (`getBean(name, T.class)`) |
| c. 지연 조회·선택적 의존·프로토타입 | `ObjectProvider<T>`, `@Lazy`, `@Lookup` | 불필요 |
| d. 이벤트 발행 | `ApplicationEventPublisher` 주입 | 불필요 |
| e. 설정 값 | `@ConfigurationProperties` / `@Value` (§4) | 동적 키만 `Environment` |
| f. 빈이 아닌 코드가 빈을 써야 함 | **그 코드를 빈으로 만들 수 없는지 먼저 질문** | 정말 불가할 때만 static 홀더 |
| g. 런타임에 빈을 **등록** | `GenericApplicationContext.registerBean`, `BeanDefinitionRegistryPostProcessor` | 프레임워크 작성자 영역. 앱 코드엔 거의 없음 |

### a. 전략 패턴 — 컨테이너가 목록을 채워준다

스프링은 `List<T>`·`Map<String, T>`(키=빈 이름) 타입의 주입 지점을 보면 **그 타입의 빈 전부**를 채워 넣는다. 그래서 "핸들러들 중 조건에 맞는 하나"를 고르는 파인더는 컨테이너 없이 쓸 수 있다.

```java
// client_api — 엑셀 타입별 핸들러 파인더
@Component
public class ExcelTypeHandlerFinder {
    private final Map<String, ExcelTypeHandler> handlers;

    public ExcelTypeHandlerFinder(List<ExcelTypeHandler> handlers) {          // ← 스프링이 ExcelTypeHandler 빈 전부를 넣어줌
        this.handlers = handlers.stream()
                .collect(toMap(ExcelTypeHandler::getSupportedType, identity()));  // 키를 "빈 이름"이 아니라 도메인 키로 재구성
    }

    public Optional<ExcelTypeHandler> find(String type) { return Optional.ofNullable(handlers.get(type)); }
}
```

한 단계 더 가면 **기동 시점 검증**까지 넣는다. 담당이 비었거나 겹치면 서버가 뜨지 않게 해서 배선 실수를 런타임까지 끌고 가지 않는다.

```java
// client_api — 스크래핑 종류별 핸들러 레지스트리
public ScrapDataHandlers(List<ScrapDataHandler<?>> handlers) {
    this.handlerByTarget = handlers.stream()
            .flatMap(h -> h.targets().stream().map(t -> Map.entry(t, h)))
            .collect(toMap(Map.Entry::getKey, Map.Entry::getValue, this::rejectDuplicate));  // 겹치면 예외
    validateAllTargetsCovered();     // enum 전부에 담당이 있는지 → 없으면 기동 실패
}
```

> 파인더가 받는 키를 **빈 이름이 아니라 도메인 키(enum·코드)**로 바꾸는 게 포인트다. 빈 이름은 클래스명에서 자동 생성되는 부산물이라 리팩토링에 약하다. `handler.getSupportedType()`처럼 핸들러가 스스로 "나는 이걸 담당한다"고 선언하게 하면 이름 변경에 안전하다.

### b. 빈 이름이 외부 데이터에서 올 때 — `getBean`이 정당한 유일한 흔한 경우, 그러나 Map이 더 낫다

pharos_batch `DynamicBatch`: DB 테이블에 `job_command = 'marketingAgreeNotifyCommand'`처럼 **빈 이름이 문자열로** 저장돼 있고, 기동 시 그 이름으로 빈을 찾아 스케줄에 묶는다 (구조 전체는 [동적 스케줄링](./dynamic-scheduling.md)).

```java
// 현재 — 정당한 조회지만 두 가지 아쉬움: static 홀더 우회(§3) + 오타 한 건이 전체를 죽임
ApplicationContext ctx = appContextProvider.getAppContext();
for (SchdJobCommandMapp mapp : mappList) {
    AbstactCommand command = ctx.getBean(mapp.getJobCommand(), AbstactCommand.class);  // 이름 오타 → 예외 → 바깥 try가 init 전체 중단
    ...
}

// 대안 — Map 주입. 없으면 null이라 "그 건만 건너뛰고 경고" 가능
private final Map<String, AbstactCommand> commands;   // 키 = 빈 이름, 스프링이 채움

for (SchdJobCommandMapp mapp : mappList) {
    AbstactCommand command = commands.get(mapp.getJobCommand());
    if (command == null) { log.warn("정의 없는 job_command: {}", mapp.getJobCommand()); continue; }
    ...
}
```

`getBean`은 "없으면 예외"가 기본 의미라 **부분 실패를 전체 실패로 승격**시킨다. DB에서 온 이름은 오타·대소문자 실수가 언제든 들어올 수 있는 입력이므로, "없을 수 있다"를 전제한 Map 조회가 의미에 맞는다.

### c. 지연·선택적·프로토타입 — `ObjectProvider<T>`

```java
// client_api — 순환 의존 회피 (마이그레이션 후 삭제 예정 FIXME가 붙어 있음)
private final ObjectProvider<SalMgmtInpService> salMgmtInpServiceProvider;

SalMgmtInpService svc = salMgmtInpServiceProvider.getObject();          // 실제 필요한 시점에 꺼냄
```

| `ObjectProvider` 메서드 | 의미 |
|---|---|
| `getObject()` | 꺼내기. 없으면 예외 |
| `getIfAvailable()` / `getIfAvailable(Supplier)` | 없으면 null / 기본값 |
| `getIfUnique()` | 여러 개면 null |
| `ifAvailable(Consumer)` | 있을 때만 실행 |
| `stream()` / `orderedStream()` | 전부 (Ordered 정렬) |

`getBean`을 감싼 것과 다르지 않지만, **"어떤 타입의 빈을, 필요할 때, 없을 수도 있음을 전제로"**라는 의도가 시그니처에 드러난다. 그게 로케이터와의 차이다. 프로토타입 스코프 빈을 매번 새로 받을 때, 순환 의존을 잠시 풀 때(Boot 2.6+는 순환 참조를 기본 금지하므로 `@Lazy`와 함께 흔한 우회), 선택적 의존(라이브러리가 있을 때만)에 쓴다. 단 순환 의존 우회로 쓴다면 위 코드처럼 **FIXME를 남기고 설계를 고치는 게 원칙** — 순환은 책임이 잘못 나뉘었다는 신호다.

### d. 이벤트 — 컨테이너 전체 대신 발행 인터페이스만

```java
private final ApplicationEventPublisher publisher;    // ApplicationContext가 이 인터페이스를 상속하므로 어느 쪽이든 되지만
publisher.publishEvent(new DemoOrgCreatedEvent(orgId));  // 필요한 능력만 받는다 (client_api 35개 클래스가 이 방식)
```

### f. 빈이 아닌 코드 — "빈으로 만들 수 없나"를 먼저

static 유틸에서 `MessageSource`가 필요하다면, 답은 static 홀더가 아니라 **그 유틸을 `@Component`로 바꾸고 주입받는 것**이다. 정말 불가능한 경우(역직렬화로 생성되는 객체, 서블릿 컨테이너가 직접 만드는 레거시 필터, 프레임워크가 `new`로 만드는 콜백)에만 ③ static 홀더를 쓰되, 클래스 이름에 정체를 드러내고(`SpringContextHolder`) 사용처를 한두 곳으로 묶어 둔다.

---

## 7. ⚠️ 함정

- **빈 이름은 대소문자 구분, 기본 이름은 `Introspector.decapitalize(클래스명)`** — `MarketingAgreeNotifyCommand` → `marketingAgreeNotifyCommand`. 단 **앞 두 글자가 모두 대문자면 그대로**(`URLParser` → `URLParser`, `DBConfig` → `DBConfig`). DB에 이름을 저장하는 구조면 이 규칙을 문서에 박아둘 것. pharos_batch DB엔 `DailyPassOrgMpaaUpdateCommand`처럼 대문자 시작 행이 있는데 `step 0`(비활성)이라 살아 있을 뿐, 활성화 순간 init 전체가 죽는다.
- **`getBean(Class)`에 후보가 둘 이상이면 `NoUniqueBeanDefinitionException`** — 인터페이스 타입으로 조회할 때 흔하다. `@Primary`/`@Qualifier`로 하나를 지정하거나, 여러 개가 정상이면 `Map<String,T>`로 받는다.
- **static 홀더의 초기화 순서** — static 필드는 `AppContextProvider` **빈이 만들어지는 ④단계**에 채워진다. 그보다 먼저 만들어지는 빈의 생성자·`@PostConstruct`에서 static을 통해 접근하면 null. 빈 생성 순서는 의존 관계로만 보장되므로 "우연히 됐다"가 배포 환경에서 깨질 수 있다. 또 테스트에서 컨텍스트가 여러 개 뜨면(슬라이스 테스트·컨텍스트 캐시) static은 **마지막 컨텍스트로 덮어써져** 다른 테스트가 엉뚱한 컨테이너를 본다.
- **`@PostConstruct` 안에서 다른 빈 `getBean`** — 그 빈이 아직 생성 중이면 미완성 참조(프록시 안 붙은 원본 등)를 받을 수 있다. 특히 순환 구조에서. 준비 완료 뒤에 해야 하는 조회는 `ApplicationReadyEvent`/`ContextRefreshedEvent` 리스너나 `SmartInitializingSingleton.afterSingletonsInstantiated()`로 옮긴다.
- **`getBean`은 프록시를 돌려준다** — `@Transactional`·AOP가 붙은 빈은 주입받든 조회하든 프록시다. 자기호출 함정 등은 그대로 → [@Transactional](./transactional.md).
- **`getBeansOfType`·`Map<String,T>`는 프로토타입 스코프 빈을 매번 새로 만든다** — 조회 자체가 생성이라 부작용이 있는 프로토타입이면 주의. 실무선 싱글턴이 대부분이라 드물지만 알고 있어야 한다.
- **완화 바인딩은 `Environment`에 없다** (§4).

---

## 💡 판단 기준

- **"주입으로 못 받는 이유를 한 문장으로 말할 수 있나?"** 못 하면 주입이다. "이름이 DB에서 온다"는 말할 수 있고, "그때그때 필요해서"는 말이 안 된다.
- **컨테이너 전체보다 좁은 인터페이스를 받는다** — 이벤트면 `ApplicationEventPublisher`, 설정이면 `Environment`(그것도 동적 키일 때만), 여러 빈 중 선택이면 `Map<String,T>`, 지연이면 `ObjectProvider<T>`. `ApplicationContext`를 받는 순간 "이 클래스는 컨테이너의 모든 능력에 의존한다"고 선언하는 셈이다.
- **동적 이름은 `Map<String,T>` 주입 + 기동 시 검증이 1순위, `getBean(name)`은 2순위** — `getBean`은 "없으면 예외"라 부분 실패를 전체 실패로 키운다. 외부에서 온 이름은 "없을 수 있는 입력"이다.
- **파인더의 키는 빈 이름이 아니라 도메인 키로** — 빈 이름은 클래스명의 부산물이라 리팩토링에 약하다. 핸들러가 스스로 담당 키를 선언하게 한다 (`ScrapDataHandlers`, `ExcelTypeHandlerFinder`).
- **static 홀더는 최후의 수단이고, 쓰면 "빈으로 만들 수 없는 이유"를 주석으로 남긴다** — 이유가 "편해서"면 빈으로 바꾼다.
- 구체 케이스: pharos_batch `DynamicBatch`는 조회 자체는 정당(이름이 DB에서 옴)하지만 ⓐ 이미 빈인데 static 홀더를 거치고 ⓑ `getBean`이라 오타 한 건이 스케줄 전체를 죽인다. `Map<String, AbstactCommand>` 주입 하나로 둘 다 사라진다.

---

## 참고

- [Spring Framework Reference — The IoC Container: Introduction](https://docs.spring.io/spring-framework/reference/core/beans/introduction.html) · [Container Overview](https://docs.spring.io/spring-framework/reference/core/beans/basics.html)
- [Spring Framework Reference — ApplicationContextAware and BeanNameAware](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html#beans-factory-aware)
- [Spring Framework Reference — Additional Capabilities of the ApplicationContext](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html) (MessageSource·이벤트·리소스)
- [Spring Framework Reference — Using @Autowired](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired.html) (배열·컬렉션·`Map<String,T>` 주입, 키=빈 이름)
- [Spring Framework Reference — Environment Abstraction](https://docs.spring.io/spring-framework/reference/core/beans/environment.html)
- [Spring Framework Reference — Container Extension Points (BeanPostProcessor·BeanFactoryPostProcessor)](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html) · [Lifecycle Callbacks](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html#beans-factory-lifecycle) (어노테이션 = 표식, processor = 읽는 쪽)
- [Spring Framework Reference — Method Injection (@Lookup)](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-method-injection.html)
- [Javadoc — ApplicationContext](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/ApplicationContext.html) · [ObjectProvider](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/ObjectProvider.html) · [GenericApplicationContext.registerBean](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/support/GenericApplicationContext.html)
- [Spring Boot Reference — Externalized Configuration (Relaxed Binding)](https://docs.spring.io/spring-boot/reference/features/external-config.html)
- [Martin Fowler — Inversion of Control Containers and the Dependency Injection pattern](https://martinfowler.com/articles/injection.html) (Service Locator vs DI 비교의 원전)
- [Mark Seemann — Service Locator is an Anti-Pattern](https://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/)
- 관련 노트: [빈 후처리기](../design/bean-post-processor.md) · [Spring 이벤트](./application-events.md) · [@Transactional](./transactional.md) · [동적 스케줄링](./dynamic-scheduling.md)

---

**학습 날짜**: 2026-09-16
**계기**: pharos_batch `DynamicBatch`의 `private final AppContextProvider appContextProvider;`가 뭔지에서 출발. "코드가 돌면서 Component를 직접 만드는 건가?" → 아니, 이미 있는 빈을 **이름으로 꺼내는** 것 → 그럼 컨테이너가 뭘 할 수 있고, 언제 직접 쥐어야 하며, 어떻게 쓰면 안티패턴인가까지 정리. client_api의 `ExcelTypeHandlerFinder`·`ScrapDataHandlers`·`ObjectProvider` 사용처를 "실제로는 이렇게"의 예로 붙임.

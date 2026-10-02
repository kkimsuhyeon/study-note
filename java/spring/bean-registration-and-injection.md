# @Bean 등록과 타입 기반 주입

`@Bean` 메서드의 매개변수는 생성에 필요한 의존성이고, 반환 객체가 새 Bean으로 등록된다. 내부에서 사용한 객체의 타입과 반환 타입을 구분한다.

## 언제 쓰나

외부 라이브러리 객체나 HTTP 인터페이스 프록시를 직접 조립해 Spring에서 주입받을 때 사용한다. `HttpClient`를 사용해 만든 객체가 모두 `HttpClient` Bean이 되는 것은 아니다.

## 사용 예시

Spring 설정 클래스 안의 예시다. `RemoteCatalogClient`는 `@GetExchange` 메서드를 선언한 사용자 정의 인터페이스이며, JDK `HttpClient`를 상속하지 않는다.

```java
@Bean
public HttpClient catalogTransport() {
    return HttpClient.newHttpClient();
}

@Bean
public RemoteCatalogClient remoteCatalogClient(
        @Qualifier("catalogTransport") HttpClient transport) {
    var factory = new JdkClientHttpRequestFactory(transport);
    RestClient client = RestClient.builder()
            .baseUrl("https://example.com")
            .requestFactory(factory)
            .build();

    return HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(client))
            .build()
            .createClient(RemoteCatalogClient.class);
}
```

| 구분 | 역할 |
|---|---|
| `HttpClient transport` | 기존 Bean을 주입받는 입력 |
| 메서드 내부 `RestClient client` | 프록시 구성에 사용하는 일반 객체. 이 선언만으로 Bean이 되지 않음 |
| 반환한 `RemoteCatalogClient` | HTTP 요청을 수행하는 프록시이며, 새로 등록되는 Bean |

기본 Bean 이름은 `@Bean` 메서드 이름이다. 위에서는 `catalogTransport`와 `remoteCatalogClient` 두 개를 등록한다. 매개변수 이름을 `client`나 `api`로 바꿔도 객체의 Java 타입은 바뀌지 않는다.

다른 Bean의 생성자 또는 `@Bean` 메서드가 `RemoteCatalogClient`를 요청하면 해당 타입의 후보를 찾는다. `HttpClient`를 요청하면 JDK 클라이언트 후보를 찾는다. HTTP 프록시의 메서드를 호출하면 설정된 RestClient와 transport를 통해 요청이 실행된다.

## RestClient·RequestFactory·HttpClient의 역할 차이

| 구성 요소 | 담당 역할 |
|---|---|
| HTTP 인터페이스 프록시 | 어노테이션과 메서드 인자를 HTTP 요청 정보로 바꿈 |
| `RestClient` | URL·헤더·본문 구성, 상태 코드 처리, 응답 본문 변환 |
| `ClientHttpRequestFactory` | Spring의 요청 객체를 만드는 공통 규격 |
| `JdkClientHttpRequestFactory` | JDK `HttpClient`를 이용하는 요청 객체를 생성하는 구현 |
| `java.net.http.HttpClient` | 실제 HTTP 통신 수행 |

`JdkClientHttpRequestFactory`의 주된 역할은 HttpClient Bean 생성이 아니다. `createRequest(URI, HttpMethod)`로 요청 객체를 만들고, 그 요청이 실행될 때 전달받은 HttpClient를 사용한다. 따라서 factory 생성 자체는 외부 API 호출이 아니다.

```text
인터페이스 메서드 호출 → 프록시 → RestClient
→ RequestFactory가 만든 요청 실행 → HttpClient → 외부 서버
```

### 명시하지 않았을 때: 자동 선택

```java
RestClient client = RestClient.builder()
        .baseUrl("https://example.com")
        .build();
```

이 코드도 내부에서 요청 factory와 통신 구현을 사용한다. `.requestFactory(...)`를 생략하면 Spring이 클래스패스의 라이브러리와 실행 환경에 따라 선택한다. JDK 구현 외에 Apache·Jetty 등의 구현도 있으므로, 짧은 설정 코드만 보고 당시 어떤 클라이언트가 선택됐는지 확정하지 않는다. 정확한 선택은 사용한 Spring 버전과 런타임 의존성을 함께 확인한다.

### 직접 지정했을 때: 구현과 설정을 고정

```java
HttpClient transport = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(2))
        .followRedirects(HttpClient.Redirect.NEVER)
        .build();

var factory = new JdkClientHttpRequestFactory(transport);
factory.setReadTimeout(Duration.ofSeconds(5));

RestClient client = RestClient.builder()
        .baseUrl("https://example.com")
        .requestFactory(factory)
        .build();
```

위 예시는 JDK 클라이언트, 연결 타임아웃, 리다이렉트 정책, 읽기 타임아웃을 명시한다. factory·RestClient를 반드시 각각 Bean으로 등록할 필요는 없다. 생성한 HttpClient의 종료 책임은 구성 방식에 맞게 정한다. 앞의 Bean 예시처럼 등록하면 Spring의 생명주기 관리를 사용할 수 있다.

| 선택 | 이점 | 확인할 점 |
|---|---|---|
| 자동 선택 | 설정 코드가 짧고 기본 동작을 쉽게 사용 | 의존성에 따라 구현이 달라질 수 있고 통신 설정은 선택된 구현의 기본값에 의존 |
| 직접 지정 | 원하는 구현과 통신 설정을 명확하게 제어 | 설정·객체 생명주기를 관리할 코드가 늘어남 |

⚠️ factory를 생략했다고 타임아웃이 없다고 단정할 수는 없다. 다만 위의 짧은 코드에는 애플리케이션이 정한 제한이 없다. 또한 읽기 타임아웃을 지정한 것과 재시도·본문 처리까지 포함한 호출 전체 시간 제한은 동일하지 않으므로, 필요한 보장은 별도로 확인한다.

💡 **호출이 되는지만 확인할 때는 기본 선택으로 시작할 수 있지만, 지연·리다이렉트 등 동작을 보장해야 한다면 해당 설정을 명시하고 검증한다.** 길어진 설정 코드는 HTTP 인터페이스를 쓰기 위한 필수 의식이 아니라 통신 정책을 표현하는 수단이다.

## 옵션 비교

| 방식 | 사용할 상황 |
|---|---|
| `@Component` 계열 | 직접 작성한 클래스를 스캔으로 등록할 때 |
| `@Bean` | 외부 라이브러리 객체나 프록시를 설정 코드로 조립할 때 |
| `@Qualifier` | 같은 타입의 후보 중 특정 용도의 Bean을 선택할 때 |
| `@Primary` | 같은 타입의 후보 중 기본 선택을 지정할 때 |

## 함정 ⚠️

- 타입 이름에 `Client`가 포함되어도 서로 호환되는 타입이라는 뜻은 아니다. 인터페이스 구현·상속 관계가 기준이다.
- `@Qualifier`는 타입에 맞는 후보를 좁힌다. 이름이 같다는 이유로 다른 타입의 객체를 주입하지 않는다. Bean 이름은 qualifier의 대체 매칭 값으로 사용할 수 있다.
- 클래스에 필드만 선언하거나 직접 `new`로 생성했다고 자동 주입되지 않는다. Spring이 처리하는 생성자·주입 지점이어야 한다.
- `@Bean` 메서드가 사용하는 모든 지역 객체가 Bean으로 등록되지는 않는다.
- 반환 타입을 필요 이상으로 넓게 선언하면 생성 전 타입 예측이 제한된다. 소비자가 주입받을 계약을 표현하는 타입을 선언한다.
- `@Configuration(proxyBeanMethods = false)`에서는 다른 Bean이 필요할 때 설정 메서드를 직접 호출하지 않고 매개변수로 주입받으면 명확하다.

Bean 초기화와 프록시 교체 과정은 [빈 후처리기](../design/bean-post-processor.md), 동적 구현체의 원리는 [동적 프록시](../design/dynamic-proxy.md)를 참고한다.

## 판단 기준 💡

HTTP 클라이언트 설정에서 무엇이 주입되는지 헷갈리면 **매개변수는 재료, 반환 타입은 제공할 계약, 반환 객체는 등록 대상**으로 나눠 읽는다. 통신에 사용하는 객체와 업무 코드가 호출하는 인터페이스를 같은 타입으로 취급하지 않는다.

## 참고

- [Spring: Using the @Bean Annotation](https://docs.spring.io/spring-framework/reference/core/beans/java/bean-annotation.html)
- [Spring: Qualifiers](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired-qualifiers.html)
- [Spring: HTTP Service Clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-http-interface)
- [Spring: RestClient.Builder.requestFactory](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestClient.Builder.html#requestFactory(org.springframework.http.client.ClientHttpRequestFactory))
- [Spring: JdkClientHttpRequestFactory](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/client/JdkClientHttpRequestFactory.html)
- 통신 구현체(JDK/Apache/Jetty/Reactor/Simple)별 동작 차이와 Boot 자동 감지 순서는 [HTTP 클라이언트 구현체 비교](./http-client-transports.md)

학습 날짜: 2026-09-19. 계기: HTTP 인터페이스 Bean을 등록하면 내부에서 사용한 JDK HttpClient 타입의 주입 지점에도 그 프록시가 들어가는지에 대한 질문. 공식 문서와 코드 구조로 확인했으며, 예시를 별도로 실행하지는 않았다.

보강 날짜: 2026-09-19. 사용자 승인으로 RestClient·RequestFactory·HttpClient의 역할과 자동 선택·명시 설정의 차이를 추가했다. Spring Framework 7.0.9 공식 문서를 확인했으며 과거 프로젝트의 실제 통신 구현은 확인하지 않았다.

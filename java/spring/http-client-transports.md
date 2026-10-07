# HTTP 클라이언트 구현체 비교 — RestClient 아래에서 실제로 통신하는 라이브러리

> **한 줄 요약**: `RestClient`는 요청을 *만들고 응답을 해석*할 뿐이고, 실제 소켓 통신은 `ClientHttpRequestFactory`가 감싼 라이브러리(JDK `HttpClient` / Apache HttpComponents / Jetty / Reactor Netty / `HttpURLConnection`)가 한다. API 사용법은 어느 구현체든 같지만 **자동 재시도, 읽기 타임아웃의 의미, 오류 본문 drain, HTTP/2, 커넥션 풀 제어**가 다르다. JDK 클라이언트는 의존성 없이 충분히 좋지만 "멱등 요청은 알아서 1회 더 보낸다"와 "읽기 타임아웃은 요청 시작부터의 총 시간"을 모르면 테스트가 이유 모르게 깨진다.

## 언제 쓰나

- `RestClient` / `RestTemplate` / HTTP 인터페이스(`@GetExchange`)로 외부 API를 부를 때 통신 라이브러리를 고르거나, 고른 이유를 설명해야 할 때
- 타임아웃·재시도·리다이렉트 동작이 기대와 다를 때. 특히 fake HTTP 서버로 호출 횟수를 검증하는 테스트에서 숫자가 2배로 나오거나, 타임아웃 테스트가 예상보다 오래 걸릴 때

## 계층 먼저 — "클라이언트"가 세 개라 헷갈린다

```text
HTTP 인터페이스 프록시(@GetExchange 메서드)   ← 어노테이션을 요청 정보로 변환
  → RestClient                                ← URL·헤더·본문 구성, 상태 코드 처리, 응답 변환
    → ClientHttpRequestFactory                ← Spring 요청 객체를 만드는 공통 규격
      → 통신 라이브러리                        ← 이 노트의 비교 대상
```

각 층의 역할과 `@Bean` 조립은 [@Bean 등록과 타입 기반 주입](./bean-registration-and-injection.md)에 있다. 이 노트는 맨 아래 층만 다룬다.

## 사용 예시

### 1) Spring Framework만으로 직접 조립

```java
HttpClient transport = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(2))
        .followRedirects(HttpClient.Redirect.NEVER)
        .build();

var factory = new JdkClientHttpRequestFactory(transport);
factory.setReadTimeout(Duration.ofSeconds(5));

RestClient client = RestClient.builder()
        .baseUrl("https://api.example.com")
        .requestFactory(factory)
        .build();
```

### 2) Spring Boot 4 빌더 — 같은 결과를 라이브러리 무관 설정으로

```java
HttpClientSettings settings = HttpClientSettings.defaults()
        .withConnectTimeout(Duration.ofSeconds(2))
        .withReadTimeout(Duration.ofSeconds(5))
        .withRedirects(HttpRedirects.DONT_FOLLOW);

ClientHttpRequestFactory factory = ClientHttpRequestFactoryBuilder.jdk().build(settings);
```

`.jdk()` 자리에 `.httpComponents()`, `.jetty()`, `.reactor()`, `.simple()`, `.detect()`가 온다. 구현체를 바꿔도 타임아웃·리다이렉트 설정 코드는 그대로다. (Boot 4.1.1 `spring-boot-http-client` jar에서 시그니처 확인)

### 3) 전역 설정 — 자동 구성된 `RestClient.Builder` 전부에 적용

```yaml
spring:
  http:
    clients:
      connect-timeout: 2s
      read-timeout: 5s
      redirects: dont-follow
      imperative:
        factory: jdk      # 생략하면 클래스패스 보고 자동 감지
```

클라이언트마다 다른 타임아웃이 필요하면(외부 공공 API 5초, LLM 서버 120초처럼) 전역 대신 2)로 각각 만든다.

## 옵션 비교 — 구현체 5종

| 구현체 (RequestFactory) | 의존성 | HTTP/2 | 커넥션 풀 제어 | 특징 |
|---|---|---|---|---|
| **JDK `java.net.http.HttpClient`** (`JdkClientHttpRequestFactory`) | 없음 (JDK 11+) | 기본 HTTP/2 협상, 1.1 폴백 | 내부 풀, 크기 조절 API 없음 | 멱등 요청 자동 1회 재시도. JDK 21+ `AutoCloseable`. 가상 스레드와 잘 맞음 |
| **Apache HttpComponents 5** (`HttpComponentsClientHttpRequestFactory`) | `httpclient5` 추가 | **HTTP/1.1만** — 이 factory는 classic API를 쓰고, HC5의 HTTP/2는 async API 전용 | `PoolingHttpClientConnectionManager`로 route별 최대 연결 등 세밀 제어 | 재시도 전략 교체 가능(`HttpRequestRetryStrategy`). `setReadTimeout`은 `RequestConfig.responseTimeout`으로 들어가 소켓 비활성 시간 의미 |
| **Jetty** (`JettyClientHttpRequestFactory`) | `jetty-client` 추가 | 가능 | 있음 | Jetty 서버 쓰는 팀이 통일 목적으로 선택 |
| **Reactor Netty** (`ReactorClientHttpRequestFactory`) | WebFlux 쓰면 이미 있음 | 가능 | 있음 | `WebClient`와 같은 엔진. MVC 프로젝트에서 굳이 넣을 이유는 적음 |
| **Simple `HttpURLConnection`** (`SimpleClientHttpRequestFactory`) | 없음 (레거시) | 1.1 | 없음, 시스템 프로퍼티로 전역 영향 | 오류 상태(401 등) 응답 접근 시 예외가 날 수 있다고 공식 문서가 경고. `RestTemplate`의 기본값 |

**자동 감지 순서**
- Spring Boot 4: Apache → Jetty → Reactor Netty → JDK → Simple. 여러 개 있으면 앞쪽 우선.
- Spring Framework 단독(`RestClient.create()`): 소스(`DefaultRestClientBuilder.initRequestFactory`) 기준 Apache → Jetty → Reactor Netty → JDK → Simple로 Boot와 같은 순서. 공식 문서 본문은 Reactor Netty를 빼고 "Apache/Jetty → JDK → Simple"로만 적는다.
- `RestTemplate` 기본은 Simple. 공식 문서가 "RestTemplate → RestClient 이전 시 HTTP 수준의 미묘한 동작 차이" 원인으로 이 기본값 차이를 꼽는다.

즉 MVC 프로젝트에 별도 HTTP 라이브러리를 안 넣었다면 자동 감지 결과도 JDK다. 그래도 명시하는 이유는 리다이렉트·타임아웃을 코드로 고정하고, 나중에 누가 Apache를 의존성에 넣어도 동작이 안 바뀌게 하기 위해서다.

### "Apache가 실시간에 좋다"는 말은 성립하지 않는다 — "실시간"을 먼저 쪼갠다

"풀 제어·idle 타임아웃이 강점"을 듣고 "그럼 실시간에 좋은 거네"로 넘어가기 쉬운데, **실시간**이 무엇을 뜻하느냐에 따라 답이 갈린다. 기법 자체는 [실시간 통신 기법 비교](../../infra/network/realtime-communication.md)에 있고, 여기서는 *클라이언트 쪽* 선택만 본다.

| "실시간"의 뜻 | 필요한 것 | Apache HttpClient | JDK HttpClient |
|---|---|---|---|
| ① 서버 푸시 (WebSocket) | WebSocket 프로토콜 클라이언트 | **없음** (HTTP 전용 라이브러리) | `java.net.http.WebSocket` 내장 (JDK 11+) |
| ② 길게 열린 스트림 (SSE, LLM 토큰 스트리밍) | 끊기지 않는 한 타임아웃 안 나는 idle 기준 read timeout | 소켓 타임아웃이 idle 기준이라 **적합** | Spring 조합에서는 read timeout이 총시간 기준이라 스트림 길이만큼 크게 잡거나 빼야 함 |
| ③ 저지연·고동시성 요청-응답 | 풀 한도 제어, 커넥션 재사용, 멀티플렉싱 | 풀 세밀 제어(route별 최대 연결) | HTTP/2 멀티플렉싱으로 연결 수 자체가 덜 중요. 풀 크기 조절 API는 없음 |

- Apache가 실제로 이기는 건 **②** 하나다. 그것도 "실시간이라서"가 아니라 "타임아웃 의미가 idle 기준이라서"다.
- **①**은 오히려 JDK가 가진 걸 Apache가 못 한다. WebSocket은 HttpClient 종류와 별개 문제.
- **③**은 둘 다 되고, 진짜 고동시성이면 Reactor Netty나 JDK + 가상 스레드 조합을 같이 놓고 본다.
- 서버 쪽 SSE 구현(`SseEmitter`)은 클라이언트 라이브러리와 무관하다. 여기서 말하는 건 우리가 *부르는* 쪽이다.

**비유로 기억하기**: HTTP 요청-응답은 *택배 주문*(보내고 답 받으면 끝), WebSocket은 *전화 통화*(양쪽이 아무 때나 말함), 스트리밍은 *라디오*(한 번 틀면 계속 흘러옴), 고동시성은 *창구가 많은 은행*. Apache와 JDK HttpClient는 둘 다 택배 주문 도구다. 전화기(①)는 JDK 상자엔 들어 있고 Apache 상자엔 없다. 라디오(②)는 둘 다 되지만 "5초 타임아웃"의 뜻이 달라서 JDK+Spring은 "틀고 5초 지나면 무조건 끔", Apache는 "5초 동안 소리가 안 나면 끔"이다. 창구(③)는 Apache가 "최대 20개"처럼 직접 정하고 JDK는 알아서 정한다. ChatGPT처럼 글자가 한 글자씩 오는 응답을 우리 서버가 *받아야* 한다면 ②가 문제가 되고, 그때만 Apache의 idle 타임아웃이 실질적 이점이다.

## ⚠️ 함정/메커니즘 — JDK 클라이언트 기준 (JDK 25 · spring-web 7.0.9 소스 확인)

**1. 멱등 요청은 JDK가 알아서 1회 더 보낸다.**
`jdk.internal.net.http.MultiExchange`의 `retryOnFailure`: 응답 헤더를 받기 전에 연결이 끊기면(`ConnectionExpiredException`) 또는 연결 자체가 실패하면(`ConnectException`) 요청이 멱등(GET·HEAD 등)일 때 `retriedOnce` 플래그로 **정확히 한 번** 같은 요청을 다시 보낸다. 서버가 요청을 처리하지 않았다고 알려준 경우(HTTP/2 `isUnprocessedByPeer`)도 재시도한다.
- POST까지 재시도하려면 `-Djdk.httpclient.enableAllMethodRetry=true`
- 연결 실패 재시도를 끄려면 `-Djdk.httpclient.disableRetryConnect=true`
- 전체 시도 상한은 `jdk.httpclient.redirects.retrylimit`(기본 5, 리다이렉트도 같이 센다)
- 응답 헤더 대기 타임아웃(`HttpTimeoutException`)은 재시도 대상이 **아니다**

→ 이 위에 "연결/읽기 오류는 최대 1회 재시도" 루프를 직접 얹으면, 연결 끊김 상황의 실제 요청 수는 `2 × 2 = 4`회다. fake 서버로 "2회 호출"을 검증하면 4가 나와서 깨진다. 선택지는 (a) JDK 재시도를 "허용된 1회"로 치고 자체 루프에서 연결 오류를 빼기, (b) 4회를 문서·테스트에 명시하기. GET이라 중복 부작용이 없으면 (b)가 단순하다.

**2. 읽기 타임아웃이 "idle 시간"이 아니라 "요청 시작부터의 총 시간"이다.**
`JdkClientHttpRequestFactory.setReadTimeout(d)`는 두 군데에 걸린다.
- `HttpRequest.Builder.timeout(d)`: 응답 **헤더**가 올 때까지의 제한 (JDK 자체 기능)
- `JdkClientHttpRequest.TimeoutHandler`: 요청 시작 시점부터 `CompletableFuture.completeOnTimeout(d)`로 카운트하고, 시간이 되면 **본문 스트림을 강제로 닫는다**

그래서 5초 read timeout이면 "느리지만 꾸준히 6초 걸리는 다운로드"도 실패한다. Apache는 같은 설정이 응답 타임아웃(classic I/O에서는 소켓 비활성 시간)으로 들어가 같은 6초 다운로드가 성공한다. 작은 XML/JSON 응답이면 차이가 없고, 대용량 스트리밍이면 구현체 선택이 곧 타임아웃 의미 선택이다.

**3. 오류 응답 본문은 "안 읽어도" close 때 drain된다.**
`JdkClientHttpResponse.close()`가 `StreamUtils.drain(body)`를 먼저 호출한다. 연결을 재사용하려면 본문을 소진해야 하기 때문이다. `RestClient`의 상태 핸들러가 본문을 읽지 않고 예외를 던져도, try-with-resources가 닫는 순간 본문을 끝까지 읽는다.
- 서버가 5xx 본문을 chunked로 천천히 흘리면 read timeout까지 스레드가 잡힌다. 자체 재시도까지 있으면 그 2배.
- 크기 제한을 `HttpMessageConverter`에서 걸었다면 그건 **성공 경로**에만 적용된다. 오류 경로의 drain은 시간으로만 제한된다.
- "오류 본문을 읽지 않는다"는 테스트는 이 스택에서 성립하지 않는다. 검증할 것은 "오류 본문을 **노출**하지 않는다"와 "read timeout 안에 끝난다"다.

**4. 리다이렉트 기본값은 `NEVER`, 프로토콜 기본값은 HTTP/2.** (JDK `HttpClient` Javadoc)
기본이 안전한 쪽이지만 그래도 명시하는 편이 낫다. 쿼리 스트링에 인증 키를 실어 보내는 공공 API라면 리다이렉트를 따라가는 순간 키가 다른 호스트로 간다. Boot 설정으로는 `HttpRedirects.DONT_FOLLOW` / `redirects: dont-follow`.

**5. 예외 메시지에 전체 URI가 들어간다.**
`RestClient`가 I/O 오류를 `ResourceAccessException`으로 감싸면서 메시지에 메서드와 **쿼리 포함 전체 URI**를 넣는다. 키가 쿼리에 있으면 그 예외를 그대로 로그에 찍는 순간 키가 남는다. 어댑터 경계에서 cause를 끊고 도메인 예외로 바꾸거나, 로그 전에 URI를 마스킹한다. 테스트에서는 `hasNoCause()`로 지킨다.

## 💡 판단 기준

- **MVC 프로젝트에서 외부 API 몇 개 부르는 정도면 JDK 클라이언트로 시작한다.** 의존성이 늘지 않고, Boot 자동 감지 결과와도 같다. 커넥션 풀 크기를 만져야 하거나 바이트 간 idle 타임아웃이 필요해질 때 Apache로 간다. 그때도 `ClientHttpRequestFactoryBuilder`로 만들었다면 `.jdk()`를 `.httpComponents()`로 바꾸는 것으로 끝난다.
- **"Apache가 기능이 더 많다"와 "Apache가 더 낫다"는 다르다.** 기능 목록만 보면 Apache가 압도하지만, 그 기능이 필요한 조건이 실제로 있는지 먼저 센다.

  | Apache가 이기는 조건 | 필요한 상황 | 저빈도·소용량 외부 API 어댑터에서는 |
  |---|---|---|
  | 풀 세밀 제어(route별 최대 연결, 큐잉) | 같은 호스트에 동시 수십 요청 | 요청이 드물고 rate limit도 낮아 동시성 거의 0 |
  | idle 기준 read timeout | 대용량·스트리밍 응답 | 응답 크기를 제한하고 실제로도 수 KB |
  | 재시도 전략 교체 | 정교한 백오프·상태코드별 정책 | 자체 루프로 이미 처리. Apache도 기본 재시도가 켜져 있고(`DefaultHttpRequestRetryStrategy` 기본 생성자 = 최대 1회·1초 간격, 429·503 응답도 재시도) "429는 재시도하지 않는다" 같은 정책이 있으면 어차피 꺼야 함 |
  | 프록시 인증·쿠키·고급 TLS | 사내 프록시, mTLS | 없음 |

  비용 쪽은 의존성 2개(`httpclient5`·`httpcore5`)와 CVE 추적, 검증해야 할 표면적 증가, 그리고 **라이브러리 자체 재시도라는 함정은 그대로**라는 점. 결국 "JDK의 단점(자동 1회 재시도, 총시간 타임아웃, close 시 drain)이 이 워크로드에서 실제 문제인가"를 물으면 셋 다 문서화·read timeout으로 관리 가능해서 JDK가 남는다.
- **선택 이유를 결정 기록에 한 줄 남긴다.** "왜 Apache가 아니라 JDK인가"는 나중에 누군가 반드시 다시 묻는다. "의존성 없음 + Boot 자동 감지 결과와 동일 + 풀·스트리밍 요구 없음"이면 충분하다.
- **호출 횟수·시간을 검증하는 테스트를 쓸 때는 "내 코드의 재시도"와 "라이브러리의 재시도"를 먼저 분리해서 센다.** 이 케이스에서는 fake 서버가 4회를 찍고 나서야 JDK 내부 재시도를 찾았다. 숫자가 정확히 2배면 아래 층을 의심한다.
- **타임아웃 숫자보다 타임아웃의 의미를 먼저 확인한다.** "read timeout 5초"가 헤더까지인지, 총 시간인지, 침묵 시간인지는 구현체마다 다르다. 스트리밍 응답을 다루기 전에 반드시 확인한다.
- **인증 키가 쿼리에 실리는 API는 리다이렉트 차단 + 예외 cause 차단을 세트로 건다.** 둘 중 하나만 하면 나머지 경로로 샌다.

## 참고

- [Spring Framework: REST Clients — Client Request Factories](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html) — 5종 목록, 자동 선택 순서, `SimpleClientHttpRequestFactory` 오류 응답 경고, RestTemplate→RestClient 기본값 차이
- [Spring Boot 4.1: Calling REST Services](https://docs.spring.io/spring-boot/reference/io/rest-client.html) — 자동 감지 순서, `spring.http.clients.*`, `HttpClientSettings`, `ClientHttpRequestFactoryBuilder`
- [JDK `java.net.http.HttpClient` Javadoc](https://docs.oracle.com/en/java/javase/25/docs/api/java.net.http/java/net/http/HttpClient.html) — 리다이렉트 기본 NEVER, 버전 기본 HTTP_2
- JDK 25 소스 `jdk.internal.net.http.MultiExchange` — `retryOnFailure`, `retriedOnce`, `jdk.httpclient.*` 프로퍼티
- spring-web 7.0.9 소스 `JdkClientHttpRequest`(`TimeoutHandler`), `JdkClientHttpResponse.close()`, `DefaultRestClientBuilder.initRequestFactory`(자동 선택 순서), `HttpComponentsClientHttpRequestFactory`(classic `HttpClient`, `setReadTimeout` → `responseTimeout`)
- httpclient5 소스 `DefaultHttpRequestRetryStrategy`(기본 1회·1초, 429·503) · [HttpClient 5.x 마이그레이션 가이드](https://hc.apache.org/httpcomponents-client-5.5.x/migration-guide/) (classic API는 HTTP/1.1 전용)
- 관련 노트: [@Bean 등록과 타입 기반 주입](./bean-registration-and-injection.md) · [포트와 어댑터](../design/ports-and-adapters.md) · [가상 스레드](../concurrency/virtual-threads.md)

학습 날짜: 2026-09-19. 계기: 외부 공공 API 어댑터가 `JdkClientHttpRequestFactory`를 명시하는 코드를 보고 "다른 구현체와 뭐가 다른가"를 물음. 같은 날 fake 서버 테스트에서 호출 횟수 4회, 오류 본문 대기 타임아웃 실패가 나왔고 원인이 각각 JDK 자동 재시도와 close 시 drain이었다. JDK·spring-web 소스와 Spring/Boot 공식 문서로 확인했다. 2026-10-02: Apache HttpClient 5의 HTTP/2 지원 범위·기본 재시도·read timeout 의미를 소스로 확인해 ❓ 표시를 정리.

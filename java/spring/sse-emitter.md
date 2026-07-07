# Spring SseEmitter 서버 구현 - 서블릿 async / send·complete·콜백 / 두 구현 패턴 / 스레드 모델

> 한 줄 요약: 컨트롤러가 `SseEmitter`를 반환하면 Spring MVC가 그 요청을 **서블릿 async로 전환** — 서블릿 스레드는 즉시 반납되고 HTTP 응답만 열린 채 남는다. 이후 **아무 스레드에서든** `emitter.send()`로 이벤트를 밀어넣고 `complete()`로 닫는다. SSE 프로토콜/와이어 포맷은 [sse.md](../../infra/network/sse.md), 중계(relay) 개념은 [realtime-communication.md](../../infra/network/realtime-communication.md) 참고 — **이 문서는 Spring 서버 구현(API·구현 패턴·스레드 모델)**을 판다.

## 언제 쓰나
- **Spring MVC(서블릿 스택)**에서 SSE 서버를 만들 때. (WebFlux라면 `Flux<ServerSentEvent<T>>` 반환이 더 자연스럽다 — emitter는 서블릿 세계의 도구)
- 두 갈래 상황:
  - **(A) 서버 내부 이벤트를 구독자에게 푸시** — 알림, 진행률. → 패턴 A
  - **(B) 외부 스트림을 받아 프론트로 중계(relay)** — LLM 토큰 스트리밍. → 패턴 B

## 핵심 객체: `SseEmitter` (`ResponseBodyEmitter`의 SSE 특화)

```java
@PostMapping(value = "/chats/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE) // ★ produces 필수
public SseEmitter stream() {
    SseEmitter emitter = new SseEmitter(10 * 60 * 1000L);   // 타임아웃 ms

    // 아무 스레드에서나 emitter.send(...) 가능 — 여기선 예시로 즉시
    emitter.send(SseEmitter.event().name("open").data("connected"));

    // cleanup 3종 (모두 "컨테이너 스레드"에서 호출됨)
    emitter.onCompletion(() -> cleanup());          // 어떤 이유로든 끝났을 때
    emitter.onTimeout(() -> emitter.complete());    // 타임아웃
    emitter.onError(e -> cleanup());                // 처리 중 에러

    return emitter;   // ★ 리턴 즉시 서블릿 스레드 반납. 응답은 안 닫힘(async)
}
```

- **`produces = MediaType.TEXT_EVENT_STREAM_VALUE`** 없으면 SSE로 협상 안 됨.
- **타임아웃 기본값**: `new SseEmitter()`(인자 없음)면 미설정 → `spring.mvc.async.request-timeout`(MVC 설정)을 따르고, 그것도 없으면 **WAS(서버) 기본값**. 무한이 아니다. LLM처럼 오래 걸리면 명시적으로 크게 준다.
- **반환 즉시 async 전환**: emitter를 리턴해도 응답이 끝나지 않는다. Spring이 커넥션을 열어둔 채 서블릿 스레드만 반납하고, 이후 `send()`가 그 열린 응답에 쓴다. `complete()`를 불러야 닫힌다.

### 메서드
| 메서드 | 의미 |
|---|---|
| `send(obj)` / `send(SseEmitter.event()...)` | 이벤트 1개 전송 |
| `complete()` | 정상 종료(커넥션 닫음) |
| `completeWithError(e)` | 에러로 종료 — **진행 중 스트림엔 부적합**(아래 함정 2) |

⚠️ `send()`가 `IOException`을 던지면(클라 끊김 등) **`completeWithError()`를 부를 필요 없다.** 서블릿 컨테이너가 알림→dispatch로 예외 리졸버를 태워 알아서 정리한다. (javadoc 명시)

### `event()` 빌더 = 와이어 필드 조립기 (`SseEventBuilder`)
| 빌더 메서드 | 생성되는 SSE 필드 |
|---|---|
| `.data(obj)` / `.data(obj, mediaType)` | `data:` |
| `.name(str)` | `event:` (이벤트 이름) |
| `.id(str)` | `id:` (재연결 시 `Last-Event-ID`로 돌아옴) |
| `.comment(str)` | `:` 로 시작하는 주석 줄 → **heartbeat 용도** |
| `.reconnectTime(ms)` | `retry:` |

> 필드 의미·재연결 동작은 [sse.md](../../infra/network/sse.md)의 와이어 포맷 표 참고. 여기선 "Java에서 어떻게 만드는가"만.

### 콜백 3종 (전부 컨테이너 스레드에서 호출)
- **`onCompletion`** — 정상·타임아웃·에러·네트워크 끊김 등 **어떤 이유로든 완료**될 때. "이 emitter는 이제 못 쓴다"를 감지하는 **정리(cleanup)의 최종 지점**.
- **`onTimeout`** — 타임아웃 시. 보통 `complete()` 호출.
- **`onError`** — 처리 중 에러.
- (Spring **6.2+**부터 각 이벤트에 콜백을 여러 개 등록 가능)

---

## 두 구현 패턴 (핵심)

emitter 자체는 똑같지만, **"이벤트를 누가 만드느냐"**에 따라 구조가 완전히 갈린다.

### 패턴 A — 세션 푸시형: emitter를 **저장소에 보관**

서버 내부에서 이벤트(알림)가 발생하면, 미리 구독해둔 클라의 emitter를 꺼내 push.

```java
// 저장소 = 그냥 동시성 맵
private final ConcurrentHashMap<String, SseEmitter> emitters = new ConcurrentHashMap<>();

// 1) 구독: GET /subscribe → emitter 만들어 저장
public SseEmitter subscribe(String clientId) {
    SseEmitter emitter = new SseEmitter(30 * 60 * 1000L);
    emitters.put(clientId, emitter);
    emitter.onCompletion(() -> emitters.remove(clientId, emitter)); // 끝나면 스스로 제거
    emitter.onTimeout(() -> emitter.complete());
    return emitter;
}

// 2) 발행: 내부 이벤트 발생 시 저장소에서 꺼내 send
public void push(String clientId, Object data) {
    Optional.ofNullable(emitters.get(clientId)).ifPresent(em -> {
        try { em.send(SseEmitter.event().data(data)); }
        catch (IOException e) { em.complete(); } // 끊긴 연결 정리
    });
}
```

- **핵심: emitter를 오래(30분 등) 보관**하고 아무 때나 꺼내 쓴다.
- **스케일아웃 문제**: 이벤트는 아무 인스턴스에서나 발생하는데 emitter는 **구독을 받은 그 인스턴스의 메모리에만** 있다. → **Redis Pub/Sub**으로 이벤트를 전 인스턴스에 브로드캐스트하고, 각 인스턴스가 자기 맵에 그 clientId가 있으면 send. (인스턴스 간 emitter 공유는 불가 — 소켓이라 직렬화 안 됨)
- heartbeat로 죽은 연결을 send 실패로 감지·제거.

### 패턴 B — 릴레이형: 요청 1건 = 외부 스트림 구독 1건

외부 SSE를 `WebClient`로 구독(`bodyToFlux` → `Flux<String>`)해서 라인마다 프론트로 흘려보내고, 특정 이벤트(예: `complete`)에서 DB 저장.

```java
// 공통 배관을 추상 클래스로: handle()=final, 라인 처리만 abstract (템플릿 메서드)
public abstract class AbstractSseStreamHandler<C extends SseStreamContext> {
    public final SseEmitter handle(Flux<String> upstream, C context) {
        SseEmitter emitter = new SseEmitter(10 * 60 * 1000L);

        upstream.subscribe(                                   // ★ 구독=시작
            line  -> handleLine(line, emitter, context),     // 라인마다 (하위 클래스)
            error -> completeQuietly(emitter),               // 에러 시 조용히 종료
            ()    -> completeQuietly(emitter));              // 정상 종료

        Disposable heartbeat = Flux.interval(Duration.ofSeconds(15))
            .subscribe(i -> send(emitter, comment("heartbeat")));

        emitter.onCompletion(() -> heartbeat.dispose());     // 정리
        emitter.onTimeout(()    -> { heartbeat.dispose(); completeQuietly(emitter); });
        emitter.onError(e       -> heartbeat.dispose());
        return emitter;
    }
    protected abstract void handleLine(String line, SseEmitter emitter, C context);
}
```

- **emitter를 저장하지 않는다** — 요청 스코프, 1:1, 끝나면 사라짐. 그래서 저장소·Pub/Sub이 필요 없다.
- **`handle()`=`final`(구독+heartbeat+타임아웃+정리 공통), `handleLine()`=`abstract`** — 새 스트리밍 연동은 `handleLine`만 새로 구현. (다른 AI 붙일 때 재사용)
- **싱글톤 핸들러 + 요청별 상태는 `Context` 객체로 전달** — 핸들러는 `@Component`(싱글톤)라 요청별 값(어느 질문에 대한 응답인지)을 **필드로 두면 동시 요청끼리 덮어쓴다.** 반드시 파라미터로.
- relay를 왜 하나(인증 보호·중간 저장), 커넥션 부담, **retry 금지**(비멱등 POST 중복) → [realtime-communication.md](../../infra/network/realtime-communication.md).

### A vs B 비교
| | 패턴 A (세션 푸시) | 패턴 B (릴레이) |
|---|---|---|
| 이벤트 출처 | **서버 내부**(알림 발생) | **외부 스트림**(LLM API) |
| emitter 관리 | 저장소에 **오래 보관** | 요청 스코프 1:1, 저장 안 함 |
| 데이터 원천 | 우리가 `send` 호출 | 외부 `Flux` 구독해서 흘려보냄 |
| 멀티 인스턴스 | **Redis Pub/Sub 필요** | 불필요(요청받은 서버가 끝까지) |
| 대표 사례 | 실시간 알림 | LLM 채팅 토큰 스트리밍 |

---

## 스레드 모델 (relay에서 특히 안 헷갈려야 함)

관여하는 스레드가 **3종**이고, 여기서 실수하면 성능이 무너진다.

| 스레드 | 하는 일 | 주의 |
|---|---|---|
| **서블릿(톰캣) 스레드** | 컨트롤러~emitter 리턴까지. 밑작업(조회·저장) | 리턴하면 반납됨 |
| **이벤트 소스 스레드** | A: 이벤트 발행 스레드 / B: **Reactor Netty 이벤트 루프**(외부 데이터 도착 시 `handleLine` 실행) | **B는 절대 블로킹 금지** |
| **블로킹 작업 스레드** | DB 쓰기 등 블로킹 I/O | `boundedElastic`로 offload |

⚠️ **이벤트 루프에서 블로킹하지 마라.** 패턴 B에서 `handleLine`은 Netty 이벤트 루프에서 돈다. 여기서 블로킹 DB I/O를 하면 그 스레드가 붙잡혀 **다른 사용자의 스트림까지 멈춘다.** → 엔티티 조립(빠름)은 콜백에서, **블로킹 DB 쓰기만** 별도 풀로:

```java
Mono.fromRunnable(() -> persistAnswer(question, answer))  // 블로킹 DB 쓰기
    .subscribeOn(Schedulers.boundedElastic())             // ★ 전용 풀로 분리
    .subscribe();
```

이 "이벤트 루프에선 절대 블로킹 안 한다"가 리액티브 코드의 제1원칙. ([boundedElastic] = 블로킹 작업용으로 준비된, 필요 시 늘어나는 스레드 풀)

---

## ⚠️ 함정 모음

1. **`send()` 스레드 안전성 — "안전하지 않다"는 옛말, 정밀하게 알자.**
   `ResponseBodyEmitter`는 내부에 **`writeLock`(쓰기 가드)**을 두어 쓰기를 직렬화한다([SPR-13224] 이후 동기화 도입). → 여러 스레드가 각자 `send()`해도 **개별 이벤트는 섞이지 않는다.** 단:
   - (a) **여러 `send()`를 "하나의 논리 단위"로 원자 전송**하려면 emitter에 직접 `synchronized` 필요(개별 send만 보장).
   - (b) **오래된 Spring 버전**은 이 보장이 없다 → 버전 의존적.
   - (c) `onError` 중 `completeWithError`와 동시 `send` 사이 **ABBA 데드락** 리포트 존재([#33421]).
   → 실무 결론: 패턴 B의 **heartbeat 스레드 + relay 스레드가 같은 emitter에 동시 `send`**하는 구조는 최신 Spring에선 안전하지만, "버전에 기대는 부분"임을 인지하고 멀티스레드 send는 의도적으로.

2. **진행 중 스트림에 `completeWithError`로 에러를 렌더링하면 스트림이 깨진다.**
   이미 `200 OK`(+ 일부 데이터)가 나간 뒤라 **HTTP 상태코드를 못 바꾼다.** 에러도 `event: error` **이벤트로** 보내거나, "**complete 이벤트 없이 종료 = 실패**"라는 계약을 프론트와 맺어야 한다. (relay 코드가 upstream 에러 시 조용히 `complete`하는 이유가 이것)

3. **cleanup 3종은 필수.** `onCompletion`/`onTimeout`/`onError`에서 heartbeat 타이머(`Flux.interval` 구독), 외부 구독 `Disposable` 등을 안 끊으면 **연결 끊긴 뒤에도 타이머·구독이 살아 누수.** `onCompletion`이 "어떤 이유로든 완료"라 최종 정리 지점.

4. **`send` `IOException` = 클라 끊김 신호.** `completeWithError` 불필요(함정 위). relay는 오히려 **클라가 끊겨도 외부 스트림을 끝까지 소비해 저장을 보장**하려고, relay용 `send` 실패를 조용히 무시하기도 한다(설계 선택).

5. **싱글톤 핸들러/서비스에 요청 상태를 필드로 두지 마라.** 동시 요청이 서로 덮어쓴다. 요청별 값은 `Context` 파라미터로.

6. **타임아웃은 이중으로.** emitter 타임아웃(클라 쪽)만으로는 부족 — 외부 스트림이 `complete` 없이 무한 지속될 수 있으니 **upstream에도 총 지속시간 상한**(예: `takeUntilOther(Mono.delay(10분))`)을 둬 좀비 커넥션 방지.

7. (프로토콜 레벨 함정 — **프록시 버퍼링 / UTF-8 인코딩 깨짐 / heartbeat 필요 / 브라우저 커넥션 제한**은 [sse.md](../../infra/network/sse.md) ⚠️ 섹션 참고. 중복 생략)

---

## 💡 판단 기준
- **이벤트를 누가 만드나**가 패턴을 가른다. **서버 내부**(알림) → 패턴 A(저장소 + 멀티 인스턴스면 Pub/Sub). **외부 스트림 중계**(LLM) → 패턴 B(구독 + relay, 저장 안 함).
- **공유되는(싱글톤) 핸들러·서비스에 요청별 상태를 필드로 두는 순간 동시성 버그.** 상태는 반드시 파라미터(Context)로 넘긴다 — SSE뿐 아니라 모든 싱글톤 빈의 원칙.
- **relay에서 블로킹 작업(DB)은 이벤트 루프 밖(`boundedElastic`)으로.** "이벤트 루프에선 절대 블로킹 안 한다."
- 구체 케이스: LLM 채팅 relay에서 `complete` 이벤트 수신 시 답변을 DB 저장해야 했는데, 이 저장을 **Netty 이벤트 루프에서 바로** 하면 다른 사용자 스트림이 밀린다 → **엔티티 조립은 콜백 스레드, 블로킹 DB 쓰기만 `boundedElastic`으로** 분리했다. 또 heartbeat(15초)와 relay가 **같은 emitter에 동시 send**하지만, `ResponseBodyEmitter` 내부 `writeLock` 덕에 이벤트가 안 섞였다 — "send는 무조건 스레드 세이프 아님"이라고 외우고 있으면 왜 안 깨지는지 설명 못 한다.

## 참고
- Spring MVC async / SseEmitter 레퍼런스: https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html
- `ResponseBodyEmitter` javadoc (send/IOException/타임아웃/콜백): https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/ResponseBodyEmitter.html
- `SseEmitter.SseEventBuilder` javadoc (빌더 메서드↔와이어 필드): https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/SseEmitter.SseEventBuilder.html
- send 스레드 안전성 이슈(SPR-13224): https://github.com/spring-projects/spring-framework/issues/17815
- SSE 에러 처리 중 ABBA 데드락(#33421): https://github.com/spring-projects/spring-framework/issues/33421
- 관련 노트: [SSE 프로토콜/포맷](../../infra/network/sse.md) · [실시간 통신 비교·relay](../../infra/network/realtime-communication.md) · [JVM 동시성 도구(Lock)](../concurrency/jvm-concurrency-tools.md)

---
학습 날짜: 2026-07-08
계기: 회사 LLM 채팅 스트리밍 MR에서 `SseEmitter` 반환 컨트롤러 + `AbstractSseStreamHandler`(구독·heartbeat·정리 템플릿) + `boundedElastic` 저장 분리 코드를 뜯어보며, 프로토콜([sse.md])이 아니라 **"Spring 서버측 구현이 어떤 부품으로 조립되고, 왜 그 스레드에서 그 일을 하는지"**를 정리.

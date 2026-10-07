# SSE (Server-Sent Events) - text/event-stream 단방향 푸시

> 한 줄 요약: HTTP 응답을 닫지 않고 `text/event-stream` 형식의 텍스트를 계속 흘려보내는 **서버→클라이언트 단방향** 푸시. 브라우저 `EventSource`가 자동 재연결까지 해주지만, **EventSource는 GET 전용**이라 POST 스트리밍은 fetch로 직접 파싱해야 한다.

전체 지형(폴링/롱폴링/WebSocket과 비교)은 [realtime-communication.md](./realtime-communication.md) 참고. 이 문서는 SSE 자체를 판다.

## 언제 쓰나
- 서버에서 클라이언트로만 계속 밀어주면 되는 경우: 알림, 작업 진행률, 대시보드 갱신, **LLM 답변 토큰 스트리밍**
- 클라이언트가 보낼 건 최초 요청 한 번뿐이고, 이후는 받기만 할 때

## 프로토콜: 그냥 HTTP다

응답 헤더가 핵심:

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

서버는 이 응답의 body를 끝내지 않고 아래 형식의 텍스트를 계속 쓴다.

### 이벤트 포맷 (와이어 형식)

```
data: 첫 번째 메시지

data: {"type":"token","content":"안녕"}

event: complete
data: {"answer":"...전체 답변..."}
id: 42

: 이 줄은 주석(comment) — 클라이언트가 무시함. heartbeat 용도

retry: 5000
```

| 필드 | 의미 |
|---|---|
| `data:` | 실제 페이로드. 연속된 여러 `data:` 줄은 `\n`으로 합쳐져 한 이벤트가 됨 |
| `event:` | 이벤트 이름 (생략 시 `message`). EventSource에서 `addEventListener("이름")으로 구분 수신 |
| `id:` | 이벤트 ID. 재연결 시 브라우저가 `Last-Event-ID` 헤더로 보내줌 → 서버가 끊긴 지점부터 재개 가능 |
| `retry:` | 재연결 대기 ms 지정 |
| `:` 로 시작 | 주석. **연결 유지용 heartbeat**로 관용적으로 사용 |

- **빈 줄(`\n\n`)이 이벤트의 구분자.** 빈 줄이 와야 이벤트 하나가 dispatch된다.
- 데이터는 **UTF-8 텍스트만** 가능. 바이너리는 못 보냄 (base64로 우회하거나 WebSocket 사용).

## 클라이언트: EventSource (표준 GET 방식)

```javascript
const es = new EventSource("/api/notifications");   // GET 요청

es.onmessage = (e) => console.log(e.data);          // event: 없는 기본 메시지
es.addEventListener("complete", (e) => { ... });    // event: complete
es.onerror = () => { /* 네트워크 오류면 자동 재연결. readyState가 CLOSED면 재연결 안 함. 중단은 es.close() */ };
```

- **자동 재연결 내장**: 네트워크 오류로 연결이 끊기면 브라우저가 `retry` 간격으로 알아서 재접속 + `Last-Event-ID` 전송. WebSocket엔 없는 공짜 기능.
- ⚠️ **재연결은 네트워크 오류일 때만이다.** 응답 상태가 200이 아니거나 `Content-Type`이 `text/event-stream`이 아니면 스펙상 "fail the connection" → `readyState = CLOSED`, 다시 붙지 않는다. 세션 만료로 401이 나면 알림이 **조용히 멈추는** 전형적 원인이므로, `onerror`에서 `readyState`를 보고 재로그인·재구독을 직접 처리한다.
- 한계: **GET만 가능, 커스텀 헤더 불가**(Authorization 못 붙임 → 쿠키 인증이거나 URL 토큰 필요), body 없음.

## 클라이언트: POST 스트리밍 (LLM 채팅 패턴)

질문 본문을 보내야 하는 LLM 채팅은 POST가 필요한데 EventSource가 안 되므로 **fetch + ReadableStream으로 직접 파싱**:

```javascript
const res = await fetch("/api/chats/stream", {
    method: "POST",
    headers: { "Content-Type": "application/json", "Authorization": token },
    body: JSON.stringify({ content: "질문" }),
});
const reader = res.body.getReader();
const decoder = new TextDecoder();
while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    const chunk = decoder.decode(value, { stream: true });
    // "data: ..." 라인 파싱은 직접 (또는 @microsoft/fetch-event-source 라이브러리)
}
```

⚠️ 이 방식은 자동 재연결·Last-Event-ID도 직접 구현해야 한다. "SSE 형식으로 응답하는 POST"일 뿐, EventSource의 편의는 못 받는다.

## 서버 구현: Spring MVC의 SseEmitter

> 아래는 개념 스케치. **구현 심화**(서블릿 async 메커니즘 / `event()` 빌더·콜백 3종 / 두 구현 패턴[세션푸시·relay] / 스레드 모델·`send` 동기화)는 → [java/spring/sse-emitter.md](../../java/spring/sse-emitter.md)

```java
@PostMapping(value = "/chats/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamChat(@RequestBody ChatRequest request) {
    SseEmitter emitter = new SseEmitter(10 * 60 * 1000L);  // 타임아웃 ms

    someAsyncSource.subscribe(
        line -> emitter.send(SseEmitter.event().data(line)),   // data: line
        error -> emitter.complete(),
        () -> emitter.complete()
    );

    emitter.onCompletion(() -> /* 구독 해제 등 정리 */);
    emitter.onTimeout(() -> emitter.complete());
    return emitter;  // 리턴 즉시 서블릿 스레드 반납 (async 처리)
}
```

- `SseEmitter`를 반환하면 Spring이 요청을 **비동기 모드로 전환** — 서블릿 스레드는 바로 반납되고, 이후 `send()`는 아무 스레드에서나 호출 가능.
- WebFlux라면 `Flux<ServerSentEvent<T>>`를 반환하는 방식도 있음.

⚠️ **여러 스레드의 `send()`** (데이터 스레드 + heartbeat 스레드): 현재 Spring의 `ResponseBodyEmitter`는 내부 `writeLock`으로 개별 send를 직렬화하므로 이벤트가 섞이지 않는다. 여러 send를 한 덩어리로 묶어야 할 때만 외부 동기화가 필요하다 → [SseEmitter 구현 §스레드 모델](../../java/spring/sse-emitter.md).

⚠️ 정리(cleanup) 3종 세트를 꼭 등록: `onCompletion` / `onTimeout` / `onError`. 안 하면 클라이언트가 창을 닫아도 서버측 구독·타이머가 살아서 누수.

## 서버 구현: WebClient로 SSE 구독 (소비자 쪽)

```java
Flux<String> stream = webClient.post()
        .uri(url)
        .accept(MediaType.TEXT_EVENT_STREAM)
        .bodyValue(body)
        .retrieve()
        .bodyToFlux(String.class);   // 이벤트의 data 값이 String으로 흘러나옴
```

- `.block()` 없이 Flux를 그대로 구독해야 스트림이 실시간으로 흐른다.
- `bodyToFlux(ServerSentEvent.class)`로 받으면 event/id/retry 필드까지 구조적으로 접근 가능.

## ⚠️ 함정 모음

- **중간 프록시 버퍼링**: nginx 등이 응답을 버퍼링하면 이벤트가 한 번에 몰아서 도착 (스트리밍이 안 됨). `X-Accel-Buffering: no` 헤더 또는 nginx `proxy_buffering off` 필요. `proxy_read_timeout`도 스트림 길이보다 크게.
- **heartbeat 필수**: LLM이 오래 생각하는 구간 등 데이터가 안 흐르는 시간이 길면 프록시/LB가 유휴 커넥션으로 판단하고 끊는다. 주석 라인(`: heartbeat`)을 15~30초 간격으로 흘려서 방지.
- **인코딩**: SSE는 UTF-8 고정인데, Spring `@EnableWebMvc` 환경의 기본 `StringHttpMessageConverter`가 ISO-8859-1이라 **한글이 ???로 깨질 수 있다**. UTF-8 컨버터를 앞순위로 등록하거나 produces에 `;charset=UTF-8` 명시.
- **HTTP/1.1 브라우저 커넥션 제한**: 도메인당 동시 커넥션 ~6개 제한에 SSE가 하나를 상시 점유. 탭 여러 개 열면 고갈될 수 있음 (HTTP/2는 멀티플렉싱이라 완화됨).
- **에러 전달 방법이 없다**: 스트림이 시작된 후엔 HTTP 상태코드를 바꿀 수 없음(이미 200 전송됨). 에러도 이벤트(`event: error`)로 보내거나, "완료 이벤트 없이 끊김"을 프론트가 에러로 해석하는 계약이 필요.

## 💡 판단 기준

- 서버→클라 단방향이면 SSE가 기본값. WebSocket 대비 인프라(프록시/인증/재연결)가 거저다.
- **GET + 구독형(알림 등) → EventSource. POST + 요청 본문 필요(LLM 채팅) → fetch 스트리밍.** 후자는 자동 재연결이 없으니 재연결·이어받기가 중요하면 "전송(POST)과 구독(GET)을 분리"하는 설계도 검토.
- 구체 케이스: LLM 채팅 스트리밍에서 질문 본문 때문에 POST SSE를 썼고, 그 대가로 EventSource의 자동 재연결을 포기 → 프론트는 "complete 이벤트 없이 종료 = 실패"로 처리하는 계약이 필요했다.

## 참고
- MDN Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- WHATWG HTML spec (SSE): https://html.spec.whatwg.org/multipage/server-sent-events.html — 비-200·잘못된 Content-Type이면 "fail the connection"(재연결 없음)
- Spring `ResponseBodyEmitter` 소스(`writeLock`): https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/mvc/method/annotation/ResponseBodyEmitter.java
- Spring SseEmitter: https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html

---
학습 날짜: 2026-07-07 (2026-10-02 재연결 조건·send 스레드 안전성 정정)
계기: LLM 채팅 스트리밍 MR에서 `SseEmitter` + `text/event-stream` + heartbeat + UTF-8 컨버터 등록 코드를 보고 각각이 왜 필요한지 파면서.

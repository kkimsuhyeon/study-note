# WebSocket - HTTP에서 업그레이드하는 양방향 전용 프로토콜

> 한 줄 요약: HTTP 핸드셰이크(`Upgrade: websocket` → `101`)로 시작해 **같은 TCP 커넥션을 양방향 메시지 채널로 전환**하는 별도 프로토콜. 자유도가 큰 만큼 재연결·인증·스케일아웃을 전부 직접 설계해야 한다.

전체 지형(폴링/SSE와 비교)은 [realtime-communication.md](./realtime-communication.md) 참고.

## 언제 쓰나
- **클라이언트도 계속 보내야 하는** 실시간 양방향: 채팅(입력·타이핑 표시), 멀티플레이 게임, 협업 편집(동시 커서), 실시간 호가 주문
- 바이너리 프레임이 필요한 경우 (SSE는 텍스트만)

## 핸드셰이크: HTTP로 시작해서 프로토콜을 갈아탄다

```http
# 클라이언트 → 서버 (일반 HTTP GET)
GET /ws/chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

# 서버 → 클라이언트
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

- `101 Switching Protocols` 이후 이 TCP 커넥션은 더 이상 HTTP가 아님 — WebSocket **프레임**(opcode: text/binary/ping/pong/close)이 오간다.
- `Sec-WebSocket-Key`/`Accept`는 "WebSocket을 이해하는 서버인지" 확인용 (보안 장치가 아님).
- URL 스킴: `ws://` (HTTP), `wss://` (HTTPS). 운영에선 사실상 wss 필수 (중간 프록시가 평문 ws를 자주 깨뜨림).
- 프로토콜 자체에 ping/pong 프레임이 내장되어 있어 연결 생존 확인에 쓴다.

## 클라이언트 API

```javascript
const ws = new WebSocket("wss://example.com/ws/chat");

ws.onopen    = () => ws.send(JSON.stringify({ type: "join", room: "42" }));
ws.onmessage = (e) => console.log(e.data);      // 서버가 보낸 메시지
ws.onclose   = (e) => { /* 자동 재연결 없음! 직접 백오프 재접속 구현 */ };
ws.onerror   = (e) => { ... };
```

⚠️ SSE의 EventSource와 달리 **자동 재연결이 없다**. 재연결 + 끊긴 동안 놓친 메시지 복구(서버측 버퍼/시퀀스 번호)를 직접 설계해야 한다 — WebSocket 운영 난이도의 본체.

## 서버 구현: Spring

### 순수 WebSocket (저수준)
```java
@Configuration
@EnableWebSocket
public class WsConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new ChatHandler(), "/ws/chat").setAllowedOrigins("*");
    }
}

public class ChatHandler extends TextWebSocketHandler {
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        session.sendMessage(new TextMessage("echo: " + message.getPayload()));
    }
}
```

### STOMP (메시징 서브프로토콜)
WebSocket은 "메시지가 오간다"까지만 정의하고 **의미(누구에게, 어떤 주제로)는 백지**다. STOMP를 얹으면 pub/sub 의미론(구독/발행/목적지 `/topic/...`)이 생기고, Spring이 `@MessageMapping`, `SimpMessagingTemplate` 등으로 지원한다. 채팅방·브로드캐스트류는 STOMP를 쓰는 게 표준적.

## ⚠️ 함정 모음

- **인프라 통과**: 프록시/LB가 `Upgrade` 헤더를 지원·전달하도록 설정 필요 (nginx `proxy_set_header Upgrade/Connection`). L7 방화벽·구형 프록시가 ws를 끊는 경우도 있어 wss 권장.
- **스케일아웃 시 세션 공유 문제**: WebSocket 세션은 **특정 서버 인스턴스의 메모리에 붙어 있다.** 서버가 2대 이상이면 A서버에 붙은 사용자에게 B서버가 직접 보낼 수 없음 → Redis pub/sub이나 외부 메시지 브로커(RabbitMQ 등)로 인스턴스 간 브로드캐스트 필요. ([scaling.md](../scaling.md)의 무상태 원칙이 깨지는 대표 지점)
- **인증**: 브라우저 WebSocket API는 커스텀 헤더를 못 붙인다 → 쿠키, URL 쿼리 토큰, 또는 연결 직후 첫 메시지로 토큰 전달 후 검증하는 패턴.
- **커넥션 수 = 메모리**: 연결당 세션 객체·버퍼가 상주. 대량 접속이면 이벤트 루프 기반(Netty/WebFlux)이 유리.

## 💡 판단 기준

- WebSocket을 고르는 순간 **재연결·놓친 메시지 복구·다중 인스턴스 브로드캐스트**가 전부 내 숙제가 된다. 이 셋을 설계할 각오(또는 필요)가 없으면 SSE/폴링으로 내려가라.
- 구체 케이스: LLM 채팅 "전송"은 질문이 POST 본문으로 들어가고 답변만 흘러나오는 단방향 → WebSocket의 양방향이 쓸 데가 없고 운영 숙제만 떠안는다 → SSE 선택이 맞았다. 반대로 "타이핑 중 표시 + 실시간 협업"이 요구사항에 들어오는 순간 WebSocket 재검토.

## 참고
- RFC 6455 The WebSocket Protocol: https://datatracker.ietf.org/doc/html/rfc6455
- MDN WebSocket: https://developer.mozilla.org/en-US/docs/Web/API/WebSocket
- Spring WebSocket/STOMP: https://docs.spring.io/spring-framework/reference/web/websocket.html

---
학습 날짜: 2026-07-07
계기: LLM 채팅 스트리밍 MR이 SSE를 쓴 이유를 이해하려고 "그럼 WebSocket이었다면 뭐가 달랐나"를 반대편에서 정리.

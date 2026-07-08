# Redis Pub/Sub - 저장 없는 실시간 방송 / 구독 전용 연결 / Spring 연동 3부품

> 한 줄 요약: `SUBSCRIBE 채널`로 **연결을 열어둔 구독자에게만** `PUBLISH 채널 메시지`가 즉시 push되는 방송 기능. **저장이 안 된다** — 그 순간 안 듣고 있으면 유실(fire-and-forget). 서버 인스턴스 간 이벤트 전파(스케일아웃 SSE/WebSocket)의 단골 해법이고, Spring에선 발행=`convertAndSend`, 구독=`RedisMessageListenerContainer`+`MessageListener` 세 부품으로 조립한다.

## 언제 쓰나
- **서버가 N대인데, 이벤트는 아무 서버에서나 발생하고, 받아야 할 클라이언트 연결(SseEmitter·WebSocket 세션)은 특정 서버 메모리에만 있을 때** — 전 서버에 브로드캐스트하고 "그 사용자 연결을 가진 서버"만 전달
- 캐시 무효화 신호, 설정 리로드 신호 등 "지금 접속 중인 프로세스들에게 알리기"
- ❌ 반대로 "반드시 전달돼야 하는" 메시지(주문 이벤트 등)에는 부적합 → Streams/Kafka

## 메커니즘: 연결의 방향부터

Redis는 우리 서버들을 모른다. **항상 앱이 Redis에 먼저 접속**하고, 구독이란 "이 연결로 그 채널 메시지를 밀어넣어달라"고 등록한 뒤 **연결을 계속 열어두는 것**이다 (전화 걸어놓고 안 끊는 상태).

```
Spring 서버1 ──연결A──▶ Redis   (일반 명령용: GET/SET — 요청/응답 후 반환)
             ──연결B──▶ Redis   (구독 전용: SUBSCRIBE SSE 후 계속 대기)
```

Redis 내부 장부: `채널 "SSE" 구독자 = [서버1 연결B, 서버2 연결B, 서버3 연결B]`

`PUBLISH SSE "..."`가 오면 Redis가 장부의 **열린 연결들에 메시지를 직접 써넣는다(push)**. 구독자가 폴링하는 게 아니라 Redis가 능동적으로 밀어넣는 것 — 지연이 밀리초 수준인 이유.

```
# 터미널 A                          # 터미널 B
127.0.0.1:6379> SUBSCRIBE SSE
Reading messages...                 127.0.0.1:6379> PUBLISH SSE "hello"
1) "message"                        (integer) 1   ← 받은 구독자 수
2) "SSE"                            # 0이면 아무도 안 들었고 메시지는 소멸
3) "hello"
```

**구독 전용 연결이 필요한 이유**: 구독 상태에 들어간 연결은 (RESP2 프로토콜 기준) SUBSCRIBE 계열 명령만 쓸 수 있다. 그래서 일반 명령용 연결과 분리해야 하고, Spring이 이걸 대신해주는 게 아래 Container다.

## Spring 연동: 3부품

```java
// ① 발행 — PUBLISH 명령 (필요할 때마다 비즈니스 코드가 호출)
redisTemplate.convertAndSend("SSE", jsonPayload);

// ② 구독 등록 — 앱 시작 시 1회, SUBSCRIBE 연결을 만들고 유지
@Bean
public RedisMessageListenerContainer container(RedisConnectionFactory cf, MessageListener listener) {
    RedisMessageListenerContainer container = new RedisMessageListenerContainer();
    container.setConnectionFactory(cf);
    container.addMessageListener(listener, new PatternTopic("SSE"));
    return container;
}

// ③ 수신 콜백 — 메시지 도착 시 Container가 호출해줌
@Component
public class MySubscriber implements MessageListener {
    @Override
    public void onMessage(Message message, byte[] pattern) {
        String channel = new String(pattern, StandardCharsets.UTF_8);
        String payload = new String(message.getBody(), StandardCharsets.UTF_8);
        // 역직렬화 후 처리
    }
}
```

| 부품 | 역할 | 비유 |
|---|---|---|
| `convertAndSend()` | `PUBLISH` 실행 | 마이크 |
| `RedisMessageListenerContainer` | 구독 전용 연결 생성·유지·재접속, 수신 스레드 관리 | 스피커(전원) |
| `MessageListener.onMessage()` | 도착 시 실행할 콜백 (연결 코드 없음 — 로직만) | 방송 들리면 할 행동 매뉴얼 |

- `ChannelTopic` = `SUBSCRIBE`(이름 그대로), `PatternTopic` = `PSUBSCRIBE`(`*` 와일드카드). `onMessage`의 두 번째 인자 `pattern`으로 어느 구독에 걸렸는지 확인.
- 발행자와 구독자는 **서로를 모른다** — 채널 이름만 공유. 서버가 몇 대로 늘어도 코드 불변 (채널명은 enum 등으로 한 곳에서 관리해 오타 어긋남 방지).

## 대표 케이스: 스케일아웃 SSE 알림

```
[서버2] 알림 발생 → convertAndSend("SSE", event JSON)
            │
         [Redis] 구독 연결 전부에 push
            │
   서버1·2·3 각자 onMessage()
            → 자기 메모리 맵에 대상 clientId의 emitter 있으면 send
            → 없으면 무시 (정상 — "내 담당 아님")
```

발행한 서버 자신도 구독자라 자기가 보낸 메시지를 받는다(정상). SseEmitter가 직렬화 불가능한 소켓이라 인스턴스 간 공유가 안 되니, **"연결 객체를 옮기는 대신 이벤트를 방송"**하는 구조. 서버측 emitter 관리는 [sse-emitter.md](../../java/spring/sse-emitter.md) 패턴 A 참고.

## ⚠️ 함정
1. **유실이 스펙이다.** 저장 안 됨, 재전송 없음, 구독자 없으면 소멸(at-most-once). 구독 연결이 잠깐 끊긴 사이의 메시지도 유실. "놓치면 안 되는" 메시지면 **Redis Streams**(저장됨, consumer group, `XREADGROUP`)나 Kafka로.
2. **onMessage는 Container의 수신 스레드에서 실행된다.** 여기서 오래 걸리는/블로킹 작업을 하면 후속 메시지 처리가 밀린다. 무거운 처리는 별도 스레드로 넘길 것.
3. **모든 구독 서버가 전부 받는다** — "한 대만 처리"가 필요한 작업(잡 큐)에 pub/sub을 쓰면 N중 실행된다. 그건 List(`LPUSH`/`BRPOP`)나 Streams consumer group의 영역.
4. 클러스터 환경에선 pub/sub이 전 노드로 전파되어 비용이 커질 수 있다 → 샤딩되는 `SPUBLISH`/`SSUBSCRIBE`(Redis 7+)가 따로 있다.

## 💡 판단 기준
- **"이 메시지, 그 순간 접속 중인 프로세스만 받으면 되나?"** — Yes(실시간 알림 중계, 캐시 무효화)면 pub/sub. No(유실 불가, 나중 소비, 한 대만 처리)면 Streams/큐. **"저장 없는 방송"이라는 성질이 기능이자 제약.**
- 구체 케이스: SSE 알림에서 "서버2에서 발생한 이벤트를 서버1에 붙은 사용자에게" 보내야 했다. emitter(소켓)는 옮길 수 없으니 pub/sub으로 전 서버에 방송하고 각자 "내 맵에 있나"로 판정 — 유실돼도 다음 알림/재조회로 복구되는 성격이라 pub/sub이 맞는 자리였다. 같은 요구가 "전자결재 문서 상태 변경"처럼 유실 불가였다면 DB 폴링이나 Streams를 골랐을 것.

## 참고
- Redis Pub/Sub 공식 문서: https://redis.io/docs/latest/develop/interact/pubsub/
- Redis Streams (유실 없는 대안): https://redis.io/docs/latest/develop/data-types/streams/
- Spring Data Redis Pub/Sub: https://docs.spring.io/spring-data/redis/reference/redis/pubsub.html
- 관련 노트: [Redis 기초](./redis-basics.md) · [SseEmitter 구현 패턴](../../java/spring/sse-emitter.md) · [스케일 아웃](../scaling.md) · [실시간 통신 비교](../network/realtime-communication.md)

---
학습 날짜: 2026-07-08
계기: SSE 알림 코드에서 `RedisMessagePublisher`(convertAndSend) / `RedisPubSubConfig`(ListenerContainer) / `RedisMessageSubscriber`(onMessage) 세 클래스가 왜 나뉘어 있는지 따라가며, "구독 = 연결을 열어두는 상태"라는 pub/sub의 동작 원리와 SSE와의 구조적 동형성(열어둔 연결에 밀어넣기 릴레이 2단)을 정리.

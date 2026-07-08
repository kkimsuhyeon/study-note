# Redis 기초 - 자료구조 서버 / 명령어로 대화 / TTL / RedisTemplate

> 한 줄 요약: Redis는 "키-값 캐시"가 아니라 **"자료구조를 담은 메모리를 네트워크로 공유하는 서버"** — 캐시·TTL·랭킹·카운터·분산 락·pub/sub까지 다 여기서 나온다. 클라이언트(앱)가 TCP로 접속해 `SET`/`GET`/`PUBLISH` 같은 **텍스트 명령어**를 보내는 구조이고, Java의 `RedisTemplate`은 그 명령어들의 포장일 뿐이다.

## 언제 쓰나
서버가 여러 대가 되는 순간 필요해지는 **"우리들만의 공용 메모리"**:
- DB까지 가기엔 아까운 조회 결과 캐시, 외부 API 토큰 보관 (TTL)
- 서버 인스턴스 간 이벤트 전파 ([pub/sub](./redis-pubsub.md))
- 실시간 랭킹, 조회수/요청수 카운팅, rate limiting
- "여러 서버 중 하나만 실행" 보장 (분산 락)

## 동작 모델: 명령어로 대화하는 서버

Redis는 별도 프로세스로 떠 있고, 클라이언트가 접속해 명령을 보내면 응답한다. SQL 대신 자기만의 명령어 세트를 쓴다. `redis-cli`로 직접 실험 가능:

```
127.0.0.1:6379> SET token:user1 "abc123"
OK
127.0.0.1:6379> GET token:user1
"abc123"
127.0.0.1:6379> EXPIRE token:user1 3600     # 1시간 뒤 자동 삭제
(integer) 1
127.0.0.1:6379> TTL token:user1             # 남은 수명 확인
(integer) 3597
```

## 자료구조와 대표 용도 (String만 있는 게 아니다)

| 자료구조 | 명령어 예 | 대표 용도 |
|---|---|---|
| String | `SET` / `GET` / `SETEX`(TTL 포함 저장) | 캐시, 토큰, 세션 |
| Hash | `HSET user:1 name kim` / `HGET` | 객체 필드 저장 |
| List | `LPUSH` / `RPOP` | 대기열(큐) |
| Set | `SADD` / `SISMEMBER` | 중복 없는 집합, 방문자 집계 |
| Sorted Set | `ZADD board 100 kim` / `ZREVRANGE` | **실시간 랭킹** (점수순 상위 N이 즉시) |
| (기능) Pub/Sub | `PUBLISH` / `SUBSCRIBE` | 실시간 방송 → [redis-pubsub.md](./redis-pubsub.md) |
| (기능) Streams | `XADD` / `XREADGROUP` | **저장되는** 메시지 큐 (경량 Kafka 느낌) |

자주 보는 패턴 명령:
- `INCR key` — 원자적 +1. 동시에 몰려도 정확 → 카운터, rate limiting
- `SETNX key val` (= `SET key val NX`) — **없을 때만** 저장 → 분산 락의 기본 재료 (스케줄러 중복 실행 방지)

## Spring에서: RedisTemplate = 만능 리모컨

`RedisTemplate`은 Spring Data Redis의 클라이언트로, Redis 명령 종류별로 메서드 그룹이 나뉜다:

| RedisTemplate | Redis 명령 | 용도 |
|---|---|---|
| `opsForValue().set/get()` | `SET`/`GET` | 키-값 |
| `opsForValue().set(k, v, 3, TimeUnit.HOURS)` | `SETEX` | TTL 포함 저장 |
| `opsForHash()` / `opsForList()` / `opsForZSet()` ... | `HSET`/`LPUSH`/`ZADD` ... | 자료구조별 |
| `expire()` / `delete()` | `EXPIRE`/`DEL` | 수명/삭제 |
| `convertAndSend(channel, msg)` | `PUBLISH` | pub/sub 발행 (convert=직렬화 + send=전송) |

제네릭(`RedisTemplate<String, String>`)과 직렬화기 설정이 "Java 객체 ↔ Redis 바이트" 변환을 결정한다. 실무에선 키=String, 값=String(JSON 직접 직렬화) 조합이 흔하다 — 값 직렬화를 라이브러리에 맡기면 클래스 패키지가 페이로드에 박히는(JdkSerialization) 등 호환성 문제가 생기기 쉬워서.

## ⚠️ 함정
1. **메모리가 본체다.** 데이터셋이 메모리에 다 올라간다. 스냅샷(RDB)·로그(AOF)로 디스크 영속화가 가능하지만 기본 감각은 "꺼지면 없어도 되는 데이터/재생성 가능한 데이터"를 두는 곳. source of truth는 RDB(관계형 DB)에.
2. **명령 실행이 (거의) 단일 스레드.** 명령 하나가 오래 걸리면 전체가 밀린다 — 운영에서 `KEYS *` 금지(`SCAN` 사용), 거대한 컬렉션 통째 조회 주의.
3. **TTL 갱신은 명시적으로.** `GET` 한다고 수명이 연장되지 않는다. 슬라이딩 만료를 원하면 조회 후 `EXPIRE`를 다시 걸어야 함 (토큰 캐시에서 조회 시 3시간 재설정하는 식).
4. `SETNX` 락은 **해제 실패(장애로 DEL 못 함) 대비 TTL 필수**, 그리고 "내가 건 락인지" 확인 후 해제해야 한다 — 제대로 하려면 Redisson 같은 라이브러리로.

## 💡 판단 기준
- **"이 데이터, Redis가 죽으면 복구 못 해도 되나?"** — Yes(캐시·토큰·랭킹)면 Redis, No(주문·결제)면 RDB. Redis는 성능·공유 계층이지 저장소의 대체가 아니다.
- 구체 케이스: 외부 API 인증 토큰을 어디 둘까 — 매번 발급하면 느리고, DB에 두기엔 만료 관리가 번거로웠다 → **Redis에 TTL 3시간으로 저장**하고, 만료 임박/401 응답 시 지우고 재발급. "수명이 있는 파생 데이터"는 Redis TTL이 정확히 맞는 자리였다.

## 참고
- Redis 자료구조 공식 문서: https://redis.io/docs/latest/develop/data-types/
- 명령어 레퍼런스: https://redis.io/docs/latest/commands/
- Spring Data Redis: https://docs.spring.io/spring-data/redis/reference/redis/template.html
- 관련 노트: [Redis pub/sub](./redis-pubsub.md) · [분산 락 개념](../../java/concurrency/locks.md) · [스케일 아웃](../scaling.md)

---
학습 날짜: 2026-07-08
계기: SSE 알림 파이프라인 분석 중 Redis가 토큰 캐시(TTL)와 서버 간 방송(pub/sub) 두 용도로 쓰이는 걸 보고, "Redis = 키-값 저장소"라는 인식을 "자료구조 공용 메모리 서버"로 교정하며 정리.

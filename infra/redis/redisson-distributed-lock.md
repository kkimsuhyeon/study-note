# Redisson 분산 락 내부 동작 - SETNX 관례 · Lua 원자성 · pub/sub 대기

> **한 줄 요약**: Redis에 "락 기능"은 없다 — 락은 **"키를 먼저 쓰는 데 성공한 쪽이 주인"이라는 관례**이고, Redisson은 그 관례를 hash + Lua 스크립트 + pub/sub으로 포장한 라이브러리다. 핵심 함정: `leaseTime`이 지나면 작업이 안 끝났어도 락이 풀린다.

## 언제 쓰나

- **여러 JVM 인스턴스가 같은 자원을 두고 경쟁**할 때 — `synchronized`/`ReentrantLock`은 한 JVM 안에서만 유효하므로, 스케일 아웃 환경([스케일 아웃 & 배포 모델](../scaling.md))에서는 모든 인스턴스가 공유하는 저장소(Redis)에 락을 둬야 한다
- 전형적 케이스: 채번(중복 없는 번호 발급), 배치와 화면 요청의 동시 실행 차단, 재고 차감
- 락 종류 전반(낙관/비관/분산의 선택 기준)은 [락 개념 종합](../../java/concurrency/locks.md) 참고

## 원리: 락의 재료가 되는 Redis 성질 2가지

1. **단일 스레드 명령 처리** — 여러 서버가 동시에 명령을 보내도 Redis는 한 번에 하나씩 처리한다. "동시"가 존재하지 않으므로 Redis가 곧 심판이다.
2. **"없으면 쓰고, 있으면 실패"의 원자 실행** — 가장 원시적 형태가 `SETNX`(SET if Not eXists):

```
서버 A: SET my_lock "A" NX PX 120000   → OK   (키가 없었다 → 락 획득)
서버 B: SET my_lock "B" NX PX 120000   → nil  (이미 있다 → 획득 실패)
```

이게 락의 전부다. **먼저 저장에 성공한 쪽이 락 주인**이고, "먼저"의 판정은 Redis의 처리 순서가 한다. 둘 다 성공하는 일은 없다는 것이 보장의 핵심. `PX`(TTL)는 주인이 죽어도 락이 영원히 안 풀리는 사태를 막는 안전장치.

## 사용 예시 (Redisson API)

```java
RLock lock = redissonClient.getLock("empl_lock:ORG123");
boolean acquired = lock.tryLock(3, 120, TimeUnit.SECONDS);  // waitTime, leaseTime
if (!acquired) throw new LockAcquisitionTimeoutException(...);
try {
    // 임계 구역
} finally {
    lock.unlock();   // 성공/예외 무관하게 반드시 해제
}
```

- **waitTime**: 락 획득을 얼마나 기다릴까 (지나면 `false` 반환)
- **leaseTime**: 락을 최대 얼마나 보유할까 (지나면 Redis가 자동 해제 = 키 TTL)
- 보일러플레이트가 여러 곳에 반복되면 [커스텀 어노테이션 + AOP](../../java/annotation/custom-annotation.md)로 선언화하는 게 관용 패턴 (`@DistributedLock(key=..., value="#param.id")`)

## Redisson이 SETNX 대신 hash + Lua를 쓰는 이유

`tryLock()` 호출 시 실제로는 대략 이런 Lua 스크립트가 실행된다:

```lua
-- (개념적으로 단순화)
if 키가 없으면 then
    HSET 락키  "클라이언트UUID:스레드ID"  1    -- 주인 기록
    PEXPIRE 락키 <leaseTime>
    return nil                                 -- 획득 성공
end
if 키의 주인이 나(같은 UUID:스레드ID)면 then
    HINCRBY 락키 <나> 1                        -- 재진입 카운트 +1
    PEXPIRE 락키 <leaseTime>
    return nil                                 -- 획득 성공 (reentrant)
end
return PTTL(락키)                              -- 실패, 남은 TTL 반환
```

- **Lua 스크립트는 통째로 원자 실행** — "확인 → 쓰기"가 별도 명령이면 그 사이에 끼어들 수 있지만, Lua로 묶으면 불가능
- **hash 필드에 주인(UUID:스레드ID)을 기록**하므로: 같은 스레드 재획득은 카운트 증가(재진입 지원), **남의 락을 unlock하면 거부** 가능 — 단순 SETNX로는 둘 다 안 됨

락이 잡힌 상태를 Redis에서 직접 보면:

```
> HGETALL empl_lock:ORG123
"a1b2c3d4-uuid:47" → "1"     (어떤 인스턴스의 47번 스레드, 재진입 1회)
> PTTL empl_lock:ORG123
118234                        (약 118초 남음)
```

## waitTime 동안 기다리는 방식 — 폴링이 아니라 pub/sub

1. 획득 실패한 대기자는 `redisson_lock__channel:{락키}` 채널을 **구독**하고 잠든다 ([Redis Pub/Sub](./redis-pubsub.md)) — "잡혔나?"를 반복해서 묻는 폴링이 아니라, 해제 방송이 올 때까지 아무것도 안 함
2. 주인이 `unlock()` — 역시 Lua: 재진입 카운트 -1, 0이 되면 `DEL` + 채널에 해제 메시지 **publish** (삭제와 방송이 원자적 한 덩어리 → "지워졌는데 방송 안 나간" 상태 없음)
3. 방송을 들은 대기자들이 깨어나 다시 tryLock Lua를 시도 — **깨어남 ≠ 획득**. 대기자가 셋이면 셋 다 깨어나 재경쟁하고, Redis가 먼저 처리한 하나만 성공, 나머지는 남은 시간 동안 다시 구독하고 잠든다
4. waitTime이 다 지나도록 못 잡으면 `tryLock`이 `false` 반환

### tryLock 대기 타임라인 — true는 즉시, false만 waitTime을 채운다

`tryLock`은 호출한 스레드를 붙잡아두는(블로킹) 메서드고, 반환 시점이 비대칭이다:

```
tryLock(3초, 120초) 호출
  │
  ├─ 즉시 1차 시도 (Lua 쓰기)
  │    └─ 성공 → 【true, 거의 0초】       ← 경합 없을 때의 대부분 경우 (Redis 왕복 1회, ms 단위)
  │
  ├─ 실패 → 채널 구독하고 잠듦
  │
  ├─ (예: 1.2초 시점) 해제 방송 수신 → 깨어나 재시도
  │    ├─ 성공 → 【true, 1.2초】
  │    └─ 실패(재경쟁 패배) → 남은 1.8초 다시 구독·잠듦
  │
  └─ 3초 소진 → 【false, 약 3초】         ← "기다려봤는데 끝내 못 잡음"이므로 항상 꽉 채움
```

- **true**: 0 ~ waitTime 사이 아무 때나, 잡히는 순간 바로
- **false**: 사실상 항상 waitTime을 꽉 채운 뒤 (예외: 대기 중 인터럽트 → waitTime 전에 `InterruptedException`)

## 함정 / 메커니즘 (⚠️)

- ⚠️ **leaseTime 만료 = 보호 소멸**: 작업이 leaseTime을 넘기면 락이 자동 해제되어 다른 요청이 들어온다. 임계 구역의 최악 소요 시간보다 넉넉하게 잡아야 한다. 반대로 만료 뒤 원래 주인이 `unlock()`하면 `IllegalMonitorStateException`이 난다(이미 내 락이 아니므로) — 이 예외는 "이미 풀렸음"으로 간주하고 무시하는 처리가 필요.
- ⚠️ **leaseTime을 명시하면 watchdog이 꺼진다**: leaseTime 없이 `lock()`/`tryLock(waitTime, unit)`을 쓰면 Redisson watchdog이 기본 30초 TTL을 걸고 **작업이 살아있는 동안 자동 연장**해준다. leaseTime을 명시하는 순간 연장은 없다 — "자동 연장 vs 예측 가능한 상한"의 트레이드오프.
- ⚠️ **비공정(non-fair) 락**: 해제 방송 후 재경쟁이라 오래 기다린 쪽이 우선권을 갖지 않는다. 순서 보장이 필요하면 `getFairLock()` (성능 비용 있음).
- ⚠️ **대기 = 호출 스레드 블로킹**: waitTime 동안 HTTP 요청 스레드가 통째로 멈춘다. 경합이 심하면 요청들이 waitTime씩 매달렸다 실패하는 그림 — waitTime은 "사용자가 기다려줄 만한 시간"으로 짧게 (수 초).
- ⚠️ **트랜잭션과 함께 쓰면 순서가 정합성을 결정**: "락 획득 → 트랜잭션 시작 → 커밋 → 락 해제"여야 한다. 반대로 락이 트랜잭션 안쪽이면 "락 해제 후 커밋 전" 틈에 다른 요청이 옛 데이터를 읽는다. AOP로 감쌀 때 `@Order`로 제어 — [커스텀 어노테이션](../../java/annotation/custom-annotation.md) 노트 참고.
- ⚠️ **Redis가 단일 장애점**: Redis가 죽으면 락 획득 자체가 안 된다(= 기능 마비). 또 단일 인스턴스 Redis 장애 조치(failover) 중 락 유실 가능성이 있어 Redlock 알고리즘(다중 인스턴스 과반 획득) 논의가 있지만, 시계 가정에 대한 반론(Kleppmann)도 유명 — 정합성이 목숨인 돈 문제라면 락에만 의존하지 말고 DB 제약(유니크 인덱스, 조건부 UPDATE)을 최후 방어선으로 두는 게 정석.

## 💡 판단 기준

- **케이스**: 사원 등록 API와 데이터 마이그레이션 배치가 서로 다른 인스턴스에서 같은 기관의 사번 채번을 동시에 수행할 수 있었다 → JVM 락으로는 막을 수 없어 `기관ID를 포함한 키`의 Redis 분산 락으로 직렬화. 키에 자원 식별자(orgId)를 넣어 **다른 기관끼리는 경합 없이 병렬** 처리.
- **두 시간 값의 의미가 다르다**: `waitTime` = 사용자(호출자)가 기다려줄 시간 → 짧게, 빨리 실패시키고 재시도 유도. `leaseTime` = 임계 구역의 최악 소요 시간 → 넉넉하게. 케이스: 배치가 락을 오래 물 수 있는 상황(leaseTime 120초)에서 화면 요청은 3초만 기다리다 "나중에 다시 시도"로 fail-fast.
- **판단**: 분산 락은 "여러 JVM + 같은 자원"일 때만 꺼내는 도구다. 인스턴스가 하나뿐이거나 자원이 DB 행 하나면 DB 락(`SELECT FOR UPDATE`, [@Lock](../../java/jpa/lock.md))이 더 단순하고 트랜잭션과 수명도 일치한다. 분산 락을 쓰더라도 **최종 정합성은 DB 제약으로 이중 방어**할 것 — 락은 성능(불필요한 실패/재시도 방지) 수단이지 유일한 방어선이 아니다.

## 참고

- [Redisson Wiki - Distributed locks](https://redisson.pro/docs/data-and-services/locks-and-synchronizers/)
- [Redis 공식 - Distributed Locks with Redis (Redlock)](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- [Martin Kleppmann - How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — Redlock 반론
- Redis 기초(자료구조·TTL): [redis-basics](./redis-basics.md)

---
학습 날짜: 2026-07-09
계기: 회사 프로젝트의 `@DistributedLock`(Redisson) 분석 중 "Redis에 락 기능이 따로 있는 건가?"라는 의문에서 — 락은 기능이 아니라 SETNX 관례이고, Redisson은 hash+Lua+pub/sub으로 재진입·TTL·대기를 얹은 것임을 확인

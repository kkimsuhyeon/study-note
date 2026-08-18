# 이벤트 유실 방지 — Transactional Outbox와 하이브리드 변형

> **한 줄 요약**: `@Async`의 큐는 JVM 메모리라 재시작·크래시·큐 포화로 이벤트가 **흔적 없이** 사라진다. 이벤트의 존재를 업무 데이터와 **같은 DB 트랜잭션**에 새겨두면(outbox), 유실이 "사라짐"이 아니라 "PENDING 행"이 되어 재처리할 수 있다.

## 언제 쓰나

- 비동기로 넘긴 이벤트에 **유실되면 안 되는 업무**(외부 요청, 후속 처리, 정산 트리거)가 실릴 때
- DB 커밋과 이벤트 발행을 함께 해야 할 때 — 아래 "이중 쓰기 문제"가 있는 모든 곳
- 반대로 유실돼도 그만인 것(캐시 워밍, 통계성 로그)이면 fire-and-forget으로 충분 — 이 패턴은 공짜가 아니다

## 문제 1: fire-and-forget의 큐는 메모리다

```java
@Async @TransactionalEventListener(phase = AFTER_COMMIT)
public void on(DomainEvent event) { process(event); }   // 처리하다 죽으면? 기록 없음
```

`@Async` 제출은 `ThreadPoolTaskExecutor`의 **인메모리 큐**에 들어간다. 배포 재시작, 크래시, 큐 포화(`RejectedExecutionException`) — 어느 경우든 이벤트가 증발하고, **증발했다는 사실조차 남지 않는다.**

## 문제 2: 이중 쓰기(dual-write)

한 업무가 두 시스템에 써야 한다: ① DB(트랜잭션 보장 O) ② 이벤트 시스템(트랜잭션 밖). 이 둘을 원자적으로 묶을 방법이 없어서 "DB는 커밋됐는데 이벤트는 유실" 또는 "이벤트는 나갔는데 DB는 롤백"이 생긴다. Kafka 같은 외부 브로커를 써도 똑같이 발생하는 분산 시스템의 고전 문제.

## 교과서 해법: Transactional Outbox

이벤트를 큐에 직접 넣지 않고, **업무 데이터와 같은 트랜잭션으로 outbox 테이블에 INSERT** 한다.

```
[업무 트랜잭션]
  UPDATE 업무테이블 ...
  INSERT INTO outbox (event_id, payload, status='PENDING')
  COMMIT              ← 원자적으로 함께 성공/실패. "업무만 커밋되고 이벤트 없음" 상태가 불가능해짐
```

별도 릴레이가 outbox를 읽어 실제 처리/발행하고 DONE으로 마킹한다. 릴레이 방식은 폴링(단순) 또는 CDC(Debezium 등 — DB 로그를 읽어 지연·부하 최소화).

**약점**: 순수 폴링은 폴링 주기만큼 지연이 생긴다 (5초 주기면 평균 2.5초).

## 하이브리드 변형 — 빠른 길 + 안전망

Spring 이벤트 인프라만으로 구현하는 절충안. 평상시엔 인메모리로 즉시 처리하고, 이력 테이블은 **보험**으로 쓴다. ([phase 메커니즘은 application-events.md](./application-events.md))

```
@Transactional 업무 메서드
  ├─ 업무 로직
  ├─ publishEvent(event)                        ← 예약만
  ├─ [BEFORE_COMMIT] 이력 INSERT (PENDING)       ← 업무 트랜잭션에 합류. DB 작업만
  ├─ COMMIT ─── 롤백이면 이력도 같이 사라짐 (유령 이력 없음)
  └─ [AFTER_COMMIT] @Async 제출                  ← 외부 작업만. 확정 후에만
        └─ (워커 스레드) 처리 → 성공 시 markDone (PENDING → DONE)

[스케줄러] PENDING 을 주기적으로 선점 → 재제출     ← 안전망
```

### 왜 적재가 BEFORE_COMMIT이어야 하나

AFTER_COMMIT에 적재하면 "커밋 성공 ~ 적재" 사이에 크래시 나는 **창(gap)**이 생긴다. 업무는 확정됐는데 이력이 없으니 재처리 목록에도 없다 — **영구 유실이고, 유실된 줄도 모른다.** BEFORE_COMMIT이면 경우의 수가 둘로 닫힌다: 업무 롤백 → 이력도 없음 / 업무 커밋 → 이력 반드시 존재.

### 사고 시나리오별 결과

| 사고 | 결과 |
|---|---|
| 처리 중 크래시·재시작 | markDone 못 감 → PENDING 잔존 → 재처리 |
| 큐 포화로 제출 거부 | `RejectedExecutionException` 잡고 로그만 — 이력이 PENDING이라 유실 아님 |
| 업무 트랜잭션 롤백 | 이력도 같이 롤백 — 유령 이벤트 없음 |
| 처리 완료 후 markDone 직전 크래시 | PENDING 잔존 → **한 번 더 실행됨** (아래 "대가") |

### 재처리 루프의 설계 포인트

- **선점은 트랜잭션 안, 실행은 밖.** 프로세서가 수백 초짜리 외부 호출을 하면 같은 트랜잭션에선 그동안 DB 커넥션을 점유한다. 상태 마킹(짧은 트랜잭션)과 실행(트랜잭션 밖)을 분리.
- **grace period**: 방금 제출돼 아직 처리 중인 건을 폴러가 또 잡지 않게, "N분 이전 것만" 조건을 건다.
- **복원 불가 행은 FAILED로 격리**: payload 역직렬화 실패 등 재시도해도 안 되는 건 PENDING에 남겨두면 폴링마다 헛돈다.

### 통용 구현: Spring Modulith

직접 만들기 전에 알아둘 것 — **Spring Modulith의 Event Publication Registry가 정확히 이 패턴**이다. `@ApplicationModuleListener`(= `@Async` + `@TransactionalEventListener` + `@Transactional(REQUIRES_NEW)` 묶음)에 이벤트 발행을 같은 트랜잭션으로 DB에 적재하고, 완료 마킹하고, 미완료 건을 재발행하는 기능까지 제공한다. 요구사항이 표준적이면 이걸 검토하는 게 먼저다.

## 보장과 대가: at-least-once

이 구조의 보장은 **최소 1회 실행**이다. exactly-once가 아니다 — markDone 직전 크래시, grace period 안의 이중 선점 등으로 중복 실행이 실제로 가능하다. 분산 시스템의 기본 정리: **exactly-once 전달은 일반적으로 불가능하므로, at-least-once + 멱등한 소비자**로 만든다. 즉 이 패턴을 켜는 순간 "핸들러가 중복 실행을 견디는가"가 청구서로 따라온다 (상태 기반 가드, 멱등 키, upsert 등).

## ⚠️ 함정

1. **payload 직렬화가 스키마다.** 이력에 payload를 JSON으로 남기고 클래스명(payload_type)으로 복원한다면, **클래스 리네임·패키지 이동·필드 변경이 미처리 행의 복원 실패**로 이어진다. 리팩토링 전 PENDING 잔량 확인이 필요해진다.
2. **컬렉션 payload는 발행 시점에 건별 분해.** 리스트째 하나의 이벤트로 만들면 "이력 1행 = 처리 1회 = 이벤트 ID 1개"가 깨져서 부분 실패 시 어디까지 됐는지 알 수 없다. 발행 창구에서 분해해 리스너 아래로는 단건만 흐르게.
3. **ID를 직접 부여하는 이력 엔티티는 select-before-insert가 붙는다.** JPA `save()`는 ID가 있으면 신규 여부를 모른다 → `Persistable<ID>` 구현 + `isNew` 플래그(`@PostLoad`/`@PostPersist`로 해제)로 불필요한 SELECT를 막는다.
4. **초기 상태는 엔티티 생성자에 박아라.** `status = PENDING`을 서비스에서 set하지 않고 private 생성자 + 정적 팩토리로 강제하면 "PENDING이 아닌 상태로 태어나는 이력"이 타입상 불가능해진다. 상태 전이도 setter 대신 `markDone()`/`markFailed()` 같은 의미 있는 메서드로.

## 💡 판단 기준

- **"이 이벤트가 조용히 사라지면 사업적으로 문제인가?"** — Yes면 outbox 계열, No면 fire-and-forget. 중간은 없다: 유실 방지는 테이블·재처리 루프·멱등성이라는 비용을 전부 동반한다.
- **유실 방지를 켜면 중복 감당이 자동으로 청구된다.** at-least-once를 골라놓고 핸들러가 멱등하지 않으면, 유실 대신 중복 장애로 바뀔 뿐이다.
- 실무에서 겪은 케이스: 재시작 때마다 스크랩 요청 이벤트가 증발하는데 로그조차 없어 원인 추적이 불가능했다 → "큐 = 메모리"라는 전제를 의심하고 나서야 구조 문제로 보였다. **비동기 핸드오프 지점마다 "여기서 죽으면 이 작업의 존재를 아는 곳이 있나?"를 물을 것.**

## 참고

- [microservices.io — Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- [Spring Modulith — Working with Application Events](https://docs.spring.io/spring-modulith/reference/events.html)
- 관련 노트: [Spring 이벤트](./application-events.md) · [@Transactional](./transactional.md) · [스레드 풀 내부(@Async·거부 정책)](../concurrency/thread-pool.md) · [영속성 컨텍스트](../jpa/persistence-context.md)

---
*학습: 2026-08-18 — 회사 프로젝트의 도메인 이벤트 신뢰성 개선 MR을 읽으며. fire-and-forget이던 이벤트 처리에 BEFORE_COMMIT 이력 적재 + PENDING 재처리를 더하는 리팩토링에서 outbox 변형을 배움.*

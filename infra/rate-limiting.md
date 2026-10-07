# Rate Limiting — "시간당 몇 번"을 세는 법과 여러 축을 함께 거는 법

> **한 줄 요약:** rate limit은 "주체(누구) × 시간 창(언제)"마다 카운터를 두고, 창 안에서 N번을 넘으면 거절(429)하는 장치다. 동시 실행 수를 제한하는 세마포어와 달리 **쓴 횟수가 돌아오지 않고 창이 바뀌어야 리셋**된다.

## 언제 쓰나

- **비용이 큰 작업의 남용 방지**: AI 호출·SMS 발송·외부 유료 API처럼 한 번에 돈이 나가는 작업.
- **익명 API 보호**: 로그인이 없어 계정 단위로 막을 수 없을 때, 세션·IP 같은 약한 식별자로 막는다.
- **공정성**: 한 사용자가 공유 자원을 독점하지 못하게.

"동시에 몇 개"가 문제면 [세마포어](../java/concurrency/jvm-concurrency-tools.md) 5절, "시간당 몇 번"이 문제면 이 노트다. 한 API에 둘이 함께 있는 경우가 흔하다.

## 사용 예시 — DB 테이블로 구현한 고정 창(fixed window)

```sql
CREATE TABLE rate_buckets (
    subject_hash  VARCHAR(64) NOT NULL,   -- 누구 (세션/IP를 HMAC한 값)
    window_start  TIMESTAMPTZ NOT NULL,   -- 언제 (창 시작 시각, 정시/자정으로 자름)
    window_kind   VARCHAR(4)  NOT NULL,   -- 'hour' | 'day'
    request_count INTEGER     NOT NULL CHECK (request_count >= 0),
    PRIMARY KEY (subject_hash, window_start, window_kind)
);
```

```java
@Transactional   // 생성 유즈케이스 전체가 한 묶음
public Response create(...) {
    // ... 멱등 키 확인 (재전송이면 여기서 기존 결과 반환 → 아래 한도 소비 안 함. 직렬화는 advisory lock 노트)
    Instant hour = now.truncatedTo(HOURS), day = now.truncatedTo(DAYS);   // Instant 기준 = UTC 경계
    String session = hmac("session-rate", ownerId);
    String ip      = hmac("ip-rate:" + day, clientIp);                     // 날짜를 purpose에 넣어 IP 추적 방지
    List<Bucket> buckets = List.of(
        new Bucket(session, hour, "hour", 5),  new Bucket(session, day, "day", 20),
        new Bucket(ip,      hour, "hour", 20), new Bucket(ip,      day, "day", 100));
    buckets.stream().sorted(bySubjectThenStartThenKind)                    // 잠금 순서 고정 → 데드락 방지
           .forEach(b -> consume(b));                                      // 하나라도 초과면 예외 → 묶음 전체 롤백
    // ... 결과 저장
}

void consume(Bucket b) {
    repo.createIfAbsent(b);                 // INSERT ... ON CONFLICT DO NOTHING  (count=0 행 보장)
    BucketRow row = repo.findForUpdate(b);  // SELECT ... FOR UPDATE
    if (row.count() >= b.limit()) throw new RateLimited(retryAfter(b));
    row.increment();                        // 변경 감지로 UPDATE
}
```

- **축 두 개(세션·IP) × 창 두 개(시간·일) = 버킷 4개.** 요청 하나가 네 개를 **전부** 소비하고, 넷 다 통과해야 성공이다("둘 중 하나"가 아니다).
- 하나라도 초과해서 예외가 나면 묶음이 롤백되므로 **앞에서 이미 올린 버킷도 함께 되돌아간다.** 동시 6건 중 1건이 429가 나도 카운터는 6이 아니라 5로 남는다 — 테스트로 확인할 만한 성질.
- 빈 행을 먼저 만들고 `FOR UPDATE`하는 패턴의 동시성 설명은 [advisory lock 노트](../database/postgres-advisory-lock.md) 판단 기준 2번.
- 응답은 `429 Too Many Requests` + `Retry-After: <초>`(창이 끝날 때까지 남은 시간, 올림).

**같은 일을 PostgreSQL 한 문장으로도 할 수 있다.**

```sql
INSERT INTO rate_buckets (subject_hash, window_start, window_kind, request_count)
VALUES (:subject, :start, :kind, 1)
ON CONFLICT (subject_hash, window_start, window_kind)
DO UPDATE SET request_count = rate_buckets.request_count + 1
WHERE rate_buckets.request_count < :limit      -- 한도에 닿았으면 올리지 않음
RETURNING request_count;                        -- 아무 행도 안 돌아오면 한도 초과
```

| 방식 | 왕복 | 장점 | 단점 |
| --- | --- | --- | --- |
| 세 단계(INSERT 무시 → `FOR UPDATE` → 엔티티 수정) | 버킷당 3번 | 한도 판단·`Retry-After` 계산을 자바 엔티티에 둠. 읽기 쉽고 테스트하기 쉬움 | 문장이 많고 행 락을 트랜잭션 끝까지 쥠 |
| upsert 한 문장(`DO UPDATE ... WHERE ... RETURNING`) | 버킷당 1번 | 빠르고 경쟁이 DB 한 문장 안에서 끝남 | 규칙이 SQL로 감. JPA 엔티티·더티 체킹을 안 씀 |
| Redis `INCR` + `EXPIRE` | 1번 | rate limit에 가장 흔함. 만료 정리가 자동 | Redis 운영 필요. DB 트랜잭션과 묶이지 않아 "하나라도 초과면 전부 되돌리기"를 따로 구현 |

**"동시에 만들거나 바꾸는" 상황별로 흔히 쓰는 방법.**

| 상황 | 흔한 방법 |
| --- | --- |
| 행이 이미 있고 값을 늘리거나 줄임(재고·잔액) | 조건부 UPDATE 한 문장: `UPDATE stock SET qty = qty - 1 WHERE id = ? AND qty > 0` → 0건이면 부족 |
| 없으면 만들고 숫자를 올림(카운터·한도) | Redis `INCR`, upsert 한 문장, 또는 세 단계 |
| 딱 한 번만 만들어야 함(중복 가입·멱등 키) | 유니크 제약 + `ON CONFLICT DO NOTHING RETURNING`. 만들기 전에 다른 검사가 필요하면 [advisory lock](../database/postgres-advisory-lock.md) |
| 판단 로직이 복잡해 자바에서 해야 함 | 행을 먼저 만들고 `FOR UPDATE` 후 엔티티로 처리 |

💡 **트래픽이 적고 규칙을 코드로 읽히게 두고 싶으면 세 단계, 요청이 많아져 왕복 수가 부담이면 upsert 한 문장, 이미 Redis가 있거나 한도 종류가 많으면 Redis.** 어느 쪽이든 "확인 후 쓰기"를 두 문장으로 나누고 락 없이 두는 것만은 피한다.

## 무엇이 세지고 무엇이 안 세지나

| 요청 | 카운트 | 이유 |
| --- | --- | --- |
| 새 생성 요청 | ○ | |
| 같은 멱등 키 재전송 | ✗ | 멱등 확인이 한도 소비보다 **앞**에 있다. 재시도·더블클릭이 한도를 먹으면 안 된다 |
| 입력 검증 실패(400) | ✗ | 한도 소비 전에 실패. 대신 이런 쓰레기 요청은 ingress(프록시·WAF) 한도로 막는다 |
| 캐시 적중(같은 조건의 이전 결과 재사용) | **순서에 달림** | 한도 소비가 캐시 조회보다 앞이면 센다. "비용이 안 드는데 왜 세냐"와 "결과를 얻은 건 같으니 센다" 중 정책 결정 |
| 한도 초과로 거절된 요청 | ✗ | 롤백 |

## 옵션 비교 — 알고리즘

| 방식 | 동작 | 장점 | 단점 |
| --- | --- | --- | --- |
| **고정 창(fixed window)** | 정시·자정으로 자른 창마다 카운터 | 가장 단순, 행 하나 = 창 하나 | **경계 버스트**: 00:59에 5번 + 01:00에 5번 = 2분에 10번 |
| 슬라이딩 로그 | 요청 시각을 전부 저장, 최근 1시간 개수를 셈 | 정확 | 요청마다 행/원소 → 저장량 큼 |
| 슬라이딩 창 카운터 | 이전 창 카운트 × 겹치는 비율 + 현재 창 카운트 | 경계 버스트 완화, 저장량 작음 | 근사치 |
| 토큰 버킷 | 일정 속도로 토큰 충전, 요청마다 1개 소비 | 순간 폭주 허용량(버킷 크기)과 평균 속도를 따로 조절 | 상태 2개(토큰 수·마지막 충전 시각) |

- **Redis**로는 `INCR` + `EXPIRE`(고정 창)나 Lua 스크립트(토큰 버킷)가 흔하다([Redis 기초](./redis/redis-basics.md)). **Java 라이브러리**는 Bucket4j(토큰 버킷, Redis·JDBC 백엔드 지원).
- 고정 창의 경계 버스트는 **긴 창을 함께 거는 것**으로 대부분 충분히 제어된다. 시간 창 버스트가 2배여도 일 창이 총량을 묶는다.

## ⚠️ 함정

- **식별자별 우회 경로가 다르다 → 축을 두 개 건다.** 세션(쿠키) 축은 쿠키를 지우면 새 식별자가 되어 리셋된다. IP 축은 그 우회를 받쳐 주지만, NAT·회사망·프록시 뒤의 여러 사람이 **한 IP를 공유**해 억울한 거절이 생긴다. 그래서 세션 한도는 빡빡하게(시간 5), IP 한도는 넉넉하게(시간 20) 둔다. 서로의 구멍을 메우는 관계.
- **프록시 뒤에서는 모든 사용자가 같은 IP.** `getRemoteAddr()`는 직접 연결한 peer(=프록시)를 준다. `X-Forwarded-For`를 믿으려면 신뢰 프록시 판별이 먼저다 — 아무나 헤더를 위조해 IP 한도를 우회할 수 있다.
- **세션 축의 주체는 세션 ID가 아니라 세션 안의 소유자 값**이어야 한다. 세션 ID 회전(세션 고정 방지)마다 한도가 리셋되면 안 된다([HMAC과 해시](../java/security/hmac-and-hashing.md) 6절).
- **창 경계의 시간대.** `Instant.truncatedTo(DAYS)`는 **UTC 자정**이다. 한국 사용자에겐 오전 9시에 일 한도가 리셋된다. 의도라면 문서에 적고, 아니면 `ZonedDateTime`으로 자른다.
- **`Retry-After`는 "창 길이"가 아니라 "창이 끝날 때까지 남은 시간"이다.** 14:40에 시간 창(14:00~15:00)이 넘치면 3600초가 아니라 1200초. 계산은 `max(1, ceil(남은 ms / 1000))` — 내림하면 0.3초 남았을 때 0이 되어 클라이언트가 즉시 재시도해 또 거절되고, 0이면 헤더 자체를 생략하는 핸들러도 많아 최소 1을 둔다.
- **여러 버킷이 동시에 넘치면 "먼저 걸린 버킷"의 `Retry-After`가 나간다.** 첫 초과에서 바로 예외를 던지는 구조라 정렬 순서가 답을 정한다. 같은 주체 안에선 시작 시각 순 정렬로 일 창이 먼저 검사돼 긴 값이 나가지만, 주체끼리(세션 vs IP)는 해시 문자열 순이라 사실상 무작위 — 세션 시간 창(20분)과 IP 일 창(18시간)이 함께 넘쳤는데 "20분 뒤"라고 안내할 수 있다. 차단은 정확하고 안내만 짧다. 정확히 하려면 소비 메서드가 예외 대신 `boolean`을 돌려주고, 호출부가 실패한 버킷들의 남은 시간 중 **최댓값**으로 예외를 만든다(어차피 예외로 롤백되므로 뒤 버킷을 더 세도 카운트는 남지 않는다). 부수 효과로 "실패할 때만 필요한 값"을 저장 계층까지 넘기지 않게 된다.
- **만료된 버킷 행 정리.** 창이 지난 행은 쓸모없지만 스스로 사라지지 않는다(PostgreSQL엔 TTL 없음). `window_start` 인덱스 + 주기적 DELETE.
- **여러 버킷을 잠글 땐 순서를 고정한다.** 요청마다 다른 순서로 잠그면 교착([데드락](../java/concurrency/deadlock.md)).
- **DB 카운터는 같은 버킷의 요청을 줄 세운다.** 같은 IP(회사망·NAT)에서 몰리면 그 버킷 행 하나의 `FOR UPDATE`에서 전부 대기하고, 요청마다 쓰기가 버킷 수만큼 생긴다. DB 방식의 장점은 "생성 트랜잭션이 실패하면 카운트도 함께 롤백된다"는 정합성이다. 트래픽이 커서 이 대기가 보이기 시작하면 Redis `INCR`나 게이트웨이 한도로 옮기고, 롤백 대신 "실패 시 되돌리기"를 직접 설계한다.

## 💡 판단 기준

**"이 식별자를 버리고 새로 얻는 비용이 얼마인가"로 축과 한도를 정한다.** 쿠키는 공짜로 버려지니 세션 축만으로는 부족하고, IP는 바꾸기 번거롭지만 공유되니 넉넉하게, 계정은 가입 비용이 있으니 가장 신뢰할 만하다. 약한 식별자 여러 개를 **모두 통과해야 성공**으로 묶으면 각 축의 구멍을 서로 메운다. 그리고 한도 소비는 **멱등 확인 뒤, 비싼 작업 앞**에 둔다 — 재전송은 세지 않고, 거절은 비용이 나가기 전에.

## 참고

- [RFC 6585 §4 — 429 Too Many Requests](https://www.rfc-editor.org/rfc/rfc6585#section-4)
- [RFC 9110 §10.2.3 — Retry-After](https://www.rfc-editor.org/rfc/rfc9110#section-10.2.3)
- [IETF draft — RateLimit header fields](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)
- [Stripe — Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters)
- [Bucket4j](https://bucket4j.com/)
- 학습일: 2026-09-30. 계기: 로그인 없는 생성 API에서 "세션별·IP별로 카운트를 다 재고 있는 건가?"라는 질문. 멱등 재전송·캐시 적중·롤백이 카운트에 어떻게 작용하는지와 두 축을 거는 이유를 정리. 예시는 일반화했고 코드는 실행하지 않았다.

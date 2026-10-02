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
    // ... 멱등 키 확인 (재전송이면 여기서 기존 결과 반환 → 아래 한도 소비 안 함)
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
- **만료된 버킷 행 정리.** 창이 지난 행은 쓸모없지만 스스로 사라지지 않는다(PostgreSQL엔 TTL 없음). `window_start` 인덱스 + 주기적 DELETE.
- **여러 버킷을 잠글 땐 순서를 고정한다.** 요청마다 다른 순서로 잠그면 교착([데드락](../java/concurrency/deadlock.md)).

## 💡 판단 기준

**"이 식별자를 버리고 새로 얻는 비용이 얼마인가"로 축과 한도를 정한다.** 쿠키는 공짜로 버려지니 세션 축만으로는 부족하고, IP는 바꾸기 번거롭지만 공유되니 넉넉하게, 계정은 가입 비용이 있으니 가장 신뢰할 만하다. 약한 식별자 여러 개를 **모두 통과해야 성공**으로 묶으면 각 축의 구멍을 서로 메운다. 그리고 한도 소비는 **멱등 확인 뒤, 비싼 작업 앞**에 둔다 — 재전송은 세지 않고, 거절은 비용이 나가기 전에.

## 참고

- [RFC 6585 §4 — 429 Too Many Requests](https://www.rfc-editor.org/rfc/rfc6585#section-4)
- [RFC 9110 §10.2.3 — Retry-After](https://www.rfc-editor.org/rfc/rfc9110#section-10.2.3)
- [IETF draft — RateLimit header fields](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)
- [Stripe — Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters)
- [Bucket4j](https://bucket4j.com/)
- 학습일: 2026-09-30. 계기: 익명 생성 API에서 "세션별·IP별로 카운트를 다 재고 있는 건가?"라는 질문. 멱등 재전송·캐시 적중·롤백이 카운트에 어떻게 작용하는지와 두 축을 거는 이유를 정리. 예시는 일반화했고 코드는 실행하지 않았다.

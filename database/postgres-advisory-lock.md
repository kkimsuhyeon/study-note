# PostgreSQL Advisory Lock — 아직 없는 행을 잠그는 법 (check-then-act 직렬화)

> **한 줄 요약:** advisory lock은 테이블·행이 아니라 **애플리케이션이 정한 64비트 정수**에 거는 락이다. "이 키의 요청은 한 번에 하나만" 같은 규칙을, 그 키의 행이 아직 없을 때부터 DB 안에서 직렬화할 수 있다.

### 무엇이 잠기고 누가 기다리나 — "advisory"는 "권고"라는 뜻

DB 전체도, 테이블도, 행도 잠기지 않는다. PostgreSQL의 락 관리자 안에 **그 숫자 하나**가 "사용 중"으로 표시될 뿐이다. 그래서 이 락은 **같은 숫자를 달라고 요청하는 코드끼리만** 서로 기다리게 만든다. DB가 데이터에 강제하는 잠금이 아니라 애플리케이션이 지키기로 약속한 잠금이라 이름이 advisory(권고)다.

| 누가 | 기다리나 |
| --- | --- |
| 같은 숫자로 `pg_advisory_xact_lock`을 부른 다른 트랜잭션 | **기다린다** |
| 다른 숫자로 부른 트랜잭션 | 안 기다린다 |
| 락을 요청하지 않고 같은 테이블을 그냥 읽고 쓰는 쿼리 | **안 기다린다.** 락이 테이블·행과 무관하다 |
| 같은 서버의 다른 데이터베이스에서 같은 숫자 | 안 기다린다. 락 이름에 데이터베이스가 포함된다 |

⚠️ 셋째 줄 때문에, 같은 데이터를 다루는 **모든 코드 경로가 같은 규칙으로 락을 요청해야** 보호가 된다. 한 경로가 락 없이 INSERT하면 그 경로는 막히지 않는다 — 그래서 유일성 자체는 기본 키·유니크 제약이 최종으로 지킨다. 지금 잡혀 있는 advisory lock은 `SELECT * FROM pg_locks WHERE locktype = 'advisory'`로 볼 수 있다.

### 다른 DB에도 있나 — 이름 붙은 잠금(named lock)

대부분의 DB에 비슷한 기능이 있지만 **풀리는 시점**이 다르다. 이 차이가 가장 중요하다(일반 지식, 각 DB 문서로 재확인 필요).

| DB | 기능 | 잠금 이름 | 트랜잭션 끝에 자동 해제 |
| --- | --- | --- | --- |
| PostgreSQL | `pg_advisory_xact_lock` / `pg_advisory_lock` | 64비트 정수(또는 정수 2개) | `xact` 버전은 예 |
| MySQL·MariaDB | `GET_LOCK(name, timeout)` / `RELEASE_LOCK(name)` | 문자열 | **아니오.** 커밋·롤백과 무관하게 연결(세션)이 끝나거나 직접 풀어야 풀린다 |
| SQL Server | `sp_getapplock` / `sp_releaseapplock` | 문자열 | `@LockOwner = 'Transaction'`이면 예 |
| Oracle | `DBMS_LOCK.REQUEST` / `RELEASE` | 이름 → 핸들 | `release_on_commit => TRUE`면 예 |
| SQLite, H2 | 없음 | — | — |

- **세 가지를 무엇에 묶이느냐로 비교하면 차이가 한눈에 보인다.**

  | | PostgreSQL `xact` | MySQL `GET_LOCK` | Redis 분산락 |
  | --- | --- | --- | --- |
  | 묶이는 곳 | 트랜잭션 | DB 연결 | 키와 만료 시간 |
  | 푸는 방법 | 커밋·롤백과 함께 자동 | `RELEASE_LOCK` 직접 호출, 또는 연결 종료 | 직접 삭제, 또는 만료 |
  | 커밋보다 먼저 풀어 버릴 위험 | 없음 | **있음** | **있음** |
  | 작업 도중 저절로 풀림 | 없음 | 연결이 끊길 때만(네트워크 단절·DB 장애 조치·풀의 연결 정리) | 만료 시간이 작업보다 먼저 오면 |
  | 잠근 서버가 죽으면 | 즉시 풀림 | 즉시 풀림(연결 종료) | 만료 시간까지 아무도 못 씀 |
  | 풀기를 잊으면 | 해당 없음 | **연결이 살아 있는 한 계속 잠김.** 풀에 돌아간 연결이 다른 요청에 쓰임 | 만료 시간이 지나면 풀림 |

  MySQL은 Redis의 "만료 시간이 작업보다 먼저 와서 두 명이 동시에 들어오는" 문제가 없는 대신, "풀기를 잊으면 만료도 없이 계속 잠긴다"는 반대 방향 위험이 있다. 커밋 후에 풀어야 한다는 순서 문제는 Redis와 똑같이 남는다.
- 💡 **MySQL이었다면?** 선택지가 셋이다. ① `GET_LOCK` — 새 인프라는 없지만 **커밋 직후, 연결을 풀에 돌려주기 전에** 풀어야 하는 순서 관리가 필요하다. ② Redis 분산락 — 커밋 후 해제는 똑같이 필요하고 만료 시간과 Redis 운영이 더해진다. ③ 유니크 제약으로 중복 잡기 — InnoDB는 중복 키 에러가 나도 **그 문장만** 되돌리고 트랜잭션은 살려 두므로(PostgreSQL은 트랜잭션 전체가 중단됨), 멱등 키 행을 먼저 넣고 중복이면 기존 결과를 돌려주는 흐름이 PostgreSQL보다 쓰기 쉽다. 단 흐름을 다시 짜야 한다. **Redis가 이미 있으면 ②, 없으면 ①이나 ③.** 실무에서 MySQL과 Redisson 분산락 조합이 흔한 건 대개 Redis가 이미 캐시용으로 있기 때문이다.
- **MySQL처럼 세션에 묶이는 잠금은 커넥션 풀에서 위험하다.** `finally`에서 빠짐없이 풀어야 하고, 놓치면 풀에 돌아간 연결이 잠금을 쥔 채 다음 요청에 쓰인다. PostgreSQL 세션 수준 락과 같은 함정이다.
- **테스트 DB가 운영 DB와 다르면 이 함수 자체가 없다.** H2로 테스트하면 `pg_advisory_xact_lock` 호출이 실패하므로 Testcontainers 등으로 진짜 PostgreSQL에서 테스트해야 한다.
- **DB를 바꿀 가능성이 있으면 포트 뒤에 숨긴다.** `lockRequest(owner, key)` 같은 포트로 의도만 드러내면, DB를 바꿀 때 어댑터 한 곳만 고치면 된다. DB에 상관없이 쓰려면 잠금용 테이블 행을 쓰는 Spring Integration `JdbcLockRegistry` 같은 라이브러리나 Redis 분산락으로 가는 방법도 있다.

## FOR UPDATE와 같은 점·다른 점

"다른 트랜잭션은 기다려라"라는 **DB 락**이라는 점은 같다. 다른 점은 **무엇을 잠그느냐** 하나다.

| | `SELECT ... FOR UPDATE` | `pg_advisory_xact_lock(숫자)` |
| --- | --- | --- |
| 잠그는 대상 | DB가 WHERE로 찾아낸 **실제 행** | 앱이 넘긴 **숫자** 하나 |
| 의미를 누가 아나 | DB (이 테이블의 이 행) | 앱만. DB는 숫자의 뜻을 모른다 |
| 행이 아직 없으면 | 잠글 게 없음 | 상관없음 |
| 해제 | 트랜잭션 끝 | 트랜잭션 끝 (같음) |
| 경합 시 | 대기 | 대기 (같음) |

비유: `FOR UPDATE`는 창고의 **특정 상자에 자물쇠**, advisory lock은 창고 입구 **칠판에 번호 적기**. 같은 번호를 적으려는 사람은 지워질 때까지 기다린다. 번호의 뜻("이 사용자의 이 멱등 키")은 앱이 정한 약속이다.

## 언제 쓰나

`SELECT FOR UPDATE`는 **이미 있는 행**만 잠근다([SELECT FOR UPDATE](./select-for-update.md)). 다음 상황은 잠글 행이 없다.

- **멱등 생성**: "같은 Idempotency-Key로 이미 만들었나 조회 → 없으면 insert." 첫 요청에는 조회할 행이 없어 두 요청이 동시에 "처음"이라고 판단한다.
- **존재하지 않는 자원 단위의 임계 구역**: 예약 슬롯·계정별 일일 카운터처럼 첫 접근에서 행을 만드는 곳.
- **여러 앱 인스턴스가 공유하는 뮤텍스**가 필요한데 Redis 같은 별도 저장소를 두기엔 과한 경우. DB에 있으므로 스케일아웃해도 함께 동작한다. JVM `synchronized`·`ReentrantLock`은 한 프로세스 안에서만 유효하다.

## 사용 예시

멱등 생성의 뼈대. 트랜잭션 메서드의 **첫 DB 문장**으로 락을 잡고, 그 아래에서 조회 → 검사 → 삽입을 끝낸다.

```java
@Transactional
public Response create(Command command, String ownerId, UUID idempotencyKey) {
    String payloadHash = hashes.payload(command);      // 순수 계산은 락 밖에서
    repository.lockRequest(ownerId, idempotencyKey);   // ① 개념을 잠근다
    Optional<RequestKey> previous = repository.findRequest(ownerId, idempotencyKey); // ② check
    if (previous.isPresent()) {
        return previous.get().payloadHash().equals(payloadHash)
                ? existing(previous.get()) : conflict();
    }
    admit(ownerId);                                    // 한도 소비 → rate-limiting.md
    return repository.insert(..., idempotencyKey, payloadHash); // ③ act
}
```

```java
// Repository 구현 — 문자열 쌍을 bigint로 바꿔 락 키로 쓴다
public void lockRequest(String ownerId, UUID key) throws NoSuchAlgorithmException {
    byte[] digest = MessageDigest.getInstance("SHA-256")
            .digest((ownerId + ":" + key).getBytes(StandardCharsets.UTF_8));
    long lockKey = ByteBuffer.wrap(digest).getLong();  // 앞 8바이트
    jdbc.query("SELECT pg_advisory_xact_lock(?)", rs -> { }, lockKey);
}
```

- `pg_advisory_xact_lock(bigint)`은 같은 숫자를 다른 트랜잭션이 잡고 있으면 **블로킹 대기**한다. 반환값은 void.
- 락 키는 저장하지 않고 비밀도 아니므로 HMAC이 아닌 일반 해시로 충분하다([HMAC과 해시](../java/security/hmac-and-hashing.md) 6절). 서로 다른 쌍이 같은 `long`이 되면(2^64분의 1) 무관한 요청이 잠깐 줄을 서는 것뿐이고, 그 요청은 자기 키로 다시 조회하므로 결과는 틀리지 않는다.
- 왜 UUID 상위 64비트를 그대로 안 쓰나 — 락의 범위가 key 단독이 아니라 `(owner, key)` 조합, 즉 멱등 테이블의 기본 키와 같아야 하기 때문이다.

락 없을 때와 있을 때의 타임라인:

```text
락 없음                                  락 있음
T1: find → 없음                          T1: lock(L) → find 없음 → admit → insert → commit → L 해제
T2: find → 없음                          T2: lock(L) 대기 ................ 획득
T1: admit → insert → commit                  → find → T1의 행 → payload 같음 → 기존 응답(200)
T2: admit → insert → PK 위반 → 500
    (한도는 2회 소모)
```

## advisory lock은 분산락인가 — 같은 점·다른 점

여러 앱 인스턴스를 조율하는 뮤텍스라는 점에서 분산락의 한 형태다. 결정적 차이는 하나고 나머지는 거기서 파생된다. **advisory lock은 락과 보호하는 데이터가 같은 트랜잭션 안에 있고, 분산락은 락과 데이터가 다른 시스템에서 다른 수명으로 산다.**

| | `pg_advisory_xact_lock` | Redis 분산락 ([Redisson](../infra/redis/redisson-distributed-lock.md)) |
| --- | --- | --- |
| 해제 | 커밋/롤백과 **동시에 자동** | 코드가 `unlock()`. `finally`·소유자 확인 필요 |
| 홀더가 죽으면 | 서버가 커넥션 끊김을 감지하는 순간 abort → 해제. 프로세스 종료는 즉시, 호스트·네트워크 단절은 감지까지 지연(아래 (c)) | 안 풀리면 영원히 → **lease(TTL)** 필수 |
| lease 만료 위험 | 없음 | 작업이 lease보다 길면 두 홀더 동시 진입. watchdog도 GC 정지·단절엔 무력 |
| 해제 vs 데이터 가시성 | 락 해제 = 커밋. 다음 홀더는 반드시 최신을 봄 | **unlock이 commit보다 먼저** 실행되는 실수(`@Transactional` 안에서 unlock)가 단골. 커밋 후 해제를 `TransactionSynchronization`으로 직접 보장해야 한다 |
| 장애 도메인 | DB 하나 | 둘. Redis failover로 락만 증발 가능 |
| 보호 범위 | 이 DB를 거치는 짧은 작업만. 락 잡은 채 외부 호출 → 긴 트랜잭션 | DB 무관 작업·이종 서비스 조율 가능 |
| 대기 비용 | 대기자마다 DB 커넥션 점유 → 풀 고갈 위험 | pub/sub 대기, 커넥션 소모 없음 |
| 의존성 | 없음 | Redis 운영 |

**분산락에서만 생기는 사고 셋 — 전부 "락과 묶음의 수명이 다르다"에서 나온다.** (a)(b)는 advisory에서 원리적으로 불가능하고 (c)는 "서버가 끊김을 알아챌 때까지"로 줄어든다. Redis 락의 장치(lease·watchdog·소유자 토큰·`finally`·커밋 후 해제 동기화)는 이 사고 하나씩에 대응한다.

```text
(a) 해제가 커밋보다 먼저 — @Transactional 메서드 안에서 unlock하면 프록시의 COMMIT이 그 뒤
T1: lock → find 없음 → insert(미커밋) → unlock
T2:                                     lock → find → T1 행 아직 미커밋 → 없음 → insert
T1:                                                                        COMMIT
T2:                                                                               COMMIT → 중복/PK 위반
→ 고치려면 TransactionSynchronization.afterCompletion에서 unlock. advisory는 "해제=커밋"이라 이 순서가 없다.

(b) lease가 묶음보다 먼저 끝남 — 락 대기·GC 정지·느린 쿼리로 묶음 35초, lease 30초
T1: lock(30s) → 묶음 진행 ........ 30s: lease 만료로 락 소멸 ... 35s: 아직 진행 중
T2:                                lock 획득 → 같은 묶음 시작 → 둘 다 insert
→ watchdog은 GC 정지·단절 중 연장 불가. advisory는 묶음이 살아 있는 한 락도 산다.

(c) 홀더 사망 — unlock 못 함
T1: lock → 묶음 중 JVM 종료
T2: lock 대기 ...... lease까지 전원 대기, lease 없으면 영원
→ advisory는 커넥션 끊김 = 묶음 abort = 해제. 단 "즉시"는 프로세스가 죽어 OS가 소켓을 닫을 때뿐이다.
  앱 호스트가 사라지거나 네트워크가 끊기면 서버는 TCP keepalive(tcp_keepalives_* 기본값 = OS 기본, 리눅스는 보통 2시간)나
  idle_in_transaction_session_timeout(기본 0 = 끔)이 끊을 때까지 트랜잭션과 락을 쥐고 있다 → 이 타임아웃을 설정해 둔다.
```

**advisory의 강점이 곧 한계: 묶음 밖으로 못 나간다.** COMMIT하면 풀리므로 "락을 잡은 채 외부 호출 40초"를 advisory로 하면 트랜잭션을 40초 열어 두는 것 = 커넥션 40초 점유([@Transactional](../java/spring/transactional.md) 6-(4)). 잠가야 하는 구간이 어디까지 뻗느냐로 도구가 갈린다:

| 잠가야 하는 구간 | 도구 |
| --- | --- |
| DB 안에서 밀리초에 끝나는 묶음 하나 (멱등 검사 → 한도 → insert) | **advisory lock.** 해제=커밋이 공짜 |
| 외부 호출을 **포함**한 구간의 상호배제 | advisory 불가. **분산락 + lease** |
| 외부 호출 구간인데 "동시 실행 금지" 대신 "늦은 결과 폐기"로 충분 | 락 없이 `run_token` + `lease_until` + 조건부 UPDATE |

셋째 줄을 택하면 둘째 줄이 필요 없어지고, 남는 락 구간은 전부 첫째 줄이 된다 — Redis가 스택에 있어도 이 구간엔 advisory가 낫다.

**"외부 호출 동시 실행을 전역 N개로 제한"의 순서.** 묶음 밖까지 뻗는 제한이라 advisory 하나로는 안 된다. ① 인스턴스 1대면 워커 스레드 수 = N, 락 불필요 ② 여러 대면 claim 트랜잭션 안에서 `pg_advisory_xact_lock(상수)`로 "세는 순간"만 직렬화 → `count(*) WHERE status='running' AND lease_until > now()`가 N 미만이면 claim → COMMIT. 긴 외부 호출은 행에 남은 lease가 지킨다 ③ 대기자를 즉시 깨워야 하거나 대상이 DB와 무관하면 Redis 세마포어(Redisson `RPermitExpirableSemaphore`) ④ 어느 단계든 마지막 방어선은 **호출받는 쪽**의 동시 요청 제한. 작업 행·lease·reaper 설계 전체는 → [DB 작업 큐](../infra/db-job-queue.md). 어느 lease 방식이든 (b) 사고가 따라오므로 **작업 상한 < lease**는 공통 필수.

💡 **락이 지키는 구간이 "이 DB 안의 짧은 트랜잭션"이면 advisory, 그 구간에 외부 호출·다른 저장소·긴 작업이 끼거나 DB를 공유하지 않는 서비스 간 조율이면 분산락.** advisory를 고르면 "해제 = 커밋"이라 멱등 검사의 "뒤 요청이 앞 요청의 커밋을 본다"가 공짜지만, 분산락으로 같은 것을 만들려면 커밋 후 unlock을 코드로 보장해야 하고 빠뜨리면 동시성 테스트가 간헐적으로 깨진다.

## 종류/옵션 비교

| 함수 | 범위 | 해제 | 대기 |
| --- | --- | --- | --- |
| `pg_advisory_xact_lock(key)` | 트랜잭션 | **커밋/롤백 시 자동**. 명시적 unlock 없음 | 블로킹 |
| `pg_try_advisory_xact_lock(key)` | 트랜잭션 | 자동 | 즉시 `boolean` 반환 |
| `pg_advisory_lock(key)` | **세션(커넥션)** | `pg_advisory_unlock(key)` 또는 세션 종료. 트랜잭션이 끝나도 남는다 | 블로킹 |
| `pg_try_advisory_lock(key)` | 세션 | 수동 | 즉시 `boolean` 반환 |

- 각 함수에 `_shared` 변형(공유 락)과 `(int, int)` 두 정수 키 형태가 있다. 두 정수 형태는 "테이블 번호, 행 id"처럼 이름 공간을 나누기에 편하다.
- 세션 수준 락은 **재진입 카운트**가 쌓인다. 두 번 잡으면 두 번 풀어야 한다. `pg_advisory_unlock_all()`로 세션의 락을 전부 푼다.
- 커넥션 풀 환경에서는 세션 수준 락이 위험하다. unlock을 빠뜨리면 풀에 반환된 커넥션에 락이 남아 **다음 대여자가 물려받는다.** 트랜잭션 안에서 끝나는 용도면 `xact` 변형을 기본으로 삼는다.

## ⚠️ 함정/메커니즘

### 1. JDBC와 JPA가 **같은 커넥션**을 타야 한다

`xact` 락은 그 커넥션의 현재 트랜잭션에 묶인다. `JdbcTemplate`이 JPA와 다른 커넥션을 쓰면 락 문장이 autocommit으로 끝나는 순간 풀려 아무 보호도 못 한다. Spring의 `JpaTransactionManager`는 `JpaDialect`(Hibernate는 `HibernateJpaDialect`)를 통해 JDBC 커넥션을 노출해서, 같은 `@Transactional` 안의 `JdbcTemplate`이 같은 커넥션을 쓴다. 트랜잭션 매니저를 따로 만들었거나 DataSource가 둘이면 이 전제가 깨질 수 있다. 의심되면 락을 잡은 뒤 다른 커넥션에서 `pg_locks`(`locktype = 'advisory'`)를 조회해 보유 여부를 확인한다.

### 2. 격리 수준이 READ COMMITTED여야 "락 뒤의 조회"가 최신을 본다

T2가 락을 얻은 다음 실행하는 조회가 T1의 커밋을 봐야 한다. READ COMMITTED는 **문장마다 새 스냅샷**을 잡으므로 본다. REPEATABLE READ 이상은 **트랜잭션의 첫 문장 시작 시점**에 스냅샷을 고정하는데, 그 첫 문장이 바로 락 문장이라 락을 **기다리는 동안 T1이 커밋한 행이 스냅샷에 없다.** 락을 얻어도 조회가 "없음"을 돌려주고 insert에서 PK 위반이 난다. **`FOR UPDATE`로 바꿔도 소용없다** — REPEATABLE READ 이상에서는 `FOR UPDATE`도 트랜잭션 스냅샷으로 대상을 찾으므로 스냅샷 뒤에 커밋된 행은 여전히 안 보인다(최신 버전 재평가는 READ COMMITTED 전용, [SELECT FOR UPDATE](./select-for-update.md) ⚠️5). PG17에서 확인: RR 트랜잭션이 락 대기 후 `SELECT ... FOR UPDATE` → 0행, 이어진 INSERT → duplicate key. 격리 수준을 올릴 거면 이 경로만 READ COMMITTED로 두거나, 트랜잭션 시작 **전에** 세션 수준 `pg_advisory_lock`으로 잡고 끝난 뒤 푼다(스냅샷이 락 획득 뒤에 잡힌다. 대신 아래 종류 비교의 unlock 누락 위험을 진다). PostgreSQL 기본값(READ COMMITTED)에서는 문제없다.

### 3. 락 위치는 "순수 계산 뒤, 첫 check 앞"

해시 계산·검증처럼 DB를 안 건드리는 일은 락 **밖**에서 끝내 보유 시간을 줄인다. 반대로 check 문장보다 **뒤**에 잡으면 경쟁 구간이 그대로 남는다. `@Transactional` 메서드 안에서 잡으면 메서드 끝의 커밋까지 유지되므로, 그 아래의 조회·한도 소비·삽입이 전부 한 임계 구역이 된다. 409·429처럼 예외로 롤백돼도 `xact` 변형은 자동 해제된다.

### 4. 데드락 순서

advisory lock을 먼저 잡고 그 아래에서 행 락(한도 버킷 등)을 **정렬된 순서**로 잡으면 어떤 두 트랜잭션도 반대 순서로 자원을 기다리지 않는다([데드락](../java/concurrency/deadlock.md)). advisory lock 여러 개를 한 트랜잭션에서 잡을 때도 키 순서를 고정한다.

### 5. 대안과 비교

| 방법 | 장점 | 한계 |
| --- | --- | --- |
| **advisory lock → check → act** | 행이 없어도 직렬화. 범위를 `(owner, key)`처럼 가늘게 | 키→정수 매핑 필요. 같은 커넥션·READ COMMITTED 전제 |
| **행을 먼저 만들고 `FOR UPDATE`** (`INSERT ... ON CONFLICT DO NOTHING` → `SELECT ... FOR UPDATE`) | 행 락만으로 해결. 익숙하다. 단 READ COMMITTED 전제는 advisory와 같다 — RR에서는 스냅샷 뒤에 커밋된 충돌 행 때문에 `ON CONFLICT DO NOTHING` 자체가 `could not serialize access`로 실패한다(PG17 확인) | **빈 행을 미리 만들어도 되는 경우만**. FK가 아직 없는 부모를 가리키거나, insert 전에 끝나야 하는 검사(한도·권한)가 있으면 불가 |
| unique 제약 + 위반 예외 잡고 재조회 | 락 없음 | PostgreSQL은 제약 위반 시 **트랜잭션 전체가 중단**되어 새 트랜잭션에서 다시 읽어야 한다. 그 전에 한 부수 작업(한도 카운트)도 함께 롤백되어 흐름이 복잡 |
| `INSERT ... ON CONFLICT DO NOTHING RETURNING` | 한 문장 | insert **전에** 검사할 것(한도·권한)이 있으면 맞지 않는다 |
| 상위 자원 행 락(예: owner 행 `FOR UPDATE`) | 단순 | 한 사용자의 서로 다른 요청까지 전부 직렬화 → 너무 굵다 |
| Redis 분산 락 | DB 밖, 짧은 TTL | 저장소 하나 더. lease 만료 시 보호 소멸([Redisson 분산 락](../infra/redis/redisson-distributed-lock.md)) |

## 💡 판단 기준

세 갈래로 기억한다.

1. 잠글 행이 **있다** → `FOR UPDATE`
2. 잠글 행이 **없지만 빈 행을 미리 만들어도 된다** → `INSERT ON CONFLICT DO NOTHING` 후 `FOR UPDATE` (예: 카운터 버킷은 `count = 0` 행을 먼저 만들어도 무해)
3. 잠글 행이 **없고 미리 만들 수도 없다** → advisory lock (예: 멱등 키 행이 아직 없는 부모를 FK로 가리키고, 한도 검사가 insert 전에 끝나야 함)

같은 서비스 안에서 2와 3이 나란히 쓰이는 일이 흔하다. "왜 저기는 행 락이고 여기는 advisory인가"의 답은 항상 **빈 행을 미리 만들 수 있는가**다.

**"미리 만들 수 있다"의 판정**: 진짜 작업 전에 (a) 모든 `NOT NULL` 컬럼 값을 이미 알고, (b) FK가 가리키는 부모가 이미 존재하고, (c) 그 빈 행이 **참이면서 무해한 상태**여야 한다. 카운터의 `count = 0`은 "아직 0번"이라는 참인 상태라 통과한다. 멱등 키 행은 `order_id NOT NULL REFERENCES orders`가 (a)(b)에서 걸린다 — 결과가 나오기 전엔 가리킬 게 없다. 근본 차이는 **행의 수명**: 카운터는 여러 요청이 공유하며 갱신하는 행이라 0에서 시작할 수 있고, 멱등 키는 요청 하나의 결과 기록이라 "아직 없음"에서 시작할 수 없다.

⚠️ 2번 패턴은 "겹칠 일이 없다"가 아니라 **"겹쳐도 깨지지 않는다"**다. 두 트랜잭션이 동시에 `INSERT ... ON CONFLICT DO NOTHING`을 실행하면 한쪽이 만들고 다른 쪽은 (상대가 커밋 전이면 끝날 때까지 기다린 뒤) 조용히 넘어간다. 어느 쪽이든 문장이 끝나면 행은 존재한다. 그 다음 `FOR UPDATE`가 줄을 세우고, 대기 후 읽는 값은 READ COMMITTED의 재평가로 최신 커밋본이다([SELECT FOR UPDATE](./select-for-update.md) ⚠️1).

```text
T1: INSERT(count=0) ON CONFLICT DO NOTHING → 생성(미커밋)
T2: INSERT(count=0) ON CONFLICT DO NOTHING → T1의 미커밋 행과 충돌 → T1이 끝날 때까지 대기 ...
T1: FOR UPDATE → 0→1 → commit
T2:   ... 대기 풀림 → 이미 있음 → 무시, 에러 없음 → FOR UPDATE → 최신 행(1) → 1→2 → commit
```

(PG17에서 확인: T1이 커밋 전이면 T2의 `INSERT ... ON CONFLICT DO NOTHING`은 T1 종료까지 막힌다. 버킷 생성을 별도 트랜잭션으로 먼저 끝내면 이 대기는 사라진다.)

**전제를 못 박는다.** advisory를 고르면 `(범위를 정하는 키들) → 해시 → bigint`로 개념을 숫자로 만들어 트랜잭션의 첫 문장으로 잡고, 락 범위는 멱등 테이블의 기본 키와 똑같이 맞춘다. "같은 커넥션·READ COMMITTED" 두 전제는 코드 주석이나 테스트(동시 요청 N개 → 행 1개)로 남긴다. 유일성 자체는 여전히 DB 제약이 지킨다 — 락은 **사용자에게 500 대신 200을 주는 장치**이고, 제약은 락이 깨졌을 때의 마지막 방어선이다.

## 참고

- [PostgreSQL — Advisory Locks (Explicit Locking)](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)
- [PostgreSQL — Advisory Lock Functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS)
- [PostgreSQL — Transaction Isolation (스냅샷 시점)](https://www.postgresql.org/docs/current/transaction-iso.html)
- [Spring — JpaTransactionManager (같은 트랜잭션 안의 JDBC 접근)](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/jpa/JpaTransactionManager.html)
- [PostgreSQL — TCP keepalive 설정](https://www.postgresql.org/docs/current/runtime-config-connection.html#RUNTIME-CONFIG-TCP-SETTINGS) · [idle_in_transaction_session_timeout](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-IDLE-IN-TRANSACTION-SESSION-TIMEOUT)
- 학습일: 2026-09-22. 계기: 멱등 생성 코드에서 조회 앞에 있는 락 한 줄이 왜 필요한지, 왜 행 락이 아니라 SHA-256으로 만든 정수를 잠그는지 질문함. 예시는 일반화했다. 동일 커넥션 항목은 문서 기반이고, 격리 수준(RR에서 FOR UPDATE·ON CONFLICT)과 ON CONFLICT 대기는 2026-10-02 PG17 컨테이너에서 실행으로 확인했다.

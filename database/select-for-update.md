# SELECT FOR UPDATE — row 락과 READ COMMITTED 재평가 (PostgreSQL)

> **한 줄 요약**: 조회하면서 대상 row에 쓰기 락을 걸어 "조회 → 검증 → 변경(read-then-act)" 로직을 직렬화하는 문법. 핵심 함정 — 락 대기에서 풀려난 트랜잭션의 조건 재평가는 **잠긴 row가 실제로 변경된 경우에만** 일어나므로, "잠근 row"와 "상태가 바뀌는 row"가 다르면 락을 걸고도 옛 값을 보고 통과한다.

## 언제 쓰나

- **read-then-act 패턴**: 조회해서 검증한 뒤 그 결과를 근거로 INSERT/UPDATE 하는 로직 (재고 차감, 쿠폰 사용 처리, 한도 검증, 순번 채번 등). 일반 SELECT는 아무것도 잠그지 않아서 검증과 변경 사이에 다른 트랜잭션이 끼어들 수 있다.
- 특정 row를 **DB 레벨 뮤텍스**처럼 써서 같은 자원을 놓고 경쟁하는 트랜잭션들을 한 줄로 세울 때.
- JPA에서는 `@Lock(PESSIMISTIC_WRITE)`이 이 SQL을 생성한다 → [@Lock 기본](../java/jpa/lock.md). MyBatis/raw SQL에서는 쿼리 끝에 직접 붙인다. 락 분류 전체 그림은 [락 개념 종합](../java/concurrency/locks.md).

## 사용 예시

```sql
BEGIN;
-- row 락 획득. 같은 row를 노리는 경쟁 트랜잭션은 여기서 대기
SELECT * FROM coupon WHERE coupon_id = 1 FOR UPDATE;
-- (애플리케이션에서 사용 수/상태 검증...)
UPDATE coupon SET used_cnt = used_cnt + 1 WHERE coupon_id = 1;
COMMIT;  -- 락 해제 (락은 반드시 트랜잭션 종료 시점에 풀린다)
```

JOIN 쿼리에서는 `OF`로 잠글 테이블을 지정할 수 있다:

```sql
SELECT c.*, cc.used_yn
FROM   coupon c
JOIN   coupon_code cc ON cc.coupon_id = c.coupon_id
WHERE  cc.code = 'ABC123'
FOR UPDATE OF c, cc   -- c, cc 양쪽 row 잠금. OF 생략 시 FROM의 모든 테이블 row 잠금
```

- `OF` 뒤에는 FROM 절에 등장한 **테이블명 또는 별칭**을 나열한다. 지정한 테이블의 row만 잠기고 나머지는 일반 조회.
- 트랜잭션 안에서만 의미가 있다 (autocommit이면 문장 끝나는 순간 락 해제 → 무의미).

## 종류/옵션 비교

문법: `FOR lock_strength [ OF 테이블 [, ...] ] [ NOWAIT | SKIP LOCKED ]`

| 락 강도 | 의미 | 쓰는 상황 |
|---|---|---|
| `FOR UPDATE` | 배타 락. 다른 트랜잭션의 UPDATE/DELETE/모든 `FOR ...` 락 시도를 차단 | 이 row를 수정·삭제할 예정 (실무 대부분) |
| `FOR NO KEY UPDATE` | 키(PK/유니크) 컬럼은 안 바꾸는 UPDATE가 잡는 약한 배타 락. `KEY SHARE`와 공존 가능 | 키 안 건드리는 수정. FK 참조 삽입을 막지 않음 |
| `FOR SHARE` | 공유 락. 변경·배타 락만 차단, 다른 SHARE와 공존 | "읽는 동안 바뀌지만 마라" |
| `FOR KEY SHARE` | 최약 락. 키 변경·삭제만 차단 | FK 검사가 내부적으로 사용 |

대기 제어 옵션:

- `NOWAIT` — 잠긴 row를 만나면 대기하지 않고 **즉시 에러**. 락 경합 시 빠른 실패가 필요할 때.
- `SKIP LOCKED` — 잠긴 row는 **건너뛰고** 나머지만 조회. 여러 워커가 같은 테이블에서 작업을 나눠 가져가는 **작업 큐 패턴**의 표준 도구 (데이터 정합 관점에선 일관성 없는 뷰이므로 일반 조회엔 부적합).

## ⚠️ 함정/메커니즘

### 1. 락에서 풀려나면 "쿼리 재실행"이 아니라 "그 row만 재평가"다

READ COMMITTED(PostgreSQL·Spring 기본)에서 `SELECT FOR UPDATE`가 잠긴 row를 만나 대기하다 풀려나면:

| 락 잡았던 트랜잭션이... | 대기하던 쪽의 동작 |
|---|---|
| 변경 없이 커밋 (락만 잡았다 풀음) | **재평가 없음** — 자기 문장 시작 시점 스냅샷 그대로 반환 |
| 그 row를 UPDATE 후 커밋 | **최신 버전으로 WHERE 재평가** — 여전히 맞으면 최신 버전 잠그고 반환, 벗어나면 결과에서 탈락 |
| 그 row를 DELETE 후 커밋 | 결과에서 제외 |
| 롤백 | 원래 버전으로 그대로 진행 |

(공식 문서: "The search condition of the command (the WHERE clause) is re-evaluated to see if the updated version of the row still matches... it is the updated version of the row that is locked and returned." — 내부 구현 이름은 EvalPlanQual)

- **row 단위 재평가**지 전체 쿼리 재실행이 아니다. 그래서 반대 방향은 없다 — 상대 커밋으로 **새로 조건에 맞게 된 row가 결과에 추가되지는 않는다.** 이미 후보였던 row의 유지/탈락만 결정.
- 같은 트랜잭션의 **다음 문장들**은 READ COMMITTED에선 어차피 새 스냅샷이므로 상대 커밋을 본다 (count 검증 등은 락 획득 후 새 문장으로 하면 정확).

### 2. 잠근 row ≠ 바뀌는 row → 락을 걸고도 뚫린다 (실제로 막힌 케이스)

쿠폰(부모) — 발급코드(자식) 구조에서, 코드 등록 로직이 부모만 잠갔던 사례:

```sql
SELECT ... FROM coupon c JOIN coupon_code cc ON ...
WHERE cc.code = 'ABC123' AND cc.used_yn = 'N'
FOR UPDATE OF c        -- 부모만 잠금. 그런데 사용 처리는 cc.used_yn을 바꾼다
```

서로 다른 두 사용자가 같은 코드로 동시 요청 시:

1. 트랜잭션 A: 부모 락 획득 → 사용 처리(`cc.used_yn='Y'`) → 커밋
2. 트랜잭션 B: 부모 락 대기 → 락 획득. 그런데 **잠긴 row(부모)는 변경된 적이 없으므로 재평가가 아예 일어나지 않는다** (위 표의 1행) → 자식의 `used_yn`은 옛 스냅샷 값 `'N'`으로 보임 → 검증 통과 → **같은 코드 2회 사용**

순차 요청은 완벽히 막히고 **동시 요청만 뚫리는** 종류의 버그라 테스트로 잡기 어렵다. 수정은 상태가 바뀌는 자식 테이블을 락 범위에 포함: `FOR UPDATE OF c, cc`. 이러면 B는 A가 변경한 자식 row에서 재평가(위 표 2행)를 타고 `used_yn='N'` 조건 탈락 → 조회 결과 없음으로 자연스럽게 거절된다.

### 3. OF 없는 `FOR UPDATE`는 FROM의 모든 테이블을 잠근다

JOIN에 공통코드 테이블 같은 **공유 조회 테이블**이 끼어 있으면 그 row까지 잠겨서, 서로 무관한 트랜잭션들이 공통코드 row 하나를 놓고 경합하게 된다. 잠글 필요가 있는 테이블만 `OF`로 지정.

### 4. outer join의 nullable 쪽은 잠글 수 없다

`FOR UPDATE cannot be applied to the nullable side of an outer join` 에러. `OF` 없이 쓰던 쿼리의 JOIN 하나를 LEFT JOIN으로 바꾸는 순간 깨진다 — `OF`로 범위를 명시해 두면 영향 없음. 집계(GROUP BY 등) 쿼리에도 못 쓴다 (결과 row가 테이블 row와 1:1 대응이 안 되므로).

### 5. 격리 수준이 올라가면 재평가 대신 에러

REPEATABLE READ/SERIALIZABLE에서는 잠긴 row가 **변경된 채** 커밋되면 재평가 대신 `ERROR: could not serialize access due to concurrent update` — 애플리케이션이 재시도해야 한다 (락만 잡았다 풀린 경우는 그대로 진행). 격리 수준별 스냅샷 차이는 [트랜잭션 격리 수준](../java/jpa/transaction-isolation.md).

### 6. 기타

- row 락은 **쓰기/락 시도만** 차단한다. 락 없는 일반 SELECT는 전혀 안 막힘 (MVCC).
- 락 해제는 커밋/롤백 시점. 락 잡은 채 외부 API 호출 같은 긴 작업을 하면 경쟁자 전원이 그만큼 대기한다.
- 여러 row를 잠글 때 트랜잭션마다 잠그는 순서가 다르면 데드락 — [데드락](../java/concurrency/deadlock.md).

## 💡 판단 기준

- **"무엇을 잠글까"는 "어느 row의 상태가 바뀌는가"로 정한다.** 검증에 참조만 하는 row가 아니라 이번 트랜잭션이 **변경하는** row가 잠겨 있어야, 대기자가 풀려날 때 재평가가 작동한다. 부모만 잠그고 자식 플래그를 바꿨다가 동시 요청이 통과한 케이스(함정 2) → 그래서 `FOR UPDATE OF 부모, 자식`.
- **JOIN이 있으면 `OF`를 항상 명시한다.** 락 범위가 코드에 문서화되고, 나중에 JOIN 추가(공유 테이블 경합)·LEFT JOIN 변경(에러)에 대한 안전장치가 된다.
- 단일 row의 플래그 선점이 전부라면 락 없이 `UPDATE ... SET used_yn='Y' WHERE id=? AND used_yn='N'` + affected rows 확인(원자적 선점)이 더 가볍다. `FOR UPDATE`는 **여러 문장에 걸친 검증·계산이 락 안에서 필요할 때** 꺼내는 도구. DB 밖 자원까지 직렬화해야 하면 [분산 락](../infra/redis/redisson-distributed-lock.md).

## 참고

- [PostgreSQL 공식: Transaction Isolation (13.2)](https://www.postgresql.org/docs/current/transaction-iso.html) — READ COMMITTED 재평가 동작
- [PostgreSQL 공식: SELECT — The Locking Clause](https://www.postgresql.org/docs/current/sql-select.html) — `OF`·NOWAIT·SKIP LOCKED 문법
- [PostgreSQL 공식: Explicit Locking (13.3.2 Row-Level Locks)](https://www.postgresql.org/docs/current/explicit-locking.html) — 락 강도 4종·충돌 표

학습 날짜: 2026-08-06 · 계기: 백오피스 API의 쿠폰 코드 등록 동시성 리뷰 중, 부모 테이블만 잠그는 `FOR UPDATE OF`가 자식 row 상태 변경을 못 막는 문제를 분석하며 락 범위와 READ COMMITTED 재평가 동작을 학습

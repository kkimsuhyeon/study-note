# LATERAL 조인과 top-N per group

> - `LATERAL` = FROM 절 서브쿼리가 **앞선 테이블의 현재 행을 참조**하게 해주는 키워드 (SQL 안의 for-each 루프)
> - 대표 용도는 **그룹마다 최신 1건의 "다른 컬럼"** 붙이기 — `GROUP BY + MAX`로는 못 푸는 문제
> - 형태: `LEFT JOIN LATERAL (... ORDER BY ts DESC LIMIT 1) ON TRUE`

📖 **읽는 경로 — 다 읽지 않아도 된다**

| 상황 | 읽을 곳 |
|---|---|
| 처음이라 뭔지 모르겠다 | **1, 2번만** (나머지는 나중에) |
| "루프라던데 느린 거 아냐?" | 3번 |
| 왜 이런 문법이 필요한지 원리가 궁금 | 4번 |
| 비슷한 도구 중에 뭘 쓸지 고민 | 6번 비교표 |
| 썼는데 결과가 이상하다 | 7번 |

---

## 1. 언제 쓰나

- **그룹마다 최신 1건의 (다른 컬럼)이 필요할 때** — 채팅방마다 마지막 메시지, 주문마다 최근 상태 이력, 사용자마다 최근 로그인 기록. (카카오톡 채팅 목록에서 방마다 "마지막 메시지 한 줄"이 보이는 구조가 정확히 이것)
- 앱 코드로 짜면 **N+1 루프**(목록 1번 + 행마다 1번)가 되고 싶어지는 조회를 **쿼리 한 방**으로 밀어넣을 때
- 바깥 행의 값을 인자로 **집합 반환 함수**나 복잡한 계산 서브쿼리를 호출할 때

## 2. 기본형 — 이 모양만 알면 된다

```sql
-- 채팅방 목록 + 방마다 "가장 최근 질문"의 mode
SELECT  cr.chat_room_id,
        cr.chat_room_name,
        q.mode
FROM    chat_room cr
LEFT JOIN LATERAL (
    SELECT  q.mode
    FROM    user_question q
    WHERE   q.chat_room_id = cr.chat_room_id   -- ★ 바깥 테이블(cr) 참조 — LATERAL이 허용
    ORDER BY q.reg_dtm DESC
    LIMIT 1                                    -- ★ 방당 정확히 1줄 보장
) q ON TRUE                                    -- 조인 조건은 이미 서브쿼리 WHERE에 있음
WHERE   cr.usr_id = :usrId
ORDER BY cr.reg_dtm DESC;
```

실행 모델은 루프다: `cr`의 각 행에 대해 → 서브쿼리 실행 → 결과를 옆에 붙임. 질문이 0건인 방은 `LEFT`라서 살아남고 `mode = NULL`.

| 요소 | 역할 |
|---|---|
| `LATERAL` | 서브쿼리에서 앞선 테이블 컬럼 참조 허용 (없으면 문법 에러) |
| `LEFT ... ON TRUE` | 매칭 0건이어도 바깥 행 유지. 조건은 서브쿼리 안에 있으니 `ON TRUE`가 관용구 |
| `ORDER BY ... LIMIT 1` | "최신 1건" — 이게 없으면 질문 수만큼 방이 **중복**돼서 나옴 |

**왜 `GROUP BY + MAX`로는 안 되나** — `MAX(reg_dtm)`은 최신 *시각*을 줄 뿐, "그 행의 mode"를 못 집는다. `MAX(mode)`는 그냥 알파벳순 최대값이고. **"그룹마다 최신 행의 컬럼"이 보이면 집계가 아니라 top-1-per-group 문제**이고, 그게 이 도구가 존재하는 이유다.

## 3. 루프라던데, N+1만큼 느린 거 아닌가?

애플리케이션 N+1과 비교하면 LATERAL이 하는 일은 개념적으로 아래 루프와 **완전히 같다**. 차이는 루프가 도는 위치뿐:

```java
List<ChatRoom> rooms = mapper.selectAll(usrId);          // 1
for (ChatRoom room : rooms) {
    mapper.selectLatestMode(room.getChatRoomId());       // + N (방마다 왕복)
}
```

**반복 횟수는 같고, 반복 1회의 비용이 다르다.** N+1이 느린 진짜 원인은 "쿼리를 N번 해서"가 아니라 **한 번마다 프로세스 경계를 넘는 것**이기 때문:

| | 반복 1회에 드는 일 | 자릿수 (환경마다 다름) |
|---|---|---|
| 앱 N+1 | 커넥션 획득 → 네트워크 왕복 → SQL 파싱 → 실행계획 수립 → 결과 직렬화 → JDBC/ORM 매핑 | **0.5~50 ms** (같은 DC ~1ms, 원격/클라우드면 수십 ms) |
| LATERAL | 이미 세워진 실행계획 안에서 인덱스 탐색 1회 | **수 µs** (shared buffer 히트 기준) |

500건 목록이면 대략 `500 × 1ms ≈ 500ms` 대 `왕복 1번 + 500 × 5µs ≈ 3ms`. 같은 "루프"라도 하나는 **프로세스 간 통신 500번**, 다른 하나는 **함수 호출 500번**이라 비교 대상이 아니다. 게다가 옵티마이저가 상황에 따라 nested loop 대신 다른 조인 전략을 고르기도 해서 "무조건 루프"도 아니다.

**EXPLAIN으로 확인하기** — `EXPLAIN (ANALYZE, BUFFERS)`를 찍고 `loops`를 본다:

```
Nested Loop Left Join            (actual time=0.020..3.120 rows=500 loops=1)
  ->  Seq Scan on chat_room cr   (actual time=0.008..0.120 rows=500 loops=1)
  ->  Limit                      (actual time=0.005..0.005 rows=1 loops=500)
        ->  Index Scan Backward using idx_q_room_dtm on user_question q
                                 (actual time=0.004..0.004 rows=1 loops=500)
              Index Cond: (chat_room_id = cr.chat_room_id)
```

- `loops=500` = 바깥 행마다 반복한 횟수
- ⚠️ 그 옆 `actual time`은 **총합이 아니라 1회 평균**이다. 실제 소요는 `actual time × loops` (EXPLAIN 읽을 때 가장 흔한 오해)
- `Index Scan`이 아니라 **`Seq Scan`이 `loops=500`과 함께** 뜨면 위험 신호 — 인덱스가 없어서 바깥 행마다 자식 테이블을 통째로 훑고 있다는 뜻

즉 LATERAL이 빠른 건 공짜가 아니라 **상관 조건+정렬 인덱스가 있다는 전제** 위에서다. 인덱스가 없거나 바깥 행이 수만 건인 배치성 쿼리라면 오히려 윈도우 함수가 낫다 (7번 참고).

JPA 쪽 N+1과 fetch 전략은 → [n-plus-one-fetch](../java/jpa/n-plus-one-fetch.md)

## 4. 서브쿼리에서 바깥 테이블을 왜 못 쓰지?

"서브쿼리는 바깥을 못 본다"가 아니라 **FROM 절 서브쿼리만 못 본다.** 같은 상관 참조를 SELECT 절에 쓰면 LATERAL 없이도 된다:

```sql
-- ❌ FROM 절: ERROR - column "o.org_id" does not exist
FROM   org o
LEFT JOIN (SELECT code FROM history WHERE org_id = o.org_id ORDER BY dtm DESC LIMIT 1) h ON TRUE

-- ✅ SELECT 절 스칼라 서브쿼리: 바깥 참조 OK
SELECT o.org_id,
       (SELECT h.code FROM history h WHERE h.org_id = o.org_id ORDER BY h.dtm DESC LIMIT 1) AS code
FROM   org o
```

이유는 절이 평가되는 순서다:

| 단계 | 절 | 하는 일 |
|---|---|---|
| 1 | **FROM (JOIN 포함)** | 테이블을 결합해 "작업 대상 행 집합"을 만든다 |
| 2 | WHERE | 행 필터 (집계 전) |
| 3 | GROUP BY | 그룹으로 접기 |
| 4 | HAVING | 그룹 필터 |
| 5 | SELECT | 표현식·별칭 계산 |
| 6 | DISTINCT | 중복 제거 |
| 7 | ORDER BY | 정렬 (여기선 SELECT 별칭 사용 가능) |
| 8 | LIMIT / OFFSET | 자르기 |

- FROM은 가장 먼저 평가돼 "어떤 행들이 있는지"를 만드는 단계 — 이 시점엔 `o`의 현재 행이라는 개념 자체가 없다. 그래서 참조 불가이고, **LATERAL이 "이 서브쿼리는 앞 항목의 행마다 다시 실행하라"고 예외를 여는 것**
- SELECT는 행이 하나씩 확정된 뒤 계산 — 그래서 그 행의 컬럼을 자유롭게 쓴다

**같은 규칙에서 나오는 실무 결과 두 가지**

- **WHERE에서는 SELECT 별칭을 못 쓴다** (`WHERE ttl_point > 0` 불가) — WHERE가 SELECT보다 먼저이기 때문. 반면 ORDER BY는 SELECT 뒤라 별칭 사용 가능
- 스칼라 서브쿼리는 **반드시 1행 1열**. 2행 이상이면 런타임 에러(`more than one row returned by a subquery used as an expression`) → `LIMIT 1` 필수. 컬럼이 2개 필요하면 서브쿼리를 2벌 써야 하고(같은 테이블 2회 스캔), 그 지점이 LATERAL로 갈아탈 신호

### JOIN은 어느 단계에서 처리되나?

JOIN은 별도 단계가 아니라 **FROM 단계 안**에서 처리된다. 그 안에도 순서가 있다:

```
a. 두 테이블의 곱(cartesian product)
b. ON 조건으로 걸러냄
c. OUTER JOIN이면 → b에서 탈락한 "보존 측" 행을 NULL로 채워 되살림   ← ★
d. JOIN이 더 있으면 c의 결과를 왼쪽 항목 삼아 a~c 반복
```

여기서 나오는 두 가지 결론:

- **`c` 단계의 존재가 `ON`과 `WHERE`의 차이를 만든다.** `ON`의 조건은 b에서 걸러도 c가 부모를 되살리지만, `WHERE`는 c가 끝난 뒤(2단계)라 NULL 행을 그대로 탈락시켜 LEFT JOIN이 사실상 INNER가 된다 → [left-join-on-vs-where](./left-join-on-vs-where.md)
- **`d`에서 "왼쪽부터 순서대로" 처리하기 때문에 LATERAL이 성립한다.** 오른쪽 항목 차례가 왔을 때 왼쪽은 이미 확정돼 있으므로 그 행을 참조할 수 있다

⚠️ **"곱을 먼저 만든다"는 논리적 정의일 뿐, 실제 실행이 아니다.** 표준은 `A JOIN B ON c` ≡ `(A × B) WHERE c`로 의미를 정의하지만, 엔진이 진짜 곱을 만들면 1만 × 1만 = 1억 행이 된다. 옵티마이저는 **Hash Join / Merge Join / Nested Loop** 중에서 골라 곱을 만들지 않고 같은 결과를 낸다. 위 순서표 전체가 마찬가지로 "결과가 어떻게 정의되는가"이지 실행 계획이 아니다 — `EXPLAIN`이 전혀 다른 순서를 보여주는 게 정상이고, WHERE 조건이 조인 전으로 내려가는 predicate pushdown이 대표적이다. (반대로 **조인 조건을 진짜 빠뜨리면** 옵티마이저도 방법이 없어 실제 곱이 발생한다 — 행 수 폭발의 전형적 원인)

## 5. top-N 말고 또 어디에 쓰나

**① 여러 값을 한 번의 스캔으로** — 스칼라 서브쿼리는 컬럼당 1벌씩 필요하지만 LATERAL은 한 번에:

```sql
LEFT JOIN LATERAL (
    SELECT COUNT(*) AS cnt, MAX(h.use_dtm) AS last_dt
    FROM   history h
    WHERE  h.org_id = o.org_id
) s ON TRUE
```

(집계 함수는 매칭 0건이어도 항상 1행을 돌려주므로 `cnt = 0`, `last_dt = NULL`이 된다)

**② 계산식에 이름 붙여 재사용** — 위의 "WHERE에서 별칭 못 씀" 문제의 해법:

```sql
-- 같은 식을 SELECT·WHERE·ORDER BY에 세 번 반복하는 대신
CROSS JOIN LATERAL (SELECT COALESCE(p.paid,0) + COALESCE(p.free,0) AS ttl) t
WHERE  t.ttl > 10000
ORDER  BY t.ttl DESC
```

FROM 단계에서 만들어진 컬럼이라 WHERE에서도 보인다. 항상 1행이므로 `CROSS JOIN LATERAL`.

**③ 집합 반환 함수로 행 펼치기** — 배열/JSON 1행을 원소 수만큼의 행으로:

```sql
CROSS JOIN LATERAL unnest(o.tags) AS t(tag)
CROSS JOIN LATERAL jsonb_array_elements(l.payload -> 'items') AS e
```

## 6. LATERAL·윈도우 함수·스칼라·DISTINCT ON — 뭘 골라야 하나

| 방법 | 형태 | 특징 |
|---|---|---|
| **LATERAL** (표준) | `LEFT JOIN LATERAL (... LIMIT n) ON TRUE` | 바깥 행마다 인덱스 top-n 집기. 여러 컬럼 한 번에. **바깥 행이 적을 때 최강** |
| 윈도우 함수 | `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...) = 1` | 서브쿼리가 독립 실행 → **파티션 대상 전체를 스캔·정렬** 후 걸러냄. 전 그룹을 다 필요로 할 때 유리 |
| 스칼라 서브쿼리 | SELECT 절 `(SELECT ... LIMIT 1)` | 실행 모델은 LATERAL과 유사하나 **컬럼 1개만** 반환 가능. 컬럼 늘면 서브쿼리도 개수만큼 반복 |
| `DISTINCT ON` | `SELECT DISTINCT ON (grp) ... ORDER BY grp, ts DESC` | PostgreSQL 전용. 문법 가장 짧지만 top-**1**만 되고 이식성 없음 |

### DISTINCT ON 자세히 (PostgreSQL 전용)

"지정한 식이 같은 행들의 묶음에서 **첫 행만** 남긴다". 그 "첫 행"을 정하는 건 `ORDER BY`다:

```sql
SELECT DISTINCT ON (org_id)
       org_id, promotion_cd, use_dtm
FROM   promotion_history
ORDER  BY org_id, use_dtm DESC;   -- org_id마다 use_dtm 최대인 1행
```

규칙과 함정:

- **`DISTINCT ON` 식은 `ORDER BY`의 맨 앞과 일치해야 한다** (공식 규정). 뒤에 오는 정렬식이 "그룹 안에서 어느 행이 첫 행인가"를 결정
- **`ORDER BY`를 빼면 어느 행이 남을지 예측 불가** — 문서가 명시하는 unpredictable. 에러가 아니라 **조용히 아무 행**이라 위험. (실무 예: `ORDER BY org_id, use_dtm`처럼 `DESC`를 빼먹으면 최신이 아니라 **가장 오래된** 이력이 뽑힌다 — 의도와 반대인데 결과는 그럴듯해서 눈치채기 어렵다)
- 최종 결과를 다른 기준으로 정렬하려면 서브쿼리로 감싸고 바깥에서 다시 `ORDER BY`
- `DISTINCT`(및 `DISTINCT ON`)는 `FOR UPDATE`/`FOR SHARE` 등 잠금 절과 **함께 쓸 수 없다** → 락이 필요하면 [select-for-update](./select-for-update.md) 쪽 방식으로 분리

윈도우 함수와의 차이: `ROW_NUMBER() OVER (PARTITION BY grp ORDER BY ts DESC)` + `WHERE rn = 1`은 **표준 SQL이고 top-N(N>1)도 되지만**, 서브쿼리로 한 겹 감싸야 한다(윈도우 함수는 WHERE에서 직접 못 씀 — 4번의 절 평가 순서 때문). `DISTINCT ON`은 짧지만 top-1 전용 + PostgreSQL 전용.

### 방언/버전

- SQL 표준(SQL:1999)의 lateral derived table. **PostgreSQL 9.3+**, **MySQL 8.0.14+** 지원
- SQL Server는 같은 개념을 `CROSS APPLY`(≒ `CROSS JOIN LATERAL`) / `OUTER APPLY`(≒ `LEFT JOIN LATERAL ... ON TRUE`)로 씀. Oracle 12c+는 LATERAL과 APPLY 둘 다 지원
- `CROSS JOIN LATERAL`은 INNER 성격 — 서브쿼리 결과 0건이면 바깥 행도 **탈락**. "없어도 목록에 나와야 하면" 반드시 `LEFT ... ON TRUE`
- PostgreSQL에서 FROM 절의 집합 반환 함수(`generate_series` 등)는 LATERAL 키워드 없이도 암묵적으로 lateral 취급

## 7. ⚠️ 결과가 이상할 때 — 자주 하는 실수

- **`LIMIT 1` 빠뜨리면 조용히 중복**: 에러가 아니라 그룹의 자식 수만큼 바깥 행이 늘어난 결과가 나온다 (목록 화면에 같은 방이 여러 줄)
- **동점(tie) 주의**: `ORDER BY reg_dtm DESC LIMIT 1`에서 동일 시각 2건이면 어느 쪽이 뽑힐지 비결정적. 재현 가능해야 하면 `ORDER BY reg_dtm DESC, id DESC`처럼 유니크 타이브레이커를 붙인다
- **상관 조건을 ON으로 빼면 의미가 달라진다**: `ON TRUE` 대신 `ON q.grp_id = 부모.grp_id`로 옮기면, 서브쿼리가 먼저 **전체에서 LIMIT 1**을 자른 뒤에 ON이 검사됨 → 전체 최신 1건이 속한 그룹만 값이 붙고 나머지는 전부 NULL. 조건이 서브쿼리 **안**에 있어야 "그룹 안에서 정렬→LIMIT"이 된다 (빼는 순간 바깥 참조가 사라져 LATERAL 의미 자체가 소멸). `ON TRUE`는 "조건은 이미 안에 있고, LEFT JOIN 문법이 요구하는 ON 자리만 채운다"는 관용구
- **실행 = nested loop → 상관 조건+정렬 인덱스 필수**: `(chat_room_id, reg_dtm)`(상관 컬럼들 + 정렬 컬럼) 복합 인덱스가 있으면 반복 1회가 "인덱스 끝 1건 집기"로 끝나지만, 없으면 바깥 행마다 자식 테이블을 스캔한다 — 바깥이 클수록 비용이 곱으로 늘어남
- **반대 케이스도 있다**: "전 테이블의 모든 그룹에 대해 top-1"처럼 결국 자식 테이블 대부분을 읽어야 하는 배치성 쿼리라면, 인덱스 probe를 그룹 수만큼 하는 LATERAL보다 **한 번 스캔하는 윈도우 함수가 더 빠를 수 있다**. LATERAL의 이점은 "바깥에서 이미 걸러진 소수 행"이 전제

## 💡 판단 기준

- 채팅방 목록에 방별 최신 질문 mode를 붙여야 했다 → `MAX()`로는 최신 행의 다른 컬럼을 못 뽑는다는 걸 확인 → **"그룹마다 최신 행의 컬럼"이 보이면 집계가 아니라 top-1-per-group 도구(LATERAL)를 꺼낸다**
- **앱에서 루프 돌며 건별 조회(N+1)를 짜고 싶어지는 순간이 LATERAL의 신호**다 — 그 루프를 쿼리 안으로 밀어넣는 도구라고 기억하면 언제 쓸지 헷갈리지 않는다
- 선택 공식: 바깥 행이 적고(WHERE로 걸러진 목록) 인덱스가 있다 → LATERAL / 전 그룹 대상 배치 집계 → 윈도우 함수 / 필요한 게 컬럼 딱 1개 → 스칼라 서브쿼리도 충분

### 행의 단위 공식 — 부모-자식 조회 도구 선택

LATERAL이 N+1 만능 해법인가 헷갈렸는데, **"결과 한 줄이 무엇을 나타내는가"**부터 정하면 도구가 저절로 갈린다:

| 결과 한 줄의 단위 | 붙이려는 값 | 도구 | 예시 |
|---|---|---|---|
| 부모 (방당 1줄) | 자식의 **집계값** (개수·합계·최신 시각 자체) | GROUP BY + 집계함수 | 방 목록 + 질문 개수 |
| 부모 (방당 1줄) | **특정 자식 행의 컬럼** (최신 행의 mode) | **LATERAL** | 방 목록 + 마지막 질문 mode |
| 자식 (질문당 1줄) | 부모 정보를 각 줄에 | 일반 JOIN | 질문 전체 목록 (방 이름 포함) — 부모 반복이 뻥튀기가 아니라 원하는 모양 |
| 중첩 (부모 안에 자식 리스트) | — | **IN 배치 + groupingBy** (쿼리 2번) | 방 상세 API `{방, questions:[...]}` — JOIN하면 부모가 자식 수만큼 반복돼 다시 접어야 함 |

- IN 배치 = 목록 1번 + `WHERE 자식FK IN (부모ID들)` 1번 → 앱에서 `Collectors.groupingBy`로 조립. N이 얼마든 쿼리 2번. JPA `@BatchSize`가 내부적으로 하는 일이 정확히 이 패턴
- 쓰기 N+1(루프 UPDATE)은 조회 도구로 못 푼다 → `foreach`로 VALUES 묶는 배치 UPDATE / JDBC batch가 해법. DB 밖 N+1(외부 API 루프)도 SQL 밖(배치 API·캐시)에서 푼다

## 참고

- PostgreSQL 공식 문서 — Table Expressions, LATERAL Subqueries: https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-LATERAL
- PostgreSQL SELECT 문서 (DISTINCT ON): https://www.postgresql.org/docs/current/sql-select.html
- MySQL 8.0 Lateral Derived Tables: https://dev.mysql.com/doc/refman/8.0/en/lateral-derived-tables.html
- 학습 날짜: 2026-08-03 (2026-08-10 보강)
- 계기: 실무 MyBatis 매퍼에서 채팅방 목록 쿼리의 `LEFT JOIN LATERAL (... ORDER BY reg_dtm DESC LIMIT 1) ON TRUE`를 만나 "이게 무슨 조인인지"부터 GROUP BY로 안 되는 이유, N+1과의 관계까지 따라가며 정리

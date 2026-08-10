# SQL의 NULL 비교와 IS DISTINCT FROM

> **한 줄 요약**: SQL의 비교 연산자는 한쪽이 NULL이면 true/false가 아니라 **NULL(unknown)** 을 돌려주고, WHERE·JOIN·CHECK는 unknown을 "통과 못 함"으로 처리한다. 그래서 `x != 'A'`는 x가 NULL일 때 **행을 탈락시킨다.** `IS DISTINCT FROM`은 NULL을 평범한 값처럼 취급해 비교하는 NULL 안전 연산자 — "이 조건에 NULL 행도 포함돼야 하나?"를 항상 따로 물어야 한다.

## 언제 쓰나

- **NULL 가능 컬럼을 부등 비교**할 때 (`!=`, `<>`) — 가장 흔한 사고 지점
- 조건부 JOIN: "채널이 A면 이 테이블, 아니면 저 테이블"처럼 값으로 갈래를 나눌 때
- 변경 감지: "이전 값과 달라졌는가"를 NULL→값, 값→NULL 전환까지 포함해 판정할 때 (`old IS DISTINCT FROM new`)

## 사용 예시

```sql
-- 일반 비교: NULL이 끼면 결과가 unknown
SELECT NULL = NULL,        -- NULL (같다고 하지 않음!)
       7    = NULL,        -- NULL
       7   <> NULL;        -- NULL

-- NULL 안전 비교
SELECT NULL IS DISTINCT FROM NULL,       -- false (둘 다 NULL = 같다)
       NULL IS DISTINCT FROM 'COMPAS',   -- true  (한쪽만 NULL = 다르다)
       NULL IS NOT DISTINCT FROM NULL;   -- true  (NULL-safe 같음)
```

공식 문서 요약: 일반 비교 연산자는 입력이 NULL이면 null(“unknown”)을 반환한다. `IS DISTINCT FROM`은 **non-null 입력에 대해서는 `<>`와 동일**하고, 둘 다 NULL이면 false, 한쪽만 NULL이면 true를 반환한다 — 즉 **NULL을 "unknown"이 아니라 하나의 정상 값처럼** 다룬다.

진리표로 보면 차이가 분명하다:

| a | b | `a = b` | `a <> b` | `a IS DISTINCT FROM b` | `a IS NOT DISTINCT FROM b` |
|---|---|---|---|---|---|
| 'X' | 'X' | true | false | false | true |
| 'X' | 'Y' | false | true | true | false |
| 'X' | NULL | **NULL** | **NULL** | true | false |
| NULL | NULL | **NULL** | **NULL** | false | true |

## 종류/옵션 비교

| 목적 | 쓸 것 | 비고 |
|---|---|---|
| NULL 여부 확인 | `IS NULL` / `IS NOT NULL` | `= NULL`은 **항상 unknown** — 절대 쓰지 않는다 |
| NULL 안전 "같음" | `IS NOT DISTINCT FROM` | MySQL은 `<=>` 연산자 |
| NULL 안전 "다름" | `IS DISTINCT FROM` | MySQL은 `NOT (a <=> b)` |
| NULL을 기본값으로 치환 | `COALESCE(a, '')` | 인덱스를 못 타게 되는 점 주의 |
| 여러 컬럼 중 첫 non-null | `COALESCE(a, b, c)` | 이종 테이블 병합에 자주 씀 (아래) |

- `!=`는 `<>`의 별칭이다 — 파싱 아주 초기에 `<>`로 변환되므로 동작이 완전히 같다 (둘을 다르게 구현하는 것 자체가 불가능)
- MyBatis XML에서는 `<`가 특수문자라 `<>` 대신 `!=`를 쓰거나 `&lt;&gt;`/CDATA로 escape해야 한다

## ⚠️ 함정/메커니즘

### 1. JOIN 조건의 부등 비교가 행을 조용히 날린다 (실무 사고)

두 종류의 기관 테이블(일반 `org`, 컴패스 `cps_org`)을 이력의 채널값으로 갈라 붙이는 쿼리:

```sql
-- ❌ frnc_id가 NULL인 과거 이력은 양쪽 다 매칭 실패 → 기관 정보가 전부 빈 행
LEFT JOIN org     o ON o.org_id = h.org_id AND h.frnc_id != 'COMPAS'   -- NULL → unknown → 매칭 X
LEFT JOIN cps_org c ON c.org_id = h.org_id AND h.frnc_id  = 'COMPAS'   -- NULL → unknown → 매칭 X

-- ✅ NULL을 "COMPAS가 아닌 것"으로 분류
LEFT JOIN org     o ON o.org_id = h.org_id AND h.frnc_id IS DISTINCT FROM 'COMPAS'
LEFT JOIN cps_org c ON c.org_id = h.org_id AND h.frnc_id = 'COMPAS'
```

INNER JOIN이었다면 행 자체가 사라지고, LEFT JOIN이면 행은 남되 컬럼이 전부 NULL이 된다. **둘 다 에러가 아니라서 "데이터가 없네?"로 오해하기 쉽다.**

### 2. `NOT IN (서브쿼리)`에 NULL이 하나라도 있으면 결과가 0건

```sql
-- 서브쿼리 결과에 NULL이 섞이면 전체가 unknown → 한 행도 안 나옴
WHERE org_id NOT IN (SELECT org_id FROM blacklist)   -- blacklist.org_id에 NULL 1개면 끝
```

`x NOT IN (a, b, NULL)`은 `x != a AND x != b AND x != NULL`이고, 마지막 항이 unknown이라 전체가 절대 true가 될 수 없다. 해법은 `NOT EXISTS` 사용(권장) 또는 서브쿼리에 `WHERE col IS NOT NULL` 추가. 참고로 `IN`(긍정형)은 이 문제가 없다 — OR 조합이라 하나만 true면 통과.

### 3. 집계·정렬에서의 NULL

- `COUNT(*)`는 모든 행을 세지만 `COUNT(col)`은 **NULL을 제외**한다. `SUM`/`AVG`도 NULL 무시 (`AVG`는 분모에서도 빠짐)
- `ORDER BY`에서 PostgreSQL 기본은 **ASC면 NULL이 마지막, DESC면 NULL이 먼저** — 필요하면 `NULLS FIRST/LAST` 명시
- `GROUP BY`는 NULL들을 **하나의 그룹**으로 묶는다 (비교와 반대로 동작하니 헷갈리기 쉬운 지점)

### 4. UNIQUE 제약은 NULL을 중복으로 보지 않는다

`UNIQUE` 컬럼에 NULL은 여러 행 들어갈 수 있다 — "unknown끼리는 같다고 판정하지 않음"의 연장. PostgreSQL 15+는 `UNIQUE NULLS NOT DISTINCT`로 이 동작을 뒤집을 수 있다.

### 5. 인덱스 관점

`IS DISTINCT FROM`은 일반 비교와 달리 **B-tree 인덱스를 잘 못 탄다**(PostgreSQL이 인덱스 조건으로 잘 변환하지 못함). JOIN 조건 보조로 쓰는 정도는 괜찮지만, 대량 테이블의 주 필터로 쓴다면 실행 계획을 확인하고 `(col IS NULL OR col <> 'X')` 형태나 부분 인덱스를 검토한다.

## 실무 패턴 — 이종 테이블 두 개를 COALESCE로 병합

같은 의미의 데이터가 테이블 두 곳에 나뉘어 있고 **컬럼명까지 다를 때**(예: `reg_dt` vs `reg_dtm`, `usr_eml` vs `mngr_mail`), 조건부 LEFT JOIN 2개 + `COALESCE`로 한 줄에 합칠 수 있다:

```sql
SELECT COALESCE(o.org_nm,  c.org_nm)          AS org_nm,
       COALESCE(o.reg_dt,  c.reg_dtm::DATE)   AS join_dt,   -- 타입까지 맞춰줘야 함
       COALESCE(o.usr_eml, c.mngr_mail)       AS usr_eml
FROM   history h
LEFT   JOIN org     o ON o.org_id = h.org_id AND h.frnc_id IS DISTINCT FROM 'COMPAS'
LEFT   JOIN cps_org c ON c.org_id = h.org_id AND h.frnc_id = 'COMPAS'
WHERE  ...
  AND  COALESCE(o.org_nm, c.org_nm) ILIKE '%' || :srchTxt || '%'   -- 검색 조건도 같이 감싸야 함
```

- 장점: 쿼리 하나로 끝나고, 채널이 늘면 LEFT JOIN + COALESCE 인자만 추가
- 대가: **검색·정렬 조건도 전부 `COALESCE`로 감싸야 하고 그 순간 인덱스를 못 탄다.** 상위 조건(위 예의 `promotion_id`)으로 이미 좁혀진 뒤라면 실무상 무해하지만, 전체 스캔에 얹으면 위험
- 대안: `UNION ALL`로 두 테이블을 공통 스키마로 정규화한 서브쿼리, 또는 MyBatis `<choose>`로 쿼리를 통째 분기(대상 채널이 파라미터로 확정될 때 가장 빠름)

## 💡 판단 기준

- **"이 컬럼이 NULL일 수 있나?"를 부등 비교(`!=`, `NOT IN`)를 쓸 때마다 묻는다.** 가능성이 있으면 `IS DISTINCT FROM` / `NOT EXISTS`로 바꾼다 — NULL이 섞였을 때 **에러가 아니라 "행이 조용히 사라지는" 형태**로 터지기 때문에 테스트 데이터로는 잘 안 잡힌다
- **컬럼이 NOT NULL로 보장돼 있으면 `!=`로 충분하다.** `IS DISTINCT FROM`을 습관적으로 쓰면 인덱스 활용만 나빠진다 — "NULL이 실제로 가능한가"가 갈림길
- 값 하나 채우는 정도는 `COALESCE`, **행을 살릴지 말지가 걸린 조건이면 `IS DISTINCT FROM`.** 조건부 LEFT JOIN 2개로 이종 테이블을 가를 때는 "어느 쪽에도 안 붙는 값(NULL)"을 어디로 보낼지 반드시 정하고 쓴다

## 참고

- [PostgreSQL: Comparison Functions and Operators (9.2)](https://www.postgresql.org/docs/current/functions-comparison.html) — `IS DISTINCT FROM` 정의, unknown 반환 규칙, `!=`는 `<>` 별칭
- [PostgreSQL: CREATE TABLE — UNIQUE NULLS NOT DISTINCT](https://www.postgresql.org/docs/current/sql-createtable.html)
- 관련 노트: [LATERAL 조인과 top-N per group](./lateral-join-top-n-per-group.md) · [LEFT JOIN 자식 조건 ON vs WHERE](./left-join-on-vs-where.md) · [SELECT FOR UPDATE](./select-for-update.md)

학습 날짜: 2026-08-10 · 계기: 백오피스 프로모션 이력 조회에서 일반 기관/컴패스 기관 테이블을 채널값으로 갈라 LEFT JOIN하다가, `frnc_id != 'COMPAS'`가 NULL 행을 양쪽 모두에서 탈락시킨다는 걸 확인하며 정리

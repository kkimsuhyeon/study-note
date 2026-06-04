# SQL 케이스 쿡북 — "이런 상황엔 어떻게?"

> **한 줄 요약**: SQL 문법 나열이 아니라, **실제로 막혔던 상황 → 해결 쿼리 → 원리/주의** 형식으로 쌓는 노트. "문법을 몰라서"가 아니라 "이 상황을 SQL로 어떻게 표현하지?"에서 막힐 때 보는 용도.

관련 노트: (DB 주제가 늘면 여기 연결)

> 작성 틀: **상황 → 흔한 시도(안 좋은 것) → 정석 → 왜/주의 → (필요시) DB별 차이 / MyBatis**. 특정 주제(NULL·조인·윈도우 함수 등)가 커지면 별도 파일로 분리.

---

## 1. DATETIME 컬럼에서 특정 연도(기간) 데이터만 조회

**상황**: `created_at`이 `DATETIME`(날짜+시간)인데, **2024년 데이터만** 뽑고 싶다.

### ❌ 흔한 시도
```sql
SELECT * FROM orders WHERE YEAR(created_at) = 2024;   -- 함수로 연도 추출
SELECT * FROM orders WHERE created_at LIKE '2024%';   -- 문자열로 비교
```
둘 다 **결과는 맞아 보여도 실무에선 나쁜 쿼리**다.

### ✅ 정석 — 범위 조건 (반열린 구간)
```sql
SELECT * FROM orders
WHERE created_at >= '2024-01-01'
  AND created_at <  '2025-01-01';   -- 다음 해 1월 1일 "미만"
```

### 왜 ❌가 나쁜가 — 인덱스를 못 탄다
- `YEAR(created_at)`: WHERE 절에서 **컬럼을 함수로 감싸면, `created_at`에 인덱스가 있어도 못 쓴다.** DB가 모든 행에 `YEAR()`를 적용해 봐야 하므로 **풀스캔**이 된다.
- `LIKE '2024%'`: `DATETIME`을 문자열로 **암묵 변환**해서 비교 → 역시 인덱스를 못 타고, 날짜 포맷/로캘이 바뀌면 깨질 수 있다.
- 반면 범위 조건은 컬럼을 **가공하지 않으니** 인덱스를 그대로 탄다(범위 스캔).

> 📌 **핵심 교훈 (sargable)**: WHERE에서 비교 대상 **컬럼은 "맨몸"으로 두고**, 가공은 **반대쪽 값**에서 하라. 컬럼을 함수로 감싸는 순간 인덱스가 죽는다. 이렇게 인덱스를 탈 수 있는 조건을 **sargable**(Search ARGument ABLE)하다고 한다. → 날짜뿐 아니라 `SUBSTRING(name)`, `UPPER(email)` 등 **모든 컬럼 가공**에 동일하게 적용되는 원칙.

### ⚠️ BETWEEN의 함정 (DATETIME에서 특히)
```sql
-- ⚠️ 12/31 낮 시간이 누락될 수 있다
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
```
`BETWEEN`은 양끝 포함이지만, `'2024-12-31'`은 `'2024-12-31 00:00:00'`으로 해석된다. 그래서 **2024-12-31 00:00:01 ~ 23:59:59 데이터가 통째로 빠진다.**
→ 그래서 `>= 시작 AND < 다음_구간_시작` 의 **반열린 구간 `[start, end)`** 이 안전하다. (월별·일별도 같은 원리: `>= '2024-03-01' AND < '2024-04-01'`)

### "연도 `'2024'` 하나만 넘어올 때" 경계 만드는 법

입력이 `'2024'` 하나여도 문제없다.

> 📌 **핵심 재확인**: 인덱스를 죽이는 건 **"컬럼 가공"**이지 **"값 가공"**이 아니다. `created_at`(컬럼)만 맨몸으로 두면, 경계값(`#{year}` → 시작/끝)은 **자바에서 만들든 SQL에서 만들든 자유**다.

**방법 A (권장) — 자바에서 경계 계산 후 바인딩**
```java
int year = Integer.parseInt("2024");
LocalDate start = LocalDate.of(year, 1, 1);   // 2024-01-01
LocalDate end   = start.plusYears(1);         // 2025-01-01
```
```xml
<select id="findByYear" resultType="Order">
  SELECT * FROM orders
  WHERE created_at &gt;= #{start}   <!-- XML이라 < > 는 이스케이프 -->
    AND created_at &lt;  #{end}
</select>
```
DB 독립적 + 테스트 쉬움 → 가장 깔끔.

**방법 B — SQL에서 경계 생성 (PostgreSQL)**
연도만 넘기고 SQL이 경계를 만든다. `created_at`은 여전히 맨몸이라 인덱스 OK.
```sql
-- 연도가 문자열 '2024'로 올 때
WHERE created_at >= to_date(#{year}, 'YYYY')
  AND created_at <  to_date(#{year}, 'YYYY') + interval '1 year'

-- 연도가 숫자로 올 때 (make_date(년,월,일))
WHERE created_at >= make_date(#{year}::int, 1, 1)
  AND created_at <  make_date(#{year}::int + 1, 1, 1)
```

> "datetime과 `'2024-01-01'` 비교 자체가 되나?" → PostgreSQL은 문자열을 timestamp로 **자동 캐스팅**해서 `created_at >= '2024-01-01'`이 작동한다. 단 `#{}` 바인딩은 `::timestamp`/`::date`를 명시하거나 위처럼 `to_date`/`make_date`로 타입을 확실히 하는 게 안전.

> ⚠️ **MyBatis XML 함정**: `<`, `>`, `&`는 XML 특수문자라 `&lt;`, `&gt;`로 쓰거나 `<![CDATA[ ... ]]>`로 감싸야 한다. (`created_at < #{end}`를 그대로 쓰면 파싱 에러)

### 🐘 PostgreSQL 메모 (이 케이스의 실제 환경)
- 범위 조건(`>= ... AND < ...`)은 PostgreSQL에서도 **그대로 최선**. 문자열 `'2024-01-01'`은 자동으로 timestamp로 캐스팅된다.
- ⚠️ **PostgreSQL엔 `YEAR()` 함수가 없다.** MySQL식 `YEAR(created_at)`을 쓰면 *function does not exist* 에러. 연도 추출은 `EXTRACT(YEAR FROM created_at)` 또는 `date_trunc('year', created_at)`. — 단 이것들도 **컬럼 가공이라 인덱스를 못 탄다** → 결국 범위가 답.
- ⚠️ **`timestamp` vs `timestamptz` 타임존 함정**: 컬럼이 `timestamptz`(타임존 포함)면, 경계값 `'2024-01-01'`은 **세션 타임존**(`SHOW TimeZone;`) 기준으로 해석된다. 한국 시각 데이터인데 세션이 UTC면 9시간 어긋나 경계가 틀어진다. 필요하면 명시적으로 `'2024-01-01 00:00:00+09'`처럼 오프셋을 박거나 세션 타임존을 확인.
- `created_at::date`로 캐스팅해 비교하는 것도 컬럼 가공 → 인덱스 못 탐(범위가 최선).

### DB별 연도 추출 함수 (참고 — 그래도 범위가 최선)
| DB | 연도 추출(비추천) |
|----|------------------|
| MySQL | `YEAR(col)` |
| PostgreSQL | `EXTRACT(YEAR FROM col)` |
| Oracle | `EXTRACT(YEAR FROM col)`, `TO_CHAR(col,'YYYY')` |

> 어느 DB든 **결론은 동일**: 함수로 연도를 뽑지 말고 **범위 조건**으로 푼다.

---

## 참고
- [Use The Index, Luke! - WHERE 절과 인덱스](https://use-the-index-luke.com/sql/where-clause)

---

**학습 날짜**: 2026-06-04
**계기**: MyBatis + PostgreSQL 환경에서 DATETIME 컬럼의 2024년 데이터를 뽑으려다, 문자열("2024")이나 함수로 접근하는 것 외에 정석(범위 조건/인덱스)을 몰라 정리. SQL 문법보다 "상황별 해법"이 필요해 쿡북 형식으로 시작.

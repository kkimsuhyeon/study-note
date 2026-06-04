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

### MyBatis 적용
연도(또는 기간)는 **애플리케이션에서 시작/끝 값을 만들어 바인딩**하는 게 깔끔하다.
```xml
<select id="findByYear" resultType="Order">
  SELECT * FROM orders
  WHERE created_at &gt;= #{start}   <!-- XML이라 < > 는 이스케이프 -->
    AND created_at &lt;  #{end}
</select>
```
```java
// 자바에서 [start, end) 계산해 넘김 — 연도(int)만 받아도 됨
LocalDate start = LocalDate.of(year, 1, 1);        // 2024-01-01
LocalDate end   = start.plusYears(1);              // 2025-01-01
```
> ⚠️ **MyBatis XML 함정**: `<`, `>`, `&`는 XML 특수문자라 `&lt;`, `&gt;`로 쓰거나 `<![CDATA[ ... ]]>`로 감싸야 한다. (`created_at < #{end}`를 그대로 쓰면 파싱 에러)

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
**계기**: MyBatis 환경에서 DATETIME 컬럼의 2024년 데이터를 뽑으려다, 문자열("2024")이나 함수로 접근하는 것 외에 정석(범위 조건/인덱스)을 몰라 정리. SQL 문법보다 "상황별 해법"이 필요해 쿡북 형식으로 시작.

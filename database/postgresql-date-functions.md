# PostgreSQL 날짜 함수 — to_date · make_date · EXTRACT · date_trunc

> **한 줄 요약**: 날짜를 **만들고(to_date·make_date)**, **분해하고(EXTRACT)**, **단위로 자르는(date_trunc)** 함수들. 역할이 셋으로 갈린다 — 헷갈리면 "입력이 뭐고 출력이 뭔지"로 구분하면 된다.

관련 노트: [SQL 케이스 쿡북](./sql-cookbook.md)

---

## 0. 한눈에 — 세 갈래로 나뉜다

| 함수 | 역할 | 입력 → 출력 |
|------|------|------------|
| `to_date(text, fmt)` | **생성**: 문자열 파싱 | 문자열 → `date` |
| `make_date(y, m, d)` | **생성**: 숫자 조립 | 정수 3개 → `date` |
| `EXTRACT(field FROM ts)` | **분해**: 일부 꺼내기 | 날짜 → 숫자 |
| `date_trunc(unit, ts)` | **절삭**: 단위로 자르기 | 날짜 → 날짜(더 거친) |

```
생성: 문자열/숫자  ──►  날짜        (to_date, make_date)
분해: 날짜  ──►  숫자(연/월/일…)     (EXTRACT)
절삭: 날짜  ──►  날짜(월초/일초…)     (date_trunc)
```

---

## 1. `to_date(text, format)` — 문자열을 날짜로 파싱

문자열과 **포맷 패턴**을 받아 `date`로 변환.

```sql
to_date('2024', 'YYYY')          -- 2024-01-01  (없는 부분은 1월 1일로)
to_date('2024-03-15', 'YYYY-MM-DD') -- 2024-03-15
to_date('15/03/2024', 'DD/MM/YYYY') -- 2024-03-15
```

- 반환: `date` (시간 없음). 시간까지 필요하면 **`to_timestamp(text, fmt)`**.
- 포맷 토큰: `YYYY`(연 4자리) `MM`(월) `DD`(일) `HH24`(24시) `MI`(분) `SS`(초).
- **언제**: 외부에서 문자열로 들어온 날짜(`'2024'`, `'20240315'`)를 날짜 타입으로 바꿀 때.

> ⚠️ to_date는 관대해서 이상한 값도 통과시킬 때가 있다(예: `to_date('2024-13-01','YYYY-MM-DD')`는 에러지만 일부 포맷은 느슨). 신뢰 못 할 입력은 검증 후 사용.

---

## 2. `make_date(year, month, day)` — 숫자로 날짜 조립

**정수 3개**로 `date`를 만든다. 파싱이 아니라 조립이라 더 명확/안전.

```sql
make_date(2024, 3, 15)   -- 2024-03-15
make_date(2024, 1, 1)    -- 2024-01-01
```

- 반환: `date`. 시간까지면 **`make_timestamp(y,m,d,h,mi,s)`**.
- **언제**: 연/월/일이 이미 숫자로 있을 때(예: 파라미터 `year`가 int). 포맷 문자열이 없어 실수 여지가 적다.
- 잘못된 값은 에러: `make_date(2024, 13, 1)` → *date field value out of range*.

> `to_date`(문자열용) vs `make_date`(숫자용) — 입력 타입으로 골라 쓰면 된다.

---

## 3. `EXTRACT(field FROM source)` — 날짜에서 일부를 숫자로 꺼내기

날짜/시간에서 **특정 필드**(연·월·일·시·요일 등)를 숫자로 추출.

```sql
EXTRACT(YEAR  FROM TIMESTAMP '2024-03-15 14:30') -- 2024
EXTRACT(MONTH FROM created_at)                    -- 3
EXTRACT(DAY   FROM created_at)                    -- 15
EXTRACT(HOUR  FROM created_at)                    -- 14
EXTRACT(DOW   FROM created_at)                    -- 요일 (0=일요일 … 6=토요일)
EXTRACT(EPOCH FROM created_at)                    -- 1970년 이후 초(타임스탬프)
```

- 반환: 숫자(`numeric`/`double precision`).
- `date_part('year', created_at)`와 **동일**(EXTRACT가 표준 SQL식 표기).
- **언제**: "몇 월인지", "무슨 요일인지" 같은 **부분 값**이 필요할 때. 요일별·시간대별 집계 등.

> ⚠️ **WHERE 절 컬럼에 EXTRACT를 쓰면 인덱스를 못 탄다.** "2024년 데이터 조회"를 `WHERE EXTRACT(YEAR FROM created_at)=2024`로 하면 안 되고 범위 조건으로 — [SQL 쿡북 §1](./sql-cookbook.md) 참고.

---

## 4. `date_trunc(unit, source)` — 단위 이하를 잘라내기(절삭)

지정 **단위 미만을 0으로** 만들어, 그 단위의 시작점으로 맞춘다.

```sql
date_trunc('month', TIMESTAMP '2024-03-15 14:30:55') -- 2024-03-01 00:00:00
date_trunc('day',   TIMESTAMP '2024-03-15 14:30:55') -- 2024-03-15 00:00:00
date_trunc('hour',  TIMESTAMP '2024-03-15 14:30:55') -- 2024-03-15 14:00:00
date_trunc('year',  TIMESTAMP '2024-03-15 14:30:55') -- 2024-01-01 00:00:00
```

- 단위: `'year' 'quarter' 'month' 'week' 'day' 'hour' 'minute' 'second'`.
- 반환: `timestamp`(잘린 날짜). EXTRACT와 달리 **숫자가 아니라 날짜**를 돌려준다.
- **언제**: **기간별 그룹핑/집계.** "월별 매출", "일별 가입자" 등.

```sql
-- 월별 매출 집계
SELECT date_trunc('month', created_at) AS month, SUM(amount)
FROM orders
GROUP BY 1
ORDER BY 1;
```

> ⚠️ 여기서도 GROUP BY/SELECT엔 좋지만, **WHERE에서 컬럼을 date_trunc로 감싸 필터하면 인덱스를 못 탄다.** 필터는 범위로.

---

## 5. EXTRACT vs date_trunc (제일 헷갈림)

둘 다 "날짜에서 뭔가 뽑는" 느낌이지만 **출력이 다르다.**

| | EXTRACT | date_trunc |
|---|---|---|
| 출력 | **숫자** (3, 2024 …) | **날짜** (2024-03-01 …) |
| 의미 | "그 부분의 값" | "그 단위로 자른 시점" |
| 3월 15일에 `month` | `3` | `2024-03-01 00:00:00` |
| 주 용도 | 요일/시간대별 분류 | 월별/일별 기간 집계 |

```sql
EXTRACT(MONTH FROM ts)      -- 3        (숫자 3)
date_trunc('month', ts)     -- 2024-03-01 00:00:00  (그 달의 시작 시각)
```

> 한 줄: **"몇 월?" → EXTRACT(숫자) / "그 달로 묶어" → date_trunc(날짜).**

---

## 6. 비교·필터에 쓸 때 — 타입을 맞춰야 한다

`>=` 같은 비교는 **같은(호환) 타입끼리만** 의미가 있다.
- `date`/`timestamp`는 **순서가 있는 타입** → 끼리끼리 크기 비교 OK.
- `EXTRACT` 결과는 **숫자** → 숫자랑만 비교. 날짜 컬럼과 직접 못 섞는다.

| 비교 | 되나? | 인덱스 |
|------|-------|--------|
| `created_at >= make_date(2024,1,1)` | ✅ 날짜 vs 날짜 | ✅ 컬럼 맨몸 |
| `created_at >= to_date('2024','YYYY')` | ✅ 날짜 vs 날짜 | ✅ |
| `created_at >= '2024-01-01'` | ✅ (문자열 자동 캐스팅) | ✅ |
| `EXTRACT(YEAR FROM created_at) >= 2024` | ✅ 숫자 vs 숫자 | ❌ 컬럼 가공 |
| `created_at >= 2024` | ❌ 날짜 vs 숫자 (타입 에러) | — |

> 결론: **날짜는 날짜로 만들어 비교**(make_date/to_date) → 타입도 맞고 인덱스도 탄다. EXTRACT 숫자 비교는 되긴 하지만 컬럼을 감싸 인덱스를 버린다. (→ [SQL 쿡북 §1](./sql-cookbook.md)의 "컬럼 가공 vs 값 가공"과 같은 결론)

### 문자열 자동 캐스팅 — 표준 포맷일 때만

`created_at >= '2024-01-01'`이 되는 건, PostgreSQL이 **문자열을 날짜 타입으로 자동 변환(캐스팅)**해 주기 때문이다(겉보기엔 문자열 vs 날짜지만, 결국 날짜 vs 날짜로 맞춰짐).

- ✅ **표준 포맷이면** 자동 캐스팅 OK: `'2024-01-01'`, `'2024-01-01 10:30:00'`.
- ⚠️ **애매한 포맷은 실패하거나 오해석**: `'2024'`(연도만), `'01/15/2024'` 등.
- → 애매한 입력은 자동 캐스팅에 기대지 말고 **`to_date('2024','YYYY')`로 포맷 명시**하거나, **자바에서 날짜로 변환해 넘긴다**(타입을 확실히 맞춤).

> 한 줄: **비교의 본질은 "같은 타입끼리".** `'2024-01-01'`은 표준이라 DB가 알아서 날짜로 맞춰주지만, 애매하면 to_date/자바 변환으로 타입을 직접 맞춘다.

---

## 7. 참고
- [PostgreSQL 공식 - Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html)
- [PostgreSQL 공식 - Data Type Formatting (to_date 포맷)](https://www.postgresql.org/docs/current/functions-formatting.html)
- 관련 노트: [SQL 케이스 쿡북](./sql-cookbook.md)

---

**학습 날짜**: 2026-06-04
**계기**: SQL 쿡북(연도 범위 조회)에서 경계 생성에 쓴 `to_date`/`make_date`와, 함수로 연도를 뽑는 `EXTRACT`/`date_trunc`가 각각 뭔지·언제 쓰는지 헷갈려서 정리

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

> ⚠️ to_date는 **포맷 모양**에 관대하다 — 템플릿의 구분자는 입력의 아무 구분자와 맞는다(`to_date('2024/03/05','YYYY-MM-DD')` → 2024-03-05, PG12부터 더 관대해짐). 반면 **값 범위**는 검사한다(`'2024-13-01'`·`'2024-02-30'`은 *date/time field value out of range*). 그래서 "형식이 맞는지"를 to_date에 맡기지 말고, 신뢰 못 할 입력은 앞단에서 형식부터 검증한다. (PG17에서 확인)

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

- 반환: `numeric`(PG14+. 이전 버전은 `double precision`).
- `date_part('year', created_at)`와 값은 같지만 **반환 타입이 다르다** — `date_part`는 역사적 이유로 `double precision`이라 정밀도 손실이 생길 수 있어 공식 문서도 `EXTRACT`(표준 SQL)를 권한다.
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
- 반환: **입력 타입을 따른다** — `timestamp` → `timestamp`, `timestamptz` → `timestamptz`, `interval` → `interval`. `date`를 넣으면 실제로는 `timestamptz`가 나온다(`pg_typeof(date_trunc('month', current_date))` → timestamp with time zone, PG17 확인). EXTRACT와 달리 **숫자가 아니라 날짜**를 돌려준다.
- **언제**: **기간별 그룹핑/집계.** "월별 매출", "일별 가입자" 등.

```sql
-- 월별 매출 집계
SELECT date_trunc('month', created_at) AS month, SUM(amount)
FROM orders
GROUP BY 1
ORDER BY 1;
```

> ⚠️ 여기서도 GROUP BY/SELECT엔 좋지만, **WHERE에서 컬럼을 date_trunc로 감싸 필터하면 인덱스를 못 탄다.** 필터는 범위로.

> ⚠️ **`timestamptz`는 세션 TimeZone 기준으로 자른다.** 세션이 UTC인데 한국 시간 기준 "월별 매출"을 원하면, 매달 1일 00~09시 KST 주문이 전달로 묶인다. PG12+는 세 번째 인자로 시간대를 고정할 수 있다: `date_trunc('month', created_at, 'Asia/Seoul')`. 날짜 경계의 같은 함정은 [SQL 쿡북 §1](./sql-cookbook.md) PostgreSQL 메모.

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

`created_at >= '2024-01-01'`이 되는 건 PostgreSQL이 문자열을 날짜로 **자동 캐스팅**하기 때문(결국 날짜 vs 날짜). 단 **표준 포맷**(`'2024-01-01'`)만 안전하고, 애매한 입력(`'2024'`·`'01/15/2024'`)은 `to_date`나 자바 변환으로 타입을 직접 맞춘다.

> 한 줄: **비교의 본질은 "같은 타입끼리".** (자동 캐스팅·sargable·인덱스 영향의 상세는 → [SQL 쿡북 §1](./sql-cookbook.md) — 이 노트는 "함수 출력 타입이 비교에 주는 영향"에 집중)

---

## 💡 판단 기준

연도 범위 조회에서 경계를 만들 함수(to_date·make_date)와 연도를 뽑는 함수(EXTRACT·date_trunc)가 섞여 헷갈렸다 → **"이 함수가 컬럼을 감싸는가, 값을 만드는가"로 자리를 정한다.** 값을 만드는 함수(to_date·make_date)는 WHERE의 비교값 쪽에, 컬럼을 감싸는 함수(EXTRACT·date_trunc)는 GROUP BY·SELECT 쪽에 둔다. 그리고 컬럼이 `timestamptz`면 "어느 시간대의 하루·한 달인가"를 먼저 정하고 쓴다.

## 7. 참고
- [PostgreSQL 공식 - Date/Time Functions](https://www.postgresql.org/docs/current/functions-datetime.html) — `date_trunc`의 `time_zone` 인자, EXTRACT vs date_part 반환 타입
- [PostgreSQL 공식 - Data Type Formatting (to_date 포맷)](https://www.postgresql.org/docs/current/functions-formatting.html)
- [PostgreSQL 12 릴리스 노트 — date_trunc 시간대 인자, to_date 관대화](https://www.postgresql.org/docs/release/12.0/)
- 관련 노트: [SQL 케이스 쿡북](./sql-cookbook.md)

---

**학습 날짜**: 2026-06-04 (2026-10-02 반환 타입·timestamptz 함정 보강, PG17에서 확인)
**계기**: SQL 쿡북(연도 범위 조회)에서 경계 생성에 쓴 `to_date`/`make_date`와, 함수로 연도를 뽑는 `EXTRACT`/`date_trunc`가 각각 뭔지·언제 쓰는지 헷갈려서 정리

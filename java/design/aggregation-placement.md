# 집계를 SQL에서 할까, 애플리케이션에서 할까

> **한 줄 요약**: 기간별 집계(월별 스냅샷 등)를 어디서 할지는 **① 대상 건수 ② 유지보수성** 두 축으로 정한다. 대상이 **수천 건 이하면 애플리케이션이 대체로 낫다** — 성능이 비슷하면서 테스트가 되고 읽히기 때문. ⚠️ 반대로 SQL에 몰아넣으면 빠를 것 같지만, **읽을 수 없는 SQL은 느려져도 아무도 못 고친다.** 성능 문제가 몇 년씩 방치되는 진짜 원인이 대개 이쪽이다.

관련 노트: [N+1과 fetch 전략 §5-1](../jpa/n-plus-one-fetch.md) · [SQL 쿡북](../../database/sql-cookbook.md) · [인덱스와 실행 계획](../../database/index-explain.md) · [Stream API](../functional/stream-api.md)

---

## 1. 언제 고민이 생기나

"최근 12개월 각 달마다 그 시점에 유효했던 건수와 합계" 같은 **기간별 스냅샷 집계**다. 달마다 조건이 달라서 `GROUP BY` 하나로 안 떨어진다.

```
2025-08 시점에 유효했던 계약 수 / 금액
2025-09 시점에 유효했던 계약 수 / 금액
...
2026-08 시점에 유효했던 계약 수 / 금액
```

SQL로 짜면 보통 **월 목록을 만들고 각 행마다 서브쿼리**를 붙이게 된다.

---

## 2. SQL에 몰아넣었을 때 생기는 일 (실제 사례)

```sql
WITH month_date AS ( SELECT ... FROM GENERATE_SERIES(...) )   -- 13개월
SELECT m.year, m.month,
    ( SELECT COUNT(*)      FROM hst WHERE ... m.date ... ) as cnt,    -- 상관 서브쿼리
    ( SELECT SUM(cost)     FROM hst WHERE ... m.date ... ) as sum     -- 조건이 위와 동일
FROM month_date AS m
```

문제가 두 겹으로 쌓였다.

**① 같은 조건을 두 번 훑는다** — `COUNT`와 `SUM`을 따로 구하려고 서브쿼리를 2개 두면 **13개월 × 2 = 26번** 테이블을 읽는다. 한 번 읽으면서 둘 다 구할 수 있는데도.

**② 컬럼을 함수로 감쌌다** — 월 단위 비교를 하려고 이렇게 썼다.
```sql
WHERE TO_DATE(TO_CHAR(assign_dtm, 'YYYYMM'), 'YYYYMM') <= m.date
```
**인덱스가 있어도 못 탄다**(→ [SQL 쿡북 §1](../../database/sql-cookbook.md)). 26번이 전부 full scan이 되고, 행마다 날짜→문자열→날짜 변환까지 돈다.

> 실측으로 이 쿼리 하나가 API 응답의 **87%**를 먹고 있었다.

**⭐ 그런데 진짜 문제는 이게 몇 년간 방치됐다는 것이다.** `TO_DATE(TO_CHAR(...))`가 인덱스를 죽인다는 걸 알아채려면 SQL에 익숙해야 하고, 경계값(배정 당월 포함? 해제 당월 제외?)이 맞는지는 **DB에 데이터를 넣어보지 않으면 확인할 방법이 없다.** 읽을 수 없으니 고칠 수도 없었다.

---

## 3. 애플리케이션으로 옮기면

조회는 **기간에 걸치는 것만, 함수 없이** 가져온다.

```java
@Query("""
        select h from AssignHst h
        where  h.orgId = :orgId and h.usrId = :usrId
          and  h.assignDate < :endExclusive
          and  (h.expiredDtm is null or h.expiredDtm >= :startInclusive)
        """)
```

판정은 값 객체가 스스로 안다.

```java
public boolean isActiveIn(YearMonth month) {
    return !YearMonth.from(assignDate).isAfter(month)
            && (expiredDtm == null || YearMonth.from(expiredDtm).isAfter(month));
}
```

집계는 루프.

```java
Stream.iterate(start, m -> m.plusMonths(1))
      .limit(MONTHS)
      .map(month -> RetainStatus.from(month, rows))
      .toList();
```

### 무엇이 좋아지나

| | SQL 안에 몰아넣기 | 애플리케이션으로 |
|---|---|---|
| 의도 | `TO_DATE(TO_CHAR(...))` — 안 읽힘 | `YearMonth` 비교 — 그대로 읽힘 |
| 경계값 검증 | DB 필요 | **단위 테스트로 가능** |
| DB 종속 | `GENERATE_SERIES` 등 벤더 문법 | 없음 |
| 전송량 | 결과 13행 | **대상 전체 행** |
| 집계 성능 | DB가 잘하는 일 | 메모리 연산(수천 건이면 ms) |

**경계값을 테스트로 고정할 수 있다는 게 가장 큰 이득**이다. 이런 조건은 한 칸 어긋나도 값이 그럴듯해서 눈으로는 안 걸린다.

```java
배정_당월은_포함된다          해제_당월은_제외된다
배정_다음달은_제외된다        배정과_해제가_같은_달이면_제외된다
```

---

## 4. ⚠️ 함정

- **전송량이 유일한 약점** — 완화하려면 **조회 창을 좁힌다.** "13개월에 걸치는 것만" 가져오면 오래된 이력은 애초에 안 온다.
  ```sql
  and h.assignDate < :endExclusive
  and (h.expiredDtm is null or h.expiredDtm >= :startInclusive)
  ```
- **오프바이원이 두 군데 생긴다.** 둘 다 틀려도 값이 그럴듯해서 안 걸린다 → 주석과 테스트로 고정할 것
  - 항목 N개면 시작은 `N-1`개월 전 (기둥 N개, 간격 N-1)
  - 상한이 배타적(`<`)이면 **마지막 달을 포함하려면 다음 달 1일**
- **엔티티로 받을지 프로젝션으로 받을지는 별개 판단** — 컬럼이 몇 개 안 되면 엔티티도 무방하다. 컬럼이 많거나 안 쓰는 연관이 붙으면 프로젝션 (→ [N+1 §5-1](../jpa/n-plus-one-fetch.md))
- **집계 규칙을 어디 둘지** — "그 달에 유효한가"는 그 데이터 자신이 답할 수 있는 질문이라 엔티티/값 객체에 두는 게 응집이 맞다. 응답 DTO에 흩어지면 재사용도 테스트도 어려워진다

---

## 5. 💡 판단 기준

| 상황 | 어디서 |
|---|---|
| 대상이 **수만~수십만 건** | **SQL** — 그걸 다 끌어올 이유가 없다 |
| 대상이 **수천 건 이하** | **애플리케이션** — 성능이 비슷하면 읽히는 쪽 |
| 조건이 **월/기간마다 달라** GROUP BY 하나로 안 떨어짐 | 애플리케이션 쪽이 편해지는 신호 |
| 집계 결과를 **다시 조인·정렬**해야 함 | SQL |
| 규칙에 **경계값 판단**이 있음(포함/제외) | 애플리케이션 — 테스트로 고정할 수 있다 |

> ⭐ **"SQL이라서 빠르다"가 아니라 "제대로 짠 SQL이 빠르다".** 지금 느린 SQL은 SQL이라서가 아니라 **잘못 짜여서** 느린 경우가 많다. 그리고 잘못 짜인 채 오래 남는 이유는 대개 **읽을 수 없어서**다. 그래서 건수가 감당 가능한 범위라면, 성능이 비슷할 때 **읽히고 테스트되는 쪽**을 고르는 게 총비용이 싸다.

> 💡 반대로 **건수가 크면 망설이지 말고 SQL**이다. 수십만 건을 애플리케이션으로 끌어와 세는 건 전송량·메모리 양쪽에서 지는 싸움이다. 이때는 SQL을 *제대로* 짜는 쪽에 시간을 쓴다 — 컬럼에 함수 씌우지 않기, 같은 조건 서브쿼리를 조인으로 접기.

---

## 6. 참고

- 관련 노트: [N+1과 fetch 전략 §5-1](../jpa/n-plus-one-fetch.md) · [SQL 쿡북](../../database/sql-cookbook.md) · [인덱스와 실행 계획](../../database/index-explain.md) · [LATERAL·top-N per group](../../database/lateral-join-top-n-per-group.md)

---

**학습 날짜**: 2026-08-13
**계기**: 월별 유지 현황 API가 느려 프로파일링하니 **쿼리 하나가 응답의 87%**였다. 원인은 상관 서브쿼리 26회 + 컬럼을 `TO_DATE(TO_CHAR(...))`로 감싸 인덱스 무효화. 고치려고 SQL을 읽다가 **경계값이 맞는지 확인할 방법이 없다**는 걸 깨닫고 애플리케이션으로 옮김. 대상이 3천 건이라 전송량도 문제가 아니었음. **"성능 문제가 오래 방치된 이유가 읽을 수 없어서였다"**가 이번의 진짜 교훈.

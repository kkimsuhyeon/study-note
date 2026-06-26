# 인덱스와 실행 계획 — EXPLAIN, range scan, full scan

> **한 줄 요약**: 인덱스는 DB가 조건에 맞는 row를 빨리 찾기 위한 정렬된 자료구조다. 쿼리가 인덱스를 잘 쓰는지는 감이 아니라 실행 계획(`EXPLAIN`)으로 확인한다.

관련 노트: [SQL 케이스 쿡북](./sql-cookbook.md) · [PostgreSQL 날짜 함수](./postgresql-date-functions.md) · [JPA 동적 조회](../java/jpa/spring-data-query.md)

---

## 1. 인덱스가 돕는 것

인덱스는 책의 색인처럼 "어디에 있는지"를 빨리 찾게 해준다.

잘 맞는 경우:

- `where user_id = ?`
- `where created_at >= ? and created_at < ?`
- `order by created_at desc limit 20`
- `(user_id, created_at)` 복합 인덱스에서 `user_id = ?` 후 날짜 범위 조회

---

## 2. 실행 계획 기본

```sql
EXPLAIN
SELECT *
FROM orders
WHERE created_at >= '2024-01-01'
  AND created_at < '2025-01-01';
```

`EXPLAIN`은 DB가 어떤 방식으로 읽을지 보여준다.

자주 보는 단어:

| 용어 | 의미 |
|---|---|
| Seq Scan / Full Table Scan | 테이블 전체를 읽음 |
| Index Scan | 인덱스를 사용해 row를 찾음 |
| Index Only Scan | 인덱스만 보고 결과를 만들 수 있음 |
| Bitmap Scan | 인덱스로 후보를 모은 뒤 테이블을 읽음 |
| rows | 예상 row 수 |
| cost | DB가 추정한 비용 |

---

## 3. sargable 조건

sargable은 인덱스를 탈 수 있는 검색 조건이라는 뜻으로 보면 된다.

좋은 조건:

```sql
where created_at >= '2024-01-01'
  and created_at < '2025-01-01'
```

나쁜 조건:

```sql
where extract(year from created_at) = 2024
```

컬럼에 함수를 씌우면 DB가 인덱스의 정렬된 값을 그대로 쓰기 어렵다. 날짜 필터는 보통 반열린 범위 조건으로 만든다.

> **선행지식 — 왜 함수를 씌우면 인덱스가 죽나.** B-tree 인덱스는 **컬럼 값을 정렬해서** 저장한다. `created_at`이 정렬돼 있으니 범위(`>= ~ <`)는 "시작점 찾아 쭉 읽기"로 빠르다. 그런데 `extract(year from created_at)`는 **원본이 아니라 계산 결과**라, 정렬된 원본 인덱스로는 그 결과를 짚을 수 없다 → 전 row를 계산해보는 full scan. "컬럼을 가공하면 정렬이 무의미해진다"가 sargable의 본질.

---

## 4. 복합 인덱스 순서

```sql
create index idx_orders_user_created_at
on orders (user_id, created_at);
```

이 인덱스는 다음에 잘 맞는다.

```sql
where user_id = ?
  and created_at >= ?
  and created_at < ?
```

복합 인덱스는 앞 컬럼부터 쓰는 게 중요하다. `(user_id, created_at)` 인덱스는 `created_at`만 단독 조건으로 조회할 때는 기대만큼 도움이 안 될 수 있다.

---

## 5. 인덱스가 항상 좋은 건 아니다

인덱스는 읽기를 빠르게 하지만 쓰기 비용을 늘린다.

- insert 때 인덱스에도 추가해야 함
- update 때 인덱스 컬럼이 바뀌면 인덱스도 갱신해야 함
- delete 때 인덱스에서도 제거해야 함
- 인덱스 자체도 저장 공간을 먹음

그래서 "혹시 모르니 다 인덱스"가 아니라, 자주 쓰는 조회 패턴 기준으로 만든다.

---

## 6. 판단 기준

| 상황 | 판단 |
|---|---|
| 자주 조회하는 equality 조건 | 인덱스 후보 |
| equality + range 조건 | 복합 인덱스 후보 |
| 낮은 선택도 컬럼 | 단독 인덱스 효과가 약할 수 있음 |
| 정렬 + limit | 정렬 컬럼 인덱스 검토 |
| 쓰기가 많은 테이블 | 인덱스 수를 더 보수적으로 |

---

## 7. 함정

- `like '%keyword'`는 일반 B-tree 인덱스를 잘 못 탄다.
- `where date(created_at) = ?`처럼 컬럼에 함수를 씌우면 인덱스를 못 탈 수 있다. (탈출구: 조건을 범위로 바꾸거나, PostgreSQL이면 **함수 기반 인덱스** `create index on t (date(created_at))`로 계산 결과 자체를 인덱싱.)
- 실행 계획의 row 추정이 틀리면 통계 갱신이 필요할 수 있다.
- 개발 DB의 작은 데이터로는 인덱스 효과가 안 보일 수 있다.

---

## 8. 참고

- PostgreSQL EXPLAIN
- MySQL EXPLAIN
- SQL 케이스 쿡북의 날짜 범위 조회

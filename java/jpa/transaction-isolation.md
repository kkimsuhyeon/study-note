# 트랜잭션 격리 수준 — 이상 현상과 락의 관계

> **한 줄 요약**: 트랜잭션 격리 수준은 "동시에 실행되는 트랜잭션이 서로의 중간 상태를 얼마나 볼 수 있나"를 정하는 규칙이다. 낮으면 동시성은 좋아지지만 이상 현상이 생기고, 높이면 정합성은 좋아지지만 대기·데드락 가능성이 커진다.

관련 노트: [@Transactional](../spring/transactional.md) · [Read-Modify-Write](./read-modify-write.md) · [@Lock 기본](./lock.md) · [데드락](../concurrency/deadlock.md)

---

## 1. 왜 알아야 하나

`@Transactional`은 트랜잭션 경계를 만든다. 하지만 **트랜잭션 안에서 무엇을 볼 수 있는지**는 DB 격리 수준이 결정한다.

동시성 버그를 볼 때는 항상 두 질문을 같이 본다.

1. 읽기와 쓰기가 같은 트랜잭션 안에 있는가?
2. 그 트랜잭션의 격리 수준에서 다른 트랜잭션의 변경을 볼 수 있는가?

---

## 2. 이상 현상

| 현상 | 의미 | 예시 |
|---|---|---|
| Dirty Read | 커밋 안 된 데이터를 읽음 | 다른 트랜잭션이 롤백하면 내가 읽은 값은 없던 값 |
| Non-repeatable Read | 같은 행을 두 번 읽었는데 값이 달라짐 | 처음엔 잔액 100, 다시 읽으니 80 |
| Phantom Read | 같은 조건으로 조회했는데 행 개수가 달라짐 | 처음엔 주문 3건, 다시 조회하니 4건 |
| Lost Update | 둘이 같은 값을 읽고 각각 수정해 한쪽 변경이 덮임 | 둘 다 재고 10을 읽고 각각 1 감소 → 최종 9 |

Lost Update는 단순 격리 수준만으로 해결된다고 외우면 위험하다. DB와 격리 수준별 동작이 다르고, 실무에서는 보통 **낙관적 락(`@Version`)**, **비관적 락(`PESSIMISTIC_WRITE`)**, **조건부 UPDATE**로 의도를 명시한다.

---

## 3. 격리 수준 한눈에

| 격리 수준 | 대략적 의미 | 막는 것 |
|---|---|---|
| READ UNCOMMITTED | 커밋 안 된 것도 볼 수 있음 | 거의 없음 |
| READ COMMITTED | 커밋된 것만 봄 | Dirty Read |
| REPEATABLE READ | 같은 트랜잭션 안에서 같은 행은 반복 조회 안정 | Dirty Read, Non-repeatable Read |
| SERIALIZABLE | 트랜잭션이 순서대로 실행된 것처럼 보장 | Dirty Read, Non-repeatable Read, Phantom Read |

주의: 이름이 같아도 DB마다 구현이 다르다. MySQL InnoDB의 `REPEATABLE READ`와 PostgreSQL의 `REPEATABLE READ`는 내부 방식과 충돌 처리 방식이 다르다.

---

## 4. 락과의 관계

격리 수준은 DB의 기본 관찰 규칙이다. 락은 특정 쿼리에서 더 강한 제어를 명시하는 도구다.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Product> findById(Long id);
```

위 쿼리는 조회 대상 행에 쓰기 락을 걸어 다른 트랜잭션의 수정을 기다리게 만든다.

격리 수준을 무작정 높이는 것보다, 실제 충돌 지점에 락이나 조건부 UPDATE를 거는 편이 더 명확한 경우가 많다.

---

## 5. 판단 기준

| 상황 | 우선 검토 |
|---|---|
| 재고 차감, 포인트 사용처럼 같은 행을 여러 요청이 수정 | `@Version` 또는 `PESSIMISTIC_WRITE` |
| 실패하면 다시 시도해도 되는 수정 | 낙관적 락 + 재시도 |
| 반드시 한 요청만 통과해야 함 | 비관적 락 또는 조건부 UPDATE |
| 범위 조건으로 "없는 것"을 확인하고 insert | 유니크 제약 + 예외 처리 |
| 조회 통계, 목록 화면 | 격리 수준보다 일관성 요구사항부터 확인 |

---

## 6. 함정

- `@Transactional`만 붙였다고 Lost Update가 자동으로 사라지는 것은 아니다.
- 격리 수준을 높이면 성능 비용, 대기, 데드락 가능성이 같이 올라간다.
- 애플리케이션에서 먼저 조회하고 나중에 수정하면 그 사이에 다른 트랜잭션이 끼어들 수 있다. 그래서 Read-Modify-Write는 같은 트랜잭션과 적절한 동시성 제어가 필요하다.
- 최종 중복 방지는 애플리케이션 체크가 아니라 DB 유니크 제약이 맡아야 한다.

---

## 7. 참고

- Spring Framework Transaction Management
- PostgreSQL Transaction Isolation
- MySQL InnoDB Transaction Isolation

# 애그리거트 소유권 & 참조 방향 — source of truth / 누가 무엇을 판정·소유하나

> **한 줄 요약**: 한 *사실*("결제됐나?")의 진실은 **딱 한 도메인이 소유**(source of truth)하고, 나머지는 그걸 *조회*하거나 *반영*만 한다. 그래서 **각 도메인은 자기 사실만 판정**하고 남의 사실은 주인에게 묻는다. 상태(state)는 양쪽이 각자 가질 수 있지만(취소/환불은 다른 사실), *판정 권한*은 하나여야 한다. 1:1/1:N **참조(FK) 방향**은 "나중에 생기고·종속적이고·N이 될 쪽이 상대 ID를 든다".

관련 노트: [도메인 검증 위치](./domain-validation.md) · [포트와 어댑터](./ports-and-adapters.md)

---

## 1. source of truth(주인) = "사실 하나당 결정·저장하는 곳 하나"

주인은 *도메인 전체*가 아니라 **사실(fact) 단위**다.
- "이 예약 결제됐나?"의 답을 **결정·저장**하는 곳이 딱 하나 = 그 사실의 주인. 나머지는 사본을 들지 말고 **주인을 보고 읽거나 반영**.
- 예: 결제 여부의 주인 = **Payment**(`status == SUCCESS`). Reservation은 그걸 *판정*하지 않는다.

> ⚠️ 같은 사실을 두 곳이 각자 판정·저장하면 **이중 진실(two sources of truth)** → 어긋나면 모순. 그래서 주인은 하나.

## 2. 각 도메인은 *자기 사실만* 판정한다 (남의 사실은 주인에게)

"판정을 하지 마라"가 아니라 **"남의 사실을 판정하지 마라"**다.

| 도메인 | 자기 사실 (판정 O) | 남의 사실 (판정 ❌, 조회) |
|---|---|---|
| Reservation | 소유자냐·만료냐·자기 상태전이 가능한가 | "결제됐나" → Payment에게 |
| Payment | 결제됐나·환불됐나 | "좌석 비었나" → Seat에게 |

- 실제 예(이 프로젝트): `Reservation.validateForPayment`에서 **"이미 결제됨(`if status==CONFIRMED → ALREADY_PAID`)" 판정을 제거** → 그 판정은 `Payment.pay()`(PENDING이 아니면 거부)로 이사. 소유자·만료 판정은 예약에 **남김**(자기 사실).
- 흔한 실수: 자기 상태(CONFIRMED)를 보고 "결제됐다"고 **추론**하는 것. CONFIRMED는 결제 후 붙지만, "결제됨"의 진실은 Payment에 있어야 한다.

### 2.1 남의 사실을 *어떻게* 엔티티에 들이나 — 조회=서비스, 판정=엔티티

"남의 사실은 판정하지 마라"의 실천편: 엔티티는 다른 애그리거트를 볼 수 없으니, **조회는 서비스가 하고 그 *결과*를 엔티티 메서드에 넘겨** 판정·예외는 엔티티 안에서 하게 한다.

> ⚠️ **엔티티에 Repository/Mapper를 주입하지 않는다.** 인프라 의존이 도메인에 새면 테스트·이식이 깨진다. 앱 서비스가 의존성을 해소해 **필요한 데이터/결과만** 엔티티에 전달한다. (MS Learn·DDD 통설)

**남의 사실을 들이는 방법 스펙트럼** (ardalis의 *11 ways* 요약):

| 방법 | 내용 | 트레이드오프 |
|---|---|---|
| DB 제약 | unique 제약 등 인프라에 위임 | 쉽고 빠르나 규칙이 도메인 밖으로 새고 영속성에 종속 |
| 도메인 서비스 | 서비스가 검사+엔티티 조작 | 로직은 도메인에 남지만 **엔티티가 빈약(anemic)해짐** — 로직이 점점 서비스로 흘러감 |
| 데이터 주입 | 비교에 필요한 원본 데이터를 메서드에 넘김 | 호출부가 *너무 많이 알아야* 함(매칭 데이터를 미리 추려 넘기는 변형은 저자도 비판) |
| **체커/함수 주입** | `IUniquenessChecker`·`Predicate`를 메서드에 주입 | 엔티티가 *언제 검사할지* 스스로 제어. 의존을 매번 주입 |
| **결과(boolean) 주입** | 서비스가 미리 판정해 **결과만** 넘김 | 가장 단순. 엔티티 조건이 단순할 때 적합 |

**구체 케이스(이 프로젝트)**: 기초코드(`WorkTimeCfg`)를 미사용/시간변경하려 할 때 "이 코드를 쓰는 활성 근로제가 있나"는 **다른 애그리거트(WorkSystem)의 사실** → 엔티티가 모름. 그래서:
```java
// 서비스: 남의 애그리거트 사실은 여기서 조회 (조회=인프라)
boolean useInWorkSystem = workSystemMapper.existsWorkSystemByWkTmCds(orgId, List.of(wkTmCd));
workTimeCfg.update(..., useInWorkSystem);          // 결과(boolean)만 주입

// 엔티티: 판정·예외는 자기 책임 (규칙=도메인)
public void update(..., boolean useInWorkSystem) {
    if (useInWorkSystem) {
        if (YNEnum.N == useYn) throw new BizException("...");   // 사용중→미사용 금지
        if (!Objects.equals(this.stTm, newStTm)) throw ...;     // 사용중 시간변경 금지
    }
    ...
}
```

> 💡 **boolean이냐 체커냐**: 검사가 *단순 존재 여부*고 항상 한다 → **boolean 결과 주입**(가장 단순). 엔티티가 *조건에 따라 검사 여부/시점을 스스로* 정해야 한다(예: 값이 바뀐 경우에만) → **함수/체커 주입**. 도메인 서비스에 다 빼는 건 엔티티를 빈약하게 만들므로 최후 수단.

> ⚠️ **check-then-act는 원자적이지 않다.** 조회→판정→저장 사이에 다른 트랜잭션이 끼면 무력화 → 진짜 강한 불변식이면 최종 보장은 **DB unique 제약/락**으로. 위 in-use 검사 같은 UX성 가드는 boolean 주입으로 충분. (→ [@Lock 기본](../jpa/lock.md) · [트랜잭션 격리 수준](../jpa/transaction-isolation.md))

## 3. 상태는 양쪽이 가져도 된다 — 판정 권한만 하나

"주인 하나"가 **"다른 도메인은 상태를 갖지 마라"가 아니다.** 서로 다른 *사실*이면 각자 상태를 가진다.
```
결제 후 취소:  Reservation.status = CANCELLED   (예약의 사실)
              Payment.status     = REFUNDED     (결제의 사실)   ← 둘 다 자기 상태 가짐 ✅
```
- 막아야 할 건 "같은 사실을 두 곳이 *각자 판정*"하는 것이지, "각자 자기 상태를 갖는 것"이 아니다.
- 결제 성공 시 `reservation.completePayment() → CONFIRMED`도 유지 — 이건 **결제 결과를 예약이 자기 상태에 *반영*(projection)**하는 것(판정이 아니라 반응). Seat가 RESERVED 되는 것도 같은 반영.

> 💡 정리: **상태(state)는 양쪽 OK, 판정(decision)은 한쪽.** 예약은 결제 결과를 자기 상태에 반영하되, "결제 가능?"은 Payment가 판정.

---

## 4. ⭐ 1:1 / 1:N 참조(FK) 방향 정하는 기준

1:1은 양쪽 다 기술적으로 가능 → 아래 기준으로 정한다(보통 한 결론으로 수렴).

| 기준 | 누가 상대 ID를 드나 |
|---|---|
| **생명주기** — 누가 먼저/없이 존재 가능? | **나중에·종속적으로 생기는 쪽**이 든다 |
| **의존(중요도)** — 핵심 vs 부수 | **부수적인 쪽**(행위·기록)이 핵심을 가리킨다 |
| **카디널리티 확장** — 나중에 N 될 쪽? | **N이 될 쪽**이 든다(1:N로 자연 확장) |
| **조회 패턴** | 주로 찾는 방향에 인덱스(보통 단방향으로 충분) |

- 헷갈리면 **"누가 먼저 생기나"** 하나만 봐도 대부분 맞다 — **먼저 생기는 게 *가리켜지는* 쪽**, 나중 생기는 게 FK를 든다.
- 예: `User ← Profile(userId)`, `Order ← Payment(orderId)`, `Article ← Comment(articleId)`, `Reservation ← Payment(reservationId)`. 전부 "먼저 생긴 핵심"을 "나중 생긴 부수"가 가리킴.
- 이 프로젝트 체인(생성 순):
  ```
  User(먼저) ← Reservation(userId)
  Seat(먼저) ← Reservation(seatId)
  Reservation(먼저) ← Payment(reservationId)
  ```
  Reservation은 중간 — 위(User·Seat)는 가리키고, 아래(Payment)에는 가리켜짐. **Reservation에 paymentId 넣으면 안 됨**(결제는 나중 생김 → 양방향·순환·이중 진실).

> ⚠️ **양방향(둘 다 ID 보유)은 기본이 아니다.** "양쪽에서 자주 탐색 + 조인 회피 성능"이 진짜 필요할 때만, 동기화 부담 감수하고. 기본은 **단방향**. "상대를 알아야 하면" ID를 새로 넣지 말고 **상대를 내 ID로 조회**(예: `payment.findByReservationId(rid)`).

---

## 5. 💡 판단 기준 (한눈에)

| 질문 | 답 |
|---|---|
| 이 사실의 진실은 어디? | 한 도메인만(source of truth). 나머진 조회/반영 |
| 이 판정을 여기서 해도 되나? | 내 사실이면 O, 남의 사실이면 ❌(주인에게 조회) |
| 남의 사실을 어떻게 엔티티에 들이나? | 서비스가 조회→결과(boolean)/체커를 메서드에 주입. **엔티티엔 Repository 주입 X** |
| 두 도메인이 상태를 다 가져도 되나? | 서로 다른 사실이면 O (판정 권한만 하나) |
| FK를 어느 쪽에? | 나중에 생기는·부수적·N 될 쪽이 상대 ID를 든다(단방향) |

> 한 줄: **"사실 하나당 주인 하나, 판정은 주인만, 상태는 각자, 참조는 나중 생긴 쪽이 한 방향으로."** 헷갈리면 *생성 순서*로 — 먼저 생긴 핵심을 나중 생긴 부수가 가리킨다.

---

## 6. 참고
- 관련 노트: [도메인 검증 위치](./domain-validation.md) · [포트와 어댑터](./ports-and-adapters.md)
- [ardalis/DDD-NoDuplicates](https://github.com/ardalis/DDD-NoDuplicates) — 중복 금지 불변식을 지키는 11가지 설계(DB제약/도메인서비스/데이터·체커·함수 주입) + 각 트레이드오프
- [Microsoft Learn — Domain model layer validations](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-model-layer-validations) · [Kamil Grzybek — Domain Model Validation](https://www.kamilgrzybek.com/blog/posts/domain-model-validation)

---

**학습 날짜**: 2026-06-20 (2026-06-30: §2.1 "남의 사실을 어떻게 엔티티에 들이나" 보강)
**계기**: Payment를 "결제 판정의 주인"으로 재설계하며 — Reservation.validateForPayment의 '이미 결제됨' 판정 제거, "상태는 양쪽 갖되 판정은 하나", Reservation에 paymentId 넣으면 안 되는 이유(참조 방향)를 정리. source of truth / 자기 사실만 판정 / 1:1 FK 방향이 한 묶음.
**계기(2026-06-30)**: 기초코드(WorkTimeCfg/WorkBreakCfg) 미사용·시간변경 시 "활성 근로제가 참조 중인가"를 막아야 했는데, 그건 다른 애그리거트(WorkSystem) 사실 → 서비스에서 `existsWorkSystemBy...`로 조회해 boolean을 엔티티 `update()`에 주입하고 throw는 엔티티에서. "조회=서비스, 판정=엔티티 / 엔티티에 Repository 주입 X" 패턴을 ardalis 11ways와 엮어 정리.

# 예제로 보는 트랜잭션 전파·롤백 — a → b → c 워크스루

> **한 줄 요약**: `a → b(REQUIRES_NEW) → c(REQUIRED)` 한 예제로, **전파(REQUIRED/REQUIRES_NEW) · rollback-only · `UnexpectedRollbackException` · catch vs rethrow · 누가 살고 누가 롤백되나**를 한 번에 따라가 본다.

관련 노트: [@Transactional (개념·전파·롤백 규칙)](./transactional.md) · [영속성 컨텍스트](../jpa/persistence-context.md)

> 용어 정의(rollback-only, `UnexpectedRollbackException`, 전파 옵션, 프록시)는 [@Transactional](./transactional.md)에 있다. 이 문서는 그걸 **예제로 체화**하는 용도.

---

## 예제 셋업

```java
@Transactional                              // a — 트랜잭션 A (최상위)
public void a() {
    repoA.save(...);   // a 자신의 작업
    b();
}

@Transactional(propagation = REQUIRES_NEW)  // b — A를 멈추고 새 트랜잭션 B 시작 (커밋 경계!)
public void b() {
    c();               // 시나리오에 따라 try-catch
}

@Transactional                              // c — REQUIRED → B에 합류 (같은 트랜잭션)
public void c() {
    throw new RuntimeException();           // 💥
}
```

### 먼저 잡고 갈 3가지 전제

1. **b = `REQUIRES_NEW`** → A와 **분리된 독립 트랜잭션 B**. b는 **물리 커밋 경계**다.
2. **c = `REQUIRED`** → 새로 안 만들고 **B에 합류**. 즉 b와 c는 **같은 트랜잭션 B**.
3. **c의 예외가 c 프록시 경계를 넘으면 → B가 `rollback-only`로 찍힌다.** 그리고 **rollback-only인 트랜잭션을 `commit`하려 하면 → `UnexpectedRollbackException`.** (롤백 자체가 이 예외인 게 아니라, "rollback-only인데 커밋 시도"가 트리거)

---

## 시나리오 1 — b가 c 예외를 **잡고 안 던짐** (삼킴)

```java
public void b() {
    try { c(); } catch (Exception e) { /* 삼킴 */ }
}
```

```
a (트랜잭션 A)
 └─ b()  [REQUIRES_NEW → A 멈추고 새 트랜잭션 B 시작]
      └─ c()  [REQUIRED → B에 합류]
           c: RuntimeException 💥
           → c 프록시 경계 넘으며 B가 rollback-only로 찍힘
      b: catch로 삼킴 → 본문 정상 종료
      b: B를 commit 시도 → B는 rollback-only → 롤백 + UnexpectedRollbackException
         (B 롤백 = b·c가 한 일 전부 취소)
 ← b가 UnexpectedRollbackException을 a로 던짐  [트랜잭션 A 재개]
```

→ b는 "잡았으니 됐겠지" 했지만 **B는 이미 rollback-only라 못 살린다.** b의 커밋이 `UnexpectedRollbackException`으로 바뀌어 a로 올라간다. ("잡아도 못 살림")

---

## 시나리오 2 — b가 c 예외를 **다시 던짐**(또는 안 잡음)

```java
public void b() { c(); }   // 안 잡음
```

```
c: RuntimeException 💥 → B rollback-only
예외가 b 밖으로 나감 → b 프록시가 B를 롤백하고 원본 예외를 그대로 전파
   (commit을 시도하지 않으므로 UnexpectedRollbackException 아님!)
 ← a로 원본 RuntimeException 전달
```

> **차이**: 시나리오 1은 b가 **커밋을 시도**해서 `UnexpectedRollbackException`. 시나리오 2는 b가 **커밋을 안 하고 예외로 빠져나가서** **원본 RuntimeException**. 어느 쪽이든 B는 롤백된다.

---

## 그래서 a는? — a가 받은 예외를 잡느냐 마느냐로 갈린다

b가 `REQUIRES_NEW`(독립)라서 **b의 실패는 A를 오염(rollback-only)시키지 않는다.** 따라서 a는 스스로를 지킬 수 있다.

| a의 처리 | 트랜잭션 A | a 자신의 작업(`saveA`) |
|---------|-----------|----------------------|
| 받은 예외를 **다시 던짐**(안 잡음) | 롤백 | 취소됨 |
| 받은 예외를 **catch하고 안 던짐** | **정상 커밋** ✅ | **남음** ✅ |

> a가 catch하면 A는 깨끗하게 커밋된다. (b가 `REQUIRED`였다면 A까지 rollback-only라 이게 불가능 — 아래 변형 참고)

---

## 변형 A — c도 `REQUIRES_NEW`였다면? (b가 살 수 있다)

```java
@Transactional(propagation = REQUIRES_NEW)   // c도 독립 트랜잭션 C
public void c() { throw new RuntimeException(); }
```

- c는 **독립 트랜잭션 C** → c 💥 → **C만 롤백**, B는 오염 안 됨.
- b가 c 예외를 잡으면 → **B는 깨끗** → b 정상 커밋 ✅ (`UnexpectedRollbackException` 없음).

> 즉 "실패할 수 있는 호출(c)을 독립 트랜잭션으로 떼면, 그게 터져도 잡아서 상위(b)를 살린다."

## 변형 B — b가 `REQUIRES_NEW`가 아니라 `REQUIRED`였다면? (a가 못 산다)

- b가 A에 합류 → b·c·a 모두 **같은 트랜잭션 A**.
- c 💥 → **A가 rollback-only**.
- a가 예외를 잡아도 → A는 이미 롤백 확정 → a 커밋 시도 시 **`UnexpectedRollbackException`** → a 못 삼.

---

## 한눈에 정리

| 무엇이 결정하나 | 결과 |
|----------------|------|
| **c의 예외가 같은 트랜잭션(B)을 rollback-only로 만드냐** | c가 `REQUIRED`(합류)면 O / `REQUIRES_NEW`(독립)면 X(자기것만) |
| **`UnexpectedRollbackException`이 뜨는 시점** | rollback-only인 트랜잭션을 **commit 시도**할 때 (= 그 트랜잭션의 커밋 경계) |
| **상위가 살 수 있냐** | 실패한 하위가 **독립(REQUIRES_NEW)**이고 상위가 **catch**하면 산다. **합류(REQUIRED)**면 같이 죽는다 |
| **rethrow vs swallow** | swallow → 커밋 시도 → `UnexpectedRollbackException` / rethrow → 커밋 안 함 → **원본 예외** 전파 |

**핵심 한 줄**: **"같은 트랜잭션이면 한 명만 깨져도 전부 롤백 운명(잡아도 못 살림). 독립 트랜잭션(REQUIRES_NEW)으로 떼고 잡으면 상위는 산다."**

---

## 참고
- 관련 노트: [@Transactional](./transactional.md) · [영속성 컨텍스트](../jpa/persistence-context.md) · [@Lock 실무 패턴(재시도)](../jpa/lock-practical.md)

---

**학습 날짜**: 2026-05-27
**계기**: `@Transactional` 전파·롤백·`UnexpectedRollbackException`을 개별로 배우다, `a→b→c` 한 예제로 전체를 꿰어 이해하려고 워크스루로 정리

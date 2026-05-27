# @Transactional — 선언적 트랜잭션, 전파(propagation), 롤백 규칙

> **한 줄 요약**: 메서드에 트랜잭션 경계를 선언적으로 부여하는 Spring 어노테이션. AOP 프록시가 메서드를 감싸 **시작 → 실행 → (정상)커밋 / (예외)롤백**을 자동 처리한다. 핵심은 **전파(propagation)** 와 **롤백 규칙**, 그리고 **프록시 기반이라 생기는 함정**.

관련 노트: [영속성 컨텍스트 · flush · 더티 체킹](../jpa/persistence-context.md) · [Read-Modify-Write와 트랜잭션 경계](../jpa/read-modify-write.md) · [@Lock 실무 패턴](../jpa/lock-practical.md)

---

## 1. 무엇을 하나

```java
@Transactional
public void transfer(Long from, Long to, long amount) {
    accountRepo.minus(from, amount);
    accountRepo.plus(to, amount);
}   // 정상 종료 → commit / 도중 예외 → 전체 rollback
```

- **선언적 트랜잭션**: `try-commit-catch-rollback`을 코드로 안 쓰고 어노테이션으로 위임.
- Spring이 **AOP 프록시**로 대상 빈을 감싼다. 메서드 호출이 프록시를 거치면서:
  ```
  프록시: 트랜잭션 시작 → 실제 메서드 실행 → 예외 없으면 commit / 있으면 rollback
  ```
- 메서드 단위. 클래스에 붙이면 그 클래스의 **모든 public 메서드**에 적용.

> JPA와의 연결: 트랜잭션 = 영속성 컨텍스트의 수명. commit 직전 **flush**로 모아둔 SQL이 나간다. ([영속성 컨텍스트](../jpa/persistence-context.md))

---

## 2. 주요 속성

| 속성 | 의미 | 비고 |
|------|------|------|
| `propagation` | 기존 트랜잭션이 있을 때 어떻게 할지 | 기본 `REQUIRED` (→ §3) |
| `isolation` | 격리 수준 | 기본 `DEFAULT`(DB 설정 따름) |
| `readOnly` | 읽기 전용 힌트 | 조회 전용 서비스에. flush 안 함 → 약간의 최적화 |
| `rollbackFor` | 이 예외에도 롤백 | 기본은 unchecked만 롤백 (→ §4) |
| `noRollbackFor` | 이 예외엔 롤백 안 함 | |
| `timeout` | 제한 시간(초) 초과 시 롤백 | 긴 트랜잭션 방어 |

---

## 3. 전파 (Propagation) — 기존 트랜잭션이 있을 때 어떻게?

| 옵션 | 동작 |
|------|------|
| **`REQUIRED`** (기본) | 있으면 **합류**, 없으면 새로 시작. 대부분 이거. |
| **`REQUIRES_NEW`** | 기존 걸 **멈추고(suspend) 완전히 새 독립 트랜잭션** 시작 |
| `NESTED` | 기존 트랜잭션 안에 **savepoint**. 부분 롤백 가능(savepoint까지만). 커넥션은 공유 |
| `SUPPORTS` | 있으면 합류, 없으면 트랜잭션 없이 실행 |
| `NOT_SUPPORTED` | 트랜잭션 멈추고 **없이** 실행 |
| `MANDATORY` | 반드시 기존 트랜잭션 있어야 함. 없으면 예외 |
| `NEVER` | 트랜잭션 있으면 예외 |

### REQUIRED vs REQUIRES_NEW (제일 중요)

```java
@Transactional                    // outer (REQUIRED)
public void outer() {
    repo.doA();
    inner();                      // 같은 빈 호출이면 프록시 안 탐 주의(§5)! 보통 다른 빈
}

@Transactional(propagation = REQUIRED)      // ← outer에 합류 (같은 트랜잭션)
@Transactional(propagation = REQUIRES_NEW)  // ← 멈추고 새 트랜잭션
```

- **REQUIRED**: inner가 outer에 **합류** → 하나의 트랜잭션. inner든 outer든 한 곳에서 실패하면 **전부 롤백**.
- **REQUIRES_NEW**: outer를 **suspend**(커넥션을 옆에 치워둠) → inner가 **새 커넥션으로 독립 실행** → inner 커밋/롤백 → outer 재개. **같은 스레드의 동기 호출**(동시성 대기 아님).

---

## 4. REQUIRES_NEW에서 inner 에러 → outer는? (핵심 질문)

REQUIRES_NEW는 **독립**이라 inner 실패가 자동으로 outer를 롤백시키지 **않는다**. **outer가 그 예외를 잡느냐**에 달렸다.

| 상황 | inner | outer |
|------|-------|-------|
| inner 예외를 outer가 **안 잡음**(전파) | 롤백 | **롤백** (예외가 올라오니까) |
| inner 예외를 outer가 **try-catch로 잡음** | 롤백 | **살아서 commit 가능** ← REQUIRES_NEW의 존재 이유 |
| outer가 나중에 롤백, inner는 이미 커밋됨 | **커밋 유지** | 롤백 |

```java
@Transactional   // outer
public void outer() {
    repo.doMainWork();
    try {
        historyService.log(...);   // REQUIRES_NEW — 독립 트랜잭션
    } catch (Exception e) {
        // inner는 자기 트랜잭션만 롤백. outer는 계속 진행 → commit OK
    }
}
```

> 정리:
> - **inner 롤백 ≠ outer 롤백** (outer가 잡으면 outer는 산다)
> - **inner 커밋은 outer가 롤백해도 살아남는다** → "실패해도 꼭 남겨야 하는 로그/이력/감사" 용도

### ⚠️ 비교: REQUIRED(합류)는 "잡아도 못 살린다"

inner가 outer에 **합류**한 상태에서 inner가 예외를 던지면, **그 하나의 트랜잭션이 `rollback-only`로 마킹**된다. outer가 예외를 try-catch로 잡아도, **commit 시점에 `UnexpectedRollbackException`** 이 터진다.

```java
@Transactional
public void outer() {
    try {
        innerRequired();   // 합류된 트랜잭션. 여기서 RuntimeException
    } catch (Exception e) {
        // 잡았지만 소용 없음 — 트랜잭션은 이미 rollback-only
    }
}   // commit 시도 → UnexpectedRollbackException 💥
```

> 같은 트랜잭션을 공유하니, 한 군데서 깨지면 전체가 롤백 운명. "일부만 살리고 싶다"면 REQUIRES_NEW(완전 분리) 또는 NESTED(savepoint).

---

## 5. 롤백 규칙 — 체크 예외는 기본 롤백 안 한다

| 예외 종류 | 기본 동작 |
|-----------|-----------|
| `RuntimeException`, `Error` (unchecked) | **롤백** ✅ |
| `Exception` 등 checked 예외 | **롤백 안 함** ❗ (커밋됨) |

```java
@Transactional
public void save() throws IOException {
    repo.insert(...);
    throw new IOException();   // checked → 롤백 안 됨! insert가 커밋되어 버림
}
```

→ checked 예외에도 롤백하려면 명시:
```java
@Transactional(rollbackFor = Exception.class)
```

> 자주 당하는 함정. "예외 던졌는데 왜 데이터가 들어갔지?" → checked 예외라 롤백 안 된 것.

---

## 6. 자주 당하는 함정

### (1) self-invocation — 같은 클래스 내부 호출은 프록시를 안 탄다
```java
@Service
public class OrderService {
    public void a() {
        b();   // ❌ this.b() — 프록시 안 거침 → b()의 @Transactional 무시됨
    }
    @Transactional
    public void b() { ... }
}
```
→ AOP 프록시는 **외부에서 빈을 호출할 때만** 개입. 내부 호출은 프록시를 우회한다. 해결: 다른 빈으로 분리하거나 self-주입/`AopContext` (보통 분리가 정석).

### (2) public 메서드에만 적용
Spring AOP 프록시 특성상 `private`/`protected`/package-private 메서드의 `@Transactional`은 **무시**된다(예외도 안 남). 반드시 `public`.

### (3) REQUIRES_NEW 커넥션 풀 데드락
outer가 커넥션을 쥔 채 suspend되고 inner가 **또 다른 커넥션**을 요구한다 → 동시에 2개 사용. 풀 크기가 작거나 동시 요청이 많으면 **커넥션 고갈 → 데드락**. REQUIRES_NEW 남발 주의.

### (4) 트랜잭션 안에서 외부 호출(HTTP/메시지) 금지
긴 외부 호출을 트랜잭션 안에 두면 그동안 커넥션·락을 잡고 있어 성능 저하. 외부 호출은 트랜잭션 밖으로.

### (5) 읽기 전용은 `readOnly = true`
조회 전용 서비스는 `@Transactional(readOnly = true)` — flush를 막아 약간의 최적화 + 의도 명시. (쓰기-읽기 분리는 [Read-Modify-Write](../jpa/read-modify-write.md))

---

## 7. 정리

- `@Transactional` = AOP 프록시가 메서드를 감싸 commit/rollback 자동화. **프록시 기반**이라 self-invocation·private은 안 먹힌다.
- 전파 기본 `REQUIRED`(합류). `REQUIRES_NEW`는 **suspend 후 독립 트랜잭션**.
- **REQUIRES_NEW**: inner 예외를 outer가 잡으면 outer는 산다 / inner 커밋은 outer 롤백에도 살아남음.
- **REQUIRED**: 합류라 한 곳만 깨져도 전체 롤백, 잡아도 `UnexpectedRollbackException`.
- 롤백은 **unchecked만 기본** → checked는 `rollbackFor` 명시.

---

## 8. 참고
- [Spring - Declarative Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html)
- [Baeldung - Transaction Propagation and Isolation in Spring @Transactional](https://www.baeldung.com/spring-transactional-propagation-isolation)
- 관련 노트: [영속성 컨텍스트](../jpa/persistence-context.md) · [Read-Modify-Write와 트랜잭션 경계](../jpa/read-modify-write.md) · [@Lock 실무 패턴](../jpa/lock-practical.md)

---

**학습 날짜**: 2026-05-27
**계기**: read-modify-write 공부 중 `REQUIRES_NEW`에서 inner/outer 트랜잭션이 어떻게 갈리고 에러 시 롤백이 어떻게 전파되는지 궁금해서 `@Transactional` 전반을 정리

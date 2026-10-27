# Lombok 생성자 애너테이션 - access·staticName·@Builder

> `@NoArgsConstructor`·`@RequiredArgsConstructor`·`@AllArgsConstructor`는 생성자를 대신 써 준다. `access`는 접근 수준이고, `staticName`을 주면 생성자는 private으로 숨기고 같은 인자의 정적 팩토리를 하나 더 만든다. ⚠️ **둘을 같이 쓰면 `access`는 생성자가 아니라 정적 팩토리에 붙는다.** 그래서 `access = PRIVATE, staticName = "of"`는 `of`까지 private이 되어 밖에서 아무것도 못 만든다.

## 언제 쓰나

| 상황 | 흔한 조합 |
|---|---|
| 스프링 빈 생성자 주입 | `@RequiredArgsConstructor` (final 필드 + `@NonNull` 필드만 인자로) |
| JPA 엔티티의 기본 생성자 | `@NoArgsConstructor(access = AccessLevel.PROTECTED)` (JPA 명세: 인자 없는 생성자는 public 또는 protected) |
| 도메인 모델을 정적 팩토리로만 만들게 하기 | `@AllArgsConstructor(access = AccessLevel.PRIVATE)` + 직접 쓴 `create`/`of` |
| 검증 없는 단순 값 묶음 | `@AllArgsConstructor(staticName = "of")` |

## 사용 예시 — 무엇이 만들어지나

```java
@Getter
@AllArgsConstructor(staticName = "of")
public class Money {
    private String currency;
    private long amount;
}
```

Lombok이 컴파일 중에 만들어 넣는 코드(개념상):

```java
private Money(String currency, long amount) {   // staticName이 있으면 생성자는 무조건 private
    this.currency = currency;
    this.amount = amount;
}

public static Money of(String currency, long amount) {   // 이 메서드의 접근 수준 = access (기본 PUBLIC)
    return new Money(currency, amount);
}
```

## 옵션 조합 비교

| 애너테이션 | 생성자 | 정적 팩토리 `of` | 밖에서 만들 수 있나 |
|---|---|---|---|
| `@AllArgsConstructor` | public | 없음 | `new` 가능 |
| `@AllArgsConstructor(access = PRIVATE)` | **private** | 없음 | 못 만듦 → 직접 쓴 팩토리가 필요 |
| `@AllArgsConstructor(staticName = "of")` | private | **public** | `of(...)`로만 |
| `@AllArgsConstructor(access = PRIVATE, staticName = "of")` | private | **private** | ⚠️ 클래스 밖에서 아무 방법도 없음 |

## 함정

### ⚠️ `access`와 `staticName`을 같이 쓰면 팩토리가 private이 된다

`access`의 Javadoc은 "생성자의 접근 수준"이라고만 적혀 있어서, `access = PRIVATE`이 생성자에 붙고 `of`는 public일 거라 기대하기 쉽다. 실제 처리 코드(1.18.36 `HandleConstructor`)는 이렇다.

```java
// 생성자: staticName이 있으면 access 값과 상관없이 PRIVATE
createConstructor(staticConstrRequired ? AccessLevel.PRIVATE : level, ...);
// 정적 팩토리: access 값(level)을 그대로 받는다
generateStaticConstructor(staticConstrRequired, typeNode, staticName, level, ...);
```

- 결과: `access = PRIVATE`을 주면 `of`가 private이 된다. 다른 클래스(유스케이스·테스트)에서 `Money.of(...)`를 부르면 **컴파일 에러**가 난다.
- 해결: `staticName`을 쓸 거면 `access`는 빼면 된다. 생성자는 어차피 private으로 숨겨진다.
- 이렇게 헷갈리는 이유: 팀 컨벤션이 "모델은 `@AllArgsConstructor(access = PRIVATE)` + 정적 팩토리"일 때, 팩토리를 손으로 쓰는 대신 `staticName`만 덧붙이면 두 규칙이 겹치면서 생긴다.

### ⚠️ `staticName` 팩토리에는 검증을 넣을 수 없다

`of`는 생성자를 그대로 감싸기만 한다. null 검사, 기본값 채우기, 상태 초기화(예: `status = PENDING`) 같은 일이 필요하면 private 생성자 + **손으로 쓴 팩토리**로 가야 한다.

```java
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class Order {
    private String id;
    private OrderStatus status;

    public static Order create(String id) {
        Objects.requireNonNull(id, "id");
        return new Order(id, OrderStatus.PENDING);   // 생성 규칙이 팩토리에 있다
    }
}
```

### ⚠️ `@Builder`는 다른 생성자 애너테이션이 있으면 생성자를 만들지 않는다

`@Builder`를 클래스에 붙이면 `build()`가 모든 필드를 받는 생성자를 부른다. 그 생성자는 **클래스에 `@XxxArgsConstructor`도, 직접 쓴 생성자도 없을 때만** package-private으로 자동 생성된다 (1.18.36 `HandleBuilder` → `SkipIfConstructorExists.I_AM_BUILDER`).

- `@Builder` + `@NoArgsConstructor`만 붙이면: 기본 생성자가 있으니 전체 인자 생성자는 안 만들어지고 → `build()`가 부를 생성자가 없어 컴파일 에러.
- 그래서 JPA 엔티티나 모델에서 `@Builder` + `@NoArgsConstructor` + `@AllArgsConstructor` 세 개가 같이 붙어 다니는 경우가 많다. 이때 `@AllArgsConstructor`가 public이면 빌더를 우회한 `new`도 열린다 → 막고 싶으면 `access = PRIVATE`.

## 💡 판단 기준

- **생성할 때 지킬 규칙(검증·기본값·초기 상태)이 하나라도 있으면** → private 생성자 + 직접 쓴 팩토리. 규칙이 팩토리 한 곳에 모인다.
- **규칙 없는 단순 값 묶음이면** → `staticName = "of"`만. `access`는 같이 쓰지 않는다.
- 생성된 코드가 의심되면 추측하지 말고 확인한다: IntelliJ의 *Delombok*, 또는 `javap -p 클래스파일`로 생성된 메서드의 접근 수준을 본다.

## 참고

- Lombok 기능 문서: https://projectlombok.org/features/constructor
- Lombok `@Builder` 문서: https://projectlombok.org/features/Builder
- 소스 확인: `lombok-1.18.36-sources.jar` — `lombok/javac/handlers/HandleConstructor.java`(`generate`), `HandleBuilder.java`
- JPA 명세(Jakarta Persistence) 2.1 Entity Class: 인자 없는 생성자는 public 또는 protected
- 관련: [record 컴팩트 생성자](./record-compact-constructor.md) — record로 같은 일을 할 때의 생성 규칙

---
학습 날짜: 2026-10-27 · 계기: 모델에 `@AllArgsConstructor(access = PRIVATE, staticName = "of")`를 붙였는데, 소스를 보니 `of`까지 private이 되어 테스트·유스케이스에서 부를 수 없는 구조였음

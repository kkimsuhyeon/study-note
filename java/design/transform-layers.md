# 변환 계층 — Factory / Mapper / Assembler (+ Command/Query)

> **한 줄 요약**: 헷갈리는 셋. **Mapper**(web: Request→Command)와 **Assembler**(app: Command→Domain Model)는 **둘 다 "변환"**인데 계층·대상이 다르고, **Factory**는 "변환"이 아니라 **도메인 객체 "생성"**(기본값·불변식)이다. Assembler가 안에서 Factory를 호출한다.

관련 노트: [도메인 검증 위치](./domain-validation.md)

---

## 1. 흐름 (한눈에)

```
Request (web DTO)
   │  ── Mapper ──>      [adapter/in/web/mapper]   HTTP 모양 → 안쪽 의도 언어
   ▼
Command / Query          [application/dto]
   │  ── Assembler ──>   [application/assembler]   의도 → 진짜 도메인 객체로 조립
   ▼
Domain Model (User)  ← 생성은 Factory(User.create), DB에서 되살리는 복원은 User.of
```

## 2. 3개 구분

| | 위치 | 역할 | 예시 |
|---|---|---|---|
| **Mapper** | `adapter/in/web/mapper` | Request → **Command/Query** (변환, web 계층) | `CreateUserRequest → CreateUserCommand` |
| **Assembler** | `application/assembler` | Command/Query → **Domain Model** (변환, app 계층) | `CreateUserCommand → User` |
| **Factory** | 도메인(모델 내 static) | 값 → **유효한 도메인 객체 *생성*** (기본값·불변식·검증) | `User.create(email, pw)` — 복원 팩토리 `User.of(...)`는 저장된 상태 재조립이라 입력 검증 X ([도메인 검증 §2-3](./domain-validation.md)) |

```java
// Assembler가 Factory를 호출 — "변환"이 "생성"을 부른다
public static User toModel(CreateUserCommand command) {
    return User.create(command.getEmail(), command.getPassword());  // ← Factory (잔액0·USER는 여기서)
}
```

## 3. 핵심 구분

- **Mapper vs Assembler** = 둘 다 변환, **계층·대상만 다름**:
  - Mapper: web 경계 (HTTP Request → Command/Query)
  - Assembler: app 경계 (Command/Query → 도메인 Model)
- **Factory** = 변환이 아니라 **생성**. Mapper/Assembler가 "옮긴다"면 Factory는 "만든다"(유효한 객체로).

## 4. 왜 Mapper/Assembler를 나눴나
- Mapper = "바깥세상(HTTP)"을 안쪽 언어로 번역하는 **어댑터 책임**.
- Assembler = 그 Command를 진짜 도메인으로 조립하는 **application 책임**.
- → **웹이 GraphQL로 바뀌어도 Assembler는 그대로** 써야 함. 그게 분리 목적(어댑터 교체에 도메인 조립이 안 흔들림).
- ⚠️ 이건 application이 web 타입을 import하지 않는 구조(헥사고날) 기준이다. 계층형 구조의 작은 프로젝트는 Request를 그대로 서비스에 넘기는 것도 흔하다 — Command(번역)로 올라갈 신호는 [책임 경계 §3-1](./responsibility-boundaries.md)에 모아뒀다.

## 5. 곁가지 — Command vs Query
| | 의미 | 트랜잭션 | 서비스 |
|---|---|---|---|
| **Command** | 상태 바꾸는 **쓰기** | `@Transactional` | `UserCommandService` |
| **Query** | 상태 안 바꾸는 **읽기** | `readOnly = true` | `UserQueryService` |

### ⚠️ 기준은 "값을 돌려주나"가 아니라 "부작용이 있나" — `findByIdForUpdate`는 Command

CQS(Command-Query Separation) 정의: Query는 값을 돌려주고 **아무것도 바꾸지 않는다**. `SELECT ... FOR UPDATE`는 데이터를 돌려주지만 커밋까지 행을 잠가서 **다른 트랜잭션의 동작을 바꾼다**. 그러니 Command 쪽이다. JDK의 `Queue.poll()`(꺼내면서 반환)과 `peek()`(들여다보기만)의 차이와 같다.

잠금 메서드를 `readOnly = true` Query 서비스에 두면 이렇게 깨진다.

- **이름표를 믿고 부른다.** "Query는 어디서 불러도 안전하다"고 보고 읽기 API나 반복문에서 부르면, 읽기 요청이 쓰기 요청을 줄 세우는 잠금을 건다.
- **혼자 불리면 잠금이 즉시 풀린다.** 바깥 `@Transactional` 안에서 불리면 그 트랜잭션에 합류해 커밋까지 잠금이 유지된다. 트랜잭션 없는 곳에서 불리면 자기 readOnly 트랜잭션을 열고 메서드가 끝날 때 커밋하면서 잠금을 바로 놓는다. 에러가 없어서 조용히 아무것도 보호하지 않는다.
- **읽기 전용 복제 DB로 라우팅하는 구성이면 잠금이 엉뚱한 서버에 걸린다.** `readOnly` 트랜잭션을 replica로 보내는 구성(`AbstractRoutingDataSource` 등)에서는 `FOR UPDATE`가 원본이 아닌 복제본에서 실행돼, 원본의 쓰기를 막지 못한다.

💡 판단 질문: **"실수로 두 번 부르거나 읽기 API에서 불러도 아무 일 없나?"** 아니면 Command다. 이름에 `ForUpdate`·`Exclusive`·`Lock`이 붙은 메서드는 Command 서비스로 간다. 잠금 노트: [@Lock 실무](../jpa/lock-practical.md)

---

## 5-1. ⚠️ 왜 엔티티를 API로 직접 반환하면 안 되나 — 설계 이유 + JPA 기술 근거

설계 이유(스펙 종속)만이 아니라 **기술적으로도 터진다** (김영한 활용2):

- **프록시 직렬화 예외**: 지연 로딩 상태의 연관 필드는 프록시 객체라 Jackson이 직렬화 못 하고 예외. `Hibernate5Module`(Hibernate 5.5+ Jakarta는 `Hibernate5JakartaModule`, 부트 3.x의 Hibernate 6은 `jackson-datatype-hibernate6`의 `Hibernate6Module` — Jackson 2.15+)로 우회 가능하지만 미초기화 필드가 null로 나가는 등 결국 **DTO 변환이 정답**.
- **양방향 무한루프**: 양방향 연관관계 엔티티를 그대로 JSON화하면 서로를 호출하며 무한루프 — 한쪽에 `@JsonIgnore`가 필요해지는 것 자체가 "화면 사정이 엔티티에 침투"한 신호.
- **엔티티 필드 추가 = API 스펙 변경**: 내부 리팩토링이 외부 계약을 깨뜨린다.

덤 (API 응답 설계 소품): **컬렉션을 JSON 최상위 배열 `[...]`로 반환하지 말 것** — count 등 필드 추가가 불가능해진다. 항상 `{ "data": [...] }` 래퍼로. / 부분 수정 API에 PUT은 부적절(PUT=전체 교체) — PATCH/POST가 맞다.

## 5-2. Request를 enum으로 바꾸면 Command도 바꿔야 하나?

**각 계층이 같은 의미의 값을 다룬다면 타입도 이어서 유지하는 편이 단순하다.** DTO를 분리한다는 이유만으로 필드까지 모두 문자열로 되돌릴 필요는 없다.

```text
HTTP JSON "online"
  → Jackson이 BookingType.ONLINE으로 변환
  → Request.bookingType: BookingType
  → Command.bookingType: BookingType
  → Domain.bookingType: BookingType
```

enum은 허용된 선택지를 표현하는 Java 타입이다. 중간에 `.name()`으로 문자열을 만들었다가 `valueOf()`로 복원하면 불필요한 변환과 실패 지점이 생긴다.

파일은 각각 독립적으로 둔다.

```java
// domain/BookingType.java
public enum BookingType {
    ONLINE, OFFLINE
}
```

```java
// web/CreateBookingRequest.java
public record CreateBookingRequest(
        @NotNull BookingType bookingType,
        @NotNull LocalDate bookingDate,
        @NotNull LocalTime bookingTime
) {
}
```

```java
// application/CreateBookingCommand.java
public record CreateBookingCommand(
        BookingType bookingType,
        LocalDate bookingDate,
        LocalTime bookingTime
) {
}
```

```java
return new CreateBookingCommand(
        request.bookingType(),
        request.bookingDate(),
        request.bookingTime()
);
```

위 예시는 타입 전달 부분이며 `java.time`·`jakarta.validation`·해당 도메인 타입 import를 생략했다. HTTP의 enum 소문자 표현은 별도 Jackson 설정으로 정할 수 있다. [Jackson 노트](../jackson/annotations.md)

### 도메인 타입을 Request가 참조해도 되나?

이 구조에서 web이 domain 타입을 참조하는 것은 바깥에서 안쪽으로의 의존이다. **도메인이 Request·Controller를 참조하는 것과 방향이 다르다.** API와 업무의 선택지가 정확히 같고 함께 바뀌어도 괜찮다면 같은 enum을 사용할 수 있다.

반대로 외부 API의 `"remote"`를 내부의 `ONLINE`으로 바꾸어야 하거나 API 버전별 선택지가 다르면 별도 웹 enum·변환이 의미 있다. 분리의 기준은 클래스 개수가 아니라 **각 계층에서 값의 의미와 변경 주기가 다른가**다.

### 날짜처럼 보여도 LocalDate가 맞지 않을 때

`LocalDate`는 시간대 없는 **ISO 달력 날짜**다. 임의의 달력 날짜를 담는 용기가 아니다. 예를 들어 음력 날짜의 월·일은 ISO 달력과 유효성 규칙이 달라서 유효한 음력 2월 30일을 `LocalDate`로 표현할 수 없다.

이런 입력은 문자열과 달력 종류로 받거나 연·월·일을 가진 전용 값 객체로 표현한다. 반면 일반 예약일처럼 ISO 날짜가 맞는 경우에는 `LocalDate`를 쓰면 문자열 파싱을 줄일 수 있다. [Java 25 LocalDate](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/time/LocalDate.html)

💡 **HTTP 표기만 다르면 경계에서 변환하고, 안쪽에서 같은 의미를 쓰는 enum·시간은 그대로 전달한다. Factory·Mapper·Assembler를 모두 별도 클래스로 만들어야 하는 것은 아니다.**

## 5-3. 도메인 안의 `XxxInput` — 계산 단계마다 입력을 "좁히는" 값

Request·Command는 **계층 경계를 넘기 위한** 모양 바꾸기다. 반면 도메인 모델 중에 `XxxInput`이라는 이름이 보이면 대개 **그 도메인 계산의 입력값**이다. 짝이 되는 출력이 있다:

```text
주문 요청 원문(OrderInput: 이름·주소·전화·카드·상품)        ← 사용자가 넣은 것 전부, 저장용
   └ toPricingInput() → PricingInput(상품·수량·쿠폰)        ← 가격 계산에 필요한 것만
                           └ 가격 계산 → Price
   └ toShippingInput() → ShippingInput(주소·무게)           ← 배송 계산에 필요한 것만
                           └ 배송 계산 → ShippingQuote
```

- **좁히기(최소 데이터):** 각 계산은 필요한 필드만 받는다. 가격 계산이 이름·전화번호를 받을 통로 자체가 없으니, 외부(AI·다른 서비스)로 보내는 단계에서 개인정보가 새지 않는다는 걸 **타입이 보장**한다.
- **경계 검증:** 입력값이 자기 불변식을 생성자에서 검사한다(있는 날짜인지, 허용된 키만 있는지). 계산 코드는 "이미 맞는 값"만 다룬다.
- **의존 방향:** 큰 쪽(주문)이 작은 쪽 입력으로 바꿔 주고(`toPricingInput()`), 작은 쪽(가격)은 주문을 모른다. 그래서 가격 계산을 다른 곳에서도 재사용할 수 있다.
- Request → Command → `OrderInput`처럼 거의 같은 모양이 여러 번 반복되면 그건 계층 규칙 때문에 생긴 비용이다(위 4절). 반면 `OrderInput → PricingInput`은 **필드가 줄어드는** 변환이라 목적이 다르다. 둘을 구분해서 보면 "왜 이렇게 많이 바꾸나"가 정리된다.
- **Request와 똑같아 보이는 도메인 값(`OrderInput`)은 왜 있나 — 요청보다 오래 살기 때문이다.** Request는 HTTP 요청 하나가 끝나면 사라지지만, 사용자가 넣은 원문은 주문에 저장되고(DB), 조회 응답에 다시 나가고, 나중에 HTTP가 없는 배치·워커가 읽어서 계산한다. 도메인 모델(주문)은 웹 DTO를 필드로 가질 수 없으니(도메인 → 웹 의존 금지) 원문을 담을 도메인 타입이 필요하다.
  - 바뀌는 이유가 다르다: Request는 **API 모양**(JSON 필드 이름, v2 API, 다른 앱의 요청 형식)이 바뀔 때, 도메인 값은 **저장 형식·업무 규칙**(기본값, 공백 정리, 유효성)이 바뀔 때 바뀐다. 지금 필드가 같아도 둘이 같이 움직일 필요는 없다.
  - 검증도 층이 다르다: Request의 `@NotNull`·`@Pattern`은 "요청 모양이 맞나"(빠른 400), 도메인 값의 생성자는 "업무적으로 유효한가"(어디서 만들든 항상).
  - 정규화(기본값·공백 정리)를 도메인 값에서 하면 해시·중복 비교·저장이 모두 같은 정규화된 값을 쓴다.

**그럼 도메인 모델을 그냥 넘기면 안 되나? — 모델끼리 아는 것 자체는 문제가 아니다.** 주문이 가격 결과를 필드로 갖고, 가격 입력이 상품 모델을 아는 건 자연스럽다. 따질 것은 세 가지뿐이다.
1. **방향:** 범용·핵심 쪽(가격 계산)이 구체적·큰 쪽(주문)을 알게 되면 안 된다. `pricing.calculate(order)`로 넘기면 가격 계산이 주문 모델에 묶여 다른 곳에서 못 쓰고, 테스트도 이름·주소를 채운 주문을 만들어야 한다. 순환은 당연히 금지.
2. **외부 경계:** AI·다른 서비스·HTTP로 나가는 지점에 큰 모델을 통째로 넘겨 직렬화하면, 나중에 그 모델에 추가된 필드까지 **자동으로 밖에 나간다.** 나가는 모양을 전용 값으로 고정하면 "무엇이 나가는가"가 코드에 명시된다.
3. **필요한 양:** 받는 쪽이 대부분의 필드를 쓰고 같은 영역 안이며 외부 경계도 아니면, 그냥 넘기는 게 맞다. 변환 타입을 만드는 비용(파일·변환 코드·테스트)이 이득보다 크다.

## 6. 💡 판단
> **"옮기냐 만드냐"로 먼저 가른다.** 옮기기(변환)면 어느 경계냐 — web면 Mapper, app이면 Assembler. 만들기(생성·기본값·불변식)면 Factory. 1:1 단순 복사라 로직이 없으면 그 변환 계층은 테스트도 스킵(프레임워크/단순 위임).

---

## 7. 참고
- 관련 노트: [도메인 검증 위치](./domain-validation.md) · [책임 경계](./responsibility-boundaries.md)
- [jackson-datatype-hibernate README](https://github.com/FasterXML/jackson-datatype-hibernate) — Hibernate 버전별 모듈(5 / 5-jakarta / 6)

- 보강: 2026-10-02. Hibernate 6용 Jackson 모듈 정정, 복원 팩토리 `of`는 검증하지 않는다는 점을 도메인 검증 노트와 맞춤, Request 직접 전달이 허용되는 구조 표시.
- 보강: 2026-09-22. Request·Command·도메인 사이의 타입 일관성, enum 공유의 의존 방향, 달력 의미에 따른 LocalDate 선택. 기존 명칭 표는 한 가지 프로젝트 관례이며 모든 프로젝트가 따라야 하는 표준은 아니다. 예시는 설명용으로 별도 실행하지 않았다.

---

**학습 날짜**: 2026-06-08
**계기**: 코드 보다 Factory/Mapper/Assembler 차이가 기억 안 나서 — "변환 2단(Mapper:web→Command, Assembler:Command→Model) + 생성 1개(Factory)"로 정리. Mapper vs Assembler는 계층 차이, Factory는 변환이 아니라 생성.

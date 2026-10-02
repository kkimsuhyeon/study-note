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
Domain Model (User)  ← 생성은 Factory(User.create / User.of)가 담당
```

## 2. 3개 구분

| | 위치 | 역할 | 예시 |
|---|---|---|---|
| **Mapper** | `adapter/in/web/mapper` | Request → **Command/Query** (변환, web 계층) | `CreateUserRequest → CreateUserCommand` |
| **Assembler** | `application/assembler` | Command/Query → **Domain Model** (변환, app 계층) | `CreateUserCommand → User` |
| **Factory** | 도메인(모델 내 static) | 값 → **유효한 도메인 객체 *생성*** (기본값·불변식·검증) | `User.create(email, pw)`, `User.of(...)` |

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

## 5. 곁가지 — Command vs Query
| | 의미 | 트랜잭션 | 서비스 |
|---|---|---|---|
| **Command** | 상태 바꾸는 **쓰기** | `@Transactional` | `UserCommandService` |
| **Query** | 상태 안 바꾸는 **읽기** | `readOnly = true` | `UserQueryService` |

---

## 5-1. ⚠️ 왜 엔티티를 API로 직접 반환하면 안 되나 — 설계 이유 + JPA 기술 근거

설계 이유(스펙 종속)만이 아니라 **기술적으로도 터진다** (김영한 활용2):

- **프록시 직렬화 예외**: 지연 로딩 상태의 연관 필드는 프록시 객체라 Jackson이 직렬화 못 하고 예외. `Hibernate5Module`(부트 3.x는 `Hibernate5JakartaModule`)로 우회 가능하지만 미초기화 필드가 null로 나가는 등 결국 **DTO 변환이 정답**.
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

## 6. 💡 판단
> **"옮기냐 만드냐"로 먼저 가른다.** 옮기기(변환)면 어느 경계냐 — web면 Mapper, app이면 Assembler. 만들기(생성·기본값·불변식)면 Factory. 1:1 단순 복사라 로직이 없으면 그 변환 계층은 테스트도 스킵(프레임워크/단순 위임).

---

## 7. 참고
- 프로젝트 규칙 원본: server-java `docs/CONVENTIONS.md §1` (변환 계층)
- 관련 노트: [도메인 검증 위치](./domain-validation.md)

- 보강: 2026-09-22. Request·Command·도메인 사이의 타입 일관성, enum 공유의 의존 방향, 달력 의미에 따른 LocalDate 선택. 기존 명칭 표는 한 가지 프로젝트 관례이며 모든 프로젝트가 따라야 하는 표준은 아니다. 예시는 설명용으로 별도 실행하지 않았다.

---

**학습 날짜**: 2026-06-08
**계기**: 코드 보다 Factory/Mapper/Assembler 차이가 기억 안 나서 — "변환 2단(Mapper:web→Command, Assembler:Command→Model) + 생성 1개(Factory)"로 정리. Mapper vs Assembler는 계층 차이, Factory는 변환이 아니라 생성.

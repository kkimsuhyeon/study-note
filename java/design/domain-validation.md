# 도메인 검증 위치 — 엔티티 / 도메인 서비스 / 앱 서비스

> **한 줄 요약**: "모든 로직을 엔티티에"는 DDD의 흔한 오해. 엔티티가 **자기 상태로 판단할 수 있는 불변식**(잔액≥0 등)은 엔티티에(rich model), **여러 행을 봐야 하는 규칙**(이메일 유니크 등)은 엔티티가 구조상 불가 → **도메인 서비스 / 체커 주입**으로(도메인 계층이지만 엔티티는 아님). 핵심 구분: **규칙(결정)은 도메인, 데이터 조회는 인프라.**

관련 노트: [Read-Modify-Write와 트랜잭션 경계](../jpa/read-modify-write.md) · [Mockito 서비스 테스트](../test/mockito-service-test.md)

---

## 1. 흔한 오해 — "DDD = 모든 로직을 엔티티에"

반은 맞고 반은 틀린 말이다.

- ✅ **맞는 알맹이**: **Anemic(빈약) 도메인 모델** 안티패턴을 피하라. 엔티티가 getter/setter 덩어리고 로직이 전부 서비스에 있으면 객체지향이 아니다. 엔티티는 **자기 로직을 가져야**(rich) 한다.
- ❌ **틀린 일반화**: "**전부** 엔티티에"는 DDD가 말한 적 없다. DDD엔 **도메인 서비스**라는 빌딩블록이 따로 있다 — 한 엔티티에 안 속하는 로직을 담는 도메인 계층 객체.

---

## 2. 검증은 두 종류 — 어디 두냐가 갈린다

| 종류 | 자기 상태로 판단? | 예 | 위치 |
|---|---|---|---|
| **단일 엔티티 불변식** | ✅ 가능 | 잔액 ≥ 0, 상태 전이(RESERVING→RESERVED) | **엔티티 안** |
| **집합/교차 규칙** | ❌ 불가 (다른 행을 봐야) | 이메일 **유니크**, 계좌 간 이체 | **도메인 서비스** |

이 프로젝트 `User`는 이미 단일 불변식을 rich하게 처리한다:
```java
public void deductBalance(BigDecimal amount) {
    if (this.balance.compareTo(amount) < 0)        // 자기 잔액만 보면 판단 가능 → 엔티티 안
        throw new BusinessException(NOT_ENOUGH_POINT);
    this.balance = this.balance.subtract(amount);
}
```
→ **유니크만 예외.** `User` 하나는 자기 옆 유저들을 몰라서 "내 이메일이 유일한가"를 **혼자 판단 못 한다.**

### 2-1. 생성 검증의 실제 배치 — 검증 3층 + "변환되는 값" 함정

`User.create`(팩토리) + `UserRegistration`(도메인 서비스) 구조에서 검증이 실제로 놓이는 곳:

| 층 | 검증 | 성격 |
|---|---|---|
| 웹 `@Valid` (Request DTO) | 형식·**비번 정책**(길이·특수문자) | **입력 거부선** — 빠른 400, UX용 |
| 팩토리 `User.create` | null/빈값, 이메일 형식 등 **값만 보고 판단 가능한 것** | **최후 방어선** — 어디서 부르든 깨진 객체 생성 불가(우회 불가능한 강제) |
| 도메인 서비스 `register` | 이메일 **유니크** (옆 행 필요) | 집합 규칙 — 약속 기반(우회 가능) |

- 웹 `@Valid`와 팩토리 검증은 **중복이 아니다** — 웹은 UX(친절한 에러), 팩토리는 무결성(웹 안 거치는 배치·내부 호출·테스트에서도 방어).
- 강제 수준이 층마다 다름: **팩토리 검증 = 컴파일/런타임 강제**(우회 불가), **도메인 서비스 검증 = 약속**(`User.create` 직접 호출 시 우회됨, §7).

> ⚠️ **변환되는 값은 검증 가능 지점이 앞으로 당겨진다 — 비밀번호 정책은 `User.create`에서 못 한다.** `create`에 도착하는 비번은 **이미 해시**(bcrypt는 60자 고정)라 "8자 이상·특수문자" 같은 정책 검증은 원본이 사라져 물리적으로 불가능. raw가 존재하는 마지막 지점(웹 `@Valid`/앱 서비스, 인코딩 **전**)에서 해야 한다. `create`가 할 수 있는 건 null/빈값 가드뿐. 반면 **이메일은 변환 없이 원본 그대로** 들어오므로 형식 검증까지 `create`에서 가능 — 같은 "필수값"이라도 **중간에 변환(해시)이 끼면 검증 위치가 갈린다.**

### 2-2. "검증이 너무 뿌려지는데?" — 배치 기준 + VO로 응집

§2-1대로 하면 비번 정책=앱, 이메일 형식=팩토리, 유니크=도메인 서비스로 **흩어져 보인다.** 두 가지로 답:

**(1) 무작위로 흩어진 게 아니다 — 배치 기준은 "그 검증에 필요한 정보가 어디 있나" 하나다.**
| 검증 | 필요한 정보 | 정보가 있는 곳 |
|---|---|---|
| 비번 정책 | raw 비번 | 인코딩 전 = 웹/앱 경계 (이후 소멸) |
| 이메일 형식 | 값 자체 | 어디든 → 도메인 가능 |
| 유니크 | 다른 행들 | repo 너머 → 도메인 서비스 |

정보가 없는 곳에선 검증 자체가 불가능 → 분산은 취향이 아니라 **제약의 결과**.

**(2) 그래도 모으고 싶으면 — Value Object가 DDD의 표준 답.** 검증을 계층이 아니라 **값의 타입 안**에 가둔다:
```java
public record Email(String value) {
    public Email { if (!value.matches("^[^@]+@[^@]+$")) throw new BusinessException(INVALID_EMAIL); }
}
public record RawPassword(String value) {
    public RawPassword { if (value.length() < 8) throw new BusinessException(INVALID_PASSWORD); }
}
// 앱 서비스 — 검증 로직이 사라지고 "타입 생성"만 남음
Email email = new Email(command.getEmail());              // 형식 검증 자동
RawPassword raw = new RawPassword(command.getPassword()); // 정책 검증 자동 ← 비번 정책이 도메인(VO)으로 내려옴!
String encoded = passwordEncoder.encode(raw.value());     // 인코딩(메커니즘)만 앱에 남음
```
- **유효하지 않은 값은 타입으로 존재 불가** — `Email` 인스턴스가 있다 = 형식이 맞다. `User.create(Email, ...)`로 받으면 재검증 불필요(타입이 증명). → "**parse, don't validate**" / "make illegal states unrepresentable".
- VO 후 남는 분산은 유니크(도메인 서비스)뿐 — 옆 행이 필요해서 어쩔 수 없는 정당한 분리.
- ⚠️ 비용: 클래스 증가 + `.value()` 변환 노이즈. 필드 2개짜리 모델엔 과할 수 있다(도메인 서비스 때와 같은 YAGNI 결). 웹 `@Valid`는 UX용으로 중복 유지 OK.

> 💡 **검증 분산이 불편하면 계층이 아니라 타입(VO)으로 모은다.** 경계에서 한 번 파싱해 "유효함이 보장된 타입"으로 바꾸면, 안쪽은 검증을 반복하지 않고 타입을 믿는다. 남는 분산(유니크)만이 진짜 구조적 분산.

| 빌딩블록 | 역할 | 비즈니스 로직? |
|---|---|---|
| **Entity** | 식별자 + 생명주기 + **자기 상태 불변식** | ✅ (rich) |
| **Value Object** | 불변 값 + 자기 검증 | ✅ |
| **Domain Service** | **한 엔티티에 안 속하는** 도메인 로직(유니크·이체 등) | ✅ (도메인 계층) |
| **Application Service** | 흐름 조합·트랜잭션 경계 | ❌ (오케스트레이션만) |
| **Repository** | 영속화 추상(포트) | ❌ |

> 유니크가 엔티티 밖에 있는 건 "DDD 위반"이 아니라, **엔티티가 구조상 못 하는 일**이라 도메인 서비스가 맡는 것. (Vaughn Vernon "Implementing DDD"도 유니크를 도메인 서비스 케이스로 다룸)

---

## 2-3. 생성 팩토리(`create`) vs 복원 팩토리(`of`) — 입력 검증은 "생성 경로"에만

§2-1이 검증을 *어느 계층*에 둘지였다면, 도메인 모델의 **정적 팩토리가 둘로 갈릴 때**(`create` / `of`) "*어느 팩토리*에 둘지"가 또 갈린다. 둘은 이름만 비슷할 뿐 역할이 다르다.

| 팩토리 | 정체 | id | 누가 부르나 |
|---|---|---|---|
| **`create(...)`** | 새 도메인 객체 **생성** (기본값·`PENDING` 등 부여) | `null` | 비즈니스 흐름 (서비스/Assembler) |
| **`of(...)`** | 전체 필드 **복원**(reconstruction) | 있음 | **`Entity.toModel()`**(DB→도메인) + `create`가 내부 위임 |

> 핵심: **`of`는 단순 생성자 대용이 아니라 *영속성 복원 경로*다.** `PaymentEntity.toModel()`이 `Payment.of(...)`를 부른다 → `of`는 "DB에 이미 있는 행을 도메인 객체로 되살리는" 입구이기도 하다.

### ⚠️ 그래서 입력 검증을 `of`에 두면 안 된다 (concrete: `Payment`)

```java
// 현재 코드 — 검증이 of(복원 경로)에 있다
public static Payment create(String reservationId, BigDecimal amount) {
    return Payment.of(null, PENDING, amount, reservationId, null);   // create가 of에 위임
}
public static Payment of(String id, PaymentStatus status, BigDecimal amount, String reservationId, String rmk) {
    if (StringUtils.isEmpty(reservationId))
        throw new BusinessException(CommonErrorCode.INVALID_INPUT);  // ⚠️ 복원 경로에 입력 검증
    return new Payment(id, status, amount, reservationId, rmk);
}
```

- **의미 충돌** — `INVALID_INPUT`은 "사용자 입력이 틀림"(보통 400 Bad Request) 뜻이다. 그런데 `of`는 **DB에 이미 저장된 결제를 `toModel()`로 되살릴 때도** 불린다 → *결제를 조회하는데 400 "입력 오류"가 터지는* 꼴. 깨진 DB 행은 "잘못된 입력"이 아니라 **데이터 정합성 문제**라 던질 예외의 종류부터 다르다.
- **중복** — `reservation_id`는 이미 `@Column(nullable = false)`라 DB가 not-null을 보장. 복원 시 재검증은 군더더기.
- **역할 위반** — 복원 팩토리는 *"있는 상태를 그대로 재조립"*하는 역할이지 거부하는 역할이 아니다. (cf. [변환 계층](./transform-layers.md): Factory는 "만든다", 복원도 그 일종)

→ **입력 검증은 "새 객체가 입력으로부터 태어나는 생성 경로"에 둔다.**

### 옵션 — 검증을 생성 경로로 옮기는 두 방법

| 방식 | 설명 | 언제 |
|---|---|---|
| **(A) `create`에만 검증** | `of`는 순수 복원으로 비움 | 생성 팩토리가 **하나**면 충분 |
| **(B) 생성 공용 통로(private)에 검증** | `create`·(미래)`fail` 등 모든 생성 팩토리가 공유, `of`(복원)는 제외 | 생성 팩토리를 **더 만들 계획**이면 |

```java
// (B) — 모든 "신규 생성"은 검증을 타고, 복원(of)은 안 탄다
private static Payment newPayment(PaymentStatus status, BigDecimal amount, String reservationId, String rmk) {
    if (StringUtils.isEmpty(reservationId)) throw new BusinessException(CommonErrorCode.INVALID_INPUT);
    return new Payment(null, status, amount, reservationId, rmk);
}
public static Payment create(String reservationId, BigDecimal amount) {
    return newPayment(PENDING, amount, reservationId, null);
}
// public static Payment fail(...) { return newPayment(FAIL, ...); }  // 미래 추가 시도 같은 통로
public static Payment of(String id, PaymentStatus status, BigDecimal amount, String reservationId, String rmk) {
    return new Payment(id, status, amount, reservationId, rmk);   // 복원 — 검증 X
}
```

> ⚠️ 생성 팩토리를 더 추가할 계획(`Payment.fail()` 등)이면 (B)를 택한다. **"단일 choke point에 불변식"** 이점은 살리되, 그 choke point를 **복원(`of`)과 분리**하는 게 요지. `create`에만 넣으면 나중에 만든 `fail`이 검증을 조용히 빠뜨린다.

이 프로젝트 `User`도 `create`/`of`가 둘 다 있고 **`of`는 검증하지 않는다**(`toModel`이 부르는 복원 경로라서) — 같은 패턴. 즉 `Payment`의 현재 코드가 컨벤션에서 벗어난 쪽이다.

> 💡 **입력 검증은 "복원 팩토리(`of`/`toModel` 경로)"가 아니라 "생성 팩토리(`create`) 경로"에 둔다.** `of`가 영속성 복원에 쓰이면 거기서 던지는 `INVALID_INPUT`은 "DB 읽다가 400" 식 의미 충돌 + `nullable` 제약과 중복이다. 생성 팩토리가 여러 개거나 늘어날 거면, 검증은 **복원과 분리된 private 생성 통로**에 모아 모든 신규 객체가 거치게 한다. (강제까지 원하면 §7 — 무검증 생성자를 private으로 닫아 "유효하지 않은 객체는 존재 불가"로.)

---

## 4. ⭐ 핵심 구분 — 규칙(결정) vs 데이터 조회

"이메일 유니크"는 사실 **두 조각**이다:

| 조각 | 정체 | 어디 |
|---|---|---|
| **규칙/결정** — "존재하면 거부(예외)" | 비즈니스 로직 | **도메인** |
| **데이터 조회** — "이 이메일 DB에 있나?" | 인프라 | **repository(포트 구현)** |

- "체커를 주입하면 구현이 서비스에 있는 거 아니냐?" → **아니다.** 체커(`existsByEmail`)는 **데이터를 주는 통로**일 뿐, **"유니크여야 한다"는 판단은 도메인 안**에 있다.
- **규칙은 도메인 / 조회는 인프라**로 갈라지는 것. 이게 의존성 역전(포트).

---

## 4-1. ⭐⭐ 규칙(결정) vs 메커니즘(수단) — 뭘 도메인에 둘지의 최상위 기준

§4의 "조회"는 사실 **메커니즘의 한 종류**다. 더 크게 보면 거의 모든 코드는 둘로 갈린다 — 이 구분이 "이걸 도메인에 둘까 앱에 둘까"의 뿌리다.

| | 규칙 (Rule / Decision / Policy) | 메커니즘 (Mechanism) |
|---|---|---|
| 정체 | "~해도 되나? / ~여야 한다"를 **판단** | "기술적으로 어떻게 처리하나"의 **수단** |
| 누가 정함 | 도메인 전문가 (비즈니스) | 개발자 (기술 선택) |
| 예 | 이메일 유니크, 잔액≥0, 상태전이 가능?, 예약 5분 만료, 권한 충분? | 비번 **해싱**, JWT 생성, JSON 직렬화, save/load, **트랜잭션**, 캐시, 메일발송, 이벤트 발행, ID(UUID) 생성 |
| 위치 | **도메인** (엔티티 / 도메인 서비스) | **앱 / 인프라** (앱서비스·어댑터) |

### 가르는 리트머스 4개
1. **요구사항 회의에서 도메인 전문가가 말하나?** "이메일 중복 금지"→말함=**규칙** / "bcrypt로 해싱"→안 말함(개발자 결정)=**메커니즘**
2. **기술 스택을 통째로 갈아도 살아남나?** DB를 Mongo로, 웹을 gRPC로 바꿔도 "유니크"는 그대로=**규칙** / `PasswordEncoder`는 라이브러리 따라 바뀜=**메커니즘**
3. **"왜?"의 답이 비즈니스냐 기술이냐?** "왜 거부?—같은 이메일 두 계정 금지(비즈니스)"=**규칙** / "왜 인코딩?—평문 유출 위험(보안 기술)"=**메커니즘**
4. **결과가 판단(true/false/예외)이냐 처리(변환·입출력)냐?** 판단=**규칙** / 데이터를 다른 형태로 바꾸거나 외부와 주고받음=**메커니즘**

> ⚠️ **한 개념 안에 둘 다 있을 수 있다.** "비밀번호"도 *형식 규칙*(길이 8자↑=규칙, 도메인 가능)과 *해싱*(메커니즘=앱)으로 갈리고, "이메일"도 *유니크*(규칙)와 *조회*(메커니즘)로 갈린다. **단어가 아니라 그 동작이 판단이냐 수단이냐**로 가른다.

> 💡 **그레이존 최종 판단: "도메인 전문가가 *어떻게* 하는지 신경 쓰나?"** 일어난다는 사실(THAT)만 신경 쓰고 방법(HOW)은 개발자에게 맡기면 → 메커니즘. (해싱을 bcrypt로 하든 argon2로 하든 전문가는 모름 → 메커니즘 → 앱서비스/인프라.) **도메인 서비스 안엔 규칙만, 메커니즘(인코딩·트랜잭션·저장·외부호출)은 앱이 처리하고 결과값만 넘긴다.**

---

## 5. 체커 주입 패턴 (포트로 도메인에 능력 주입)

도메인이 "이런 게 필요해"를 포트로 선언하고, 구현(repo)은 밖에서 주입:

```java
// 도메인 계층 — 포트 (함수형 인터페이스)
@FunctionalInterface
public interface EmailUniquenessChecker {
    boolean exists(String email);
}

// 도메인 팩토리/서비스 — 규칙(결정)은 여기 (도메인 안)
public static User register(String email, String password, EmailUniquenessChecker checker) {
    if (checker.exists(email)) {
        throw new BusinessException(UserErrorCode.EXISTS_EMAIL);   // ← 규칙 = 도메인
    }
    return new User(null, email, password, BigDecimal.ZERO, UserRole.USER);
}
```
```java
// 호출부(앱/어댑터) — repo 메서드를 함수로 주입
User user = User.register(email, pw, repository::existsByEmail);  // 메서드 레퍼런스
```

- "함수 전달" = **함수형 인터페이스 / 메서드 레퍼런스**(`repository::existsByEmail`)를 넘기는 것.
- 표준 타입 `Predicate<String>`로 받아도 되지만, **이름 있는 인터페이스**(`EmailUniquenessChecker`)가 의도가 드러나 DDD에선 선호.
- 도메인은 repo를 **import 안 함** → 포트(추상)에만 의존 → 순수 유지. (`User`가 `UserEntity`를 모르게 한 것과 같은 원리)

### 테스트 이점 — mock 불필요
체커가 함수라, 도메인 단위 테스트에 Mockito 없이 람다로 주입:
```java
assertThatThrownBy(() -> User.register("a@", "pw", email -> true))   // 존재한다 치고
    .extracting("errorCode").isEqualTo(UserErrorCode.EXISTS_EMAIL);
User u = User.register("a@", "pw", email -> false);                  // 없다 치고 → 정상
```

---

## 5-1. ⚠️ 안티패턴 — Command에 검증용 필드 넣기

"규칙을 엔티티에 넣자"를 이렇게 시도하기 쉬운데, 권장되지 않는다:
```java
CreateUserCommand { email, password, boolean emailExists }   // ← 검증용 필드를 Command에
service: command.setEmailExists(repository.existsByEmail(email));
entity:  if (command.isEmailExists()) throw ...
```

**문제:**
1. **Command 오염** — Command는 *호출자의 입력/의도*인데, `emailExists`는 *계산된 사실(세상의 상태)*. 입력과 계산값을 한 DTO에 섞으면 의미가 흐려진다.
2. **신뢰 문제** — 엔티티가 "누가 미리 채워준 플래그"를 믿어야 함. 서비스가 깜빡하면 `false` 기본값 → **검증이 조용히 통과**(버그). 강제할 수단 없음.
3. **스케일** — `List<String> existingEmails`를 통째로 넘기는 변형은 유저가 많으면 불가.

**대안 (같은 "규칙은 엔티티에"를 더 깨끗이):**
- **체커(포트) 주입** — 엔티티가 플래그를 *읽는* 게 아니라 `checker.exists(email)`로 **스스로 확인**. 안 넘기면 컴파일 에러라 빠뜨릴 수 없고 Command도 안 건드림. (§5)
- **boolean을 Command 말고 별도 팩토리 파라미터로** — `User.create(email, pw, emailAlreadyExists)`. Command는 깨끗하게 유지.

> 핵심: **검증에 필요한 데이터는 Command(입력 DTO)에 넣지 말고, 체커(포트)로 주입하거나 별도 인자/도메인 서비스로 전달한다.** Command는 "호출자 의도"만.

---

## 5-2. "의존"이란 — 값 전달 ≠ 의존

| 엔티티가 하는 일 | 의존? |
|---|---|
| `boolean`/`List` 같은 **값을 받기만** | ❌ 의존 아님 (값의 타입에만 의존, repo엔 무관) → **순수 유지** |
| 포트(`checker`)를 **호출** / 타입을 **참조** | ✅ 의존 (게다가 호출이 I/O를 유발) |

> **"값을 받는다" ≠ "의존한다". "호출한다/타입을 참조한다" = "의존한다".**
> - **값 채워 넘기기**(서비스가 repo 호출 후 boolean/리스트를 엔티티에 전달): 엔티티는 데이터만 받아 판단 → **repo 의존 0, I/O 0, 순수.** 단점은 의존이 아니라 §5-1(Command 오염·신뢰).
> - **체커 주입**(엔티티가 `checker.exists()` 호출): 도메인 포트에 의존 + 호출이 DB를 침 → 엔티티가 **I/O를 유발**(순수 깨짐). 대신 규칙을 못 빠뜨림.

## 5-3. Application Service vs Domain Service vs UseCase (이름은 같은 "서비스")

| 계층 | 이름 | 역할 |
|---|---|---|
| web | Controller | HTTP I/O |
| application | **UseCase** | 교차 도메인 흐름 조율 (예: `PaymentUseCase`) |
| application | **Application Service** | 단일 도메인 흐름·트랜잭션 (예: `UserCommandService`, `AuthService`) |
| domain | **Domain Service** | 한 엔티티에 안 담기는 **도메인 규칙**(유니크·이체) — 조율/트랜잭션 없음, 도메인 개념에 이름 |
| domain | Entity / VO | 단일 엔티티 로직 (`User`) |
| infra | Repository | 영속화 포트 |

- **UseCase / Application Service = 조율**(load→도메인→save, 트랜잭션). 컨트롤러가 직접 호출하는 그 계층.
- **Domain Service = 규칙 자체**(조율 아님). 도메인 계층에 둠.
- 차이: Application Service는 "이 작업을 위해 **조율한다**", Domain Service는 "**도메인 규칙을 표현한다**".
- 현실: 작은 프로젝트는 Domain Service를 **안 만들고** 그 로직을 Application Service에 합치기도 함(지금 `AuthService`가 그럼) — 실용적 선택, 틀린 것 아님.

### 호출 흐름 — 직선이 아니라 트리 (signup, Option 1 기준)

```
Controller (web)
 └─→ [UseCase]                          ← 교차 도메인일 때만 (단순 signup은 생략 가능)
      └─→ Application Service (AuthService)    조율: 비밀번호 인코딩 · 트랜잭션
           ├─→ Domain Service (UserRegistration)   규칙: 중복 검증
           │     ├─→ Repository.existsByEmail        ← 검증용 조회
           │     └─→ User.create(...)                ← 엔티티 *생성*
           └─→ Repository.save(user)                 ← 엔티티 *저장*
```

> ⚠️ **Entity는 "repo 다음 계층"이 아니다.** 엔티티는 **서비스가 만들고(`create`) repo가 저장/조회하는(`save`/`find`) *대상***. repo↔entity는 직렬(→)이 아니라, 서비스가 **둘 다 쓰는** 관계. 흐름은 직선이 아니라 트리 — 한 서비스가 repo도 부르고 엔티티도 만든다.
> 의존 방향(헥사고날): `service → Repository(포트/도메인 추상) → Adapter(인프라) → DB`. 바깥(controller)→안(domain)으로 호출이 흐르고, 안쪽은 바깥을 모른다. (포트·어댑터·의존성 규칙·"소스 의존 ≠ 호출 방향" 상세 → [포트와 어댑터](./ports-and-adapters.md))

> ⚠️ **Domain Service ≠ 유틸 클래스.** 유틸(`SpecificationUtils` 같은)은 무상태 기술 헬퍼(static). Domain Service는 **도메인 개념을 표현하는 객체(빈)**, 도메인 언어로 이름 붙이고 **repo 포트를 주입받을 수 있다.** 위치는 도메인 계층(`domain/.../service`)의 별도 클래스 — "엔티티 폴더 안 유틸"이 아님.

### 어떤 로직을 어느 서비스에? — "규칙이냐 조율이냐"

| 로직 | 위치 |
|---|---|
| `@Transactional`, save/load, DTO 매핑 | App Service (조율) |
| 비밀번호 인코딩, 권한 체크, 외부 호출(메일·이벤트) | App Service (인프라) |
| 단일 엔티티 불변식(잔액≥0) | Entity |
| 교차 엔티티 규칙(유니크·이체) | Domain Service |

냄새 테스트 3개:
1. **도메인 전문가가 알아들을 규칙인가?** (요구사항에 적힐 말) → 도메인.
2. **트랜잭션·DTO·보안·외부시스템을 건드리나?** → 앱.
3. **DB·웹을 싹 갈아치워도 살아남나?** 살아남음 → 도메인 / 바뀜 → 앱.

> ⚠️ **미리 쪼개지 마라.** "어디 넣지" 고민이 곧 2계층의 비용. **기본은 엔티티 + 앱 서비스 2단으로 충분**하고, Domain Service는 **교차 규칙이 충분히 무거워질 때 추출**한다(규칙이 늘어 앱 서비스가 비즈니스 로직으로 뚱뚱해지는 게 신호). 작은 규칙 하나(유니크)면 앱 서비스에 둬도 정당.

> 💡 **규칙/결정 → 도메인, 절차/조율 → 앱.** 단 Domain Service는 *필요해질 때* 추출(YAGNI) — 미리 만들면 고민만 늘고 이득 없다.

### App Service가 위임(CRUD)만 가지면 빈약 — 애그리거트 연산을 줘라 (concrete: `PaymentCommandService`)

§1의 Anemic은 *엔티티* 얘기였지만, **서비스 계층도 빈약해질 수 있다.**

```java
// PaymentCommandService — repository 호출만, 행위가 없음 = 빈약한 서비스 계층
@Transactional public Payment create(Payment p) { return repository.save(p); }
@Transactional public Payment update(Payment p) { return repository.update(p); }
```

"결제한다(`pay`)"를 어디 둘지가 헷갈리는데, 통째로 한 곳에 넣는 게 아니라 **두 조각으로 갈린다**(§5-3 표 그대로):

| 부분 | 성격 | 위치 |
|---|---|---|
| 예약 검증·잔액 차감·좌석 확정 | **여러 애그리거트** 건드림 | **UseCase** (교차 도메인 조율) |
| Payment 생성 → `pay()` 전이 → 저장 | **payment 애그리거트 한정** | **App Service** (단일 애그리거트 조율) ← `pay`가 여기 |

```java
// App Service — payment 애그리거트의 "결제" 연산을 응집 (create→도메인호출→save)
@Transactional
public Payment pay(String reservationId, BigDecimal amount) {
    Payment payment = Payment.create(reservationId, amount);
    payment.pay(amount);                 // 도메인 전이는 엔티티가
    return repository.save(payment);
}
// UseCase는 교차 도메인만 조율하고 이 메서드에 위임 → 책임 분리
```

- 빈약 서비스 = **Repository 위 얇은 래퍼**(`create`/`update`처럼 위임만). 흩어져 있던 `Payment.create + payment.pay + save`를 **도메인 동사로 이름 붙인 한 메서드(`pay`)**로 모으면 응집이 생긴다.
- 위임만 하던 `create`/`update`는 `pay`가 흡수하면 **호출처가 사라져 제거 가능**(실패 이력·상태변경 등 다른 용도 없으면).
- ⚠️ §5-3 "미리 쪼개지 마라"와 충돌 아님 — **판단 축은 "그게 이 도메인의 의미 있는 연산이라 이름값을 하나 + 재사용/응집"**. 결제·충전처럼 핵심 연산이면 이름값을 하니 빼고, 정말 사소한 한 줄이면 UseCase에 둬도 된다.

> 💡 **UseCase = 교차 도메인 조율 / App Service = 단일 애그리거트 조율(load/create→도메인 호출→save) / Entity = 상태 전이.** App Service가 `create`/`update` 같은 CRUD 위임만 갖고 있으면 빈약 신호 — 애그리거트의 의미 있는 연산(`pay`)을 줘서 UseCase에 흩어진 조율을 끌어내린다. (단 사소하면 YAGNI로 UseCase에 둬도 무방)

## 5-4. IO는 못 없앤다 — pull vs push

유니크 검증은 DB 쿼리(IO)가 무조건 필요하다. 아키텍처는 **"누가 IO를 유발하고 규칙을 누가 소유하느냐"**만 정할 뿐, IO 자체는 못 없앤다. 실제 IO는 **항상 repository(인프라 어댑터)**에서 일어난다. 형태는 둘뿐:

| | **Pull** (도메인이 유발) | **Push** (상위가 내려줌) |
|---|---|---|
| 흐름 | 도메인이 **포트 호출** → 필요할 때 IO | 상위(앱서비스)가 **먼저 조회** → boolean/리스트를 도메인에 전달 |
| IO 위치 | repository(어댑터) | repository(어댑터) |
| 도메인 순수성 | 깨짐(IO 유발) | 유지(데이터만 받음) |
| 단점 | 도메인이 I/O에 묶임 | 상위가 채워줘야 함(§5-1 Command 오염 주의) |

> 헥사고날: Pull이라도 도메인이 직접 JDBC를 치는 게 아니다. 도메인은 **포트(추상)**만 호출하고, **실제 DB 호출은 어댑터**가 한다 → 도메인 코드엔 DB 코드가 없다. "도메인이 IO를 유발"의 정확한 뜻.

> 💡 **IO는 못 피한다 — 항상 repository에.** 선택지는 **pull(도메인이 포트로 불러옴) vs push(상위가 불러서 데이터로 내려줌)** 둘뿐. 트레이드오프는 "도메인 순수성 vs 규칙을 도메인에 묶기".

---

## 5-5. 도메인 서비스 만들 때 — 실전 판단 모음

`UserRegistration`을 실제로 빼보며 나온 질문들. (이 프로젝트: 유니크 검증을 `AuthService`에서 도메인 서비스로 추출 시도)

### (a) 폴더로는 도메인서비스 / 앱서비스 구분이 안 된다
이 프로젝트는 포트(`UserRepository`)도 서비스도 전부 `application/`에 둔다 → 도메인 서비스도 `application/service`에 살게 되어 **위치로는 구분 불가.** 구분 신호:
- **이름**: 앱서비스 = 애그리거트+작업(`UserCommandService`) / 도메인서비스 = 도메인 개념(`UserRegistration`)
- **`@Transactional` 유무**: 붙으면 앱(경계 소유), 안 붙으면 "규칙만"인 도메인 서비스 신호
  - ⚠️ 도메인 서비스에 `@Transactional`이 **없어도** 호출자(앱서비스) 트랜잭션 안에서 돈다 — 스프링 트랜잭션은 **스레드 바인딩 + 기본 전파 `REQUIRED`**라 새로 안 열고 *합류*. 그래서 register 안의 `existsByEmail`도 호출자 트랜잭션에 참여. **없는 게 정상이자 권장**(경계는 앱이 소유, 안 붙어야 "규칙만"이 드러남). ⚠️ 단 "검증+저장을 한 트랜잭션으로 묶기"는 **호출하는 앱서비스 책임** — 도메인 서비스를 트랜잭션 밖에서 단독 호출하면 그 조회는 auto-commit이라 저장과의 atomicity가 안 보장된다.
- → DDD 빌딩블록은 *물리 위치*가 아니라 *책임*으로 정의된다. 폴더는 그걸 드러내는 보조 수단일 뿐.

### (b) 서비스 쪼개는 단위가 종류마다 다르다 (폭발 방지)
| | 쪼개는 단위 | 개수 |
|---|---|---|
| **앱서비스** | 애그리거트별로 묶음 (CQRS면 읽기/쓰기) | 도메인당 1~2개. 작업은 *메서드*로 추가(클래스 ❌) |
| **도메인서비스** | **무거운 교차 규칙당** | 대부분 0개, 있어도 드묾 |

→ "등록"처럼 **액션마다 서비스 클래스를 만들면 폭발한다.** 등록의 조율은 `UserCommandService.createUser` **메서드**로 들어가지, 클래스가 따로 안 생긴다.

### (c) 도메인 서비스는 메서드가 보통 1개 — 정상이다
도메인 서비스는 "연산/규칙 하나"를 표현하니 단일 메서드(`register`)가 자연스럽다. 늘릴지는 **개수가 아니라 응집도(같은 개념이냐)**로 — 억지로 채우면 앱서비스로 퇴화. ⚠️ 단일 메서드가 *사소하면*(우리 `existsByEmail` 한 줄) "클래스로 뺄 값어치 있었나"(YAGNI) 의심 신호. 무거운 규칙 덩어리면 메서드 하나라도 정당.

### (d) 이름 — 애그리거트명 금지, 규칙명으로 좁게
- `UserPolicy` / `UserManager` ❌ — 범위가 너무 넓어 온갖 규칙이 빨려드는 god class.
- `RegistrationPolicy`, `PasswordPolicy` ✅ — **무슨 규칙인지** 이름에 박음.
- ⚠️ **repo를 부르면 "Policy" 말고 도메인 서비스 이름**(`UserRegistration`). Policy/Specification은 보통 **IO 없는 순수 규칙**(`isSatisfiedBy`) 뉘앙스라, repo 조회가 들어가면 이름이 거짓말이 됨.

### (e) model/엔 repo가 필요한 검증을 두지 마라
`model/`은 순수(프레임워크 무관, `new`로 찍는 객체) 자리다. repo가 필요한 검증(유니크)을 `UserValidation`이라며 model/에 두면 순수 레이어가 인프라 포트에 오염된다 — **그건 이미 도메인 서비스**다(→ `application/service`). **repo가 필요 없는 순수 검증**(이메일 형식·비번 길이·잔액≥0)만 엔티티 메서드 / VO로 model/에 둔다.

> 💡 한 줄: **클래스는 애그리거트 단위로 묶고(앱서비스), 구분은 메서드 이름으로. 도메인 서비스는 "규칙이 메서드 하나에 안 담길 만큼 무거울 때"만 꺼내는 카드 — 이름은 규칙명으로 좁게, repo 부르면 Policy가 아니라 도메인 서비스, 위치는 (이 프로젝트선) application/service.**

---

## 6. 현실 — 유니크 최종 보장은 DB 제약

`existsByEmail` 체크 → save는 **check-then-act 레이스**가 있다. 두 명이 동시에 같은 이메일로 가입하면 둘 다 `exists=false` 통과 후 둘 다 save 시도. **엔티티에서 검증하든 도메인 서비스로 하든 이 레이스는 못 막는다** — `@Column(unique=true)` DB 제약(또는 락)만 막는다. 코드 검증은 "친절한 에러 메시지"용, DB 제약이 "진짜 방어선".

---

## 7. 💡 판단 기준

| 검증 | 위치 |
|---|---|
| 자기 상태로 판단 가능 (잔액·상태전이) | **엔티티** (rich) |
| 집합/교차 (유니크·이체) | **도메인 서비스 / 체커 주입** (도메인 계층) |
| 흐름 조합·트랜잭션 | 앱 서비스 (로직 아님) |
| 동시성 유니크 최종 보장 | **DB 제약** |

> 한 줄: **"엔티티에 넣어라"의 진짜 뜻은 "빈약 모델 만들지 마라"다.** 엔티티가 *할 수 있는* 건 엔티티가, *구조상 못 하는*(여러 행을 봐야 하는) 건 도메인 서비스가. 그때도 **규칙은 도메인, 조회는 인프라**로 가른다.

> 현재 프로젝트는 유니크를 `AuthService`(앱 서비스)에서 체크한다 — 실용적이고 멀쩡한 선택. 도메인 순수성을 더 원하면 위 체커 주입으로 규칙을 도메인에 내릴 수 있다(트레이드오프: 포트·주입 증가). "틀린 것"은 아니다.

### "체커를 엔티티에 주입" vs "서비스에서 create 전에 체크" — 차이는?

| | (A) 엔티티에 체커 주입 | (B) 서비스에서 create 전 체크 (현재) |
|---|---|---|
| 쿼리·예외·레이스 | 동일 | 동일 (**런타임 결과 같음**) |
| 규칙 소유 | 엔티티(도메인) | 앱 서비스 |
| **우회 가능?** | ❌ 시그니처가 강제 → 못 빠뜨림 | ⚠️ `User.create()`/`save()` 직접 호출하면 우회됨 |
| 도메인에 I/O 의존 | 생김(포트) | 없음(엔티티 순수) |

> 💡 **차이의 본질 = "우회 가능성 + 유저 생성 지점 수".** 기능 결과는 같다. 생성 경로가 **하나**(signup만)면 우회될 일이 없어 (B)로 충분. **여러 곳**에서 만들거나 "이 규칙은 절대 못 빠뜨려야 한다"가 중요하면 규칙을 도메인으로 내린다(엔티티 체커 주입 또는 **도메인 서비스** — 엔티티에 repo 꽂기 싫으면 후자).

### ⭐ "강제"는 도메인 서비스가 아니라 *팩토리 시그니처*에서 나온다 (헷갈림 주의)

Domain Service를 만들어도 `User.create(email, password)`가 **여전히 public이면 우회된다** — 누가 그걸 직접 부르면 검증이 빠짐. **Domain Service는 규칙을 도메인 *계층*으로 올릴 뿐, "못 빠뜨리게" 강제하지 않는다.** 강제는 "약속(convention)".

| 방식 | 규칙 위치 | 강제 수단 | 우회 |
|---|---|---|---|
| 앱 서비스 체크 | 앱 계층 | 약속 | `User.create()` 직접 호출 시 ⚠️ |
| **Domain Service** | 도메인 계층 | **약속** | `User.create()` 직접 호출 시 ⚠️ (여전히) |
| **체커 주입(팩토리)** | 엔티티 팩토리 | **컴파일러** | **불가** (체커 없으면 컴파일 에러) |

> **"모든 생성에 검증을 무조건 태운다"가 목표면 — 규칙을 도메인 서비스로 옮기는 게 아니라, *팩토리가 체커를 요구하게*(시그니처) 만들어야 컴파일러가 강제한다.** Domain Service로 강제하려면 무검증 `User.create`의 가시성을 좁혀 그 서비스만 부르게 해야 하는데(package-private 등), 보통 패키지가 달라 번거롭다 → 강제가 목표면 체커 주입이 자연스러움. (그래도 최종 보루는 DB 유니크 제약)

---

## 8. 참고
- [Martin Fowler - Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html)
- 관련 노트: [Read-Modify-Write와 트랜잭션 경계](../jpa/read-modify-write.md) · [Mockito 서비스 테스트](../test/mockito-service-test.md)

---

**학습 날짜**: 2026-06-07
**계기**: `UserCommandService.create`는 fail이 없는데 중복 이메일 검증이 `AuthService`에 있는 걸 보고 — "검증은 엔티티에 있어야 DDD 아니냐"는 의문에서 출발. 단일 불변식 vs 집합 유니크, Anemic vs Rich, 도메인 서비스/체커 주입(규칙=도메인·조회=인프라), 유니크 최종보장=DB 제약을 정리.

**추가(2026-06-25)**: §2-3 생성(`create`) vs 복원(`of`) 팩토리 — 입력 검증은 복원 경로(`of`/`toModel`)가 아니라 생성 경로에 둔다. `Payment.of`에 `INVALID_INPUT` 검증이 들어가 있는데 그 `of`를 `PaymentEntity.toModel()`이 복원에 쓰는 걸 보고 정리.

**추가(2026-06-25)**: §5-3에 "빈약한 App Service" 보강 — `PaymentCommandService`가 `create`/`update` 위임만 가진 걸 보고, 애그리거트 연산(`pay`=생성→도메인호출→저장)을 줘서 UseCase(교차 도메인)에 흩어진 조율을 끌어내리는 판단. UseCase=교차 도메인 / App Service=단일 애그리거트 조율 경계 정리.

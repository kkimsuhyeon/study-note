# 도메인 검증 위치 — 엔티티 / 도메인 서비스 / 앱 서비스

> **한 줄 요약**: "모든 로직을 엔티티에"는 DDD의 흔한 오해. 엔티티가 **자기 상태로 판단할 수 있는 불변식**(잔액≥0 등)은 엔티티에(rich model), **여러 행을 봐야 하는 규칙**(이메일 유니크 등)은 엔티티가 구조상 불가 → **도메인 서비스 / 체커 주입**으로(도메인 계층이지만 엔티티는 아님). 핵심 구분: **규칙(결정)은 도메인, 데이터 조회는 인프라.**

관련 노트: [Read-Modify-Write와 트랜잭션 경계](../jpa/read-modify-write.md) · [Mockito 서비스 테스트](../test/mockito-service-test.md)

---

## 1. 흔한 오해 — "DDD = 모든 로직을 엔티티에"

반은 맞고 반은 틀린 말이다.

- ✅ **맞는 알맹이**: **Anemic(빈약) 도메인 모델** 안티패턴을 피하라. 엔티티가 getter/setter 덩어리고 로직이 전부 서비스에 있으면 객체지향이 아니다. 엔티티는 **자기 로직을 가져야**(rich) 한다.
- ❌ **틀린 일반화**: "**전부** 엔티티에"는 DDD가 말한 적 없다. DDD엔 **도메인 서비스**라는 빌딩블록이 따로 있다 — 한 엔티티에 안 속하는 로직을 담는 도메인 계층 객체.

> Anemic Domain Model = Martin Fowler가 이름 붙인 안티패턴. 그 반대(rich)를 지향하라는 게 핵심이지, "엔티티에 우겨넣어라"가 아니다.

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

---

## 3. 계층별 책임 (DDD 빌딩블록)

| 빌딩블록 | 역할 | 비즈니스 로직? |
|---|---|---|
| **Entity** | 식별자 + 생명주기 + **자기 상태 불변식** | ✅ (rich) |
| **Value Object** | 불변 값 + 자기 검증 | ✅ |
| **Domain Service** | **한 엔티티에 안 속하는** 도메인 로직(유니크·이체 등) | ✅ (도메인 계층) |
| **Application Service** | 흐름 조합·트랜잭션 경계 | ❌ (오케스트레이션만) |
| **Repository** | 영속화 추상(포트) | ❌ |

> 유니크가 엔티티 밖에 있는 건 "DDD 위반"이 아니라, **엔티티가 구조상 못 하는 일**이라 도메인 서비스가 맡는 것. (Vaughn Vernon "Implementing DDD"도 유니크를 도메인 서비스 케이스로 다룸)

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

> ⚠️ **Domain Service ≠ 유틸 클래스.** 유틸(`SpecificationUtils` 같은)은 무상태 기술 헬퍼(static). Domain Service는 **도메인 개념을 표현하는 객체(빈)**, 도메인 언어로 이름 붙이고 **repo 포트를 주입받을 수 있다.** 위치는 도메인 계층(`domain/.../service`)의 별도 클래스 — "엔티티 폴더 안 유틸"이 아님.

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

---

## 8. 참고
- [Martin Fowler - Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html)
- 관련 노트: [Read-Modify-Write와 트랜잭션 경계](../jpa/read-modify-write.md) · [Mockito 서비스 테스트](../test/mockito-service-test.md)

---

**학습 날짜**: 2026-06-07
**계기**: `UserCommandService.create`는 fail이 없는데 중복 이메일 검증이 `AuthService`에 있는 걸 보고 — "검증은 엔티티에 있어야 DDD 아니냐"는 의문에서 출발. 단일 불변식 vs 집합 유니크, Anemic vs Rich, 도메인 서비스/체커 주입(규칙=도메인·조회=인프라), 유니크 최종보장=DB 제약을 정리.

# 포트와 어댑터 — 콘센트 규격과 플러그

> **한 줄 요약**: **포트 = 도메인이 정하는 콘센트 규격**(인터페이스, "여기 꽂히려면 이 모양"), **어댑터 = 그 규격에 맞게 깎은 플러그**(구현, 반대쪽 끝은 실제 기술에 연결), **DI = 꽂는 행위**. 핵심 판단: **인터페이스(포트)는 의존 방향을 역전해야 할 때만** — service→repo는 안→밖이라 역전 필요(포트 ✅), controller→service는 밖→안이라 이미 올바름(인터페이스 불필요 ❌).

관련 노트: [도메인 검증 위치](./domain-validation.md) · [변환 계층](./transform-layers.md)

---

## 1. 비유 — 콘센트와 플러그

```
벽(도메인)          콘센트(포트)            플러그(어댑터)          기계(기술)
UserRegistration ─ PasswordHasher 규격 ─ PasswordHasherAdapter ─ Spring Security
UserService      ─ UserRepository 규격 ─ UserRepositoryAdapter ─ JPA/MySQL
```

- **규격은 항상 벽(도메인) 쪽이 정한다** — 기계 제조사가 "내 플러그에 맞춰 벽을 뚫어라"가 아니라, 벽이 규격을 선언하면 기술 쪽이 어댑터를 만들어 맞춰 들어온다. 이게 **의존성 역전(DIP)**.
- **플러그 교체 자유** — bcrypt→argon2, JPA→Mongo로 바꿔도 플러그(어댑터)만 새로 깎으면 벽(도메인)은 안 건드림. 포트&어댑터(=헥사고날)의 존재 이유.
- **꽂는 행위 = Spring DI** — 런타임에 `implements PasswordHasher`인 빈을 찾아 끼워줌.
- **테스트 = 임시 플러그** — 포트가 함수형이면 람다로: `new UserRegistration(repoMock, raw -> "hashed:" + raw)`. 진짜 기계 없이 벽 배선만 검사.

## 1-1. 의존성 규칙 — 모든 화살표는 안쪽으로 (직선이 아니라 부채꼴)

**의존성 규칙(Dependency Rule)**: 소스코드 의존성(import)은 항상 **안쪽(핵심)으로만** 향한다. 단 한 줄 직선이 아니라 adapter가 두 갈래인 **부채꼴**:

```
adapter/in (controller) → application → service(도메인 서비스) → port → model
adapter/out (persistence·security) ─────────────────────────→ port → model
```

- in 어댑터는 application을 *경유*해 들어오고, **out 어댑터는 port에 직접 꽂힌다**(`implements UserRepository`) — 문(in)과 플러그(out)의 차이.
- 검증법: 안쪽 패키지(model/port/service)가 바깥(application/adapter)을 **import하는 순간 위반**.

### ⭐ 소스 의존 방향 ≠ 런타임 호출 방향

| | 방향 |
|---|---|
| **소스 의존** (import·컴파일) | 밖→안 (adapter가 port를 import) |
| **런타임 호출** | 안→밖 (`register()` → `port.existsByEmail()` → 실행되는 건 바깥의 어댑터 → DB) |

이 비틀림을 만드는 장치가 **포트(인터페이스)+DI** — 그게 의존성 역전(DIP)의 정체. 전통 레이어드(controller→service→repo구현→DB, 의존이 DB로 흘러내림)와의 결정적 차이.

---

## 2. 실제 코드 (예시)

```java
// 포트 — 도메인이 "이 능력이 필요해"를 규격으로 선언 (Security 모름)
@FunctionalInterface
public interface PasswordHasher {
    String hash(String rawPassword);
}

// 어댑터 — 여기만 실제 기술을 앎. 변환/위임
@Component
@RequiredArgsConstructor
public class PasswordHasherAdapter implements PasswordHasher {
    private final PasswordEncoder passwordEncoder;        // ← 반대쪽 끝(기계)
    public String hash(String raw) { return passwordEncoder.encode(raw); }
}
```

> `UserRepository`/`UserRepositoryAdapter`도 **완전히 같은 구조** — 폴더명이 `repository`라 안 보일 뿐 처음부터 포트였다. 어댑터가 `UserEntity ↔ User` 변환까지 하는 것도 "모양 맞추기"(GoF Adapter 패턴)의 일부. controller도 사실 "HTTP 모양→서비스 모양"을 맞추는 **in adapter**(`adapter/in/web`).

## 3. ⭐ 판단 — 인터페이스(포트)는 언제 만드나

**"의존 방향을 역전해야 할 때만."**

| 호출 | 방향 | 역전 필요? | 인터페이스 |
|---|---|---|---|
| Service → Repository/외부기술 | 안→밖 (잘못된 방향) | ✅ | **포트로** (out port) |
| Controller → Service | 밖→안 (이미 올바름) | ❌ | 구체 클래스 직접 호출 OK |

- 교과서 헥사고날엔 service 앞에도 포트(**in port**, `CreateUserUseCase` 인터페이스)가 있지만, 의존 방향이 이미 맞아 이득이 적어 **실무에선 대부분 생략**. "service는 비즈니스 그 자체(안쪽)라 포트감이 아니다"는 직감과 같은 결론.
- 📖 여기서 UseCase = in port **인터페이스**. 이와 달리 "교차 도메인을 조율하는 별도 application 클래스"를 UseCase라 부르는 관례도 있다 → [도메인 검증 §5-3](./domain-validation.md).
- 그래서 포트는 사실상 **out**(영속화·외부 시스템)에 집중된다.

## 4. 실무에서 뭘 포트로 빼나 (빈도순)

1. **영속화**(repository) — 거의 항상
2. **외부 시스템 클라이언트**(PG·메일·외부 API) — 실익 최대(테스트에서 fake로 교체)
3. **메시징**(이벤트 발행)
4. **시간**(`Clock`) — 시간 의존 로직 테스트용
5. `PasswordEncoder` 류 래핑 — 드묾(이미 인터페이스라). 학습/순수성 신호 목적이면 OK

## 5. 포트의 주소 — 소유권과 승격

- **포트는 "필요를 선언한 쪽"이 소유** — `PasswordHasher`는 `UserRegistration`(user 도메인)이 필요로 하니 `domain/user/application/port`. 한 도메인만 쓰는데 미리 공용 자리에 두는 건 반(反)YAGNI.
- **쓰는 도메인이 둘 이상 되면 그때 공용 위치로 승격.**
- 포트를 application에 두냐 domain(model 옆)에 두냐는 **학파 차이** — 헥사고날/클린=application 경계, DDD=도메인 계층. 둘 다 정당.

### 5-0. 포트 패키지 구성 — "계약 + 계약의 어휘", 나눌 땐 in/out

- `port/`에 인터페이스(UserRepository·PasswordHasher)와 `UserCriteria`가 섞여 보여도 **이질적이지 않다** — UserCriteria는 일반 dto가 아니라 **포트 시그니처에 등장하는 타입**(계약의 어휘). 계약과 그 어휘는 함께 둔다. (로직은 전부 service/·model/에)
- 커져서 나눌 땐 application처럼 종류별(dto/service)이 아니라 **방향별 `port/in`(유스케이스)·`port/out`(repo·외부)** — buckpal식 교과서 컨벤션. in port 생략 중이면 빈 폴더만 생기니, **파일 몇 개일 땐 평평하게**(미리 쪼개지 마라).

### 5-1. 포트 시그니처의 프레임워크 타입 — 자체 타입 vs `Page` 허용

포트를 도메인으로 내리려면 시그니처의 `Page`/`Pageable`(Spring Data)이 걸림돌. 정석은 **자체 페이징 타입 + 어댑터 변환**:
```java
public record PageQuery(int page, int size) {}                                  // 도메인 어휘
public record PageResult<T>(List<T> content, long totalElements, int totalPages) {}
// 어댑터에서: PageQuery → PageRequest 변환, Page<Entity> → PageResult<Model> 변환
```
하지만 **포트에 `Page`/`Pageable`을 허용하는 절충도 실무에서 흔하고 근거가 있다**:
- `Page`/`Pageable`은 Spring Data **Commons**(JPA 비종속 — Mongo에도 그대로) → 사실상 준표준, 결합 비용 낮음
- 자체 타입은 결국 `Page`를 어설프게 베끼게 됨(정렬·hasNext 등 기능 따라가기)

> 💡 목표는 순수성 100%가 아니라 "도메인이 JPA·웹을 모르는 것". **멀티모듈로 도메인을 스프링 없이 컴파일할 게 아니면 페이징은 `Page` 허용으로 선 긋는 것도 합리적.**

### 5-2. 폴더를 "도메인 먼저" 자르나 "계층 먼저" 자르나 — 같은 헥사고날, 다른 주소

| | 도메인 먼저 (package by feature) | 계층 먼저 (package by layer) |
| --- | --- | --- |
| 모양 | `domain/user/{model, port, adapter/in/web, adapter/out/persistence}` | 최상위 `application / domain / infrastructure / config / shared`, 어댑터는 `infrastructure/{web, persistence, 외부API명, 라이브러리명}` |
| 강점 | 한 기능을 고칠 때 한 폴더 안에서 끝남. 기능 삭제가 폴더 삭제 | 외부 기술별로 모여서 "HTTP 클라이언트 설정이 어디 있나"가 한눈에. 도메인 폴더가 순수해 보임 |
| 약점 | 외부 기술 설정이 도메인마다 흩어짐 | 한 기능을 고치려면 최상위 폴더 3~4개를 오간다 |

의존 방향 규칙(어댑터 → 포트 ← 도메인)은 둘이 **똑같다**. 주소만 다르다. 처음 배운 구조와 달라 보여 헷갈릴 때는 "포트 인터페이스가 어디 있고, 그걸 구현한 클래스가 어디 있나" 두 개만 찾으면 대응이 잡힌다.

**규칙은 테스트로 강제할 수 있다 — ArchUnit.** "domain 패키지는 `java..`·domain·공통 예외·허용한 어노테이션에만 의존", "application은 infrastructure·config를 모른다", "최상위 패키지 간 순환 없음"을 JUnit 테스트로 쓰면, 규칙을 어기는 import가 생기는 순간 빌드가 깨진다. 코드 리뷰에서 매번 눈으로 잡을 필요가 없어진다.

**어댑터를 `@Component` 대신 `@Configuration`에서 `new`로 조립하는 이유** — 어댑터가 설정값(API 키·타임아웃)과 무거운 준비물(RestClient·메시지 컨버터·리다이렉트 정책)을 필요로 할 때, 그 준비를 설정 클래스 한곳에 모으고 어댑터 자체는 평범한 클래스로 둔다. 테스트에서 `new`로 바로 만들기 쉽다. `@Component` + 생성자 `@Value`도 똑같이 동작하니 **취향과 일관성의 문제**다 — 섞어 쓰면 "이 빈은 어디서 생기지?"를 매번 찾게 되니 한 프로젝트 안에서는 규칙을 하나로.

### 5-3. 저장소 포트 하나에 관심사가 모일 때 — "이름과 메서드가 맞나", "에러를 누가 만드나"

한 유즈케이스가 테이블 여러 개를 쓰면, 편해서 포트 하나(`XxxRepository`)에 메서드를 다 몰아넣기 쉽다. 신호 세 가지:

1. **포트 이름과 메서드가 안 맞는다.** "보고서 저장소"에 보고서와 무관한 요청 한도 카운터 메서드가 있다. 나중에 다른 기능(재시도 API 등)이 한도만 쓰려 해도 보고서 저장소에 의존해야 한다.
2. **비즈니스 규칙과 에러가 영속성 엔티티 안에 있다.** JPA 엔티티가 "한도 초과면 429 + Retry-After 예외"를 직접 던진다. 저장 계층이 응답 정책을 결정하는 셈이다.
3. **에러를 만들기 위한 값이 저장소까지 내려간다.** `retryAfterSeconds`처럼 저장과 무관하고 예외 메시지에만 쓰이는 인자가 포트 시그니처에 있으면, 에러를 만드는 위치가 틀렸다는 신호다.

나누는 방법: 관심사별로 포트를 분리하고(`RateLimitRepository`), 포트는 **의도만**(`boolean tryConsume(key, limit)`), 어댑터는 **DB에서 원자적으로 해내는 방법만**(`INSERT ... ON CONFLICT` → `FOR UPDATE` → 증가) 맡는다. 결과가 `false`면 **안쪽(유즈케이스·도메인)이 에러를 만든다.** "어떻게 저장하나"는 바깥, "넘으면 무엇이 되나"는 안쪽이다.

### 5-4. application 폴더가 어색해 보일 때 — 폴더를 늘리기 전에 "섞인 관심사"부터

"파일이 많아서 폴더로 묶고 싶다"는 느낌의 원인이 개수가 아닐 때가 많다. 먼저 볼 신호:

- **계층 간 비대칭.** 같은 관심사(요청 한도)가 domain·infrastructure에선 자기 패키지(`ratelimit/`)로 나뉘었는데 application에서만 다른 기능(`report/`) 안에 섞여 있다.
- **다른 관심사의 타입이 끼어 있다.** 보고서 폴더에 한도 버킷 record가 있다.
- **유즈케이스 하나가 두 가지 일을 한다.** 생성 흐름 메서드 아래에 한도 계산·해시·정렬 헬퍼가 30줄 붙어 있다.

이때 해법은 종류별 하위 폴더(`command/ response/ usecase/`)가 아니라 **섞인 관심사를 협력 객체로 빼서 제 패키지에 두는 것**이다(`application/ratelimit/RateLimiter`). 유즈케이스에는 `rateLimiter.consume(...)` 한 줄과 생성 흐름만 남는다.

종류별 폴더를 미루는 이유:
- **Java에는 "하위 패키지 접근"이 없다.** `report/`와 `report/command/`는 서로 남남인 패키지라, 같은 폴더라서 package-private으로 숨겨 두던 타입을 쪼개는 순간 `public`으로 열어야 한다.
- 파일 6개를 폴더 4개로 나누면 폴더당 1~2개 — 찾는 비용만 는다.
- 나중에 유즈케이스가 3개 이상으로 늘어 각자 command·response를 가지면, 그때 **유즈케이스별**(`report/create/`, `report/get/`)로 나눈다. 한 기능을 고칠 때 보는 파일이 한 폴더에 모인다.

트랜잭션 안의 로직을 협력 객체로 뺄 때 주의:
- 협력 객체에 **`REQUIRES_NEW`를 붙이지 않는다.** 호출자 트랜잭션에 합류해야 생성이 실패할 때 늘어난 카운트도 같이 롤백된다. 별 트랜잭션이면 실패한 요청도 한도를 깎는다.
- **호출 위치를 그대로 둔다.** 예: 멱등 재전송 확인 **뒤**, 비싼 작업 **앞**. 옮기다 순서가 바뀌면 재전송도 한도를 소비한다.
- 동작이 같으니 기존 테스트를 고치지 않고 통과하는 것이 성공 기준이다.

---

## 6. 💡 판단 기준

> **"이 호출, 안에서 밖으로 나가나?"** 나가면(DB·외부API·인프라) 포트+어댑터로 역전. 안 나가면(controller→service) 인터페이스 없이 직접. 포트 규격은 도메인 언어로(벽이 정함), 어댑터만 기술을 알게. 포트 주소는 필요 선언한 도메인 — 공용은 둘째 사용자가 나타나면.

---

## 7. 참고
- 관련 노트: [도메인 검증 위치](./domain-validation.md) · [변환 계층](./transform-layers.md) · [책임 경계 §3-1](./responsibility-boundaries.md)(Request를 안쪽에 넘겨도 되는 구조)
- [Alistair Cockburn - Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) — 포트&어댑터 원문
- [ArchUnit User Guide](https://www.archunit.org/userguide/html/000_Index.html) — 패키지 의존 규칙을 테스트로 강제

---

**학습 날짜**: 2026-06-10
**계기**: `PasswordHasher` 포트+어댑터를 직접 만들며 — "adapter가 진짜 (돼지코) 어댑터구나" + "내 `UserRepository`도 포트랑 다를 게 없네" 깨달음에서. 콘센트/플러그 비유, in/out 포트와 "역전 필요할 때만 인터페이스" 기준, 포트 소유권(필요 선언한 쪽→둘째 사용자 때 승격) 정리.
**보강(2026-10-02)**: UseCase 용어가 도메인 검증 노트와 다르게 쓰이는 점 표시, 프로젝트 한정 문장 정리, 참고 URL 추가.

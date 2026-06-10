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

## 2. 실제 코드 (이 프로젝트)

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
- 포트를 application에 두냐 domain(model 옆)에 두냐는 **학파 차이** — 헥사고날/클린=application 경계, DDD=도메인 계층. 둘 다 정당. (이 프로젝트는 전자)

---

## 6. 💡 판단 기준

> **"이 호출, 안에서 밖으로 나가나?"** 나가면(DB·외부API·인프라) 포트+어댑터로 역전. 안 나가면(controller→service) 인터페이스 없이 직접. 포트 규격은 도메인 언어로(벽이 정함), 어댑터만 기술을 알게. 포트 주소는 필요 선언한 도메인 — 공용은 둘째 사용자가 나타나면.

---

## 7. 참고
- 관련 노트: [도메인 검증 위치](./domain-validation.md) · [변환 계층](./transform-layers.md)

---

**학습 날짜**: 2026-06-10
**계기**: `PasswordHasher` 포트+어댑터를 직접 만들며 — "adapter가 진짜 (돼지코) 어댑터구나" + "내 `UserRepository`도 포트랑 다를 게 없네" 깨달음에서. 콘센트/플러그 비유, in/out 포트와 "역전 필요할 때만 인터페이스" 기준, 포트 소유권(필요 선언한 쪽→둘째 사용자 때 승격) 정리.

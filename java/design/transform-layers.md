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

## 6. 💡 판단
> **"옮기냐 만드냐"로 먼저 가른다.** 옮기기(변환)면 어느 경계냐 — web면 Mapper, app이면 Assembler. 만들기(생성·기본값·불변식)면 Factory. 1:1 단순 복사라 로직이 없으면 그 변환 계층은 테스트도 스킵(프레임워크/단순 위임).

---

## 7. 참고
- 프로젝트 규칙 원본: server-java `docs/CONVENTIONS.md §1` (변환 계층)
- 관련 노트: [도메인 검증 위치](./domain-validation.md)

---

**학습 날짜**: 2026-06-08
**계기**: 코드 보다 Factory/Mapper/Assembler 차이가 기억 안 나서 — "변환 2단(Mapper:web→Command, Assembler:Command→Model) + 생성 1개(Factory)"로 정리. Mapper vs Assembler는 계층 차이, Factory는 변환이 아니라 생성.

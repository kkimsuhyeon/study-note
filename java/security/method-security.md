# Spring Security 메서드 보안 — @PreAuthorize · @EnableMethodSecurity

> **한 줄 요약**: URL 패턴 보안(필터 체인)과 별개로, 스프링 빈의 **메서드 단위**에 SpEL 조건식으로 인가(Authorization)를 거는 기능. 최대 함정은 활성화 어노테이션 없이 `@PreAuthorize`만 붙이면 **에러 없이 조용히 무시**된다는 것.

## 언제 쓰나

URL 레벨 보안(`antMatchers("/api/**").hasAnyAuthority(...)`)은 "**이 경로에 누가** 들어올 수 있나"만 표현할 수 있다. 아래처럼 **파라미터·반환값·도메인 상태**가 판정에 필요하면 메서드 보안이 필요하다:

- "이 API는 **백오피스 서버 호출자만**" (호출자 종류 구분 — 경로로는 구분 불가)
- "**본인 데이터만** 조회/수정 가능" (`#userId == principal.id`)
- "조회 **결과가 본인 소유일 때만** 응답" (`returnObject.ownerId == ...`)

비유: `antMatchers`가 건물 입구의 경비라면, `@PreAuthorize`는 각 방 문마다 붙는 세부 조건 잠금장치.

## 켜는 법 (활성화 어노테이션)

`@Configuration` 클래스에 붙인다. **이게 없으면 아래 모든 어노테이션은 붙어 있어도 무시된다.**

| | 구식 `@EnableGlobalMethodSecurity` | 신식 `@EnableMethodSecurity` |
|---|---|---|
| 도입/상태 | Spring Security 초기~, **5.6부터 deprecated** | 5.6 도입, 6.x 표준 |
| `prePostEnabled` 기본값 | **false** (명시적으로 켜야 함) | **true** (붙이기만 하면 됨) |
| 내부 구조 | metadata source + AccessDecisionManager/Voter | `AuthorizationManager` 기반, 네이티브 Spring AOP |
| 커스터마이즈 | `GlobalMethodSecurityConfiguration` 상속 | 빈 등록 방식 |

```java
// Spring Security 5.x (구식)
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class SecurityConfig { ... }

// Spring Security 5.6+ / 6.x (신식) — prePostEnabled 기본 true라 옵션 생략 가능
@Configuration
@EnableMethodSecurity
public class SecurityConfig { ... }
```

## 사용 예시 — @PreAuthorize

메서드 실행 **전**에 SpEL 표현식을 평가해서, `false`면 본문 진입 없이 `AccessDeniedException`(→ 403)을 던진다.

```java
// 역할 체크
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { ... }

// 파라미터 참조 (#파라미터명) — 본인 데이터만
@PreAuthorize("#userId == principal.id")
public UserDetail getMyInfo(Long userId) { ... }

// 파라미터 객체의 메서드 호출 — 백오피스 호출자만
@PreAuthorize("#authUser != null and #authUser.isBackOffice()")
public void withdraw(@AuthenticationPrincipal AuthUser authUser, ...) { ... }

// 다른 스프링 빈에 판정 위임 — 표현식이 복잡해지면 이쪽
@PreAuthorize("@orderPermissionChecker.canAccess(#orderId, principal)")
public Order getOrder(Long orderId) { ... }
```

### SpEL에서 쓸 수 있는 것들

| 표현식 | 의미 |
|---|---|
| `hasRole('ADMIN')` / `hasAnyRole(...)` | 역할 보유 — **`ROLE_` 접두사를 자동으로 붙여서** 비교 |
| `hasAuthority('X')` / `hasAnyAuthority(...)` | 권한 문자열 **그대로** 비교 (접두사 없음) |
| `isAuthenticated()` / `isAnonymous()` | 인증 여부 |
| `permitAll` / `denyAll` | 무조건 허용/거부 |
| `principal` / `authentication` | SecurityContext의 인증 주체/인증 객체 |
| `#paramName` | **메서드 파라미터** 참조 (이름 기반) |
| `@beanName.method(...)` | 다른 **스프링 빈** 메서드 호출 |
| `returnObject` | 반환값 (**@PostAuthorize 전용**) |
| `filterObject` | 컬렉션 원소 (**@PreFilter/@PostFilter 전용**) |

## 종류 비교 — 메서드 보안 어노테이션 4형제

| 어노테이션 | 활성화 옵션 | 표현력 |
|---|---|---|
| `@PreAuthorize` / `@PostAuthorize` | `prePostEnabled` | **SpEL 전체** — 파라미터·반환값·빈 호출 가능 |
| `@PreFilter` / `@PostFilter` | `prePostEnabled` | 컬렉션 파라미터/반환값에서 조건 맞는 원소만 필터링 |
| `@Secured("ROLE_ADMIN")` | `securedEnabled` | 권한 문자열 나열만 (SpEL 불가) |
| `@RolesAllowed("ADMIN")` | `jsr250Enabled` | 자바 표준(JSR-250), 역할 나열만 |

→ 사실상 `@PreAuthorize`가 상위호환. `@Secured`/`@RolesAllowed`는 레거시 호환이나 표준 준수가 필요할 때 정도.

`@PostAuthorize`는 메서드 실행 **후** `returnObject`를 보고 판정:

```java
@PostAuthorize("returnObject.ownerId == principal.id")
public Document getDocument(Long id) { ... }  // 남의 문서면 응답 직전에 403
```

## ⚠️ 함정 / 메커니즘

1. **활성화 안 하면 조용히 무시**. `@EnableMethodSecurity`(구식은 `prePostEnabled=true`) 없이 `@PreAuthorize`를 붙이면 컴파일도 되고 에러도 없이 **그냥 통과**한다. 보안 코드가 무음으로 죽는 최악의 패턴 — 팀에서 "AT 프로젝트에서 이렇게 했다"는 예시가 나왔을 때 첫 질문이 "`@EnableGlobalMethodSecurity` 켰어요?"였던 이유. (같은 패턴: `@EnableCaching` 없는 `@Cacheable` → [spring-cache.md](../spring/spring-cache.md))
2. **AOP 프록시 기반** → `@Transactional`과 동일한 프록시 함정을 공유한다 ([transactional.md](../spring/transactional.md)):
   - **자기 호출(self-invocation)은 검사를 우회**한다 — 같은 클래스 안에서 `this.securedMethod()` 호출 시 프록시를 안 거침.
   - **스프링 빈에만** 동작. `new`로 만든 객체엔 무효.
   - `private` 메서드엔 못 붙인다 (프록시가 가로챌 수 없음).
3. **`hasRole`은 `ROLE_` 접두사를 자동으로 붙인다.** `hasRole('ROLE_ADMIN')`이라고 쓰면 `ROLE_ROLE_ADMIN`을 찾는 이중 접두사 사고. DB에 저장된 권한 문자열에 접두사가 없다면 `hasAuthority`를 쓰는 게 안전.
4. **`#paramName` 참조는 컴파일 시 파라미터 이름 보존이 전제.** `-parameters` 컴파일 옵션이 필요하다. Spring Framework **6.1부터는 바이트코드 파싱 폴백(`LocalVariableTableParameterNameDiscoverer`)이 아예 제거**되어 이 옵션이 사실상 필수 (Spring Boot 플러그인 빌드는 기본으로 켜줌). 이름을 못 찾으면 `@P("authUser")`로 명시하는 우회로가 있다.
5. **`@PostAuthorize`는 메서드가 이미 실행된 후에 거부한다.** 쓰기 작업에 쓰면 부수효과(외부 API 호출, 이벤트 발행 등)는 이미 발생한 상태. `AccessDeniedException`이 RuntimeException이라 트랜잭션 내 DB 변경은 롤백되지만, **트랜잭션 밖 부수효과는 안 돌아온다**. 조회 전용으로만.
6. **실행 시점이 URL 보안과 다르다.** URL 보안은 서블릿 **필터 체인**에서(컨트롤러 진입 전), 메서드 보안은 **빈 메서드 호출 시점**(AOP)에서 평가된다. 인증 필터(JWT 필터 등)가 SecurityContext를 먼저 세팅해줘야 `principal`/`#authUser`가 의미를 가진다.
7. **표현식은 문자열이라 컴파일 검증이 없다.** 오타(`isBackOffce()`)는 런타임에 해당 메서드가 호출될 때야 터진다. 표현식을 쓰는 메서드는 거부 케이스 테스트를 같이 두는 게 안전.

## 💡 판단 기준

**구체 케이스**: 파로스 탈퇴 API는 백오피스 서버에서만 호출 가능해야 하는데, URL 보안이 `/api/** → 권한 목록` 방식뿐이라 일반 사용자 JWT로도 호출되는 상태였다. "호출자 종류" 구분 장치가 필요 → 팀에서 커스텀 어노테이션(`@AllowedCallers` + 자작 AOP, → [custom-annotation.md](../annotation/custom-annotation.md)) vs 기본 제공 `@PreAuthorize` 논의 → **`@PreAuthorize("#authUser != null and #authUser.isBackOffice()")` 채택**.

- **프레임워크 기본 제공으로 표현 가능하면 자작보다 기본 제공 먼저.** 커스텀 어노테이션+AOP는 인터셉터·예외 변환·테스트를 전부 직접 유지보수해야 하지만, `@PreAuthorize`는 이미 검증된 인프라에 표현식 한 줄이다. 자작이 정당해지는 건 기본 제공의 표현력을 벗어날 때(예: 어노테이션 속성으로 선언적 메타데이터를 줘야 할 때)뿐.
- **URL vs 메서드 보안 분담**: "이 **경로**에 누가 들어오나" = URL 보안(공통 관문), "이 **동작**을 어떤 조건에서 허용하나" = 메서드 보안(개별 규칙). 전자로 다 막고, 전자가 표현 못 하는 것만 후자로.
- **표현식이 도메인 규칙 수준으로 길어지면** SpEL 문자열에 로직을 넣지 말고 `@beanName.check(...)`로 빈에 위임 — 문자열은 IDE 지원·컴파일 검증이 없어서 로직 담기엔 부적합.

## 참고

- [Spring Security 공식 — Method Security](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html)
- [EnableMethodSecurity API 문서](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/config/annotation/method/configuration/EnableMethodSecurity.html) — `prePostEnabled` 기본 true
- [Baeldung — Spring @EnableMethodSecurity](https://www.baeldung.com/spring-enablemethodsecurity) — 구식→신식 마이그레이션
- [Spring Framework 6.1 Release Notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-6.1-Release-Notes) — `-parameters` 필수화 배경
- 학습 날짜: 2026-08-04
- 계기: 서버 to 서버 API(탈퇴) 호출자 제한 논의 — 커스텀 어노테이션 명칭 추천으로 시작해서 "기본 제공(`@PreAuthorize`) 쓰자"로 결론 난 팀 대화. 현재 프로젝트는 URL 보안만 켜져 있어 `@EnableGlobalMethodSecurity` 추가가 선행 작업.

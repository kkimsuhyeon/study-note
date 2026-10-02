# Spring Security 도입 — 로그인이 없는 앱에 먼저 들일 때 깨지는 것들

> **한 줄 요약:** `SecurityFilterChain` 빈을 직접 정의하면 Boot의 기본 체인(전부 로그인 필요, 폼 로그인)은 빠지지만 CSRF·보안 헤더·요청 캐시 같은 기본 필터는 남는다. 로그인 전에 들일 때는 **"지금 동작을 하나도 바꾸지 않는 체인"**을 먼저 만들고, CSRF 방식 전환은 다음 단계로 미룬다.

## 언제 쓰나

- **나중에 로그인을 넣을 계획이 있고, 보안 경계를 한곳에 모아 두고 싶을 때.** 인증·인가·세션 고정 방지·CSRF가 같은 필터 체인 위에서 동작하게 된다.
- **직접 만든 보안 필터(Origin·Content-Type 검사 등)를 표준 위치로 옮기고 싶을 때.**
- 반대로 로그인 계획이 없고 직접 만든 필터가 잘 동작한다면 들일 이유가 약하다. 핵심 가치가 인증·인가다.

## 사용 예시 — 1단계: 동작을 바꾸지 않는 체인

```java
@Configuration(proxyBeanMethods = false)
public class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http, OriginCheckFilter originFilter) throws Exception {
        http
            .securityMatcher("/api/**")
            .authorizeHttpRequests(a -> a.anyRequest().permitAll())   // 로그인이 생기면 여기서 경로별로 나눈다
            // CSRF는 Origin 검사 + JSON 전용 + SameSite=Lax가 맡는다. 토큰 방식은 2단계에서 결정.
            .csrf(csrf -> csrf.disable())
            .addFilterBefore(originFilter, CsrfFilter.class)
            .requestCache(cache -> cache.disable());                   // 익명 요청이 세션을 만들지 않게
        return http.build();
    }

    // 필터를 빈으로 두면 Boot가 서블릿 필터로도 등록한다 → 두 번 실행되지 않게 끈다
    @Bean
    FilterRegistrationBean<OriginCheckFilter> originFilterRegistration(OriginCheckFilter filter) {
        FilterRegistrationBean<OriginCheckFilter> registration = new FilterRegistrationBean<>(filter);
        registration.setEnabled(false);
        return registration;
    }
}
```

- `csrf.disable()`은 **대신 막는 장치가 있을 때만** 쓰고, 그 이유를 바로 옆 주석에 남긴다. 이유 없이 복사된 `disable()`이 가장 흔한 사고다([Origin 헤더](../../infra/network/origin-header.md)).
- 구동 로그의 `Will secure ... with filters: [...]` 줄에서 실제 필터 순서를 확인할 수 있다. 직접 넣은 필터가 `CsrfFilter` 앞에 한 번만 있어야 한다.

### 로그인이 없을 때 끄는 기본 장치들 — 각각 무엇인가

Spring Security가 기본으로 깔아 주는 장치는 대부분 **"브라우저 화면 + 로그인"**을 전제로 한다. 로그인이 없는 JSON API에서는 쓸 일이 없거나 오히려 방해가 된다.

| 설정 | 그 장치가 원래 하는 일 | 로그인 없는 API에서 끄는 이유 | 로그인을 넣으면 |
| --- | --- | --- | --- |
| `csrf.disable()` | POST·PUT·DELETE마다 CSRF 토큰을 요구, 없으면 403 | 토큰을 주고받지 않으므로 전부 403. 다른 방어(Origin 검사 등)가 있을 때만 끈다 | 토큰 방식으로 바꿀지 결정 |
| `cors.disable()` | 다른 origin에 응답 읽기를 허용하는 헤더 부착, preflight 응답 | 같은 origin 서비스. 나중에 CORS 빈이 생겨도 자동으로 열리지 않게 잠금 | 앱·웹뷰 등 다른 origin이 생길 때만 그 origin을 명시 |
| `requestCache.disable()` | 로그인 안 한 사람이 보호 페이지에 오면 원래 요청을 **세션에 저장** → 로그인 후 그 페이지로 되돌려 보냄 | 로그인 페이지도 리다이렉트도 없다. 켜 두면 익명 방문자에게 세션이 생긴다 | 화면 기반 로그인이면 다시 켤 수 있음. JSON API면 보통 끈 채로 |
| `sessionCreationPolicy(NEVER)` | Security가 세션을 언제 만들지 정함. 기본 `IF_REQUIRED`(필요하면 만듦) | Security는 세션을 **절대 만들지 않고, 있으면 쓰기만** 한다. 세션을 만드는 시점은 앱 코드가 정한다 | 세션 로그인이면 `IF_REQUIRED`로. JWT 같은 헤더 토큰이면 `STATELESS`(만들지도 쓰지도 않음) |
| `formLogin.disable()` | `/login` HTML 로그인 폼과 아이디·비밀번호 POST 처리(`UsernamePasswordAuthenticationFilter`) | HTML 화면이 없는 JSON API | 폼 대신 JSON 로그인 API를 직접 만드는 경우가 많다 |
| `httpBasic.disable()` | 브라우저 팝업으로 아이디·비밀번호를 받아 매 요청 `Authorization: Basic base64(id:pw)` 전송 | 쓰지 않는 인증 방식 | 내부 관리 도구 정도에서만 |
| `logout.disable()` | `/logout` 요청 시 세션 무효화·쿠키 삭제·리다이렉트 | 익명 세션이 곧 보고서 소유권이라, 세션이 의도치 않게 무효화되면 결과를 잃는다 | 로그아웃 API로 다시 켬 |
| `headers.disable()` | 보안 헤더 자동 부착(`Cache-Control: no-store` 계열, `X-Content-Type-Options`, `X-Frame-Options`, HTTPS면 HSTS) | 기존 필터가 같은 헤더를 직접 붙이므로 중복 방지 | HSTS·`X-Frame-Options`를 다시 켤지 결정 |

`SessionCreationPolicy`의 `NEVER`는 **Security 자신**의 행동만 정한다. 앱 코드의 `request.getSession()`은 그대로 세션을 만든다.

## 옵션 비교 — 2단계: CSRF를 어떻게 할까

| 선택 | 프론트 변화 | 맞는 상황 |
| --- | --- | --- |
| 1단계 그대로(Origin 필터가 CSRF 담당) | 없음 | 같은 origin 웹만 있을 때. 계약 유지 |
| `csrf(c -> c.spa())` 토큰 방식 | `XSRF-TOKEN` 쿠키를 먼저 받고 POST마다 `X-XSRF-TOKEN` 헤더 | 회사 표준과 맞추고 싶을 때, 로그인 폼까지 같은 방식으로 지킬 때 |
| 토큰 + Origin 필터 둘 다 | 위와 같음 | 피해가 큰 동작(결제·계정 변경)이 생겼을 때 |

## ⚠️ 함정

1. **기본 CSRF가 켜진 채로 두면 기존 POST가 전부 403이 된다.** 직접 체인을 정의해도 CSRF 필터는 기본으로 붙는다. 토큰을 보내지 않던 클라이언트와 테스트가 한꺼번에 깨진다 → 1단계에서는 명시적으로 결정한다.
2. **필터가 두 번 실행된다.** `@Component`나 `@Bean`으로 둔 `Filter`는 Boot가 컨테이너에 등록하고, `addFilterBefore`로 Security 체인에도 들어가 **순서가 다른 두 곳에서** 돈다. `FilterRegistrationBean#setEnabled(false)`로 한쪽을 끈다.
   - **`OncePerRequestFilter`를 상속했다면 본문은 한 번만 돈다.** 처음 실행될 때 요청 속성에 "이미 실행됨" 표시(`필터이름.FILTERED`)를 남기고, 같은 요청에서 다시 들어오면 `doFilterInternal`을 건너뛴다. 그래도 끄는 이유는 **주인을 하나로** 하기 위해서다 — 컨테이너 쪽 복사본이 살아 있으면, 나중에 Security 체인 설정(`securityMatcher` 등)을 실수로 바꿔도 컨테이너 쪽이 대신 검사해 줘서 **실수가 테스트에 드러나지 않는다.** 일반 `Filter`를 구현한 필터는 이런 표시가 없어 정말로 두 번 돈다.
   - `setEnabled(false)`는 "등록하지 않는다는 등록"이다. `FilterRegistrationBean`은 필터를 어떤 URL·순서로 컨테이너에 넣을지 적는 설명서인데, 이 필터에 대한 설명서가 있으면 Boot는 자동 등록을 하지 않고, `enabled=false`라 결국 아무것도 등록되지 않는다. 필터 빈 자체는 그대로 있어서 Security 체인에 주입할 수 있다.
   - 대안: 필터를 **빈으로 만들지 않고** 체인 메서드 안에서 `new`로 만들면 Boot가 볼 일이 없어 이 설정이 필요 없다(공식 문서도 "그래서 필터는 흔히 빈이 아니다"라고 쓴다). 대신 테스트에서 `@MockitoSpyBean`으로 바꿔 끼우거나 다른 빈에 주입할 수 없다.
   - 테스트로 못 박는 법: `verify(filter, times(1)).doFilter(...)` — 건너뛴 호출도 `doFilter` 호출로 세어지므로, 자동 등록이 살아 있으면 2가 되어 실패한다.
3. **요청 캐시가 익명 세션을 만든다.** 기본 `HttpSessionRequestCache`는 인증이 필요한 요청을 막을 때 원래 요청을 **세션에 저장**하고, 세션 생성을 허용하는 게 기본값이다. "성공한 요청에서만 세션을 만든다" 같은 원칙이 있으면 `requestCache.disable()`.
4. **보안 헤더가 겹친다.** Spring Security는 기본으로 `Cache-Control: no-store` 계열, `X-Content-Type-Options: nosniff`, `X-Frame-Options` 등을 붙인다. 직접 붙이던 헤더와 겹치면 한쪽으로 정리하고, 기본에 없는 것(`Referrer-Policy`, `X-Robots-Tag` 등)만 남긴다.
5. **거절 응답 형식이 달라진다.** Security가 직접 내는 401·403은 프로젝트의 JSON 에러 형식이 아니다. 로그인을 넣는 시점에 `AuthenticationEntryPoint`·`AccessDeniedHandler`를 맞춘다.
6. **필터에서 `getRequestURI()`로 경로를 비교하면 인코딩으로 우회될 수 있다.** `getRequestURI()`는 퍼센트 인코딩이 **풀리지 않은 원본**이다. 반면 Spring MVC는 경로를 **디코딩해서** 컨트롤러를 찾는다. 그래서 `/api/v1/%65xternal/...`(`%65`=`e`)처럼 글자 하나만 인코딩하면, `AntPathMatcher`로 원본 URI를 비교하던 필터는 "내 대상이 아니다"라고 건너뛰는데 컨트롤러에는 그대로 도착한다. Spring Security 방화벽은 `%2F`·`%2E` 같은 위험 문자만 거절하고 일반 글자의 인코딩은 통과시킨다. 막는 법: Spring이 디코딩한 경로 기준으로 비교한다(`PathPattern` + `PathContainer`, 또는 Security의 `RequestMatcher`/`securityMatcher`로 대상 지정), 그리고 `/api/%72eports` 같은 인코딩 경로로 막히는지 **테스트를 하나 둔다.**
7. **CORS는 설정 빈이 생기는 순간 자동으로 켜진다.** Spring Security는 `UrlBasedCorsConfigurationSource` 빈이 **하나** 있으면 체인에 CORS를 자동 적용한다(둘 이상이면 어느 걸 쓸지 몰라 적용하지 않는다). 같은 origin 서비스라면 `cors(cors -> cors.disable())`를 명시해 두는 게 잠금 역할을 한다 — 나중에 누가 예제를 복사해 CORS 빈을 추가해도 이 체인에는 열리지 않고, 열려면 이 줄을 고쳐야 한다.
8. **"Origin 기능"으로 CORS를 켜지 말 것.** 같은 origin 서비스라면 CORS 설정은 필요 없고, 켜면 허용한 origin에 응답 읽기 권한까지 열린다. CORS는 정말 다른 origin(앱 웹뷰 등)을 받아야 할 때 그 origin만 명시해서 켠다.

## 💡 판단 기준

**들이는 단계와 바꾸는 단계를 나눈다.** 1단계는 "체인 위로 옮기기만" 해서 기존 테스트가 하나도 바뀌지 않아야 성공이다. 동작 변화(토큰 방식, 경로별 인증)는 이유가 생겼을 때 하나씩 한다. 그래야 문제가 생겼을 때 "도입 때문인지 정책 변경 때문인지"가 갈린다.

## 참고

- [Spring Security 7.0 — Servlet Architecture (필터를 빈으로 선언할 때)](https://docs.spring.io/spring-security/reference/7.0/servlet/architecture.html)
- [Spring Security 7.0 — HttpSecurity (requestCache 비활성화)](https://docs.spring.io/spring-security/reference/7.0/api/java/org/springframework/security/config/annotation/web/builders/HttpSecurity.html)
- [Spring Security 7.0 — HttpSessionRequestCache](https://docs.spring.io/spring-security/reference/7.0/api/java/org/springframework/security/web/savedrequest/HttpSessionRequestCache.html)
- [Spring Security 7.0 — CsrfConfigurer (`spa()`)](https://docs.spring.io/spring-security/reference/7.0/api/java/org/springframework/security/config/annotation/web/configurers/CsrfConfigurer.html)
- 학습일: 2026-10-01. 계기: 로그인 없는 익명 쿠키 세션 API에서 직접 만든 Origin 필터를 Spring Security로 옮겨도 되는지 검토. 필터 이중 등록·요청 캐시의 세션 생성은 공식 문서로 확인했고, 예시 코드는 컴파일·실행하지 않았다.

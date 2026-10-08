# Origin 헤더 — 브라우저가 붙이는 "이 요청을 시작한 페이지의 주소"

> **한 줄 요약:** `Origin`은 프론트 코드가 아니라 **브라우저가 자동으로** 붙이는 헤더이고, 값은 요청을 보낸 페이지의 `스킴://호스트:포트`다. 자바스크립트로 바꿀 수 없어서, 서버는 이 값을 보고 "우리 프론트 화면에서 시작된 요청인가"를 판단할 수 있다.

## 언제 쓰나

- **쿠키로 사용자를 구분하는 API의 CSRF 방어.** 브라우저는 쿠키를 자동으로 붙이므로, 다른 사이트가 사용자의 브라우저를 이용해 우리 API를 부르면 서버는 사용자가 직접 보낸 요청과 구분하지 못한다. Origin을 보면 구분된다.
- **CORS 판단.** 다른 origin에서 온 요청에 대해 서버가 `Access-Control-Allow-Origin`으로 허용 여부를 답할 때 기준이 되는 값이다([세션과 쿠키](./sessions-and-cookies.md) CORS 절).

## 사용 예시 — 브라우저가 언제 붙이나

```http
POST /api/reports HTTP/1.1
Host: api.example.com
Origin: https://app.example.com        ← 브라우저가 붙임. 경로·쿼리는 없음
Referer: https://app.example.com/form  ← 이것도 브라우저가 붙임. 경로까지 들어감
Sec-Fetch-Site: same-site              ← 브라우저가 붙임. 요청 출발지와 목적지의 관계
Content-Type: application/json
```

| 요청 | Origin이 붙나 |
| --- | --- |
| 다른 origin으로 보내는 요청(CORS) | 붙음 |
| 같은 origin의 POST·PUT·PATCH·DELETE·OPTIONS | 붙음 |
| **같은 origin의 GET·HEAD** | **안 붙음** |
| 다른 origin으로 보내는 `no-cors` 모드 GET·HEAD | 안 붙음 |
| 주소창 입력·링크 클릭 같은 페이지 이동 GET | 안 붙음 |
| WebSocket 핸드셰이크 | 붙음 — 단 CORS가 막아 주지 않으므로 서버가 직접 검사해야 한다([WebSocket](./websocket.md) ⚠️) |

같은 origin에서 GET을 보낸 요청을 개발자 도구로 보면 Origin이 없고 Referer와 `Sec-Fetch-Site: same-origin`만 있는 게 정상이다.

- **값은 `null`일 수도 있다.** `file://`·`data:` 페이지, sandbox iframe, origin을 넘는 리다이렉트, 그리고 일부 `Referrer-Policy`에서의 HTML 폼 전송. `fetch()`는 기본 모드가 `cors`라서 같은 origin POST도 항상 실제 origin을 보낸다.
- **자바스크립트로 못 바꾼다.** `Origin`은 forbidden request header라서 `fetch(..., { headers: { Origin: "..." } })`로 넣어도 브라우저가 무시한다. 그래서 브라우저 안에서는 위조할 수 없다.
- **브라우저 밖에서는 아무 값이나 넣을 수 있다.** curl·Postman·서버 간 호출은 Origin을 안 보내거나 원하는 값을 보낸다. 하지만 그런 클라이언트에는 **피해자의 쿠키가 없다** — CSRF는 "남의 브라우저와 쿠키를 빌리는" 공격이라 Origin 검사로 충분하고, 스크립트 남용은 [rate limit](../rate-limiting.md)이 막는다.

## 옵션 비교 — CSRF를 막는 방법들

| 방법 | 동작 | 장점 | 한계 |
| --- | --- | --- | --- |
| `SameSite=Lax` 쿠키 | 다른 **사이트**에서 시작된 POST에는 쿠키를 안 실음 | 설정 한 줄 | 같은 사이트의 다른 서브도메인(다른 origin)은 못 막음. 오래된 브라우저 |
| **Origin 검사** | 상태를 바꾸는 요청에서 Origin이 허용 목록과 정확히 같은지 | 토큰 발급·보관이 필요 없음. 서버 필터 하나 | Origin이 `null`·누락인 정상 요청을 어떻게 볼지 정해야 함 |
| `Sec-Fetch-Site` 검사 | `cross-site`면 거절 | 브라우저가 관계를 직접 알려 줌 | 이 헤더를 안 보내는 오래된 브라우저 |
| CSRF 토큰 | 서버가 준 토큰을 요청에 다시 실어 보내게 함 | 가장 전통적, Spring Security 기본 | 발급·전달·검증 흐름이 필요 |
| JSON 전용 Content-Type | 상태를 바꾸는 API가 `application/json`만 받음(그 외 415) | 다른 origin이 JSON을 보내려면 브라우저가 **preflight(OPTIONS)**를 먼저 보내고, CORS를 허용하지 않으면 본 요청을 아예 안 보낸다. HTML 폼은 `urlencoded`·`multipart`·`text/plain`만 보낼 수 있어 415 | 단독으로 믿지 말 것(OWASP). 과거 브라우저·플러그인 버그로 우회된 전례가 있다 |

**CORS는 원래 CSRF 방어 장치가 아니지만, 설정이 있으면 부수 효과로 일부를 막는다.** CORS는 "다른 origin이 응답을 **읽게** 해 줄지"를 정하는 장치다.

- **설정이 없을 때**: Spring MVC는 다른 origin 요청을 **거절하지 않고** CORS 헤더를 안 붙일 뿐이며, 막는 건 브라우저다. 단순 요청(폼 POST 등)은 서버에 도착해 **실행된 뒤** 응답만 못 읽는 것이라 CSRF는 이미 성공한 상태다.
- **설정이 있을 때**: `DefaultCorsProcessor`가 Origin이 허용 목록에 없는 교차 출처 요청을 **403 "Invalid CORS request"로 거절**한다. 다른 사이트에서 온 폼 POST도 여기서 막힌다. 이 기능은 Spring Security가 아니라 **Spring Framework** 것이고, Spring Security의 `http.cors()`는 같은 설정을 보안 필터 체인 앞쪽에 끼워 넣을 뿐이다.

CORS 설정을 Origin 검사 대신 쓸 때 직접 만든 필터와 달라지는 점:

| | 직접 만든 Origin 필터 | CORS 설정 |
| --- | --- | --- |
| Origin이 없는 요청 | 거절할 수 있음 | **통과.** Origin이 없으면 CORS 요청으로 보지 않는다 (브라우저 POST엔 항상 붙으므로 CSRF 관점에선 큰 차이 아님) |
| "다른 origin인가" 판단 | 허용 목록과 문자열 비교 | 서버가 보는 **자기 주소**와 비교. 리버스 프록시 뒤에서 전달 헤더를 안 믿으면 같은 origin 요청도 교차로 판정 → 프론트 주소를 허용 목록에 넣어야 함 |
| 거절 응답 형식 | 프로젝트 JSON 에러 형식 | 기본은 일반 텍스트 403 |
| 부작용 | 없음 | 허용한 origin에 **응답 읽기·쿠키 전송 권한까지** 열어 줌. 나중에 앱·웹뷰용 origin을 추가하면 잠금 목록이 아니라 문 여는 목록이 늘어난다 |

⚠️ **CORS 예제에 흔히 붙어 있는 `csrf(csrf -> csrf.disable())`를 그대로 복사하지 말 것.** 그건 `Authorization` 헤더 토큰(JWT 등)으로 인증해서 브라우저가 자격 증명을 자동으로 붙이지 않을 때만 안전하다. 쿠키 세션 앱에서 끄면 CSRF 방어가 사라진다.

**세 가지 질문을 구분한다.** 섞이면 "토큰으로 하냐 세션으로 하냐"와 "Origin을 보냐"가 같은 선택지처럼 보인다.

| 질문 | 장치 | 예 |
| --- | --- | --- |
| 누구냐 (인증) | 세션 쿠키 / 헤더 토큰 | `JSESSIONID`·`SESSION`, JWT |
| 본인이 의도한 요청이냐 (CSRF) | CSRF 토큰 / Origin 검사 / SameSite / JSON 전용 | Spring Security CSRF, 직접 만든 필터 |
| 다른 origin이 응답을 읽어도 되냐 (CORS) | CORS 설정 | `allowedOrigins`, `allowCredentials` |

CSRF 토큰도 저장 위치가 두 가지라 "세션으로 vs 토큰으로"처럼 들릴 수 있다 — Spring Security의 `HttpSessionCsrfTokenRepository`(서버 세션에 보관)와 `CookieCsrfTokenRepository`(쿠키에 보관, 이중 제출). 둘 다 CSRF 토큰 방식이고 인증 방식과는 별개다.

OWASP는 이것들을 서로의 대체가 아니라 **겹쳐 쓰는 방어**(defense in depth)로 본다.

**세 겹(SameSite=Lax · JSON 전용 · Origin 검사)이 각각 무엇을 막나** — "다른 둘로 막히니 하나는 빼도 되지 않나?"를 판단할 때:

| 공격 상황 | SameSite=Lax | JSON 전용 | Origin 검사 |
| --- | --- | --- | --- |
| 다른 사이트의 폼 POST | 막음 (쿠키 안 실림) | 막음 (415) | 막음 |
| 다른 사이트의 `fetch` JSON | 막음 | 막음 (preflight 실패) | 막음 |
| 같은 사이트의 다른 서브도메인에서 폼 POST | **못 막음** (같은 사이트라 쿠키 실림) | 막음 (415) | 막음 |
| SameSite를 `None`으로 바꾼 뒤 (다른 사이트의 인앱 웹뷰 지원 등) | **못 막음** | 막음 | 막음 |
| JSON이 아닌 업로드 API(multipart)를 추가한 뒤 | 막음 | **못 막음** | 막음 |
| SameSite를 지원하지 않는 옛 브라우저 | **못 막음** | 막음 | 막음 |

SameSite=Lax와 JSON 전용 두 겹만으로도 표의 앞 두 줄(가장 흔한 경우)은 막힌다. Origin 검사는 **나머지 두 겹 중 하나가 약해지는 순간 혼자 남는 마지막 겹**이다 — 약해지는 계기(웹뷰 지원, 파일 업로드 추가)가 대개 나중의 기능 추가라서, 그때 누가 보안 영향을 떠올리지 못해도 버티게 해 준다.

### Spring Security로 하면 무엇이 달라지나

Spring Security(7.0 기준)의 CSRF 방어는 **토큰 방식**이다. 문서에 Origin이나 `Sec-Fetch-Site`로 검사하는 내장 옵션은 없다. GET·HEAD·TRACE·OPTIONS를 뺀 요청에 토큰을 요구하는 게 기본이고, SPA용으로는 `http.csrf(csrf -> csrf.spa())`가 쿠키 기반 토큰 저장소(`XSRF-TOKEN` 쿠키, `X-XSRF-TOKEN` 헤더)를 설정해 준다.

| 선택 | 바뀌는 것 | 비용 |
| --- | --- | --- |
| 커스텀 Origin 필터 유지 | 없음 | 경로 매칭·인코딩 우회 같은 함정을 직접 막아야 한다 |
| Spring Security + 토큰(`spa()`) | 프론트가 **토큰을 먼저 받아** 헤더로 보내야 함. 쿠키가 하나 늘고 API 계약이 바뀜 | 첫 POST 전에 토큰을 받는 요청이 필요. 토큰을 세션에 저장하는 기본 저장소를 쓰면 **익명 방문자에게도 세션이 생긴다** |
| Spring Security + Origin 필터를 체인에 추가(`csrf.disable()` + `addFilterBefore`) | 계약 그대로, 기본 보안 헤더를 얻음 | Origin 검사는 여전히 직접 작성. 체인을 직접 정의하면 "전부 인증 필요·폼 로그인"은 빠지지만, 기본으로 켜지는 CSRF·요청 캐시와 기본 401/403 응답 형식은 정리해야 함 → [Spring Security 도입](../../java/security/spring-security-filter-chain.md) |
| CORS 설정으로 대신(`http.cors()` 또는 Spring MVC 전역 CORS) | 허용 안 된 Origin의 교차 요청을 프레임워크가 403으로 거절 | Origin 없는 요청은 통과, 거절 응답 형식이 다름, 허용 origin에 응답 읽기 권한까지 열림 — 위 "CORS 설정을 Origin 검사 대신 쓸 때" 표 |

💡 **CSRF만을 위해 Spring Security를 들일 이유는 약하다.** 인증(로그인)이 없는 동안 핵심 가치(인증·인가·세션 고정 방지)는 쓰이지 않고, CSRF만 위해 들이면 토큰 흐름이 추가되거나 결국 같은 커스텀 필터를 체인 안에 넣게 된다. **회원 로그인이 정해지는 시점**이 들일 때다. 그보다 먼저 들인다면 동작을 하나도 바꾸지 않는 1단계 체인으로 옮기는 것부터 한다([Spring Security 도입](../../java/security/spring-security-filter-chain.md) 💡).

## ⚠️ 함정

- **XSS가 있으면 CSRF 방어는 전부 소용없다.** 우리 페이지 안에서 도는 스크립트의 `fetch`는 Origin이 우리 주소이고 SameSite 쿠키도 실리며, 일부러 HttpOnly가 아닌 `XSRF-TOKEN`도 읽힌다. 그래서 순서는 "XSS를 먼저 막고, 그 위에 CSRF 방어"다 → [XSS와 CSP](./xss-and-csp.md).
- **Origin 검사(와 CSRF 토큰)는 훔친 쿠키를 막지 못한다.** 쿠키 값을 손에 넣은 공격자는 Origin을 직접 적고 토큰도 새로 받아 온다. 이건 CSRF(값을 **모른 채** 피해자 브라우저가 붙이게 함)가 아니라 세션 탈취(값을 **알고** 직접 붙임)이고, 방어는 "못 훔치게" 하는 쪽이다 → [웹 공격 지도 §1](./web-attacks-map.md).
- **GET에서 Origin을 필수로 요구하면 정상 요청이 막힌다.** 같은 origin GET에는 원래 안 붙는다. 그래서 Origin 검사는 상태를 바꾸는 메서드에만 건다. GET은 다른 origin에서 보내도 CORS 때문에 응답을 읽을 수 없어 CSRF로 얻는 게 적다 — 단 **GET이 상태를 바꾸지 않을 때만** 성립한다.
- **GET을 예외로 둘 땐 HEAD도 같이 둔다.** HEAD는 "본문 없는 GET"이다. 서버는 GET과 똑같이 처리하고 상태 코드·헤더(`Content-Length`·`Content-Type`·`ETag` 등)만 돌려준다. 파일 크기 확인, 링크 생존 확인, 캐시 검증에 쓰인다. **Spring MVC는 `@GetMapping`에 HEAD를 자동으로 연결**하므로(본문은 버리고 `Content-Length`만 계산), HEAD로 보내도 같은 컨트롤러·같은 DB 조회가 실행된다. 둘 다 상태를 바꾸지 않는 **안전한 메서드**이고 같은 origin에서는 Origin이 안 붙는다. 필터가 GET만 빼고 HEAD를 빼지 않으면 정상 HEAD가 403이 되고, 반대로 HEAD에만 다른 규칙을 걸면 GET 검사를 HEAD로 돌아가는 구멍이 된다. 그래서 `method != GET && method != HEAD`처럼 짝으로 다룬다.
- **허용 목록은 정확히 일치로 비교한다.** `startsWith("https://app.example.com")`이면 `https://app.example.com.evil.com`이 통과한다. 스킴·호스트·포트까지 문자열 전체가 같아야 한다.
- **Host와 헷갈리지 말 것.** `Host`는 요청이 **도착하는** 서버 이름, `Origin`은 요청을 **출발시킨** 페이지의 주소다.
- **Referer로 대신하지 말 것.** 경로까지 담아 개인정보가 새기 쉽고, `Referrer-Policy`로 아예 빠지는 경우가 많다.
- **로컬 개발에서는 origin이 포트까지 포함한다.** 프론트 `http://localhost:3000`과 API `http://localhost:8080`은 **다른 origin**이다(쿠키는 포트를 구분하지 않는 것과 대비 — [세션과 쿠키](./sessions-and-cookies.md)).
- **클래스 이름만 보고 Spring Security가 켜져 있다고 가정하지 말 것.** `SecurityConfig`라는 이름의 직접 만든 필터일 수도 있다. 의존성에 `spring-boot-starter-security`가 없으면 CSRF 토큰도 없다 — 의존성과 실제 필터 목록으로 어떤 방어가 켜져 있는지 확인한다.

## 💡 판단 기준

**"이 요청을 사용자의 브라우저가 대신 보내게 만들 수 있나?"** 쿠키가 자동으로 붙는 구조면 그렇다 → 상태를 바꾸는 메서드에 Origin(또는 `Sec-Fetch-Site`) 검사 + `SameSite=Lax`를 겹쳐 건다. 로그인 폼·결제처럼 피해가 큰 동작이거나 오래된 브라우저까지 지켜야 하면 CSRF 토큰을 더한다. 쿠키가 아니라 `Authorization` 헤더 토큰으로 인증하면 브라우저가 자동으로 붙이지 않으므로 CSRF 자체가 성립하지 않는다.

## 참고

- [MDN — Origin](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Origin)
- [MDN — Referrer-Policy: Effect on the Origin header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Referrer-Policy#effect_on_the_origin_header)
- [MDN — Forbidden request header](https://developer.mozilla.org/en-US/docs/Glossary/Forbidden_request_header)
- [MDN — Sec-Fetch-Site](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Sec-Fetch-Site)
- [Spring Framework — CORS Processing](https://docs.spring.io/spring-framework/reference/web/webmvc-cors.html)
- [Spring Security 7.0 — CsrfConfigurer](https://docs.spring.io/spring-security/reference/7.0/api/java/org/springframework/security/config/annotation/web/configurers/CsrfConfigurer.html)
- [OWASP — CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- 학습일: 2026-10-01. 계기: 로그인 없이 쿠키 세션으로 방문자를 구분하는 API가 POST에서 Origin을 검사하는 코드를 보고 "Origin은 누가 던지는 거야?", "평소 보던 요청엔 없던데"라는 질문. 같은 origin GET에는 원래 Origin이 없다는 규칙을 MDN으로 확인. 예시 요청은 일반화했다.

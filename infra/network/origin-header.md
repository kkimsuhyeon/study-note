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

오늘 구성에서는 앞의 두 겹만으로도 대부분 막힌다. Origin 검사는 **나머지 두 겹 중 하나가 약해지는 순간 혼자 남는 마지막 겹**이다 — 약해지는 계기(웹뷰 지원, 파일 업로드 추가)가 대개 나중의 기능 추가라서, 그때 누가 보안 영향을 떠올리지 못해도 버티게 해 준다.

### Spring Security로 하면 무엇이 달라지나

Spring Security(7.0 기준)의 CSRF 방어는 **토큰 방식**이다. 문서에 Origin이나 `Sec-Fetch-Site`로 검사하는 내장 옵션은 없다. GET·HEAD·TRACE·OPTIONS를 뺀 요청에 토큰을 요구하는 게 기본이고, SPA용으로는 `http.csrf(csrf -> csrf.spa())`가 쿠키 기반 토큰 저장소(`XSRF-TOKEN` 쿠키, `X-XSRF-TOKEN` 헤더)를 설정해 준다.

| 선택 | 바뀌는 것 | 비용 |
| --- | --- | --- |
| 커스텀 Origin 필터 유지 | 없음 | 경로 매칭·인코딩 우회 같은 함정을 직접 막아야 한다 |
| Spring Security + 토큰(`spa()`) | 프론트가 **토큰을 먼저 받아** 헤더로 보내야 함. 쿠키가 하나 늘고 API 계약이 바뀜 | 첫 POST 전에 토큰을 받는 요청이 필요. 토큰을 세션에 저장하는 기본 저장소를 쓰면 **익명 방문자에게도 세션이 생긴다** |
| Spring Security + Origin 필터를 체인에 추가(`csrf.disable()` + `addFilterBefore`) | 계약 그대로, 기본 보안 헤더를 얻음 | Origin 검사는 여전히 직접 작성. 기본값(전부 인증 필요, 폼 로그인, 기본 403 응답 형식)을 하나씩 꺼야 함 |
| CORS 설정으로 대신(`http.cors()` 또는 Spring MVC 전역 CORS) | 허용 안 된 Origin의 교차 요청을 프레임워크가 403으로 거절 | Origin 없는 요청은 통과, 거절 응답 형식이 다름, 허용 origin에 응답 읽기 권한까지 열림 — 아래 표 |

💡 **인증(로그인)이 없는 동안 Spring Security의 핵심 가치는 대부분 쓰이지 않는다.** CSRF만 위해 들이면 토큰 흐름이 추가되거나, 결국 같은 커스텀 필터를 Spring Security 안에 넣게 된다. **회원 로그인이 생기는 시점**이 들일 때다 — 그때는 인증·인가·세션 고정 방지까지 한 번에 얻는다.

## ⚠️ 함정

- **XSS가 있으면 CSRF 방어는 전부 소용없다.** 공격자 스크립트가 **우리 페이지 안에서** 돌면, 그 스크립트가 보내는 `fetch`는 Origin이 우리 주소라 통과하고, 같은 사이트라 SameSite 쿠키가 실리고, 같은 origin이라 JSON도 preflight 없이 나가고, 페이지에서 읽을 수 있는 CSRF 토큰(`XSRF-TOKEN`은 일부러 HttpOnly가 아니다)도 읽힌다. HttpOnly는 세션 쿠키 값을 **읽어 가는 것**만 막을 뿐, 그 쿠키로 **요청을 보내는 것**은 못 막는다. 그래서 순서는 "XSS를 먼저 막고, 그 위에 CSRF 방어"다 — 출력할 때 이스케이프(React·Next.js는 기본으로 해 준다. `dangerouslySetInnerHTML`이나 마크다운을 HTML로 바꿔 넣는 곳이 구멍), 그리고 CSP로 허용한 출처의 스크립트만 실행되게 하는 두 번째 겹. **사용자 입력이 AI 프롬프트에 들어가고 AI 출력을 화면에 보여 주는 서비스**라면 AI 출력도 사용자 입력처럼 취급해서 이스케이프한다. [OWASP — XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html), [MDN — CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP)
- **Origin 검사(와 CSRF 토큰)는 훔친 쿠키를 막지 못한다.** 공격자가 세션 쿠키 값 자체를 손에 넣어 Postman에 넣으면, Origin도 허용된 값으로 직접 적을 수 있고 CSRF 토큰도 그 쿠키로 새로 받아 올 수 있다. 서버 입장에선 진짜 사용자와 같다. 이건 CSRF가 아니라 **세션 탈취(session hijacking)**이고, 방어는 "못 훔치게" 하는 쪽이다 — `HttpOnly`(스크립트가 못 읽음), `Secure`(HTTPS로만 전송), 세션 ID를 URL·로그에 남기지 않기, 만료 시간. 정리하면 CSRF = 공격자가 쿠키 값을 **모른 채** 피해자 브라우저가 붙이게 만드는 것, 탈취 = 쿠키 값을 **알고** 직접 붙이는 것.
- **GET에서 Origin을 필수로 요구하면 정상 요청이 막힌다.** 같은 origin GET에는 원래 안 붙는다. 그래서 Origin 검사는 상태를 바꾸는 메서드에만 건다. GET은 다른 origin에서 보내도 CORS 때문에 응답을 읽을 수 없어 CSRF로 얻는 게 적다 — 단 **GET이 상태를 바꾸지 않을 때만** 성립한다.
- **허용 목록은 정확히 일치로 비교한다.** `startsWith("https://app.example.com")`이면 `https://app.example.com.evil.com`이 통과한다. 스킴·호스트·포트까지 문자열 전체가 같아야 한다.
- **Host와 헷갈리지 말 것.** `Host`는 요청이 **도착하는** 서버 이름, `Origin`은 요청을 **출발시킨** 페이지의 주소다.
- **Referer로 대신하지 말 것.** 경로까지 담아 개인정보가 새기 쉽고, `Referrer-Policy`로 아예 빠지는 경우가 많다.
- **로컬 개발에서는 origin이 포트까지 포함한다.** 프론트 `http://localhost:3000`과 API `http://localhost:8080`은 **다른 origin**이다(쿠키는 포트를 구분하지 않는 것과 대비 — [세션과 쿠키](./sessions-and-cookies.md)).
- **필터 클래스 이름이 `SecurityConfig`라도 Spring Security가 아닐 수 있다.** 의존성에 `spring-boot-starter-security`가 없으면 CSRF 토큰도 없다 — 이름이 아니라 실제로 어떤 방어가 켜져 있는지 확인한다.

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
- 학습일: 2026-10-01. 계기: 익명 쿠키 세션 API가 POST에서 Origin을 검사하는 코드를 보고 "Origin은 누가 던지는 거야?", "평소 보던 요청엔 없던데"라는 질문. 같은 origin GET에는 원래 Origin이 없다는 규칙을 MDN으로 확인. 예시 요청은 일반화했다.

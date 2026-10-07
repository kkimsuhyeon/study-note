# XSS와 CSP — 남의 스크립트가 우리 페이지에서 돌지 못하게

> **한 줄 요약:** XSS는 공격자가 넣은 스크립트가 **우리 페이지 안에서** 실행되는 공격이다. 1차 방어는 **HTML을 만드는 곳에서 출력을 이스케이프**하는 것이고, CSP는 "이 페이지에서 실행해도 되는 스크립트 목록"을 브라우저에 알려 주는 **응답 헤더**로, 이스케이프가 한 곳에서 실수했을 때 버티는 2차 방어다.

## 언제 쓰나

- 사용자 입력(이름, 댓글, 검색어)이나 **외부에서 받은 글**(AI 출력, 다른 API 응답)을 화면에 보여 주는 모든 곳.
- 특히 **저장된 데이터가 다른 사람에게 보이는 곳**(공유 페이지, 게시판). 한 번 심으면 보는 사람마다 실행된다.
- XSS가 뚫리면 CSRF 방어(Origin 검사·SameSite·CSRF 토큰)는 전부 무력해진다 — 스크립트가 우리 origin에서 돌기 때문이다([Origin 헤더](./origin-header.md) ⚠️).

## XSS의 세 종류

| 종류 | 공격 경로 | 예 |
| --- | --- | --- |
| 저장형(stored) | DB에 저장된 값이 나중에 화면에 출력 | 공유 페이지에 보이는 이름·AI가 생성한 글에 `<img src=x onerror=...>` |
| 반사형(reflected) | 요청 값이 응답에 그대로 다시 출력 | 검색 결과 페이지 "`<검색어>`에 대한 결과" |
| DOM 기반 | 프론트 코드가 URL·입력을 DOM에 직접 넣음 | `element.innerHTML = location.hash` |

## 누가 무엇을 막나

| 위치 | 하는 일 |
| --- | --- |
| **HTML을 만드는 곳**(SPA면 프론트, 서버 렌더링이면 그 서버) | 출력 이스케이프. **1차 방어의 주인** |
| 백엔드 API | 입력 규칙으로 공격 표면 줄이기(허용 문자 제한), JSON 응답에 `Content-Type: application/json` + `X-Content-Type-Options: nosniff`로 JSON이 HTML로 해석되지 않게 |
| 페이지(HTML)를 내려 주는 서버 | CSP 헤더. 2차 방어 |

## 사용 예시 1 — React/Next.js의 이스케이프

```tsx
// 안전: {}로 넣은 문자열은 React가 텍스트로 이스케이프한다
<p>{analysis.summary}</p>

// 위험: 문자열을 HTML로 해석시킨다. 마크다운→HTML 변환 결과를 넣는 것도 같다
<div dangerouslySetInnerHTML={{ __html: analysis.summary }} />
```

- 꼭 HTML로 넣어야 하면 DOMPurify 같은 새니타이저를 거친다.
- 링크 `href`에 사용자 값을 넣을 때는 `javascript:` 스킴이 아닌지 확인한다.

## 사용 예시 2 — CSP 헤더

```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-r4nd0m' 'strict-dynamic';
  object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none'
```

| 지시어 | 뜻 |
| --- | --- |
| `default-src 'self'` | 따로 정하지 않은 리소스는 같은 origin에서만 |
| `script-src 'nonce-…'` | 이 요청마다 바뀌는 난수(nonce)가 붙은 `<script>`만 실행. 공격자가 넣은 스크립트엔 nonce가 없어서 막힘 |
| `'strict-dynamic'` | nonce로 허락된 스크립트가 불러오는 스크립트도 허락 |
| `object-src 'none'` | 플러그인(`<object>`) 금지 |
| `base-uri 'self'` | `<base>` 태그로 상대 경로를 가로채는 것 금지 |
| `form-action 'self'` | 폼을 다른 곳으로 전송 금지 |
| `frame-ancestors 'none'` | 다른 사이트가 이 페이지를 iframe에 넣지 못함(클릭재킹 방어, `X-Frame-Options`의 후속) |

- `onerror=`, `onclick=` 같은 **인라인 이벤트 핸들러**도 nonce가 없으니 막힌다 — 대부분의 XSS 페이로드가 여기서 죽는다.
- 처음엔 `Content-Security-Policy-Report-Only`로 걸어 **막지 않고 위반만 보고**받으며 정상 기능이 깨지는지 본다.

## Next.js에서의 CSP

- Next.js(16 기준)는 `proxy`에서 요청마다 nonce를 만들어 CSP 헤더에 넣으면, 렌더링할 때 그 헤더를 읽어 **자기 프레임워크 스크립트·번들·인라인 스크립트에 nonce를 자동으로 붙인다**.
- 그러려면 페이지가 **동적 렌더링**이어야 한다. 정적 페이지는 빌드 시점에 만들어져 요청 헤더가 없으므로 nonce를 넣을 수 없다 → CDN 캐시 이점을 일부 포기하는 트레이드오프.
- 개발 모드에서는 React 디버깅 때문에 `'unsafe-eval'`이 필요하고, 운영에서는 필요 없다.

## ⚠️ 함정

- **nonce·hash 없이 `'unsafe-inline'`만 넣으면 XSS 방어 효과가 거의 사라진다.** "일단 화면이 깨지니까" 넣는 순간 공격자의 인라인 스크립트도 허락된다. 반대로 nonce나 hash가 있는 정책에서는 CSP2 이상 브라우저가 `'unsafe-inline'`을 **무시**하고, `'strict-dynamic'`이 있으면 `'self'`·호스트 허용 목록도 무시한다. 그래서 MDN은 구형 브라우저 호환용으로 `script-src 'unsafe-inline' https: 'nonce-…' 'strict-dynamic'` 조합을 예로 든다 — 같은 `'unsafe-inline'`이라도 옆에 무엇이 있느냐로 의미가 바뀐다.
- **CSP는 HTML 문서 응답에 걸어야 의미가 있다.** JSON API 응답에만 걸면 페이지는 보호되지 않는다. 프론트와 API가 다른 서버라면 프론트(또는 앞단 프록시) 쪽 설정이다.
- **입력 검증은 XSS 방어의 대체가 아니다.** 허용 문자를 제한하면 표면은 줄지만, 다른 경로(AI 출력, 외부 API, 나중에 규칙이 느슨해진 필드)로 들어온 값은 막지 못한다. 출력 시 이스케이프는 값의 출처와 상관없이 항상 한다.
- **AI 출력도 신뢰하지 않는 입력이다.** 사용자 입력이 프롬프트에 섞이면(프롬프트 인젝션) AI가 HTML·스크립트 조각을 그대로 내보낼 수 있다.

## 💡 판단 기준

**"이 값은 결국 화면 어디에 어떻게 그려지나?"를 값마다 묻는다.** 텍스트로 그려지면(React `{}`) 이미 안전하고, HTML로 그려지면(innerHTML·마크다운 변환) 새니타이저가 필수다. 그 위에 CSP를 nonce 방식으로 걸어 "한 곳의 실수"가 실행으로 이어지지 않게 한다. 저장된 값이 **다른 사람에게 보이는** 화면(공유 페이지)부터 점검한다.

## 참고

- [OWASP — Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [MDN — Content Security Policy (CSP)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP)
- [MDN — script-src (`'strict-dynamic'`이 `'unsafe-inline'`·`'self'`를 무시, 하위 호환 예시)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/script-src)
- [Next.js — Content Security Policy guide](https://nextjs.org/docs/app/guides/content-security-policy)
- 학습일: 2026-10-01 (2026-10-02 `'unsafe-inline'` 조건 정정). 계기: CSRF 방어를 정리하다 "XSS가 뚫리면 CSRF 방어가 무력"이라는 점에서 "XSS는 프론트에서 막나? CSP는 뭐야?"라는 질문. Next.js의 nonce·동적 렌더링 조건은 공식 문서로 확인했고 예시 코드는 실행하지 않았다.

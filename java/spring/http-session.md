# HttpSession — 서버 세션을 Java 코드에서 읽고 쓰는 API

> **한 줄 요약:** `HttpSession`은 세션 ID·속성·수명을 다루는 Servlet API다. `getSession(false)`는 기존 세션만 찾고, `getSession()`은 없으면 새로 만든다.

읽는 순서: [세션·쿠키 기초](../../infra/network/sessions-and-cookies.md) → **이 노트** → [Spring Session JDBC](./spring-session-jdbc.md)

## 1. 언제 쓰나

방문자의 언어 설정, 로그인 후 사용자 식별자, 비회원 장바구니 ID처럼 **다음 요청에서도 서버가 알아야 하는 작은 값**을 저장할 때 사용한다.

```java
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpSession;
```

`HttpSession`은 Spring 전용 클래스가 아닌 Jakarta Servlet의 **인터페이스**다. Tomcat 같은 컨테이너가 구현을 제공하며, Spring Session을 도입하면 저장소와 구현을 바꿔 사용할 수 있다.

여기서 인터페이스란 개발자가 호출할 수 있는 메서드의 약속이다. `getAttribute()`·`setAttribute()`라는 사용법은 같아도, 아래 구현은 메모리에서 값을 꺼낼 수도 있고 DB에서 복원한 값을 꺼낼 수도 있다.

### request와 session의 수명이 다르다

| 객체 | 담당하는 범위 | 담는 정보 예시 |
| --- | --- | --- |
| `HttpServletRequest` | 이번 HTTP 요청 | URL, 헤더, 쿠키, 요청 본문 |
| `HttpSession` | 같은 세션 ID로 이어지는 여러 요청 | 언어 설정, 사용자·장바구니 식별자 |

페이지를 새로고침하면 서버는 새 요청을 처리한다. 하지만 쿠키가 같은 유효한 세션을 가리키면, 새 요청에서도 이전 세션 상태를 사용할 수 있다.

**“같은 세션”은 논리적으로 같은 ID·상태라는 뜻이다.** DB에서 매 요청마다 복원할 수도 있으므로 Java 객체의 메모리 주소가 항상 같다는 뜻은 아니다. `HttpSession` 객체를 직접 싱글턴 Bean으로 만들어 모든 방문자가 공유하게 하면 안 된다.

## 2. `getSession(false)`의 false는 무엇인가?

```java
HttpSession session = request.getSession(false);
```

뜻은 **“이번 요청에 연결된 유효한 세션을 가져오되, 없으면 생성하지 마”**다. 쿠키가 없거나 그 ID에 해당하는 세션이 만료·삭제됐다면 `null`일 수 있다.

| 호출 | 유효한 세션이 있음 | 유효한 세션이 없음 |
| --- | --- | --- |
| `request.getSession(false)` | 기존 세션 반환 | `null` |
| `request.getSession(true)` | 기존 세션 반환 | 새 세션 생성 |
| `request.getSession()` | 기존 세션 반환 | 새 세션 생성 |

`false`는 읽기 전용 트랜잭션이나 로그인 여부를 의미하지 않는다. **없을 때 만들지 말라**는 옵션이다. 세션 접근에 따른 마지막 접근 시각 갱신은 별개다. [HttpServletRequest API](https://jakarta.ee/specifications/servlet/6.1/apidocs/jakarta.servlet/jakarta/servlet/http/httpservletrequest)

### 한 줄을 순서대로 해석하기

```java
HttpSession session = request.getSession(false);
```

1. `request`: 현재 방문자가 보낸 요청을 나타내는 객체다.
2. `getSession(...)`: 현재 요청과 연결된 세션을 구한다. 현재 사용하는 세션 구현이 ID 해석과 조회를 맡는다.
3. `false`: 못 찾았다고 새 세션을 만들지 않는다.
4. `HttpSession session`: 찾은 세션을 지역 변수에 담는다. 못 찾으면 `null`이 들어간다.

이 줄이 반환하는 것은 **쿠키 문자열이 아니라 세션을 다루는 객체**다. `request.getCookies()`와는 목적이 다르다.

## 3. 저장 → 다음 요청에서 조회 → 제거

아래는 웹 요청 처리 메서드 내부에 들어가는 학습용 코드 조각이다. `request`는 `HttpServletRequest` 매개변수다.

**저장:**

```java
HttpSession session = request.getSession();
session.setAttribute("preferredLanguage", "ko");
```

`setAttribute(이름, 값)`는 Map에 값을 넣듯 사용한다. 같은 이름에 다시 넣으면 기존 값을 교체한다. 사용자 요청이 끝나도 유효한 세션에 속성이 남아 다음 요청에서 읽을 수 있다.

**조회:**

```java
HttpSession session = request.getSession(false);
String language = session == null
        ? null
        : (String) session.getAttribute("preferredLanguage");

if (language == null) {
    language = "en";
}
```

`getAttribute()`의 반환 타입은 `Object`여서 저장한 타입으로 변환한다. **세션이 없는 경우와 속성이 없는 경우는 다르며 둘 다 처리**해야 한다.

```text
세션 자체가 없음                 → session == null
세션은 있지만 언어를 저장 안 함 → getAttribute(...) == null
세션에 언어를 저장함            → "ko"
```

이름은 대소문자까지 일치해야 한다. `preferredLanguage`로 저장한 값을 `language`로 찾으면 다른 속성이므로 `null`이 나온다. 같은 이름에 정수를 저장한 뒤 `String`으로 캐스팅하면 타입 오류가 발생한다. 여러 곳에서 사용하는 속성 이름은 상수로 두면 오타를 줄일 수 있다.

**속성 하나만 제거:**

```java
HttpSession session = request.getSession(false);
if (session != null) {
    session.removeAttribute("preferredLanguage");
}
```

**세션 전체 무효화:**

```java
HttpSession session = request.getSession(false);
if (session != null) {
    session.invalidate();
}
```

무효화한 세션 객체를 계속 사용하면 `IllegalStateException`이 발생할 수 있다. 기존 쿠키가 요청에 남아 있어도 무효화된 서버 세션의 권한이 되살아나지는 않는다. [HttpSession API](https://jakarta.ee/specifications/servlet/6.1/apidocs/jakarta.servlet/jakarta/servlet/http/httpsession)

### `request.setAttribute()`와는 무엇이 다른가?

```java
request.setAttribute("preferredLanguage", "ko");
```

위 코드는 **이번 요청에만 값**을 붙인다. 같은 요청을 처리하는 필터·Controller 등이 공유할 수 있지만, 다음 HTTP 요청까지 이어지는 세션 속성은 아니다.

```java
request.getSession().setAttribute("preferredLanguage", "ko");
```

이 코드는 **세션에 값**을 넣는다. 유효한 같은 세션으로 다음 요청이 들어오면 다시 읽을 수 있다. 이름이 같은 `setAttribute`라도 호출 대상이 다르면 수명이 다르다.

## 4. 자주 쓰는 메서드

| 메서드 | 의미 |
| --- | --- |
| `getId()` | 세션 식별자 조회. 업무 사용자 ID와 구분 |
| `setAttribute(name, value)` | 이름으로 값 저장·교체 |
| `getAttribute(name)` | 값 조회, 없으면 `null` |
| `removeAttribute(name)` | 속성 하나 제거 |
| `invalidate()` | 세션 전체 무효화 |
| `setMaxInactiveInterval(1800)` | 요청 사이 최대 비활성 시간 1,800초 설정 |
| `request.changeSessionId()` | 세션 속성을 유지하며 세션 ID 변경 |

타임아웃은 일반적으로 **마지막 요청 접근 후의 비활성 시간**이다. 생성 후 무조건 30분이 지나면 끝나는 절대 만료와 구분한다. 속성 메서드를 여러 번 호출하는 것 자체가 요청 접근 시각을 계속 갱신한다는 뜻도 아니다.

## 5. Spring MVC Controller에서는 어떻게 받나?

다음 메서드는 `@RestController` 안에 있다고 가정한다.

```java
@GetMapping("/language")
public String language(HttpSession session) {
    return (String) session.getAttribute("preferredLanguage");
}
```

MVC가 `HttpSession` 매개변수를 처리할 때는 세션을 얻기 위해 생성할 수 있다. **없는 세션을 만들고 싶지 않은 조회 API**라면 `HttpServletRequest`를 받고 `getSession(false)`를 명시하는 편이 의도가 분명하다. 반환·기본값 처리는 API에 맞게 추가한다.

```java
@GetMapping("/language")
public String language(HttpServletRequest request) {
    HttpSession session = request.getSession(false);
    if (session == null) {
        return "en";
    }

    String language = (String) session.getAttribute("preferredLanguage");
    return language == null ? "en" : language;
}
```

위 두 메서드는 같은 경로의 **대안**이므로 한 Controller에 동시에 넣지 않는다. 아래 방식은 처음 방문한 사람에게 기본 언어만 알려주면서 불필요한 세션을 만들지 않는다.

세션 접근은 Controller나 웹 전용 컴포넌트에 두고, 업무 서비스에는 `cartId`·`userId`처럼 필요한 값만 전달할 수 있다. 업무 계산이 쿠키와 Servlet API를 알 필요는 없다.

### 세션을 언제 만들지는 업무 정책이다

입력 검증 전에 `getSession()`을 호출하면, 나중에 입력 오류를 반환해도 세션이 이미 생성될 수 있다. “성공한 처리에만 새 쿠키를 발급한다”는 요구가 있으면 생성 시점을 성공 이후로 옮겨야 한다.

반대로 방문 시점부터 장바구니·사용 언어를 기억해야 한다면 먼저 생성하는 것도 가능하다. **`getSession()` 자체가 나쁜 게 아니라, 필요한 시점에 호출하는지가 중요하다.**

## 6. ⚠️ 함정과 메커니즘

**세션이 있어도 로그인했다고 판단하면 안 된다.** `getSession()`은 인증 검사 없이 세션을 만들 수 있다. 인증 여부는 인증 정보로 확인해야 한다. Spring Security를 사용한다면 직접 로그인 상태 플래그를 만들기보다 그 인증·로그아웃 흐름에 맡긴다.

**세션 ID를 영구 소유자 ID로 쓰지 않는다.** 로그인 과정 등에서 ID가 바뀔 수 있다. 안정적인 사용자·익명 소유자 식별자는 세션 속성으로 별도 보관한다. Spring Security는 세션 고정 공격 방지를 위해 인증 시 세션 ID 변경을 지원한다. [세션 보안](https://docs.spring.io/spring-security/reference/servlet/authentication/session-management.html)

**같은 세션의 요청도 동시에 실행될 수 있다.** 세션의 값을 읽고 1을 더해 다시 저장하는 작업은 자동으로 원자적이지 않다. 결제·재고·정확한 요청 횟수 같은 규칙은 DB 트랜잭션 등 적합한 수단으로 처리한다. 관련: [Read-Modify-Write](../jpa/read-modify-write.md).

**큰 객체·JPA Entity를 통째로 넣지 않는다.** 메모리 비용, 직렬화, 오래된 데이터 문제를 줄이려면 식별자와 작은 값 위주로 저장하고 업무 데이터는 필요할 때 조회한다.

## 7. 작은 예시로 전체 흐름 복습

언어를 저장하는 요청과 조회하는 요청은 서로 다른 요청이다.

```text
첫 요청
  request.getSession(false) → null
  request.getSession()      → 새 세션 S1
  setAttribute("preferredLanguage", "ko")
  응답: S1을 가리키는 쿠키 발급

다음 요청 (같은 쿠키 전송)
  request.getSession(false) → 세션 S1
  getAttribute("preferredLanguage") → "ko"

세션 만료 후 요청 (쿠키가 남아 있어도)
  request.getSession(false) → null
```

확인 질문:

1. `getSession(false)`가 `null`이면 반드시 “이 사람은 회원이 아니다”라는 뜻인가?
2. `removeAttribute("preferredLanguage")`를 호출하면 세션 ID도 사라지는가?
3. `HttpSession`을 Controller 매개변수로 직접 받으면 세션 생성 정책이 달라질 수 있는가?

답: **아니다, 이 요청에서 유효한 세션을 못 찾은 것 / 아니다, 속성만 제거 / 그렇다, 세션이 없을 때 생성할 수 있다.**

## 참고·학습 기록

- [Jakarta Servlet — HttpServletRequest](https://jakarta.ee/specifications/servlet/6.1/apidocs/jakarta.servlet/jakarta/servlet/http/httpservletrequest)
- [Jakarta Servlet — HttpSession](https://jakarta.ee/specifications/servlet/6.1/apidocs/jakarta.servlet/jakarta/servlet/http/httpsession)
- [Spring MVC — Controller 메서드 매개변수](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/arguments.html)
- [Spring Security — 세션 관리](https://docs.spring.io/spring-security/reference/servlet/authentication/session-management.html)
- 기준: Jakarta Servlet 6.1, Spring MVC 7 계열. 코드 조각은 API 설명용이며 독립 실행 애플리케이션은 아니다.
- 학습일: 2026-09-21. 계기: `request.getSession(false)`가 무엇을 찾고 왜 `null`을 반환하는지 이해하기.

💡 **기존 방문자의 권한만 확인할 때는 `getSession(false)`, 실제로 이어갈 상태를 만들기로 결정했을 때는 `getSession()`을 선택한다.**

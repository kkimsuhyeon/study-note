# 앞뒤 공백 제거 정규식의 ReDoS — `\s+$`는 입력 길이의 제곱으로 느려진다

> **한 줄 요약:** `value.replaceAll("^\\s+|\\s+$", "")` 같은 앞뒤 공백 제거 정규식은 **긴 공백 구간 뒤에 공백이 아닌 글자가 오면** 시작 위치마다 공백 구간 끝까지 먹고 `$`에서 실패한 뒤 되돌아가므로 O(n²)이 된다. 길이 제한이 없는 입력에 쓰면 요청 하나로 스레드를 수십 초씩 묶을 수 있다. 선형 루프나 `String.strip()`으로 바꾸고, 입력 길이 상한을 먼저 둔다.

---

## 1. 언제 쓰나

- 사용자 입력(이름, 제목 등)의 앞뒤 공백을 정리할 때.
- 정규식으로 "끝에 붙은 X" (`X+$`)를 찾는 코드를 리뷰할 때. `\s+$`, `[ \t]+$`, `,+$`처럼 **같은 문자 클래스의 반복 + 끝 앵커**는 모두 같은 함정이다.

## 2. 사용 예시

```java
// ❌ 위험: 공백 구간 길이 n에 대해 O(n²)
private static final String EDGE_SPACES = "^[\\p{javaWhitespace}\\p{Zs}]+|[\\p{javaWhitespace}\\p{Zs}]+$";
String trimmed = value.replaceAll(EDGE_SPACES, "");
```

JDK 25에서 `"a" + " ".repeat(n) + "b"`에 위 코드를 돌려 측정한 결과(직접 실행):

| 공백 수 n | 시간 |
|---|---|
| 5,000 | 62 ms |
| 10,000 | 226 ms |
| 20,000 | 869 ms |
| 40,000 | 3,455 ms |

n이 2배가 되면 시간은 약 4배가 된다(제곱). 이 추세라면 200KB 본문 하나로 1분 이상 걸린다.

**왜 느린가:** `replaceAll`은 문자열의 각 위치에서 매칭을 시도한다. 공백 구간 안의 위치 i에서 `[공백]+`가 구간 끝까지 탐욕적으로 먹은 뒤 `$`를 확인하는데, 그 뒤에 `b`가 있어 실패한다. 그러면 한 글자씩 되돌아가며 다시 `$`를 확인하고 모두 실패한다. 위치 i마다 (구간 끝 − i)번 일하므로 합계가 n²/2다.

```java
// ✅ 선형: 앞에서 한 번, 뒤에서 한 번만 훑는다 (원래 정규식과 같은 공백 정의)
static String stripEdgeSpaces(String value) {
    int start = 0;
    int end = value.length();
    while (start < end) {
        int cp = value.codePointAt(start);
        if (!isEdgeSpace(cp)) break;
        start += Character.charCount(cp);
    }
    while (end > start) {
        int cp = value.codePointBefore(end);
        if (!isEdgeSpace(cp)) break;
        end -= Character.charCount(cp);
    }
    return value.substring(start, end);
}

static boolean isEdgeSpace(int cp) {
    return Character.isWhitespace(cp) || Character.getType(cp) == Character.SPACE_SEPARATOR;  // \p{javaWhitespace} | \p{Zs}
}
```

## 3. 옵션 비교

| 방법 | 공백으로 보는 범위 | 시간 | 비고 |
|---|---|---|---|
| `trim()` | `<= U+0020` 제어문자·공백 | O(n) | 전각 공백(U+3000)·NBSP(U+00A0)를 못 지운다 |
| `strip()` (Java 11+) | `Character.isWhitespace` | O(n) | 전각 공백은 지우지만 NBSP·U+2007·U+202F(줄바꿈 없는 공백)는 남긴다 |
| 정규식 `^\s+\|\s+$` | 패턴에 따라 | **O(n²)** | 위 함정 |
| 직접 루프(위 예시) | 원하는 대로(`isWhitespace` ∪ `Zs`) | O(n) | 기존 정규식과 정의를 정확히 맞출 수 있다 |

- 정규식을 꼭 써야 하면 끝 쪽 앵커 반복을 피하거나, **입력 길이 상한을 먼저 검사**해 n 자체를 작게 만든다.

## 4. ⚠️ 함정

- **검증보다 정리가 먼저 돈다.** "이름은 1~20자"라는 검증이 있어도, 공백 제거를 *먼저* 하고 길이를 재는 코드라면 길이 검사가 공격을 막지 못한다. 원문 길이 상한(예: 요청 DTO의 `@Size(max = …)`)은 정리 **전**에 걸어야 한다.
- **JSON 본문 크기는 기본으로 제한되지 않는 경우가 많다.** Tomcat의 `max-http-form-post-size`는 폼 전송용이라 JSON 본문에는 적용되지 않는다(일반 지식). 프록시나 서버 설정으로 본문 상한을 따로 둔다.
- **요청 한도(rate limit)보다 앞에서 돌면** 한도로도 못 막는다. 입력 정규화는 보통 유스케이스 맨 앞에서 일어나기 때문이다.
- 실제 사례: 2016년 Stack Overflow 장애가 정확히 이 모양의 정규식(`^[\s‌]+|[\s‌]+$`)과 공백 2만 개짜리 게시물 하나 때문이었다(사후 보고서 공개).

## 5. 💡 판단 기준

- **사용자 입력에 도는 정규식을 보면 "가장 나쁜 입력은 뭐고, 길이 제한은 어디서 걸리나"를 묻는다.** 반복(`+`, `*`) 뒤에 실패할 수 있는 조건(`$`, 다른 문자)이 붙어 있고 길이 상한이 정리보다 뒤에 있으면 의심한다.
- 앞뒤 정리 같은 단순 작업은 정규식보다 `strip()`이나 직접 루프가 빠르고 안전하다.

## 참고·학습 기록

- 2026-10-12: 코드 리뷰에서 표시 이름 정리 정규식이 요청 한도보다 앞에서, 길이 상한 없이 도는 것을 발견하고 JDK 25로 시간 증가를 측정했다.
- Stack Overflow 2016-07-20 장애 사후 보고서(stackstatus "Outage Postmortem - July 20, 2016").

# 문자열 길이 — `length()` vs `codePointCount()` vs 사람이 보는 글자

> **한 줄 요약:** Java `String.length()`는 글자 수가 아니라 UTF-16 칸(`char`) 수다. "몇 글자"를 제한하려면 `codePointCount()`로 유니코드 문자 수를 세고, 이모지·결합 문자까지 사람 눈의 글자로 세야 하면 `BreakIterator`를 쓴다.

## 1. 언제 쓰나

- 이름·제목·닉네임처럼 **"N자 이내"** 규칙을 검사할 때
- AI·외부 API 응답의 문자열 길이 상한을 검사할 때 (JSON Schema의 `maxLength`도 유니코드 문자 수 기준)
- DB 열 길이와 비교할 때 — 단, DB의 `VARCHAR(n)`이 문자 수인지 바이트 수인지는 DB마다 다르다

## 2. 사용 예시

Java의 `String`은 내부적으로 UTF-16이다. 대부분의 글자(한글 음절, 영문, 기본 한자)는 `char` 1칸이지만, **U+FFFF를 넘는 문자(이모지, 확장 한자 등)는 `char` 2칸(서로게이트 쌍)**을 쓴다.

```java
String name = "수현";
name.length();                              // 2
name.codePointCount(0, name.length());      // 2   — 한글 음절은 둘이 같다

String rare = "𠮷";                          // 확장 한자(U+20BB7)
rare.length();                              // 2   ← 한 글자인데 2
rare.codePointCount(0, rare.length());      // 1

String smile = "😀";
smile.length();                             // 2
smile.codePointCount(0, smile.length());    // 1
```

`codePointCount(시작, 끝)`은 `[시작, 끝)` 범위의 `char`들을 읽어 **유니코드 문자(code point) 몇 개인지** 센다. 서로게이트 쌍 두 칸을 하나로 센다. 전체를 셀 때는 `s.codePointCount(0, s.length())` 또는 `(int) s.codePoints().count()`.

(JDK 25 jshell로 위 값 확인)

## 3. 옵션 비교

| 방법 | 세는 단위 | "😀" | "👍🏽"(피부색 이모지) | NFD로 쪼개진 "한" |
| --- | --- | --- | --- | --- |
| `length()` | UTF-16 칸 | 2 | 4 | 3 |
| `codePointCount()` | 유니코드 문자 | 1 | 2 | 3 |
| `BreakIterator.getCharacterInstance()` | 사람이 보는 글자(grapheme) | 1 | 1 | 1 |

- 한글 이름처럼 BMP 안의 글자만 받는 규칙이면 `codePointCount()`로 충분하다.
- 이모지 조합·결합 문자까지 "보이는 대로" 세야 하면 `BreakIterator`(또는 정규식 `\X`).

## 4. ⚠️ 함정

- **`length()`로 "20자 제한"을 걸면** 확장 한자·이모지가 섞인 이름은 실제보다 일찍 잘린다. `substring(0, 20)`으로 자르면 서로게이트 쌍 중간을 끊어 깨진 문자가 생길 수도 있다.
- **같은 모양, 다른 문자 수 — 유니코드 정규화.** "한"은 보통 한 문자(NFC, U+D55C)지만, 초성·중성·종성 세 문자(NFD)로 들어올 수도 있다. 화면엔 똑같이 보이는데 `codePointCount()`가 3이다. 길이 규칙이 중요하면 검사 전에 `Normalizer.normalize(s, Normalizer.Form.NFC)`로 맞춘다.
- **바이트 수와 혼동 금지.** HTTP 본문·DB 바이트 제한은 `s.getBytes(UTF_8).length`로 따로 센다. 한글 음절은 UTF-8로 3바이트다.
- 정규식 `[\p{L}]`은 "문자(letter)"만 허용하므로 이모지는 애초에 걸러진다. 길이 규칙과 허용 문자 규칙을 함께 보면 실제로 들어올 수 있는 입력이 좁혀진다.

## 5. 💡 판단 기준

**"글자 수" 규칙은 기본을 `codePointCount()`로 둔다.** `length()`는 "UTF-16 칸 수"가 필요할 때만, 사람이 보는 글자 그대로가 필요할 때(이모지 허용 입력)는 `BreakIterator`. 길이가 계약(JSON Schema `maxLength`, API 문서)에 적혀 있으면 그 계약이 무엇을 세는지 먼저 확인한다.

## 참고·학습 기록

- [Java 25 — String.codePointCount](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/String.html#codePointCount(int,int))
- [Java 25 — BreakIterator](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/text/BreakIterator.html)
- [JSON Schema — string length는 문자 수(code point) 기준](https://json-schema.org/understanding-json-schema/reference/string#length)

**학습 날짜**: 2026-10-11
**계기**: 이름 1~20자 검사 코드에서 `length()` 대신 `codePointCount()`를 쓰는 이유가 궁금해서.

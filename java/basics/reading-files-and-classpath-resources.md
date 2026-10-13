# 파일 읽기 — 디스크 파일 vs 클래스패스 리소스, 한 번에 vs 스트림

> **한 줄 요약**: Java의 파일 읽기 방법이 많아 보이는 건 **두 질문의 조합**이라서다. ① 어디에 있나 — 디스크의 파일(`Path`)인가, 앱과 함께 묶여 배포되는 **클래스패스 리소스**인가 ② 어떻게 읽나 — 한 번에 문자열로 읽나, 스트림으로 나눠 읽나. `src/main/resources`에 둔 파일은 배포 jar 안에 들어가서 **디스크 파일이 아니게 된다** — 그래서 `getFile()`·`Path`로 읽는 코드는 IDE에서는 되고 jar로 실행하면 실패한다.

## 언제 쓰나

- 앱과 함께 배포되는 고정 텍스트(프롬프트, JSON 스키마, SQL, 템플릿)를 시작할 때 읽을 때 → 클래스패스 리소스
- 실행 환경마다 바뀌는 파일(업로드 파일, 외부 설정, 로그)을 읽을 때 → 디스크 파일(`Path`)
- 큰 파일을 줄 단위로 처리할 때 → 스트림

## 두 축으로 나눈 방법 지도

| | 한 번에 다 읽기 | 스트림으로 읽기 |
|---|---|---|
| **디스크 파일** (JDK) | `Files.readString(path, UTF_8)` (Java 11+), `Files.readAllBytes(path)`, `Files.readAllLines(path)` | `Files.lines(path)`(try-with-resources 필수), `Files.newBufferedReader(path)`, `Files.newInputStream(path)` |
| **클래스패스** (JDK) | `in.readAllBytes()`로 직접 | `Class.getResourceAsStream(...)`, `ClassLoader.getResourceAsStream(...)` |
| **클래스패스** (Spring) | `new ClassPathResource("a/b.txt").getContentAsString(UTF_8)` (6.0.5+), `getContentAsByteArray()` | `resource.getInputStream()` |
| **위치를 설정으로** (Spring) | `@Value("classpath:a/b.txt") Resource r` → `r.getContentAsString(UTF_8)` | `r.getInputStream()` |

- 옛 방식 `new FileReader(file)`·`new FileInputStream(file)`은 디스크 파일 전용이다. `FileReader`는 Java 18 전에는 OS 기본 문자셋을 썼다(JEP 400으로 18부터 기본 UTF-8). 어느 버전이든 문자셋은 명시한다.
- Spring의 `Resource`는 위치를 문자열 접두사로 바꿔 끼운다: `classpath:`(앱 안), `file:`(디스크), `https:`(URL). 읽는 코드는 같다.

## 클래스패스 리소스가 디스크 파일이 아닌 이유

```text
개발:  src/main/resources/prompts/p.txt
빌드:  build/resources/main/prompts/p.txt          ← IDE·gradle 실행은 여기서 읽음 (디스크 파일 O)
배포:  app.jar 안의 BOOT-INF/classes/prompts/p.txt ← java -jar 실행은 zip 안의 항목 (디스크 파일 X)
```

클래스패스 = "클래스와 리소스를 찾는 경로 목록"이다. 폴더일 수도, jar(zip) 파일일 수도 있다. jar 안의 항목은 **스트림으로 열 수는 있지만 `File`·`Path`로 가리킬 수는 없다.**

## 실행으로 확인한 결과

같은 코드를 ① 폴더 클래스패스(IDE 실행과 같음) ② jar 클래스패스(배포 실행과 같음)로 돌렸다. JDK 25, spring-core 7.0.9.

| 읽는 방법 | 폴더 | jar |
|---|---|---|
| `new ClassPathResource("prompts/p.txt").getContentAsString(UTF_8)` | ✅ | ✅ |
| `new ClassPathResource("prompts/p.txt").getFile()` | ✅ | ❌ `FileNotFoundException: … cannot be resolved to absolute file path because it does not reside in the file system: jar:file:…!/prompts/p.txt` |
| `X.class.getResourceAsStream("/prompts/p.txt")` (앞에 `/`) | ✅ | ✅ |
| `X.class.getResourceAsStream("prompts/p.txt")` (`/` 없음) | ⚠️ `null` | ⚠️ `null` |
| `X.class.getClassLoader().getResourceAsStream("prompts/p.txt")` | ✅ | ✅ |
| `Files.readString(Path.of("src/main/resources/prompts/p.txt"))` | ❌ 실행 폴더가 프로젝트 루트가 아니면 `NoSuchFileException` | ❌ 같음 |
| 없는 파일을 `ClassPathResource`로 | ❌ `FileNotFoundException: class path resource [prompts/nope.txt] cannot be opened because it does not exist` | 같음 |

## ⚠️ 함정

### 1. `getFile()`·`Path`로 읽는 코드는 IDE에서만 된다

개발 중에는 리소스가 진짜 파일이라 통과하고, jar로 배포한 뒤에 처음 터진다. 클래스패스 리소스는 **스트림 계열**(`getInputStream`, `getContentAsString`, `getResourceAsStream`)로만 읽는다.

참고로 `Resource.getContentAsString`의 Javadoc은 "파일 시스템 경로로 풀 수 없으면 `FileNotFoundException`"이라고 적혀 있어 jar에서 안 될 것처럼 읽힌다. 하지만 구현은 `getInputStream()`을 읽는 것이라 jar 안에서도 동작한다(위 표에서 실행 확인).

### 2. `getResourceAsStream`은 못 찾으면 예외가 아니라 `null`

- `Class.getResourceAsStream("prompts/p.txt")` — `/`가 없으면 **그 클래스의 패키지 기준 상대 경로**다. `demo.X`에서 부르면 `demo/prompts/p.txt`를 찾는다.
- `Class.getResourceAsStream("/prompts/p.txt")` — `/`가 있으면 클래스패스 루트 기준.
- `ClassLoader.getResourceAsStream("prompts/p.txt")` — 항상 루트 기준이라 `/`를 붙이지 않는다.
- 셋 다 못 찾으면 `null`을 돌려주고, 나중에 엉뚱한 곳에서 `NullPointerException`이 난다. `ClassPathResource`는 앞의 `/`를 떼고 루트 기준으로 찾으며, 없으면 경로가 적힌 `FileNotFoundException`을 던진다.

### 3. 상대 `Path`는 "실행한 폴더" 기준이다

`Path.of("src/main/resources/...")`는 JVM을 띄운 작업 디렉터리(cwd)에서 찾는다. IDE, `gradle bootRun`, `java -jar`, 컨테이너마다 cwd가 달라서 같은 코드가 어떤 실행 방식에서만 된다. 배포물 안의 파일을 소스 경로로 읽는 코드는 쓰지 않는다.

### 4. 문자셋은 항상 명시한다

한글 텍스트를 기본 문자셋으로 읽으면 환경에 따라 깨진다. `getContentAsString(Charset)`처럼 문자셋을 필수 인자로 받는 API가 실수를 막는다.

### 5. 스트림은 닫는다

`getInputStream()`·`Files.lines()`·`Files.newBufferedReader()`는 호출한 쪽이 닫아야 한다(try-with-resources). `Files.readString`·`getContentAsString`처럼 한 번에 읽는 API는 내부에서 닫는다.

### 6. 언제 읽을지도 정한다

요청마다 리소스를 읽으면 매번 I/O와 디코딩이 반복된다. 고정 텍스트는 **앱 시작 시 한 번** 읽어 문자열로 들고 있는다. 시작 시점에 읽으면 파일이 빠졌을 때 앱이 아예 뜨지 않아서(fail-fast), 첫 요청에서야 발견하는 일도 없다. 대신 jar 안 파일이라 내용을 바꾸려면 다시 빌드·배포해야 한다.

## 💡 판단 기준

**먼저 "이 파일은 앱과 함께 배포되나, 실행 환경에 따로 놓이나"를 묻는다.** 함께 배포되면 클래스패스 리소스로 두고 스트림 계열 API(`ClassPathResource.getContentAsString(UTF_8)`)로 시작 시 한 번 읽는다. 환경마다 바뀌어야 하면 디스크 경로를 설정값으로 받아 `Files`로 읽는다. 둘 다 필요해지면(평소엔 내장, 가끔 외부 파일로 교체) 그때 `@Value("${…:classpath:…}") Resource`로 위치를 설정으로 뺀다 — 처음부터 만들 필요는 없다.

## 참고

- [Spring: Resources](https://docs.spring.io/spring-framework/reference/core/resources.html) — `Resource`, `ClassPathResource`, `classpath:`/`file:` 접두사
- [Spring: Resource.getContentAsString (6.0.5+)](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/io/Resource.html#getContentAsString(java.nio.charset.Charset))
- [JDK: Class.getResourceAsStream](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/Class.html#getResourceAsStream(java.lang.String)) — 상대·절대 이름 규칙
- [JDK: Files.readString](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/nio/file/Files.html#readString(java.nio.file.Path,java.nio.charset.Charset))
- [JEP 400: UTF-8 by Default](https://openjdk.org/jeps/400)
- 컨테이너에서 `getResource("classpath:...")`·`classpath*:`로 여러 개 찾기는 [ApplicationContext](../spring/application-context.md)

학습 날짜: 2026-10-13 · 계기: 설정 클래스에서 프롬프트·스키마 파일을 `new ClassPathResource(...).getContentAsString(UTF_8)`로 읽는 코드를 보고 "파일 읽는 방법이 엄청 많던데 이건 뭔가" 질문. 폴더/jar 두 클래스패스로 직접 실행해 비교.

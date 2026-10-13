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

## java.io 스트림 쌓기 — `FileInputStream` 사용법

`java.io`(Java 1.0부터)는 **관(stream)을 겹쳐 끼우는** 방식이다. 바깥 관이 안쪽 관에 기능을 하나씩 더한다(데코레이터 패턴).

```text
FileInputStream       파일에서 바이트를 꺼냄      read() → 0~255, 끝이면 -1
  └ InputStreamReader 바이트 → 문자 (문자셋 필요)  read() → 문자 하나, 끝이면 -1
     └ BufferedReader 버퍼 + 줄 단위 읽기         readLine() → 한 줄, 끝이면 null
```

| 계열 | 단위 | 대표 클래스 | 쓰는 곳 |
|---|---|---|---|
| `InputStream` | 바이트 | `FileInputStream`, `ByteArrayInputStream`, `BufferedInputStream` | 이미지·엑셀·zip 같은 바이너리, 다른 라이브러리에 넘길 때 |
| `Reader` | 문자 | `InputStreamReader`, `FileReader`, `BufferedReader` | 텍스트 |

```java
// ① 버퍼 배열로 반복 — 가장 흔한 FileInputStream 모양
try (FileInputStream in = new FileInputStream(file)) {
    ByteArrayOutputStream collected = new ByteArrayOutputStream();
    byte[] buffer = new byte[8192];
    int n;                                   // 이번에 실제로 읽은 바이트 수
    while ((n = in.read(buffer)) != -1) {    // -1 = 파일 끝
        collected.write(buffer, 0, n);       // buffer 전체가 아니라 0~n만
    }
    String text = collected.toString(StandardCharsets.UTF_8);
}

// ② 한 번에 (Java 9+)
try (FileInputStream in = new FileInputStream(file)) {
    String text = new String(in.readAllBytes(), StandardCharsets.UTF_8);
}

// ③ 텍스트를 줄 단위로 — 관 세 겹
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(new FileInputStream(file), StandardCharsets.UTF_8))) {
    String line;
    while ((line = reader.readLine()) != null) { ... }   // 줄바꿈 문자는 빠진 채로 옴
}

// ④ 다른 곳으로 그대로 흘려보내기 (Java 9+) — 파일 다운로드 응답 등
try (InputStream in = new FileInputStream(file)) {
    in.transferTo(response.getOutputStream());
}
```

- 바깥 관(`BufferedReader`)을 닫으면 안쪽 관까지 차례로 닫힌다.
- `Files.newInputStream(path)`·`Files.newBufferedReader(path, UTF_8)`는 같은 관을 NIO(`Path`) 쪽에서 만들어 주는 방법이다. 없는 파일이면 `NoSuchFileException`처럼 원인이 드러나는 예외를 준다.

### 옛 코드의 close 모양 → try-with-resources

Java 7 전 코드는 이렇게 닫는다. 지금 보면 길지만 하는 일은 "끝나면 반드시 닫기" 하나다.

```java
FileInputStream fis = null;
try {
    fis = new FileInputStream(file);
    ...
} catch (IOException e) {
    ...
} finally {
    if (fis != null) {
        try { fis.close(); } catch (IOException ignored) { }
    }
}
```

```java
try (FileInputStream fis = new FileInputStream(file)) {   // Java 7+: 블록이 끝나면 자동 close
    ...
}
```

닫지 않으면 OS가 열린 파일마다 내주는 번호(파일 디스크립터)가 반납되지 않는다. 쌓이면 `Too many open files`로 새 파일·소켓을 못 연다.

### 실행으로 확인한 흔한 실수 두 개

파일 내용 `사주 프롬프트⏎두 번째 줄⏎` (UTF-8, 35바이트, 한글 한 글자 = 3바이트).

```java
// 실수 A — 읽은 개수 n을 무시하고 buffer 전체를 문자열로
while (in.read(buffer) != -1) {                       // buffer = 16바이트
    sb.append(new String(buffer, StandardCharsets.UTF_8));
}
// 결과: 사주 프롬프트⏎두 번째 ���⏎⏎두 번째 �
// 마지막 회차는 3바이트만 새로 읽었는데, 나머지 13바이트는 직전 회차 내용이 그대로 남아 있다
```

```java
// 실수 B — 바이트 덩어리마다 따로 문자로 바꿈
while ((n = in.read(buffer)) != -1) {                 // buffer = 4바이트
    sb.append(new String(buffer, 0, n, StandardCharsets.UTF_8));
}
// 결과: 사��� �����프트⏎두 번��� ���⏎
// 3바이트짜리 한글이 4바이트 경계에서 잘려, 반쪽 글자가 각각 깨진다

// 고침 — 문자 변환은 Reader에 맡긴다 (덩어리 경계를 넘어 이어 붙여 줌)
try (Reader reader = new InputStreamReader(new FileInputStream(file), StandardCharsets.UTF_8)) {
    char[] chars = new char[4];
    while ((n = reader.read(chars)) != -1) sb.append(chars, 0, n);
}
// 결과: 사주 프롬프트⏎두 번째 줄⏎
```

### 문자셋에 안 맞는 바이트가 있을 때 — 방법마다 반응이 다르다

UTF-8로 해석할 수 없는 바이트(`ab\xffcd`)가 든 파일을 각 방법으로 읽은 결과 (JDK 25):

| 방법 | 결과 | 성격 |
|---|---|---|
| `new String(bytes, UTF_8)` | `ab�cd` | 깨진 자리를 �로 바꾸고 조용히 진행 |
| `InputStreamReader` + `BufferedReader` | `ab�cd` | 같음 |
| `Files.readString` / `readAllLines` / `newBufferedReader` | `MalformedInputException` | 깨진 바이트가 있으면 예외 (엄격) |
| `Scanner.nextLine()` | `NoSuchElementException: No line found` | 읽기 오류를 "입력 끝"으로 취급해 원인이 가려짐 |

없는 파일도 다르다: `new FileInputStream("nope.txt")` → `FileNotFoundException: nope.txt (No such file or directory)`, `Files.newInputStream(Path.of("nope.txt"))` → `NoSuchFileException: nope.txt`(예외 타입만으로 원인 구분).

데이터가 깨졌다는 사실을 알아야 하는 곳(업로드 파일 검증, 정산 파일)은 엄격한 `Files` 쪽이, 일단 끝까지 보여 주는 게 나은 곳(로그 뷰어)은 관대한 쪽이 맞다.

실수 B는 버퍼가 8192바이트여도 생긴다 — 파일이 8192바이트를 넘으면 경계에 걸린 한 글자가 깨진다. 작은 테스트 파일에서는 안 보이다가 큰 파일에서만 나타난다. 텍스트는 ① 바이트를 전부 모은 뒤 한 번에 변환하거나 ② Reader로 읽는다.

## 그 밖의 방법

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
보강: 2026-10-13 — `FileInputStream` 사용법(관 쌓기·read 반복·readAllBytes·transferTo), 옛 finally close와 try-with-resources, 실수 두 개(읽은 개수 무시·바이트 덩어리별 변환으로 한글 깨짐)를 JDK 25로 실행해 결과 첨부.

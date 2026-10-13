# 임시 파일과 파일 쓰기 — createTempFile·deleteOnExit·FileOutputStream

> **한 줄 요약**: 서버에서 큰 파일(엑셀 내보내기 등)을 만들 때는 "임시 파일을 만들고 → 거기에 쓰고 → 다 쓴 파일을 읽어 보내고 → 지운다". `createTempFile`은 겹치지 않는 이름의 **빈 파일을 즉시** 만들고, `FileOutputStream`은 그 파일에 바이트를 쓰는 관이다. ⚠️ `deleteOnExit()`은 JVM이 정상 종료할 때만 지우고, 호출할 때마다 경로를 JVM 종료 때까지 메모리에 쌓는다 — 몇 주씩 도는 서버에서는 정리 수단이 아니다. 요청이 끝날 때 `finally`에서 직접 지운다.

## 언제 쓰나

- 응답으로 보내거나 저장소에 올릴 파일을 끝까지 다 만든 뒤 내보내야 할 때 (도중 실패 시 반쪽 파일이 나가지 않게)
- 라이브러리가 `File`/`OutputStream`을 요구할 때
- 메모리에 다 올리기엔 큰 결과물을 만들 때

## 쓰기 쪽 짝 — 읽기 API와 1:1로 대응

| 읽기 | 쓰기 |
|---|---|
| `FileInputStream` | `FileOutputStream` |
| `FileReader` / `InputStreamReader` | `FileWriter` / `OutputStreamWriter` |
| `BufferedReader` | `BufferedWriter` |
| `Files.readString` / `readAllBytes` | `Files.writeString` / `Files.write` |
| `Files.newInputStream` / `newBufferedReader` | `Files.newOutputStream` / `newBufferedWriter` |
| `in.transferTo(out)` — 읽은 것을 바로 쓰기 | |

읽기 쪽은 [파일 읽기](./reading-files-and-classpath-resources.md).

## 사용 예시

```java
// 1) 임시 파일 만들기 — 이 시점에 0바이트 파일이 실제로 생긴다
File file = File.createTempFile("report", ".xlsx");
// → /tmp(또는 OS 임시 폴더)/report14140716367314861851.xlsx

// 2) 쓰기 — try-with-resources로 쓰는 동안만 열어 둔다
try (OutputStream out = new FileOutputStream(file)) {
    workbook.write(out);
}

// 3) 다 쓴 파일을 읽어 내보내기
try (InputStream in = new FileInputStream(file)) {
    in.transferTo(response.getOutputStream());
} finally {
    Files.deleteIfExists(file.toPath());   // 4) 요청이 끝나면 직접 지운다
}
```

NIO 쪽으로 쓰면:

```java
Path path = Files.createTempFile("report", ".xlsx");
try {
    try (OutputStream out = Files.newOutputStream(path)) { workbook.write(out); }
    try (InputStream in = Files.newInputStream(path)) { in.transferTo(response.getOutputStream()); }
} finally {
    Files.deleteIfExists(path);
}
```

## 실행으로 확인한 동작 (JDK 25, macOS, umask 022)

| 확인 | 결과 |
|---|---|
| `File.createTempFile("report", ".xlsx")` | `report14140716367314861851.xlsx`, 존재=true, 크기=0 |
| 같은 prefix로 한 번 더 | `report16824675780354441104.xlsx` — 숫자가 달라 겹치지 않음 |
| suffix `null` | `abc13275545746032164237.tmp` — 기본 `.tmp` |
| prefix `"ab"` (2글자) | `IllegalArgumentException: Prefix string "ab" too short: length must be at least 3` |
| prefix 한글 `"매출보고서"` | `매출보고서10020836090161568122.xlsx` — 가능 |
| 위치 | `java.io.tmpdir` 시스템 속성 (macOS는 `/var/folders/…/T/`, 리눅스는 보통 `/tmp`) |
| `new FileOutputStream(f)`로 "hello" 쓴 뒤 다시 열어 "hi" | 크기 5 → 2. **기본은 덮어쓰기**(열 때 내용을 비움) |
| `new FileOutputStream(f, true)`로 "!!" | 2 → 4. 두 번째 인자 `true` = 이어 쓰기 |
| `File.createTempFile` 권한 | `rw-r--r--` — 같은 서버의 다른 사용자도 읽기 가능 |
| `Files.createTempFile` 권한 | `rw-------` — 소유자만 |
| `deleteOnExit()` 1000번 호출 후 파일을 직접 지워도 | JVM 내부 등록 목록 크기 1000 그대로 |

## ⚠️ 함정

### 1. `deleteOnExit()`은 서버의 정리 수단이 아니다

- **정상 종료 때만** 지운다. `kill -9`, OOM으로 죽으면 남는다.
- 서버는 몇 주씩 안 꺼지므로, 그동안 임시 파일이 디스크에 계속 쌓인다.
- 호출할 때마다 경로를 JVM 내부 목록에 추가하고, 파일을 직접 지워도 목록에서 빠지지 않는다(위 표). 요청마다 호출하면 이 목록이 재시작 전까지 계속 커진다.
- 그래서 요청 처리가 끝나는 `finally`에서 `Files.deleteIfExists`로 직접 지운다. `deleteOnExit`은 CLI 도구처럼 금방 끝나는 프로그램용이다.

### 2. 생성자에서 스트림을 열어 필드에 들고 있으면 새기 쉽다

```java
Export(String name) throws IOException {
    this.file = File.createTempFile(name, ".xlsx");
    this.out = new FileOutputStream(file);   // 여기서 열고
    this.style = createStyle();              // 이 줄이 예외를 던지면 객체가 안 만들어져 out을 닫을 사람이 없다
}
```

스트림은 **실제로 쓰는 순간에** try-with-resources로 열고 닫는 편이 안전하다. 꼭 필드로 들고 있어야 하면 클래스가 `AutoCloseable`을 구현하고, 생성자 안에서 예외가 나면 이미 연 것을 닫아야 한다.

### 3. `File.createTempFile`은 다른 사용자도 읽을 수 있게 만든다

umask가 022인 리눅스·macOS에서 `rw-r--r--`로 생긴다. 고객 개인정보가 든 엑셀이라면 같은 서버의 다른 계정이 읽을 수 있다. `Files.createTempFile`은 기본이 소유자 전용(`rw-------`)이다.

### 4. prefix는 3글자 이상

파일 이름을 그대로 prefix로 넘기는 코드는 짧은 이름(`"매출"` 2글자)에서 `IllegalArgumentException`이 난다.

### 5. 라이브러리도 자기 임시 파일을 만든다 — 예: POI `SXSSFWorkbook`

대용량 엑셀 쓰기용 `SXSSFWorkbook`은 메모리에 최근 행만 두고(기본 100행, `DEFAULT_WINDOW_SIZE`), 창 밖으로 밀려난 행은 **자체 임시 파일에** 써 버린다. 그래서:
- 밀려난 행은 `getRow()`로 다시 접근할 수 없다(쓰기 전용에 가깝다).
- 공식 문서: SXSSF가 만든 임시 파일은 **`dispose()`를 호출해 직접 정리해야 한다**. `close()`의 Javadoc에는 임시 파일 삭제가 적혀 있지 않다.
- 즉 정리할 것이 둘이다: 내가 만든 임시 파일(`deleteIfExists`) + 라이브러리가 만든 임시 파일(`dispose()`).

```java
SXSSFWorkbook workbook = new SXSSFWorkbook();
Path path = Files.createTempFile("export", ".xlsx");
try {
    // ... 행 추가 ...
    try (OutputStream out = Files.newOutputStream(path)) { workbook.write(out); }
    // ... path를 읽어 내보내기 ...
} finally {
    workbook.dispose();          // POI 임시 파일
    workbook.close();
    Files.deleteIfExists(path);  // 내 임시 파일
}
```

## 임시 파일 vs 응답에 바로 쓰기

| | 임시 파일에 쓴 뒤 보내기 | `workbook.write(response.getOutputStream())` |
|---|---|---|
| 디스크 | 씀 (정리 필요) | 안 씀 |
| 도중 실패 | 아직 아무것도 안 보냄 → 깔끔한 오류 응답 | 이미 일부 전송됨 → 응답 헤더는 성공, 파일은 깨짐 |
| Content-Length | 파일 크기로 알 수 있음 | 모름 |
| 저장소 업로드·재전송 | 쉬움 | 다시 만들어야 함 |

## 💡 판단 기준

**임시 파일은 "누가, 언제 지우나"를 만들 때 같이 정한다.** 요청 하나의 수명이면 그 요청의 `finally`에서 지운다. JVM 종료(`deleteOnExit`)에 맡기는 순간 서버에서는 "안 지운다"와 같다. 라이브러리가 내부적으로 임시 파일을 쓰는지(`dispose` 같은 메서드가 있는지)도 함께 확인한다.

## 참고

- [JDK: File.createTempFile](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/io/File.html#createTempFile(java.lang.String,java.lang.String)) · [File.deleteOnExit](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/io/File.html#deleteOnExit())
- [JDK: Files.createTempFile](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/nio/file/Files.html#createTempFile(java.lang.String,java.lang.String,java.nio.file.attribute.FileAttribute...)) — `File.createTempFile`보다 제한된 권한일 수 있다고 명시
- [Apache POI: SXSSF (Streaming Usermodel API)](https://poi.apache.org/components/spreadsheet/how-to.html#sxssf) — 창 크기 기본 100, 임시 파일은 `dispose`로 직접 정리
- [Apache POI: SXSSFWorkbook Javadoc](https://poi.apache.org/apidocs/dev/org/apache/poi/xssf/streaming/SXSSFWorkbook.html)

학습 날짜: 2026-10-13 · 계기: 엑셀 내보내기 코드에서 `File.createTempFile`·`deleteOnExit`·`new FileOutputStream(file)`을 생성자에서 여는 모양을 보고 "파일을 이렇게 읽는 것인가" 질문 — 실제로는 쓰기 준비였고, 임시 파일 정리 책임을 함께 정리. JDK 25로 이름 규칙·권한·덮어쓰기·deleteOnExit 목록 증가를 실행 확인.

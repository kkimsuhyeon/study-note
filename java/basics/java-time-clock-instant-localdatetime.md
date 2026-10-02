# Clock·Instant·LocalDateTime — 시계와 시각 값을 구분하기

> **한 줄 요약:** `Clock`은 현재 시각을 공급하는 시계, `Instant`는 시간선 위의 한 순간, `LocalDateTime`은 시간대 정보 없이 날짜와 시각을 담은 값이다.

## 1. 언제 쓰나

| 타입 | 답하는 질문 | 사용 예 |
| --- | --- | --- |
| `Clock` | 지금을 어디서 읽을까? | 운영에서는 시스템 시계, 테스트에서는 고정 시계 |
| `Instant` | 정확히 언제 일어난 사건인가? | 요청 접수·만료 시점 |
| `LocalDateTime` | 달력과 시계에 적힌 날짜·시각은? | 사용자가 입력한 방문 희망 일시. 실제 순간으로 해석하려면 시간대가 필요 |
| `LocalDate` | 날짜만 무엇인가? | 생년월일·한국 기준 오늘 |

`ZoneId`는 `Asia/Seoul` 같은 지역의 시간대 규칙을 나타낸다. `Instant`에 이를 적용하면 지역의 날짜·시각·시간대를 함께 가진 `ZonedDateTime`을 얻는다. 거기서 `toLocalDateTime()`을 호출하면 시간대 정보가 빠진 날짜·시각만 남는다.

### LocalDateTime만 써 왔다면 — 잘못인지보다 시간대 약속부터 확인

`LocalDateTime.of(2030, 10, 5, 14, 0)`은 "2030년 10월 5일 14시"라는 달력 일시를 표현한다. 방문 희망 일시처럼 사람이 고른 일시를 담을 수 있다. 실제 예약 시점으로 확정할 때는 매장의 시간대 등 업무 규칙을 적용한다. 이미 예약을 확정했다면 순간과 지역 정보를 어떤 형태로 보존할지도 별도로 정한다.

생성일·만료일을 `LocalDateTime`으로 다루는 것도, 저장·조회·비교하는 모든 곳이 같은 시간대 약속을 지키면 가능하다. 문제는 그 약속이 값 자체에는 없다는 것이다. 기존 데이터가 한국 시각인데 새 서버의 `LocalDateTime.now()`가 UTC 날짜·시각을 만들면 기준이 섞인다. UTC는 세계 시각을 맞추는 기준이며, 이 예시에서 한국은 UTC보다 9시간 빠르다.

예를 들어 한국 13시에 "10분 뒤 만료"를 `13:10`으로 저장했다. 11분 뒤 UTC 서버가 자기 지역 시각 `04:11`과 이를 비교하면 `04:11 < 13:10`이라 아직 유효하다고 잘못 판단할 수 있다. 타입의 비교 기능이 틀린 것이 아니라, 서로 다른 기준의 숫자를 넣은 것이 원인이다.

따라서 기존 필드를 일괄 교체하기 전에 **그 값이 어느 시간대 기준으로 저장돼 왔는지** 먼저 확인한다. 시간대를 붙이는 일은 단순한 표시 변경이 아니라 데이터의 의미를 해석하는 일이다.

### Instant는 사건이 발생한 순간을 보존한다

`Instant`는 공통 기준인 `1970-01-01T00:00:00Z`로부터의 초와 나노초로 순간을 표현한다. 지역 시간대를 저장하지 않아도 기준이 정해져 있어 순간을 식별한다. 생성·수정·만료·전송 시각처럼 "언제 일어났는가"를 기록할 때 의미가 잘 맞는다.

```java
Instant acceptedAt = Instant.parse("2026-09-29T04:00:00Z");
Instant expiresAt = acceptedAt.plusSeconds(600);
Instant checkedAt = Instant.parse("2026-09-29T04:11:00Z");
boolean expired = !checkedAt.isBefore(expiresAt); // true, 만료 순간부터 무효라는 정책
```

한국 시각으로는 13:00 접수·13:10 만료·13:11 확인이지만 비교 대상은 같은 시간선 위의 순간이다. 화면 표시는 사용자나 서비스의 시간대를 적용해 별도로 만든다. `Instant`가 서버들의 물리적 시계를 자동 동기화해 주는 것은 아니다.

## 2. 하나의 순간을 시계에서 읽고 지역 시각으로 표시하기

아래는 실행 가능한 형태로 작성한 학습 예시다. 주석의 값은 공식 API 규칙에 따른 예상값이며 이 노트를 작성하면서 빌드·실행하지는 않았다.

```java
import java.time.Clock;
import java.time.Instant;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.time.temporal.ChronoUnit;

public class TimeExample {
    public static void main(String[] args) {
        ZoneId seoul = ZoneId.of("Asia/Seoul");
        Clock clock = Clock.fixed(
                Instant.parse("2026-09-29T04:37:42.123Z"), seoul);

        Instant now = clock.instant();
        LocalDateTime local = LocalDateTime.ofInstant(now, seoul);
        Instant hour = now.truncatedTo(ChronoUnit.HOURS);

        System.out.println(now);   // 2026-09-29T04:37:42.123Z
        System.out.println(local); // 2026-09-29T13:37:42.123
        System.out.println(hour);  // 2026-09-29T04:00:00Z
    }
}
```

`Z`는 UTC 오프셋 0을 나타낸다. 위 예시에서 UTC 04:37과 한국 13:37은 **같은 순간의 다른 지역 표현**이다. `Instant` 자체가 `Asia/Seoul` 같은 지역 시간대를 저장하는 것은 아니다.

`LocalDateTime`에 `2026-09-29T13:37`만 있으면 한국 13:37인지 다른 지역의 13:37인지 알 수 없다. `Local`은 "자동으로 한국 시간"이라는 뜻이 아니다. 매개변수 없는 `LocalDateTime.now()`는 시스템 기본 시간대를 사용하지만, 만들어진 값에는 그 시간대가 남지 않는다.

### Clock을 따로 받는 이유

운영에서는 `Clock.system(seoul)`로 실제 시각을 읽고, 테스트에서는 `Clock.fixed(...)`로 항상 같은 시각을 돌려주는 시계로 교체할 수 있다. 예를 들어 만료 검사에서 "지금은 만료 1초 전"이라는 조건을 실제로 기다리지 않고 구성한다.

`Clock`에 지역 시간대가 있어도 `clock.instant()`는 순간을 반환한다. `LocalDateTime.now(clock)`이나 `LocalDate.now(clock)`처럼 지역 달력 값을 만들 때 시계의 시간대가 사용된다. `Clock`을 받는 의존성 설계는 [포트와 어댑터](../design/ports-and-adapters.md)도 참고한다.

### Clock은 LocalDateTime과도 같이 쓴다

**어떤 값을 사용할지**와 **현재 시각을 어디서 얻을지**는 별도 선택이다.

```java
// 위 TimeExample의 고정 시계를 그대로 사용한다고 가정
Instant point = clock.instant();
LocalDateTime localNow = LocalDateTime.now(clock);
```

두 줄 모두 같은 시계를 사용한다. 첫째는 순간, 둘째는 그 시계에 설정된 지역의 날짜·시각을 얻는다. `Instant.now()`와 `LocalDateTime.now()`도 내부적으로 시스템 시계를 읽지만, 코드에 `Clock`을 받도록 만들면 그 시간 공급원을 교체할 수 있다.

예를 들어 만료 여부를 검사하는 코드가 `clock.instant()`를 읽도록 작성되어 있다면, 테스트마다 `Clock.fixed(...)`를 만료 직전·정각·직후로 바꿔 경계를 확인한다. `Clock`을 쓰기 위해 저장 필드를 전부 `Instant`로 바꿀 필요는 없으며, `LocalDateTime.now(clock)`도 고정 시계를 이용할 수 있다.

```text
Clock: 현재 시각을 공급하는 의존성
  ├─ clock.instant()          → Instant: 한 순간
  └─ LocalDateTime.now(clock)  → LocalDateTime: 지역 날짜·시각, 시간대는 값에 남지 않음
```

## 3. truncatedTo(HOURS) — 같은 시간 구간에 묶기

`ChronoUnit.HOURS`는 시간 단위다. `truncatedTo(HOURS)`는 UTC 기준으로 그보다 작은 분·초·소수 초를 0으로 만든 값을 반환한다. 반올림하거나 한 시간을 빼는 연산이 아니다.

| 요청 시점(UTC) | 시간 구간의 시작값 |
| --- | --- |
| `2026-09-29T04:01:00Z` | `2026-09-29T04:00:00Z` |
| `2026-09-29T04:59:59Z` | `2026-09-29T04:00:00Z` |
| `2026-09-29T05:00:00Z` | `2026-09-29T05:00:00Z` |

요청 횟수 집계에서 `(대상, 구간 시작값)`을 키로 삼으면 같은 시간 구간의 요청들이 같은 카운터를 사용한다. 이것은 정시마다 구간이 바뀌는 **고정 구간**이며, "현재부터 과거 60분"을 계속 계산하는 방식과 다르다.

### 날짜 단위로 자르면 UTC 자정이다

```java
// now와 seoul은 위 예제와 같은 값
Instant utcDayStart = now.truncatedTo(ChronoUnit.DAYS);
// 2026-09-29T00:00:00Z → 한국에서는 9월 29일 09:00

Instant seoulDayStart = now.atZone(seoul)
        .toLocalDate()
        .atStartOfDay(seoul)
        .toInstant();
// 2026-09-28T15:00:00Z → 한국에서는 9월 29일 00:00
```

UTC 날짜별 집계라면 첫 번째가 맞고, 한국 달력 날짜별 집계라면 두 번째가 맞다. 시계의 시간대 설정만 바꿔도 `Instant.truncatedTo(DAYS)`가 한국 자정으로 바뀌지는 않는다.

## 4. ⚠️ 자주 섞이는 개념

- **시계와 시각:** `clock.instant()`를 다시 호출해야 새 시각을 읽는다. 이미 받은 `now`가 시간이 흐른다고 자동으로 변하지 않는다.
- **원본과 계산 결과:** `Instant`·`LocalDateTime`은 불변 값이다. `now.truncatedTo(...)`가 기존 `now`를 수정하지 않으므로 반환값을 사용해야 한다.
- **지역 날짜와 순간:** 같은 순간에도 지역에 따라 날짜가 다를 수 있다. "오늘까지 입력 가능" 같은 검증은 어느 지역의 오늘인지 정한다.
- **지역 일시를 순간으로 변환:** `local.atZone(zone).toInstant()`처럼 시간대를 적용한다. 일광절약시간이 있는 지역에는 존재하지 않거나 두 번 나타나는 지역 시각이 있으므로 예약 정책에서 해석 규칙을 정해야 한다.

## 5. 💡 판단 기준

요청 만료처럼 **사건의 선후와 경과를 비교하면 `Instant`**, 방문 희망 일시처럼 **사람이 입력한 달력 일시를 다루면 `LocalDateTime`과 시간대 정책**, "지금"을 테스트에서 바꿔야 하면 **`Clock`을 주입**한다. 일별 집계에서는 타입 선택과 별개로 **어느 시간대의 자정이 경계인지** 먼저 정한다.

| 구체적인 요구 | 선택과 이유 |
| --- | --- |
| 주문이 접수된 순간·10분 뒤 만료 | `Instant`: 지역 표시와 분리해 순간끼리 비교 |
| 사용자가 입력한 방문 희망 일시 | `LocalDateTime`: 달력 입력을 보존하고 매장 시간대로 해석 |
| 생년월일 | `LocalDate`: 시각이나 UTC 순간으로 바꿀 필요가 없음 |
| 시간이 바뀌어도 재현되는 만료 테스트 | `Clock`: 현재 시각 공급원을 고정 시계로 교체 |
| 기존 LocalDateTime 기반 생성일 컬럼 | 시간대·DB 매핑·API 계약을 먼저 확인. 타입 이름만 교체하지 않음 |

## 참고·학습 기록

- [Java 25 — Instant](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/time/Instant.html)
- [Java 25 — Clock](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/time/Clock.html)
- [Java 25 — LocalDateTime](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/time/LocalDateTime.html)
- [Java 25 — LocalDate](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/time/LocalDate.html)
- [Java 25 — java.time 타입 선택 가이드](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/time/package-summary.html)
- 학습일: 2026-09-29. 계기: 시간별 요청 제한에서 `now.truncatedTo(HOURS)`를 읽다가 시계·순간·지역 일시가 혼동됨. 예시는 일반화했으며 공식 문서로 동작을 확인했다. 예제 실행과 프로젝트 테스트는 하지 않았다.
- 보강: 2026-09-29. 계기: LocalDateTime만 사용하던 관점에서 사건 시각·달력 입력·시간 공급원의 목적을 구분하고, 서로 다른 시간대의 만료 비교 예시를 추가함.

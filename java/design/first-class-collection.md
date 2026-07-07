# 일급 컬렉션 (First-Class Collection) — 컬렉션 하나만 감싼 클래스

> **한 줄 요약**: `List`/`Map` 등 컬렉션을 **다른 필드 없이 딱 하나만** 감싼 클래스. 목적은 "이 컬렉션으로 할 수 있는 도메인 질의/검증"을 그 타입 안에 모으는 것(응집). 여러 곳이 같은 리스트를 `stream().filter(...)`로 반복 조작하던 걸 **컬렉션 객체의 메서드 한 곳**으로 모으고, 생성 시 방어적 복사 + 불변으로 만들어 공유해도 안전하게 한다. 마틴 파울러 "객체지향 생활체조" 규칙 중 하나.

관련 노트: [도메인 검증 위치](./domain-validation.md) · [애그리거트 소유권](./aggregate-ownership.md)

---

## 1. 언제 쓰나 — "리스트를 여러 곳이 같은 방식으로 주무를 때"

`List<T>`를 날것으로 여기저기 넘기면 이런 문제가 생긴다.

- **로직 중복**: 이 리스트로 "특정 조건 필터", "합계", "검증"을 하는 코드가 리스트를 쓰는 곳마다 복붙된다.
- **변경 위험**: 아무나 `list.add()`/`list.remove()` 할 수 있어, 한 곳이 바꾸면 그 리스트를 공유하는 다른 곳이 깨진다.
- **의미 상실**: 그냥 `List<Integer>`면 이게 로또 번호인지 점수인지 이름에서 안 드러난다. 개발자마다 변수명도 제각각(`nums`, `lottoList`...)이라 검색·소통이 어렵다.

→ **컬렉션을 클래스로 감싸(Wrapping) 이름을 주고, 그 컬렉션에 대한 로직을 그 안에 모으면** 위 문제가 한 번에 풀린다. 그게 일급 컬렉션.

---

## 2. 정의 — "컬렉션 하나 + 다른 필드 없음"

Collection을 Wrapping하면서 **그 외 다른 멤버 변수가 없는** 클래스가 일급 컬렉션이다. **필드가 컬렉션 딱 하나여야** 한다. 다른 필드(카운터, 플래그 등)가 같이 있으면 일급 컬렉션이 아니다(→ §5 대비).

```java
// 일급 컬렉션 O — 필드가 List 하나뿐
public final class Lotto {
    private final List<LottoNumber> numbers;   // ← 이거 하나만
}

// 일급 컬렉션 X — matchCount라는 다른 필드가 섞임
public class Lotto {
    private final List<LottoNumber> numbers;
    private int matchCount;                     // ← 이게 있으면 위배
}
```

---

## 3. 실제 예시 — `AttendanceTraceSegments` (근태마감, client_api)

하루치 근태를 판정하려면 4개 소스(근로제 기준 / 교대 패턴 / 전사휴일 / 근태신청)의 세그먼트가 필요했다. 이 `List<AttendanceTraceSegment>`를 날것으로 넘기지 않고 일급 컬렉션으로 감쌌다.

```java
public final class AttendanceTraceSegments {

    private final List<AttendanceTraceSegment> values;   // ← 필드 이거 하나뿐 = 일급 컬렉션

    private AttendanceTraceSegments(List<AttendanceTraceSegment> values) {
        this.values = Collections.unmodifiableList(new ArrayList<>(values));  // 방어적 복사 + 불변 (§4-1)
    }

    // 소스별 리스트들을 하나로 합치는 정적 팩토리 (4로더 결합)
    @SafeVarargs
    public static AttendanceTraceSegments combine(List<AttendanceTraceSegment>... sources) { ... }

    // ── 이 컬렉션에 대한 "도메인 질의"를 타입 안에 모음 ──
    public AttendanceTraceSegment getConfigAppl() { ... }   // 설정 가진 신청(출장/외근/간주) 찾기
    public AttendanceTraceSegment getPattern()     { ... }  // 패턴 세그먼트 찾기
    public AttendanceTraceSegment getByTrcDiv(TraceDiv d){...} // 특정 종류 조회
    public AttendanceTraceSegments excludeExtraWork() { ... }  // 연장/야간/휴일 제외한 정규 기준
}
```

**핵심 효과 — 두 경로가 같은 컬렉션·같은 질의를 공유**
근태조정 적용(`WorkAttendanceCalculator`)과 월별 마감(`DailyClosingBuilder`)이 둘 다 `AttendanceTraceSegments.combine(...)`으로 입력을 만들고 `excludeExtraWork()` 같은 질의를 그대로 쓴다. 만약 리스트를 날것으로 넘겼다면 "연장/야간/휴일을 빼는 필터"를 두 경로가 각자 `stream().filter(...)`로 짰을 것이고, 한쪽만 고치면 두 경로 결과가 어긋난다. 컬렉션에 로직을 모으니 **한 곳만 고치면 두 경로가 같이 반영**된다.

---

## 4. 장점 4가지 (통설)

| 장점 | 내용 |
|---|---|
| **불변성 보장** | 값 변경 메서드(add/remove/set)를 안 만들면 밖에서 못 바꿈. 공유해도 안전 |
| **상태 + 로직 응집** | 컬렉션과 "그 컬렉션으로 하는 계산/검증"이 한 곳에 → 중복 제거, 변경 시 한 곳만 수정 |
| **검증 강제** | 생성자에 검증을 넣으면 "이 컬렉션을 쓰려면 반드시 검증을 통과"가 강제됨 (히스토리 모르는 사람도 실수 못 함) |
| **이름 부여** | `List<Pay>` → `NaverPays`처럼 이름이 생겨 의미가 드러나고, 팀 용어·검색이 통일됨 |

`List`에 담고 Service/Util에서 로직을 돌리면, 컬렉션과 계산 로직이 서로 관계가 있는데도 그 관계가 코드로 표현되지 않는다. 일급 컬렉션은 그 관계를 타입으로 표현하는 것.

### 4-1. ⚠️ 불변의 함정 — `final`만으론 부족, 방어적 복사가 핵심

가장 자주 틀리는 부분. **`final`은 재할당만 막지 컬렉션 내용 변경은 못 막는다.**

```java
private final List<T> values;   // final이어도 values.add(...)는 됨! (재할당만 금지)
```

두 군데를 다 막아야 진짜 불변이 된다.

```java
// (1) 생성자 — 받은 리스트를 그대로 참조하면 밖에서 원본을 바꿀 때 내부도 바뀜
this.values = new ArrayList<>(values);              // 방어적 복사 (다른 주소)
// 또는 unmodifiable까지: Collections.unmodifiableList(new ArrayList<>(values))

// (2) getter — 내부 리스트를 그대로 return하면 밖에서 add/remove 가능
public List<T> getValues() {
    return Collections.unmodifiableList(values);    // 또는 new ArrayList<>(values)
}
```

`AttendanceTraceSegments`는 생성자에서 `Collections.unmodifiableList(new ArrayList<>(values))`로 (1)을 처리했다 — 복사 + 불변 래핑 둘 다.

> ⚠️ 생성자 방어적 복사와 getter 방어를 **둘 다** 해야 한다. 하나만 하면 나머지 구멍으로 원본이 샌다.

---

## 5. ⭐ 일급 컬렉션이 "아닌 것"과의 구분 — 값 객체 / Tell Don't Ask

필드가 컬렉션 하나가 아니면 일급 컬렉션이 아니다. 하지만 **여전히 좋은 설계**일 수 있다 — 다른 이름의 패턴일 뿐. 근태 코드의 `Schedule`이 그 예.

```java
public class Schedule {
    private final LocalDate startDate;                    // 컬렉션 아님
    private final Map<Integer, ScheduleEntry> byOrdinal;  // 컬렉션
    private final boolean skipHoliday;                    // 컬렉션 아님
    private final Set<LocalDate> skipHolidayDates;        // 컬렉션
    // → 컬렉션 외 필드(startDate, skipHoliday)가 있으므로 일급 컬렉션 X
}
```

`Schedule`은 일급 컬렉션이 아니라 **값 객체(Value Object) / 도메인 모델**이고, 두 가지 좋은 패턴을 품고 있다.

**(a) Tell, Don't Ask** — 데이터를 꺼내 밖에서 계산하지 말고, 가진 객체에게 시킨다.
```java
// ❌ Ask: 밖에서 필드 꺼내 계산
LocalDate st = schedule.getStartDate();
long offset = ChronoUnit.DAYS.between(st, workDate);
// ... 호출부가 스케줄 계산 규칙을 다 앎

// ✅ Tell: "이 날짜의 entry 줘" 라고 시키면 스스로 계산
ScheduleEntry entry = schedule.resolveEntry(workDate);   // 일차 계산·휴일 이월을 자기가 처리
```
`resolveEntry`가 "workDate가 몇 번째 일차인지, 휴일이면 얼마나 이월되는지"를 **자기 안에서** 계산한다. 계산 규칙이 `Schedule` 안에 응집되어, 호출부는 규칙을 몰라도 된다.

**(b) 널 객체 패턴 (Null Object Pattern)** — `null` 대신 "빈 객체"를 반환해 `if (x != null)` 분기를 없앤다.
```java
public static Schedule empty() { return EMPTY; }   // 스케줄근로제 아니면 empty()

// 호출부: null 체크 없이 그냥 호출 — empty면 resolveEntry가 알아서 null 반환
schedule.resolveEntry(workDate);   // if (schedule != null) 불필요
```

### 일급 컬렉션 vs 값 객체 — 한눈에

| | 일급 컬렉션 | 값 객체 (`Schedule`) |
|---|---|---|
| 필드 | **컬렉션 1개만** | 컬렉션 + 다른 값들 |
| 예 | `AttendanceTraceSegments` | `Schedule` |
| 핵심 | 컬렉션 질의를 응집 + 불변 공유 | 판정 책임을 자기가 가짐(Tell Don't Ask) + 널 객체 |
| 공통점 | **관련된 걸 객체로 모아 응집↑ + 불변** | (좌동) |

> ⚠️ **면접 함정 방어**: `Schedule`을 "일급 컬렉션"이라 부르면 틀린다. "필드가 컬렉션 하나가 아니라서 엄밀히는 값 객체에 가깝고, 판정 책임을 객체 안에 둔 Tell Don't Ask + 널 객체 패턴"이라고 답하면 개념을 정확히 안다는 인상을 준다.

---

## 6. ⚠️ 함정·반대 의견

- **불변은 일급 컬렉션의 "자동 효과"가 아니다.** 어떤 이는 "일급 컬렉션을 통해 불변의 장점을 가질 수 있다"보다 "일급 컬렉션을 불변으로 만들어라"가 맞다고 지적한다. 즉 불변은 방어적 복사·getter 방어를 **직접 해야** 얻어지는 것이지, 감싸기만 하면 생기는 게 아니다. `List`는 일급 컬렉션 없이도 불변으로 만들 수 있다.
- **YAGNI**: 필드 하나짜리 단순 리스트를 감싸면 클래스만 늘고 `.getValues()` 변환 노이즈가 생긴다. "이 컬렉션에 붙는 도메인 로직·검증·공유가 실제로 있을 때" 도입한다(도메인 검증의 VO 도입 판단과 같은 결 → [도메인 검증 위치 §2-2](./domain-validation.md)).
- **원시값 포장(Wrapper Class)과의 관계**: 일급 컬렉션은 "컬렉션판 원시값 포장"이다. 값 하나를 `Position` 같은 타입으로 감싸는 게 원시값 포장, 컬렉션을 감싸는 게 일급 컬렉션. 목적(응집·검증·타입 안전)은 같다.

---

## 7. 💡 판단 기준

| 질문 | 답 |
|---|---|
| 필드가 컬렉션 하나뿐인가? | 예 → 일급 컬렉션 / 아니오(다른 값도 있음) → 값 객체·도메인 모델 |
| 언제 감싸나? | 리스트에 붙는 도메인 질의·검증이 **여러 곳에서 반복**되거나, 여러 경로가 **공유**할 때 |
| 불변은 어떻게? | `final`론 부족 → 생성자 방어적 복사 + getter 방어 **둘 다** |
| 단순 리스트인데 감쌀까? | 로직·검증·공유가 없으면 YAGNI — 감싸지 마라 |

> 한 줄: **"이 리스트로 뭘 하는가"가 여러 곳에 흩어지면 컬렉션을 클래스로 감싸 그 로직을 안으로 모은다. 감쌌으면 방어적 복사로 불변까지 챙긴다. 필드에 컬렉션 말고 다른 게 섞이면 그건 일급 컬렉션이 아니라 값 객체 — 이름을 정확히.**

---

## 8. 참고
- 향로(jojoldu) - 일급 컬렉션의 소개와 써야 할 이유 (국내 대표 정리글, 4가지 장점 출처)
- 마틴 파울러, 『소트웍스 앤솔러지』 "객체지향 생활체조" 규칙 (일급 컬렉션 사용)
- 관련 노트: [도메인 검증 위치](./domain-validation.md)(VO·YAGNI 결) · [애그리거트 소유권](./aggregate-ownership.md)

---

**학습 날짜**: 2026-07-03
**계기**: client_api 근태마감(9292) 코드를 포트폴리오용으로 정리하다가 — 근태조정 적용과 월별 마감 두 경로가 `List<AttendanceTraceSegment>`를 각자 필터링하던 걸 `AttendanceTraceSegments` 일급 컬렉션으로 모아 공통화한 작업을 설명하며 정리. 곁들여 `Schedule`은 필드가 컬렉션 하나가 아니라 일급 컬렉션이 아니고 값 객체(Tell Don't Ask + 널 객체)라는 구분, `final`만으론 불변이 안 되고 방어적 복사가 필요하다는 함정을 함께 정리.

# record 컴팩트 생성자 — 고치는 건 매개변수, 필드는 맨 끝에 대입된다

> **한 줄 요약:** record의 컴팩트 생성자(`public Point { … }`) 안에서 다루는 이름은 **매개변수**다. 필드는 본문이 끝난 뒤 한꺼번에 대입되므로, 본문 안에서 접근자(`x()`)나 인스턴스 메서드로 필드를 읽으면 아직 기본값(`null`·0·false)이다.

## 1. 언제 쓰나

- record 값을 만들 때 **검증**(범위·형식)하거나 **정규화**(앞뒤 공백 제거, 방어적 복사)할 때
- 생성자 안에서 검증 헬퍼를 부르는데, 그 헬퍼가 여러 필드를 써야 할 때 — 어떻게 넘겨야 하는지가 이 노트의 핵심

## 2. 사용 예시

```java
public record Member(String name, List<String> tags) {
    public Member {                          // 괄호 없는 "컴팩트" 정식 생성자
        name = name.strip();                 // 매개변수 name을 바꾼다 (this.name 아님)
        if (name.isEmpty()) throw new IllegalArgumentException();
        tags = List.copyOf(tags);            // 방어적 복사도 매개변수에
    }                                        // ← 여기서 this.name = name; this.tags = tags; 가 자동으로
}
```

`this.name = …`처럼 필드에 직접 대입하면 컴파일 오류다. 바꾼 매개변수 값이 마지막에 필드로 들어간다.

## 3. 본문 안에서 필드를 읽으면?

```java
public record R(String s) {
    public R {
        s = s.trim();
        System.out.println(s);     // "a"   — 매개변수
        System.out.println(s());   // null  — 접근자는 아직 대입 전 필드를 읽는다
    }
}
new R("  a ");
```

(JDK 25 jshell로 확인: 매개변수는 `a`, 접근자는 `null`)

그래서 생성자에서 쓰는 검증 헬퍼는 **`static` + 필요한 값을 매개변수로** 받는다. 인스턴스 메서드로 만들어 필드를 읽게 하면 생성자 안에서는 기본값만 본다.

```java
public record Order(String date, int qty) {
    public Order {
        toSchedule(date, qty);                // 결과는 버린다 — "만들 수 있는가"를 미리 확인하는 용도
    }
    public Schedule schedule() {
        return toSchedule(date, qty);         // 생성 후에는 필드로 같은 헬퍼 재사용
    }
    private static Schedule toSchedule(String date, int qty) { … }   // 실패하면 예외
}
```

## 4. ⚠️ 함정

- **접근자로 읽은 값이 기대와 다르다** — 위 3절. NPE로 터지면 원인을 찾기 어렵다.
- **헬퍼 이름이 하는 일과 맞지 않기 쉽다.** 생성자에서 결과를 버리는 쓰임 때문에 `validateXxx`라고 짓기 쉽지만, 실제로 값을 **만들어 반환**하면 `toXxx`/`parseXxx`가 맞다. 버리는 호출에는 "만들 수 있는지 미리 확인" 주석을 한 줄 남긴다.
- record에는 구성 요소 외의 인스턴스 필드를 둘 수 없다. "한 번 계산해 저장"이 필요하면 그 값을 구성 요소로 올리거나(직렬화 모양이 바뀜), 매번 계산한다.

## 5. 💡 판단 기준

**컴팩트 생성자 안에서는 "매개변수만 있다"고 생각한다.** 여러 값을 엮어 검사·변환하는 헬퍼는 `static`으로 두고 값을 넘긴다. 헬퍼 이름은 반환하는 것 기준(`toXxx`), 검사만 하면 `requireXxx`/`isXxx`.

## 참고·학습 기록

- [JLS 21 §8.10.4.2 Compact Canonical Constructors](https://docs.oracle.com/javase/specs/jls/se21/html/jls-8.html#jls-8.10.4.2)

**학습 날짜**: 2026-10-11
**계기**: record 생성자 안의 `validateBirthInfo(...)`가 검증이 아니라 값을 만들어 반환하는데 왜 static 메서드에 매개변수를 다 넘기는지 궁금해서.

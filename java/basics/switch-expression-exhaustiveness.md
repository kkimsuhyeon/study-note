# switch 문 vs switch 식 — enum exhaustiveness와 default의 판단 기준

> **한 줄 요약**: switch 식(expression, Java 14 표준화)은 값을 반드시 내놓아야 해서 컴파일러가 **모든 enum 상수 커버(exhaustiveness)를 강제**한다. 이 강제는 "enum에 상수가 추가되면 매핑 누락을 컴파일 에러로 잡아주는 안전망"인데, `default`를 넣는 순간 영구히 꺼진다 — default는 "앞으로 추가될 모든 값에 이 공통 처리가 옳을 때"만 넣는다.

## 언제 쓰나

- enum → 값 매핑처럼 **분기 결과를 변수에 대입**할 때 (식이 문보다 안전하고 간결)
- 채널별 설정, 타입별 핸들러, 상태별 메시지 등 "값마다 반드시 대응이 있어야 하는" 매핑

## 사용 예시

```java
// switch 문(statement) — 전통 방식. 누락돼도 컴파일 OK (조용히 지나감)
switch (frncId) {
    case PHAROS:
        channel = channels.get(0);
        break;             // break 없으면 fall-through
    case STELLA:
        channel = channels.get(1);
        break;
}

// switch 식(expression, Java 14+) — 결과를 대입. 전체 커버 강제
Channel channel = switch (frncId) {
    case PHAROS -> channels.get(0);   // 화살표: fall-through 없음
    case STELLA -> channels.get(1);
};  // enum 상수를 하나라도 빠뜨리면 컴파일 에러

// 블록이 필요하면 yield로 값 반환
int result = switch (grade) {
    case A -> 100;
    case B -> {
        log.info("B grade");
        yield 80;
    }
};
```

## 문 vs 식 비교

| | switch 문 | switch 식 |
|---|---|---|
| 결과 | 없음 (부수효과로 대입) | **값을 반환** (대입/리턴에 사용) |
| exhaustiveness | 강제 안 함 → 누락 조용히 통과 | **enum 전체 커버 또는 default 필수** (컴파일 에러) |
| fall-through | `:` 라벨은 break 누락 시 발생 | `->` 화살표는 없음 |
| 버전 | 항상 | Java 14 (JEP 361) |

- Java 21의 패턴 매칭 switch(JEP 441, sealed 타입/패턴 case)부터는 **문(statement)도** 패턴을 쓰면 exhaustiveness가 강제된다.

## ⚠️ 함정/메커니즘

### 1. enum에 상수를 추가하면 default 없는 switch 식들이 일제히 컴파일 에러

"the switch expression does not cover all possible input values" — 이건 버그가 아니라 **설계된 안전망**이다. 새 상수에 대한 매핑을 채우라고 컴파일러가 강제하는 것.

실제로 겪은 케이스: 장부 채널 enum(PHAROS/STELLA)에 무관한 서비스 채널 상수를 추가하려다, 알림톡 채널 매핑 switch 식 2곳이 즉시 깨지는 걸 확인. **채울 수 있는 올바른 매핑이 존재하지 않는 값이라면, 그 값은 그 enum에 속하지 않는다는 신호**다 (도메인 불일치). 결국 enum 추가를 포기하고 별도 리터럴로 처리했다.

### 2. default를 넣으면 안전망이 영구히 꺼진다

default가 있으면 이후 어떤 상수가 추가돼도 컴파일러는 침묵하고, 새 값은 조용히 default로 흘러간다 — 값마다 달라야 하는 매핑에서는 default가 곧 버그 은닉처.

### 3. 전체 커버 + default 없음이어도 런타임 안전장치가 있다

컴파일러가 **암묵 default를 삽입**해두는데, 컴파일 후에 enum만 새 버전(상수 추가)으로 교체된 채 실행되면 `IncompatibleClassChangeError`를 던진다 — 조용히 잘못 매핑되는 대신 즉사시키는 방향.

### 4. 엑셀/목록 검증 같은 `values()` 순회도 함께 오염된다

enum 상수 추가의 영향 범위는 switch만이 아니다 — `values()`, `getNms()` 같은 전체 순회 기반 허용값 목록·드롭다운·검증 로직에 새 상수가 자동으로 끼어든다. enum에 값을 추가하기 전에 `values()` 사용처를 함께 훑어야 한다.

## 💡 판단 기준

- **default를 넣을지는 "앞으로 추가될 모든 값에 이 처리가 옳은가"로 정한다.** 옳다(공통 폴백이 존재) → default. 값마다 달라야 한다 → default 금지, 전체 나열로 안전망을 유지한다. 알림톡 채널 매핑처럼 "새 채널엔 새 채널의 설정이 있어야 하는" 매핑이 후자의 전형.
- **enum 상수 추가가 컴파일 에러를 내면, 에러를 우회(default 추가)하기 전에 "이 값이 이 enum의 계약을 채울 수 있는가"부터 묻는다.** 못 채우면 enum이 아니라 값이 잘못 온 것 — 별도 상수/enum으로 분리한다.

## 참고

- [JEP 361: Switch Expressions (Java 14 표준화)](https://openjdk.org/jeps/361) — exhaustiveness·암묵 default·yield
- [JEP 441: Pattern Matching for switch (Java 21)](https://openjdk.org/jeps/441) — 패턴 switch의 exhaustiveness
- JLS §14.11.2 (Exhaustive Switch Blocks)

학습 날짜: 2026-08-07 · 계기: 채널 enum에 서비스 채널 상수를 추가하려다 default 없는 switch 식 2곳의 컴파일 에러 안전망을 만나며 — "default를 달면 되나?"에 대한 답 정리

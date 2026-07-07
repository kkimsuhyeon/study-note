# 람다 ≠ 비동기 — 실행 타이밍은 "받는 메서드"가 정한다

> **한 줄 요약**: 람다 `() -> ...`는 **"코드를 값으로 포장한 것"**일 뿐, 그 자체론 동기도 비동기도 아니다. **언제·어느 스레드에서 실행될지는 그 람다를 넘겨받은 메서드가 100% 정한다.** `forEach`에 주면 지금, `onCompletion`에 주면 나중(다른 스레드). 문법이 똑같아도 타이밍은 딴판이라, 코드 읽을 때 **"이 람다 누가·언제·어느 스레드에서 부르지?"**를 받는 메서드 기준으로 봐야 한다.

## 언제 헷갈리나 (이 노트가 필요한 이유)

```java
emitter.onCompletion(() -> repository.removeIfMatches(clientId, emitter));
```
이걸 보고 *"람다니까 비동기로 돌겠네?"* 라고 넘겨짚기 쉽다. 하지만 람다라서 비동기인 게 **아니다.** 실행 타이밍/스레드를 잘못 가정하면 → 실행 순서, 트랜잭션 경계, ThreadLocal 접근을 틀리게 짜서 **동시성 버그**가 난다. (아래 구체 케이스)

## 람다가 실제로 뭔가

- 람다는 **함수형 인터페이스(추상 메서드 1개)의 인스턴스** — "코드 조각을 값처럼 담아서 넘기는" 문법.
- `list.forEach(x -> print(x))` 는 `list.forEach(new Consumer<>(){ public void accept(x){ print(x);} })` 의 축약일 뿐.
- **메서드 레퍼런스** `repository::remove` 도 똑같은 것(람다의 다른 표기). 실행 의미는 동일하게 "받는 메서드가 정함".
- 즉 람다 **자체엔 "언제 실행"이라는 정보가 없다.** 그건 순전히 호출자 몫.

## 실행 타이밍 3부류

| 부류 | 언제 실행 | 대표 메서드 | 스레드 |
|---|---|---|---|
| **① 즉시 (동기)** | 그 메서드가 리턴하기 **전에**, 그 자리에서 | `forEach`, `computeIfAbsent`, `ifPresent`, `Comparator.compare`, `replaceAll` | **호출한 그 스레드** |
| **② 등록 후 나중 (콜백)** | 어떤 **이벤트가 터질 때** | `onCompletion`, `add~Listener`, `thenApply`, `subscribe` | 이벤트 주체의 스레드 (내 스레드 아닐 수 있음) |
| **③ 명시적 비동기** | 다른 스레드로 **던짐** | `executor.submit`, `CompletableFuture.supplyAsync`, `new Thread(...)` | **다른 스레드** |

```java
// ① 즉시 — 지금 하나씩 실행하고 리턴 (동기, 같은 스레드)
list.forEach(x -> print(x));
map.computeIfAbsent(k, k -> load(k));

// ② 콜백 — "예약"만. 실제 실행은 이벤트 때 (지금 아님)
emitter.onCompletion(() -> cleanup());     // "완료되면"
future.thenApply(r -> transform(r));       // "결과 오면"

// ③ 비동기 — 스레드풀/새 스레드로
executor.submit(() -> heavyWork());
```

> ⚠️ **두 개의 축을 분리해서 보라.** "지금 실행 vs 나중 실행"과 "같은 스레드 vs 다른 스레드"는 **별개 축**이다. 대개 ①은 같은 스레드, ③은 다른 스레드지만, ②(콜백)는 둘 다 가능하다 — `ifPresent`는 콜백처럼 넘기지만 지금·같은 스레드고, `onCompletion`은 나중·다른 스레드다.

## ⚠️ 함정 / 뉘앙스

- **"콜백이면 비동기"가 아니다.** `Optional.ifPresent(v -> ...)`, `Map.computeIfAbsent(k, ...)` 도 람다(콜백)를 받지만 **지금, 같은 스레드에서 동기로** 실행된다. "람다를 넘긴다 = 비동기"도, "콜백 = 비동기"도 둘 다 틀림. 축은 오직 **"받는 메서드가 지금 부르나 / 나중에 부르나".**
- **스트림 중간 연산은 lazy — `.map()` 시점에 실행 안 된다.** `stream.map(x -> f(x))` 의 람다는 `.map()`을 호출할 때가 아니라 **terminal 연산(`collect`/`forEach`/`count` 등)이 파이프라인을 당길 때** 실행된다. (Oracle 문서: *"Intermediate operations are always lazy... traversal does not begin until the terminal operation is executed."*) → 중간 연산만 쓰고 terminal이 없으면 **람다는 아예 안 돈다.**
- **같은 `forEach`라도 `parallelStream`이면 다른 스레드.** 병렬 스트림의 람다는 **공용 `ForkJoinPool`** 스레드들에서 실행된다 → 순서·스레드 가정이 깨진다. `thenApply` vs `thenApplyAsync` 도 마찬가지(같은 이름 계열도 Async면 풀로 감).
- **콜백 스레드엔 "요청 스레드의 상태"가 없다.** `@Transactional`, `SecurityContext`, `AuthUser.get()` 같은 **ThreadLocal 기반 값**은 ②③의 다른 스레드 콜백에선 **안 잡힌다.** 콜백 안에서 이런 걸 꺼내 쓰면 null/빈 트랜잭션이 된다. → 필요한 값은 **람다 캡처(파라미터)로 미리 넘겨야** 한다.
- **②는 호출 순서와 실행 순서가 다르다.** `onCompletion`은 "완료 시 컨테이너 스레드에서 호출"이라, 코드상 뒤에 있는 줄(`put`)이 **먼저** 실행될 수 있다 → 순서에 기대는 로직은 CAS(조건부) 연산으로 방어. (→ [map-methods: 조건부(CAS)](../collections/map-methods.md), [SseEmitter 노트](../spring/sse-emitter.md))

## 그래서 어떻게 구분하나

받는 메서드의 **이름 낌새 + javadoc**으로 판단:

| 이름 낌새 | 보통 | 부류 |
|---|---|---|
| `forEach`, `map`, `filter`, `compute*`, `ifPresent`, `sort` | 지금 실행 | ① |
| `on~`, `add~Listener`, `then~`, `subscribe`, `when~` | 나중 실행(콜백) | ② |
| `submit`, `~Async`, `new Thread`, `parallelStream` | 다른 스레드 | ②·③ |

애매하면 그 메서드 javadoc을 본다 — `onCompletion`은 *"async 요청이 완료될 때 **컨테이너 스레드에서** 호출된다"* 라고 명시돼 있어서 "나중 + 다른 스레드"임을 알 수 있었다.

## 💡 판단 기준

- **람다를 보면 실행이 아니라 "등록/전달"로 읽어라.** "이 코드 조각을 누구한테 넘기는 거지? 걔가 언제·어느 스레드에서 부르지?" — 받는 메서드가 답이다. `forEach`면 지금 여기, `onCompletion`이면 나중 딴 스레드.
- **콜백(②③) 안에서 "지금 이 스레드"를 가정하지 마라.** ThreadLocal(트랜잭션·인증), 실행 순서, "이 변수 아직 안 바뀌었겠지"를 콜백에 넣으면 지뢰. 필요한 값은 **캡처로 넘기고**, 순서 의존은 **조건부(CAS) 연산으로** 방어.
- 구체 케이스 (둘 다 지금 보는 PHAROS SSE 코드):
  1. **순서**: `onCompletion(() -> map.remove(clientId, emitter))` 콜백이 뒤의 `map.put(clientId, newEmitter)` 보다 **늦게/다른 스레드에서** 돌 수 있어, `remove`를 1인자가 아닌 **2인자(값 일치 시에만)** 로 써서 새 emitter를 보호했다.
  2. **ThreadLocal**: 채팅 인증에서 원래 `AuthUser.get()`(ThreadLocal)로 orgId/usrId를 꺼냈는데, 스트리밍 처리는 **Reactor/boundedElastic 콜백 스레드**에서 돌아 그 ThreadLocal이 비어버린다 → 그래서 helper를 `getToken(orgId, usrId)` 로 바꿔 **값을 파라미터로 넘기도록** 리팩터링했다. ("콜백 스레드엔 요청 스레드 상태가 없다"의 실제 사례.)

## 참고
- JLS §15.27 Lambda Expressions: https://docs.oracle.com/javase/specs/jls/se17/html/jls-15.html#jls-15.27
- Oracle `java.util.stream` 패키지 (중간 연산 lazy·직렬/병렬): https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/package-summary.html
- `ResponseBodyEmitter.onCompletion` javadoc ("container thread"): https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/ResponseBodyEmitter.html
- 관련 노트: [Map 조건부(CAS) 메서드](../collections/map-methods.md) · [SseEmitter 서버 구현](../spring/sse-emitter.md) · [스레드 기초](../concurrency/threads.md)

---
학습 날짜: 2026-07-08
계기: SSE `emitter.onCompletion(() -> ...)` 콜백을 보다가 "대부분의 람다는 다 비동기 처리냐?"는 질문에서 출발. 람다는 실행 의미가 없고 "받는 메서드가 타이밍을 정한다"를, 마침 보던 SSE 코드의 onCompletion(순서)·AuthUser.get()→파라미터 전환(ThreadLocal) 두 사례로 묶어 정리.

# Flux / Mono 기초 - 논블로킹이 본질 / lazy·신호 3종·스레드 모델 / MVC vs WebFlux에서의 위치

> 한 줄 요약: `Mono<T>`(나중에 올 값 0~1개) / `Flux<T>`(나중에 흘러올 값 0~N개)는 Reactor의 비동기 스트림 타입. **본질은 "실시간"이 아니라 논블로킹** — 외부 I/O를 기다리는 동안 스레드를 붙잡지 않기 위한 표현이다. 핵심 함정: **subscribe 전엔 아무 일도 안 일어나고(lazy)**, 에러는 throw가 아니라 **신호로 흘러오며**, 콜백은 이벤트 루프에서 돌아 **블로킹·ThreadLocal 금지**.

## 언제 만나나
- **WebFlux 프로젝트**: 모든 곳. 단순 단건 조회 API도 `Mono<UserResponse>` 리턴이 기본.
- **Spring MVC 프로젝트**(서블릿·블로킹): **가장자리에서만** —
  - `WebClient` 호출 (RestTemplate 후계자가 리액티브 네이티브라서. 보통 받자마자 `.block()`으로 동기 세계 복귀)
  - 스트리밍 relay (외부 SSE를 `bodyToFlux`로 구독해 흘려보내기 — 블로킹 모델로는 "조금씩 도착하는 걸 조금씩 전달"을 표현 못 함)
  - `Flux.interval` 같은 유틸 (heartbeat 타이머)

## 타입의 의미 — "언제·몇 개"의 조합

| 타입 | 몇 개 | 언제 |
|---|---|---|
| `String` | 1 | 지금 |
| `List<String>` | N | 지금 (다 모일 때까지 블로킹) |
| `CompletableFuture<String>` | 1 | 나중에 |
| `Mono<String>` | 0~1 | 나중에 |
| `Flux<String>` | 0~N | **나중에 하나씩** |

왜 필요한가 — MVC 모델은 외부 API가 3초 걸리면 스레드 1개가 3초간 점유된다(동시 500요청 = 스레드 500개). 논블로킹은 "응답 오면 알려줘" 하고 스레드를 놔버려 소수의 이벤트 루프 스레드로 수천 동시 요청을 버틴다. `Mono`/`Flux`는 그 "나중에 올 결과"를 담는 그릇.

## 사용 예시 — 조립과 구독은 분리된다

```java
// [조립 단계] 파이프 설계도만 만든다 — HTTP 요청 아직 안 나감!
Flux<String> stream = webClient.post()
        .uri("/chat/stream")
        .accept(MediaType.TEXT_EVENT_STREAM)
        .bodyValue(request)
        .retrieve()
        .bodyToFlux(String.class)                 // 응답 본문 → 라인 단위 스트림
        .doOnError(e -> log.warn(...))            // 엿보기 창문 (구독 아님, 신호는 그대로 통과)
        .takeUntilOther(Mono.delay(ofMinutes(10))); // 10분 밸브 (무한 스트림 방지)

// [구독 단계] 여기가 시작 버튼 — 이 순간 HTTP 요청이 나가고 물이 흐른다
stream.subscribe(
        line  -> handle(line),      // onNext:     값 도착 (0~N번)
        error -> cleanup(),         // onError:    에러 종료 (1번, 끝)
        ()    -> complete()         // onComplete: 정상 종료 (1번, 끝)
);
```

- **신호 규약: `onNext* → (onError | onComplete)`** — 값이 여러 번 오다가 반드시 둘 중 하나로 끝난다. List엔 없는 개념("정상적으로 다 왔다" vs "중간에 터졌다"의 구분).
- `Flux.interval(Duration)` — 시간 자체를 스트림으로. 타이머를 같은 패러다임(subscribe/dispose)으로 처리.
- `subscribe(...)`가 돌려주는 **`Disposable`** = 구독을 끊는 손잡이. 타이머·구독 정리는 `disposable.dispose()`.

## ⚠️ 함정
1. **lazy — subscribe 전엔 아무 일도 안 일어난다.** `Flux`를 리턴받았다고 "연결됐다"가 아니라 "연결하는 방법을 받았다". 시작 시점을 착각하면 로그·에러 위치를 오해한다.
2. **에러는 throw가 아니라 신호다.** 조립 메서드를 try-catch로 감싸도 스트림 에러는 안 잡힌다 — 에러는 나중에 subscribe 이후 발생해 onError 콜백으로 배달된다. 실제 사례: 블로킹 requester(`.block()` — 예외가 진짜 던져짐)를 복사해 만든 스트리밍 requester에 `catch (WebClientResponseException)`이 남아 있었는데, **잡힐 일 없는 죽은 코드**였다. 블로킹 코드를 리액티브로 옮길 땐 에러 처리도 catch → onError로 이사시켜야 한다.
3. **콜백은 이벤트 루프(Netty) 스레드에서 돈다.**
   - **블로킹 금지**: 콜백에서 블로킹 DB I/O를 하면 그 이벤트 루프가 처리하던 다른 사용자 요청까지 멈춘다. 블로킹 작업은 `Mono.fromRunnable(...).subscribeOn(Schedulers.boundedElastic()).subscribe()`로 분리.
   - **ThreadLocal 없음**: 요청 스레드의 SecurityContext(`AuthUser.get()` 류)가 콜백에선 비어 있다. 필요한 값은 파라미터로 명시 전달. ([람다 실행 타이밍](../functional/lambda-execution-timing.md)의 "콜백 스레드엔 ThreadLocal 없음"과 같은 원리)
4. **이벤트 루프 위에서 `.block()` 금지.** 자기가 처리해줄 결과를 자기가 기다리는 꼴 — 최악은 데드락. MVC 컨트롤러(서블릿 스레드)에서의 `.block()`은 허용되는 타협이지만, 리액티브 콜백 안에서는 절대 금지.
5. `doOn~` 계열(doOnNext/doOnError...)은 **구독이 아니라 관측**이다. 신호를 소비하지 않고 엿본 뒤 그대로 아래로 흘려보낸다. 로그용이지 에러 "처리"가 아니다.

## MVC vs WebFlux — 어디까지 알아야 하나

| | Spring MVC | Spring WebFlux |
|---|---|---|
| 요청 모델 | 요청 1 = 스레드 1 (블로킹 OK) | 이벤트 루프 소수가 전 요청 (블로킹 금지) |
| Mono/Flux | 가장자리(WebClient·스트리밍)에서만 | 모든 곳 |
| 필요한 지식 | **부분집합이면 충분**: 타입 의미 / lazy / 신호 3종 / 스레드 모델·boundedElastic / Disposable | 전체 패러다임 (연산자 조합, 백프레셔, Context API...) |

MVC 프로젝트라면 flatMap/zip 같은 조합 연산자나 백프레셔 세부는 당장 몰라도 코드를 읽고 고칠 수 있다.

## 💡 판단 기준
- **"Mono/Flux = 실시간용"이라고 기억하지 말 것 — "스레드가 기다리지 않게 하는 타입"이다.** 실시간 스트리밍은 응용 사례 중 하나일 뿐, WebFlux에선 평범한 단건 조회도 Mono다. 이렇게 잡아둬야 WebFlux 코드를 만났을 때 안 헷갈린다.
- 리액티브 코드를 읽을 때 항상 두 가지를 물을 것: **"이 줄은 조립인가 실행(구독)인가"**, **"이 콜백은 어느 스레드에서 도는가"**. 구체 케이스: LLM 스트리밍 relay에서 라인 처리 콜백(이벤트 루프)에 블로킹 DB 저장이 섞이면 안 되는 이유도, 인증 헬퍼가 ThreadLocal을 버리고 orgId/usrId를 파라미터로 받게 리팩토링된 이유도 전부 두 번째 질문에서 나왔다.

## 참고
- Project Reactor 공식 레퍼런스: https://projectreactor.io/docs/core/release/reference/
- Flux javadoc: https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html
- Schedulers (boundedElastic): https://projectreactor.io/docs/core/release/reference/#schedulers
- Spring WebClient: https://docs.spring.io/spring-framework/reference/web/webflux-webclient.html
- 관련 노트: [SseEmitter 구현 패턴·스레드 모델](../spring/sse-emitter.md) · [람다 실행 타이밍](../functional/lambda-execution-timing.md) · [SSE 프로토콜](../../infra/network/sse.md)

---
학습 날짜: 2026-07-08
계기: MVC 프로젝트의 LLM 채팅 스트리밍 relay 코드에서 `WebClient.bodyToFlux` → `subscribe`로 이어지는 경로를 따라가며, "requestPost 안에 onNext가 왜 없지?"(조립/구독 분리), "catch가 왜 안 잡히지?"(에러=신호), "Flux는 실시간용인가?"(아니, 논블로킹용)를 차례로 정리.

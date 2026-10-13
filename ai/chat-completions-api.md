# Chat Completions API 응답 다루기 — 두 겹 응답, 무상태 재요청, 스키마 강제의 한계

> **한 줄 요약**: Chat Completions 응답은 **두 겹**이다. 바깥은 API 봉투(`choices[0].message`·`finish_reason`)이고, 모델이 쓴 결과물은 그 안의 `content` **문자열**에 들어 있다. 서버는 이전 요청을 기억하지 않으므로(stateless) "방금 답을 고쳐"는 **대화 전체를 다시 보내는 것**이다. `json_schema` 강제를 켜도 거절(`refusal`)이나 토큰 한도(`length`)로 스키마를 벗어날 수 있다 — AI 응답은 받는 쪽에서 봉투와 내용을 따로 검사하는 신뢰하지 않는 입력이다.

## 언제 쓰나

- OpenAI 또는 OpenAI 호환 서버(같은 `/chat/completions` 모양을 흉내 내는 로컬·자체 호스팅 추론 서버 등)를 HTTP로 직접 호출할 때
- 모델에게 JSON 결과물을 받아 업무 데이터로 저장할 때
- 결과가 규칙에 안 맞으면 모델에게 고쳐 달라고 다시 요청하는 흐름을 만들 때

## 요청 모양

```json
{
  "model": "some-model",
  "messages": [
    {"role": "system", "content": "역할·출력 규칙"},
    {"role": "user",   "content": "입력 데이터"}
  ],
  "max_tokens": 2048,
  "response_format": {"type": "json_schema", "json_schema": {"name": "result", "strict": true, "schema": { }}}
}
```

| 역할 | 뜻 |
|---|---|
| `system` / `developer` | 개발자가 주는 지시. 공식 문서: o1 이후 모델에서는 developer 메시지가 system 메시지를 대신한다 |
| `user` | 사용자 입력 |
| `assistant` | **모델이 했던 답**. 이전 대화를 다시 보낼 때 이 역할로 넣는다 |
| `tool` | 도구 호출 결과 |

## 응답 모양 — 두 겹

```json
{
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": "{\"summary\":\"...\",\"items\":[...]}",
      "refusal": null
    },
    "finish_reason": "stop"
  }],
  "usage": {"prompt_tokens": 19, "completion_tokens": 10, "total_tokens": 29}
}
```

- **바깥 겹(봉투)**: API가 정한 모양. 생성이 정상 종료됐는지, 거절인지, 도구 호출인지를 알려 준다.
- **안쪽 겹(content)**: 모델이 쓴 결과물. JSON을 요청했어도 **문자열**로 들어 있어서 한 번 더 파싱해야 한다. 공식 레퍼런스상 `content`는 `string or null`이다.

### `finish_reason` 값 (공식 레퍼런스)

| 값 | 뜻 | 받는 쪽 처리 |
|---|---|---|
| `stop` | 자연스럽게 끝남 또는 stop 시퀀스 도달 | 정상. content 검사로 진행 |
| `length` | 요청의 최대 토큰 수에 도달 | **잘린 결과**. JSON이면 닫는 괄호가 없을 수 있다 |
| `content_filter` | 콘텐츠 필터로 내용이 빠짐 | 실패 처리 |
| `tool_calls` | 모델이 도구를 호출함 | 도구를 안 쓰는 흐름이면 실패 처리 |
| `function_call` | (deprecated) 함수 호출 | 위와 같음 |

## 받는 쪽 처리 순서

```text
HTTP 응답
 ├─ ① 상태 코드: 429면 Retry-After 읽기, 200이 아니면 본문을 읽지 않고 실패
 ├─ ② 크기 상한: 본문을 N바이트까지만 읽기 (무제한 readAllBytes 금지)
 ├─ ③ 봉투 검사: choices 존재, finish_reason == "stop", content가 비어 있지 않은 문자열,
 │              refusal 없음, tool_calls 없음 → content 문자열만 꺼냄
 └─ ④ 내용 검사: content를 도메인 타입으로 엄격 파싱(모르는 필드·중복 키·타입 변환 거부)
                + 업무 규칙(길이·개수·참조 무결성)
```

①②는 HTTP 통신 문제라 클라이언트 계층에, ③은 API 형식 지식이라 공급자 어댑터에, ④는 업무 규칙이라 검증기에 두면 각자 바뀌는 이유가 하나씩만 남는다. 요청 본문을 만드는 일(프롬프트 + 입력 → 요청 JSON)도 따로 떼면, "보내는 쪽"과 "받는 쪽"이 섞이지 않는다.

```text
RequestBuilder(요청 JSON 만들기) → HttpClient(①②) → ProviderAdapter(③) → Validator(④)
```

## 다시 묻기 = 대화 전체 재전송

공식 대화 상태 가이드는 각 생성 요청이 독립적이고 stateless라고 설명한다. 그래서 여러 턴을 이어 가려면 이전 메시지를 요청에 직접 넣어야 한다. 검증에 실패한 답을 고쳐 달라고 할 때도 같다.

```json
"messages": [
  {"role": "system",    "content": "같은 지시"},
  {"role": "user",      "content": "같은 입력"},
  {"role": "assistant", "content": "<거부된 직전 답 원문>"},
  {"role": "user",      "content": "직전 답이 규격 검사에 실패했다. 아래 위반을 고쳐 JSON 하나만 다시 출력하라.\n[{\"path\":\"$.summary\",\"rule\":\"TEXT_LENGTH\"}]"}
]
```

- 직전 답은 **파싱하지 않고 원문 그대로** `assistant` 역할로 넣는다. 모델 입장에서는 "내가 이렇게 답했는데 이게 틀렸다"는 맥락이 된다.
- 위반 목록은 "어디(`path`)가 무슨 규칙(`rule`)에 걸렸는지"처럼 기계적으로 만든 것을 넣는다. 사람 말로 길게 쓰는 것보다 재현 가능하다.
- 서버 쪽에 상태를 맡기는 방법도 있다. 같은 회사의 Responses API는 `previous_response_id`나 Conversations API로 이전 턴을 이어 붙일 수 있다. 다만 가이드에 따르면 이 경우에도 이전 입력 토큰은 다시 과금된다.

## ⚠️ 함정

### 1. 스키마 강제(`strict: true`)를 켜도 스키마 밖의 응답이 온다

구조화 출력 가이드는 스키마 준수를 보장한다고 설명하면서도, 두 예외를 직접 적어 둔다.
- **거절**: 안전상 이유로 거절하면 스키마를 따르지 않는 `refusal` 필드가 대신 온다.
- **토큰 한도**: 최대 토큰에 걸리면 응답이 불완전하다(`finish_reason: "length"`).

OpenAI 호환 서버는 `response_format`·`strict` 지원 범위가 구현마다 다를 수 있다(서버별 확인 필요). 그래서 스키마 강제는 "실패율을 낮추는 장치"이고, 받는 쪽 검증을 대신하지 않는다.

### 2. `content`를 바로 도메인 타입으로 읽으면 실패 원인이 섞인다

봉투 검사 없이 `content`만 꺼내 파싱하면, 잘린 응답(`length`)·거절·도구 호출이 모두 "JSON 파싱 실패" 하나로 보인다. 원인을 구분해야 재시도할지(일시 장애), 프롬프트를 고칠지(형식 위반), 포기할지(거절)를 정할 수 있다.

### 3. 다시 묻기는 공짜가 아니다

재요청에는 처음 입력 + 직전 답 + 위반 목록이 모두 들어가므로 입력 토큰이 커진다. 교정 횟수 상한(예: 1회)과 요청 크기 상한을 같이 둔다. 직전 답이 비정상적으로 길면 교정 요청 자체가 상한을 넘을 수 있다 — 이 경우는 우리 코드 버그가 아니라 AI 출력 문제로 분류한다.

### 4. AI 출력은 화면에서도 신뢰하지 않는 입력이다

검증을 통과한 문자열도 HTML로 그대로 끼워 넣으면 XSS 통로가 된다 → [XSS와 CSP](../infra/network/xss-and-csp.md).

## 💡 판단 기준

**AI 응답은 "외부 API가 돌려준 JSON 안에 들어 있는, 사용자가 쓴 것 같은 문자열"로 다룬다.** 봉투(API 형식)와 내용(업무 규칙)을 서로 다른 단계에서 검사하고, 다시 묻기는 대화를 통째로 다시 보내는 일이라 횟수·크기 상한을 처음부터 같이 설계한다. 응답 처리 코드가 "요청 만들기 / HTTP / 봉투 / 내용" 중 두 가지 이상을 한 메서드에서 하고 있으면 나눌 신호다.

## 참고

- [OpenAI: Create chat completion (API reference)](https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create) — `finish_reason` 값, `content: string or null`, developer 역할
- [OpenAI: Conversation state](https://developers.openai.com/api/docs/guides/conversation-state) — 요청은 독립·stateless, 이전 메시지를 직접 넣기, `previous_response_id`
- [OpenAI: Structured model outputs](https://developers.openai.com/api/docs/guides/structured-outputs?api-mode=chat) — 스키마 준수 보장, refusal 필드, max tokens로 불완전한 응답
- HTTP 쪽 함정(오류 본문 drain, read timeout 의미)은 [HTTP 클라이언트 구현체 비교](../java/spring/http-client-transports.md)
- 키가 박힌 클라이언트를 Bean으로 내놓는 방법은 [@Bean 등록과 타입 기반 주입](../java/spring/bean-registration-and-injection.md)

학습 날짜: 2026-10-13 · 계기: AI 요청 본문을 만드는 클래스가 "AI 응답을 다시 파싱하는 것인지" 헷갈렸던 질문에서 — 요청 만들기·HTTP·봉투·내용 검사가 어떻게 나뉘는지와, 교정 요청이 왜 직전 답을 assistant 역할로 다시 보내는지 정리. 공식 문서 세 곳은 2026-10-13에 확인.

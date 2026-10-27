# 같음 판정 - `equals`(정체성) vs 이름 붙인 비교 메서드(업무 규칙)

> `equals`는 Java 전체가 "이 둘이 **같은 것**인가"로 읽는 약속이다. 엔티티라면 id, 값 객체라면 모든 필드가 기준이다. "같은 요청인가", "같은 좌석인가"처럼 **업무 규칙으로서의 같음**은 `isSameRequest(other)` 같은 이름 붙인 메서드로 따로 둔다. `equals`를 일부 필드만 비교하도록 재정의하면, 그 규칙이 `HashSet`·AssertJ·Mockito까지 새어 나간다.

## 언제 이 판단이 필요한가

- 저장된 기록과 이번 요청을 비교해서 갈래를 나눌 때 (멱등키 재요청: 같은 내용이면 무시, 다르면 거절)
- `record`나 Lombok `@EqualsAndHashCode`가 "공짜로" 비교를 주니까 그걸 쓰고 싶을 때
- 비교에 쓸 필드가 객체의 일부뿐일 때 → `equals`를 직접 재정의하고 싶어질 때

## 세 가지 방법

예시 객체: 멱등키 요청 기록. DB에서는 PK가 `(userId, idempotencyKey)`이고, 모델은 `operation`, `payloadHash`만 갖는다.

```java
// ① 모든 필드 equals — record 또는 @EqualsAndHashCode
public record RequestRecord(String operation, String payloadHash) {}

// ② 일부 필드만 equals를 직접 재정의
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof RequestRecord other)) return false;
    return operation.equals(other.operation) && payloadHash.equals(other.payloadHash);
}
@Override public int hashCode() { return Objects.hash(operation, payloadHash); }   // 반드시 같은 필드로

// ③ 이름 붙인 비교 메서드 (equals는 Object 기본 = 같은 인스턴스인지)
public boolean isSameRequest(RequestRecord other) {
    return operation.equals(other.operation) && payloadHash.equals(other.payloadHash);
}
```

| | ① 모든 필드 equals | ② 일부 필드 equals | ③ 이름 붙인 메서드 |
|---|---|---|---|
| 나중에 필드 추가(예: 저장된 응답 본문) | ⚠️ 뜻이 바뀜: 처리 전 요청(응답 없음)과 저장된 기록이 영원히 다름 → 정상 재시도가 거절됨 | 영향 없음 | 영향 없음 |
| 서로 다른 유저의 "1만 원 충전" 기록 두 개 | `equals` → true | `equals` → true | `equals` → false (다른 인스턴스) |
| `HashSet`·`isEqualTo`·Mockito 인자 비교 | 규칙을 따라감 | 규칙을 따라감 | 영향 없음 |
| 작성량 | 한 줄 | `equals` + `hashCode` 10줄 이상, 계약 준수 | 메서드 하나 |
| 테스트에서 새로 만든 객체로 `verify` | 그대로 됨 | 그대로 됨 | `ArgumentCaptor` / `argThat` 필요 |

## 함정

### ⚠️ 일부 필드 `equals`는 다른 것을 "같은 것"이라고 말한다

정체성(`userId`, `idempotencyKey`)이 모델 밖에 있으면, 남은 필드로 만든 `equals`는 서로 다른 기록을 같다고 판정한다.

```text
유저 A: RequestRecord(CHARGE, hash(10000))
유저 B: RequestRecord(CHARGE, hash(10000))

a.isSameRequest(b) → true   "요청 내용이 같다" — 사실
a.equals(b)        → true   "같은 기록이다"   — 사실이 아님
```

호출하는 지점의 결과는 같아도, `equals`의 거짓 답은 아래처럼 다른 곳에서 문제가 된다.

### ⚠️ `equals`를 쓰는 라이브러리가 전부 그 규칙을 따른다

- `HashSet`/`HashMap`: A와 B의 기록이 원소 하나로 합쳐진다.
- AssertJ `isEqualTo`: 기본 비교가 `equals`다. 비교에서 빠진 필드(예: 응답 본문)가 틀려도 **테스트가 통과**한다. 테스트가 버그를 숨긴다.
- Mockito: `verify(mock).save(x)`는 인자를 `equals`로 비교한다 ("natural java style"). 그래서 빠진 필드는 검증되지 않는다.

### ⚠️ `equals`만 고치고 `hashCode`를 빼먹으면 해시 컬렉션이 깨진다

`Object` 계약: `equals`가 true인 두 객체는 `hashCode`도 같아야 한다. 이걸 어기면 이렇게 된다.

```java
Set<RequestRecord> set = new HashSet<>();
set.add(new RequestRecord("CHARGE", "abc"));
set.contains(new RequestRecord("CHARGE", "abc"));   // hashCode 미재정의 → 다른 버킷을 찾아 false
```

그 밖의 계약도 직접 지켜야 한다: 반사성, 대칭성, 추이성, 일관성, `equals(null)`은 false. 타입 검사(`instanceof`와 `getClass()`)를 잘못 고르면 상속 관계에서 대칭성이 깨진다 (Effective Java Item 10).

### ⚠️ 문자열 필드는 `==`가 아니라 `equals`

해시 문자열은 실행 중에 계산된 새 `String` 객체다. 그래서 내용이 같아도 `==`는 false다. 테스트에서 리터럴끼리 비교하면 문자열 풀 때문에 우연히 통과할 수 있어서 더 늦게 발견된다. 같은 메커니즘: [오토박싱 & 래퍼 캐시](../basics/autoboxing-wrapper-cache.md).

## 테스트에서 ③을 검증하는 방법

`equals`가 기본(인스턴스 비교)이면, 유스케이스 안에서 만든 객체와 테스트에서 만든 객체는 다르다고 판정된다.

```java
// 1) 넘어간 객체를 꺼내서 필드 확인
ArgumentCaptor<RequestRecord> captor = ArgumentCaptor.forClass(RequestRecord.class);
verify(service).save(eq("user1"), eq(key), captor.capture());
assertThat(captor.getValue().operation()).isEqualTo("CHARGE");

// 2) 비교 규칙 자체를 매처로 재사용
verify(service).save(eq("user1"), eq(key), argThat(r -> r.isSameRequest(expected)));

// 3) 필드 단위 비교 (equals를 쓰지 않음)
assertThat(actual).usingRecursiveComparison().isEqualTo(expected);
```

`equals`를 재정의해서 얻는 실익은 "`verify`가 그대로 된다"는 것 하나뿐이다. 위 방법이면 몇 줄로 해결되므로, 그것 때문에 `equals`의 뜻을 바꿀 이유는 없다.

## 💡 판단 기준

- **`equals` = 정체성.** 엔티티는 id, 값 객체(`Money(currency, amount)`처럼 필드 전체가 곧 그 값이고 앞으로도 늘지 않는 것)는 모든 필드. 이때는 `record`나 `@EqualsAndHashCode`로 자동 생성하고 손으로 쓰지 않는다.
- **일부 필드만 비교하는 `equals`가 떠오르면**, 그게 엔티티의 id 비교가 아닌 이상 업무 규칙을 `equals`에 숨기려는 신호다 → 이름 붙인 메서드로.
- 이름 붙인 메서드는 호출부에서 뜻이 보인다: `previous.isSameRequest(incoming)` vs `previous.equals(incoming)`. 도메인 모델의 `isOwnedBy`, `isPayable`과 같은 결이다.

## 참고

- `Object.equals` / `hashCode` 계약 (Java 21 API): https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Object.html#equals(java.lang.Object)
- Effective Java 3판 Item 10(equals 일반 규약), Item 11(equals 재정의 시 hashCode도)
- Mockito 인자 매처 ("verifies argument values in natural java style: by using an equals() method"): https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/ArgumentMatchers.html
- AssertJ 재귀 비교: https://assertj.github.io/doc/#assertj-core-recursive-comparison
- 관련: [값 타입 §4 — 값 타입의 equals/hashCode](../jpa/value-types.md) · [컬렉션 계층 — Set 중복 판별](../collections/collection-hierarchy.md) · [Lombok 생성자](../basics/lombok-constructors.md)

---
학습 날짜: 2026-10-27 · 계기: 멱등키 기록 비교를 `isSameRequest`로 만들다가 "`@EqualsAndHashCode`면 되지 않나", "`equals`를 직접 재정의하면?"으로 이어진 질문

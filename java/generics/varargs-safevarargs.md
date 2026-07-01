# 가변인자(varargs) & @SafeVarargs — 제네릭 varargs의 heap pollution

> **한 줄 요약**: `타입... 이름`은 "인자 0개 이상"을 받는 문법이고 내부적으로 **배열**로 변환된다. 제네릭(`T...`, `List<X>...`)과 결합하면 **배열은 실체화(reifiable)·제네릭은 소거(non-reifiable)**라 안 맞아 **heap pollution**(타입 안전성 오염) 경고가 뜬다. `@SafeVarargs`는 "이 varargs를 안전하게만 다룬다"는 개발자의 **약속**으로 경고를 억제한다 — **경고를 없앨 뿐 안전을 보장하진 않는다.** 안전 기준 = varargs 배열을 **읽기만** 하고 **저장·노출·반환하지 않기.**

관련 노트: [리스트 생성](../collections/list-creation.md) (List.of 등 varargs 팩토리)

---

## 1. varargs(가변인자)가 뭔가
- `메서드(타입... 이름)` — 같은 타입 인자를 **0개 이상** 받는 문법. 호출 쪽은 `f()`, `f(a)`, `f(a, b, c)` 처럼 개수 자유.
- 규칙: **마지막 파라미터 하나만** varargs로 가능, `타입...` 형태. `f(String prefix, int... nums)`처럼 앞에 일반 파라미터가 올 수 있다.
- 컴파일러가 내부적으로 **배열**로 바꾼다. `int... nums` → 메서드 안에서는 `int[] nums`이고, 호출 `f(1, 2, 3)` → `f(new int[]{1, 2, 3})`.

```java
static int sum(int... nums) {          // nums 는 사실 int[]
    int s = 0;
    for (int n : nums) s += n;          // 배열처럼 순회
    return s;
}
sum();          // 0  (빈 배열)
sum(1, 2, 3);   // 6
```

## 2. 언제 문제되나 — 제네릭 varargs만
varargs **원소가 제네릭(매개변수화) 타입**이면 컴파일러가 경고를 낸다:
```java
static <T> List<T> listOf(T... items) { ... }          // ⚠️ unchecked
static Segments combine(List<Seg>... sources) { ... }  // ⚠️ unchecked
```
> ⚠️ *Possible heap pollution from parameterized vararg type*

이유: **배열은 reifiable**(런타임에 원소 타입을 유지) 인데 **제네릭은 non-reifiable**(소거되어 런타임엔 타입 정보가 없음). 이 둘을 섞으면 컴파일러가 타입 안전을 보장하지 못한다. → 원시타입 `int...`, 구체타입 `String...`은 reifiable이라 경고가 **없고**, **제네릭 varlargs만** 경고 대상.

## 3. heap pollution = "힙 낭비"가 아니다 (메커니즘)
**heap pollution = 매개변수화 타입 변수가 실제로는 *다른* 매개변수화 타입 객체를 가리키는 상태** = 타입 안전성 오염. 메모리 낭비와는 무관하다.

`T... args`는 메서드 안에서 `T[]`이고, 소거 때문에 런타임엔 사실상 `Object[]`. 이 틈으로 엉뚱한 타입이 끼어들 수 있다:

```java
// (a) 배열을 그대로 노출 — 위험
static <T> T[] toArray(T... args) { return args; }
static <T> T[] danger(T a, T b) { return toArray(a, b); }

String[] ss = danger("x", "y");   // 실제 만들어진 건 Object[] → ClassCastException!

// (b) 배열에 저장 — 위험
static void pollute(List<String>... lists) {
    Object[] arr = lists;          // List<String>[] → Object[] (허용됨)
    arr[0] = List.of(1, 2, 3);     // List<Integer> 를 슬쩍 심음 (heap pollution!)
    String s = lists[0].get(0);    // 꺼낼 때 ClassCastException
}
```
→ 오염은 **심는 순간엔 조용하고**, 나중에 **엉뚱한 곳에서 ClassCastException**으로 터진다. 그래서 위험하다.

## 4. @SafeVarargs — "안전하게만 쓴다"는 약속
- 하는 일: 선언부 + **모든 호출부**의 unchecked 경고를 억제. (호출자가 일일이 `@SuppressWarnings`를 안 달아도 됨 → 라이브러리 API에서 유용: `List.of`, `Arrays.asList`, `Stream.of`.)
- **경고만 끈다. 실제 안전은 개발자 책임.** 잘못 붙이면 위의 ClassCastException을 오히려 감춰버린다.
- 붙일 수 있는 위치 = **오버라이드가 불가능한** 메서드(안전 약속을 하위 클래스가 깰 수 없어야 하므로):

| 위치 | 가능? |
|---|---|
| `static` 메서드 | ⭕ |
| `final` 인스턴스 메서드 | ⭕ |
| 생성자 | ⭕ |
| `private` 인스턴스 메서드 | ⭕ (**Java 9+**) |
| 일반(non-final) 인스턴스 메서드 | ❌ (오버라이드로 약속이 깨질 수 있어 금지) |

- `@SuppressWarnings("unchecked")`와의 차이: 그건 **그 지점만** 끄고 호출부는 여전히 경고. `@SafeVarargs`는 **호출부까지** 끈다.

## 5. 💡 판단 기준 — 붙여도 되는가
**varargs 배열을 읽기만(iterate) 하고 저장·수정·외부 노출·반환을 안 하면 안전** → 붙여도 된다.

구체 케이스(이 프로젝트 `Segments.combine`):
```java
@SafeVarargs
static Segments combine(List<Seg>... sources) {
    List<Seg> merged = new ArrayList<>();
    for (List<Seg> s : sources) {          // ✅ 읽기만
        if (s != null) merged.addAll(s);   // ✅ 값만 새 리스트로 복사
    }
    return new Segments(merged);           // ✅ sources 배열 자체는 노출 안 함
}
```
반대로 **그 배열을 반환/저장/다른 메서드로 넘기면** 붙이면 안 된다(위 `toArray`처럼).

> 한 줄: **제네릭 varargs는 "읽기 전용"으로만 쓰고 그 배열을 밖으로 흘리지 마라.** 그 조건이면 `@SafeVarargs`로 경고를 끈다. 애매하면 varargs 대신 `List<T>` 파라미터를 받는 편이 더 안전하다.

## 6. 참고
- [Oracle Java Tutorials — Non-Reifiable Types & Varargs](https://docs.oracle.com/javase/tutorial/java/generics/nonReifiableVarargsType.html)
- [Baeldung — Java @SafeVarargs Annotation](https://www.baeldung.com/java-safevarargs)
- [Oracle — Improved Compiler Warnings When Using Non-Reifiable Formal Parameters with Varargs](https://docs.oracle.com/javase/8/docs/technotes/guides/language/non-reifiable-varargs.html)

---

**학습 날짜**: 2026-07-01
**계기**: `Segments.combine(List<AttendanceTraceSegment>... sources)`에서 `@SafeVarargs`를 처음 보고 — varargs 개념 자체부터, 제네릭 varargs가 왜 heap pollution 경고를 내는지(배열 reifiable vs 제네릭 non-reifiable), 붙일 수 있는 위치와 안전 기준(읽기만/노출 금지)을 정리.

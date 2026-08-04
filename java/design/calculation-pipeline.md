# 계산 파이프라인 구조 — Pipes and Filters + supports-execute + Collecting Parameter

> **한 줄 요약**: 여러 단계로 이루어진 도메인 계산을 "불변 입력(Factor) → 정책(Policy)으로 변환 → 순서가 고정된 계산기(Calculator)들이 공유 컨텍스트(Context)를 누적 갱신 → 결과(Result)"로 조립하는 구조. 하나의 패턴이 아니라 **GoF/POSA 패턴 4~5개의 조합**이며, 핵심 함정은 "단계 간 의존이 컨텍스트 뒤에 숨는다 — **실행 순서가 곧 스펙**"이라는 점.

## 언제 쓰나

이런 신호가 **겹칠 때** 꺼내는 구조다 (하나만으로는 부족):

- 계산이 **3단계 이상**이고 단계 사이에 **순서 의존**이 있다 (예: 금액 확정 → 과세 분해 → 세금 → 보험 → 합계)
- 단계 일부를 **조건부로 켜고 꺼야** 한다 (예: "근태 반영해서 계산" 옵션, "간이 계산" 모드)
- 입력을 만들기 위한 **조회가 무겁고 다양**해서, 조회(I/O)와 계산(순수 로직)을 분리하고 싶다
- 각 단계를 **DB/Mock 없이 단위 테스트**하고 싶다
- 입력 종류에 따라 **계산 방식 자체가 갈린다** (예: 급여/상여, 회원/비회원)

반대로 단계가 1~2개고 옵션 분기도 없다면 이 구조는 과잉이다 — 그냥 메서드 2개로 충분.

## 전체 구조

```
Service 계층 (조회/조립 담당)              도메인 계층 (순수 계산)
──────────────────────────        ─────────────────────────────────
DB 조회를 전부 끝내고               Executor  ← 진입점
불변 Factor에 담아 전달   ───────►    │ ① PolicyFinder.find(factor).apply(factor)
                                     │    (입력 종류별 정책이 factor 변환·계산유형 결정)
                                     │ ② Pipeline.execute(appliedFactor)
                                     ▼
                                  Pipeline (calculator 순서 고정)
                                    1. ACalculator ── supports? ──► doCalculate(ctx)
                                    2. BCalculator ── supports? ──► doCalculate(ctx)
                                    3. SummaryCalculator ...
                                     │      (모두가 Context 하나를 누적 갱신)
                                     ▼
                                  Context.toResult() → Result
```

| 부품 | 역할 | 패턴 이름 |
|---|---|---|
| Factor | 계산에 필요한 **모든** 입력을 담은 불변 객체. `withXxx()` 복사 메서드로만 변형 | Parameter Object + 불변 값 객체 |
| Policy | 입력 종류(지급구분 등)별로 factor를 변환, 어떤 계산유형을 실행할지 결정 | Strategy (+ Finder가 목록에서 `supports()`로 선택) |
| Pipeline | calculator들을 **고정 순서**로 실행 | Pipes and Filters (POSA) |
| AbstractCalculator | `calculate()`를 `final`로 잠그고 "지원 검사 → `doCalculate()` 위임" 골격 정의 | Template Method (GoF) + supports-execute 관용구 |
| Context | 모든 단계가 돌려 쓰며 결과를 누적하는 가변 중간 상태 | Collecting Parameter (Kent Beck, *Smalltalk Best Practice Patterns*) |

⚠️ **Chain of Responsibility가 아니다**: CoR은 "체인을 따라가다 **누군가 하나가 처리하면 멈추는**" 구조(예: 예외 핸들러 선택). 여기는 **모든 단계가 순서대로 전부 실행**되고 각자 자기 몫만 계산한다 → Pipes and Filters.

## 사용 예시 — 주문 정산 계산

```java
// 1. 실행할 계산 유형 — 단계 on/off 스위치
public enum CalculationType {
    DISCOUNT, SHIPPING, TAX;

    public static final Set<CalculationType> DEFAULT_TYPES = EnumSet.of(SHIPPING, TAX);
    // 프로모션 적용 주문만 DISCOUNT를 추가로 켠다
}

// 2. Factor — 조회 결과까지 전부 담은 불변 입력 (Parameter Object)
@Value(staticConstructor = "of")
public class OrderCalculationFactor {
    Set<CalculationType> calculationTypes; // 실행할 계산 유형
    OrderInput order;                      // 주문 라인·수량·단가
    MemberGrade grade;                     // DB 조회 결과 (서비스가 미리 조회)
    ShippingPolicySnapshot shippingPolicy; // DB 조회 결과

    public boolean hasCalculationType(CalculationType type) {
        return calculationTypes.contains(type);
    }

    public OrderCalculationFactor withCalculationTypes(Set<CalculationType> types) {
        return OrderCalculationFactor.of(types, order, grade, shippingPolicy);
    }
}

// 3. Context — 단계들이 누적 갱신하는 중간 상태 (Collecting Parameter)
public class OrderCalculationContext {
    private final Map<String, Long> amountByItem = new LinkedHashMap<>();
    @Getter private long discountAmt;
    @Getter private long shippingAmt;
    @Getter private long taxAmt;

    public void replaceItemAmt(String itemId, long amt) { amountByItem.put(itemId, amt); }
    public long totalItemAmt() { return amountByItem.values().stream().mapToLong(Long::longValue).sum(); }

    public OrderCalculationResult toResult() {
        return OrderCalculationResult.of(totalItemAmt(), discountAmt, shippingAmt, taxAmt);
    }
}

// 4. 공통 골격 — Template Method + supports-execute
public abstract class AbstractOrderCalculator {
    private final Set<CalculationType> supportedTypes;

    protected AbstractOrderCalculator(CalculationType... types) {
        this.supportedTypes = types.length == 0
                ? Collections.emptySet()
                : Collections.unmodifiableSet(EnumSet.copyOf(Arrays.asList(types)));
    }

    public final void calculate(OrderCalculationFactor factor, OrderCalculationContext ctx) {
        if (!supports(factor)) {
            return; // 유형이 안 켜져 있으면 조용히 통과
        }
        doCalculate(factor, ctx);
    }

    private boolean supports(OrderCalculationFactor factor) {
        return supportedTypes.isEmpty()
                || supportedTypes.stream().anyMatch(factor::hasCalculationType);
    }

    protected abstract void doCalculate(OrderCalculationFactor factor, OrderCalculationContext ctx);
}

// 5. 구체 계산기 — 자기 담당 유형을 생성자에서 선언
public class DiscountCalculator extends AbstractOrderCalculator {
    public DiscountCalculator() { super(CalculationType.DISCOUNT); }

    @Override
    protected void doCalculate(OrderCalculationFactor factor, OrderCalculationContext ctx) {
        // 등급 할인율 적용 — DB 접근 없음, factor와 ctx만 사용
        long discount = Math.round(ctx.totalItemAmt() * factor.getGrade().discountRate());
        ctx.setDiscountAmt(discount);
    }
}

// 6. Pipeline — 실행 순서가 여기 한 곳에 스펙으로 고정된다
public class OrderCalculationPipeline {
    private final DiscountCalculator discountCalculator = new DiscountCalculator();
    private final ShippingCalculator shippingCalculator = new ShippingCalculator();
    private final TaxCalculator taxCalculator = new TaxCalculator();

    public OrderCalculationResult execute(OrderCalculationFactor factor) {
        OrderCalculationContext ctx = OrderCalculationContext.from(factor);
        discountCalculator.calculate(factor, ctx); // 할인 먼저 (과세 기준 금액 확정)
        shippingCalculator.calculate(factor, ctx);
        taxCalculator.calculate(factor, ctx);      // 확정된 금액 위에서 세금
        return ctx.toResult();
    }
}
```

테스트는 단계별로 순수하게 가능하다 — DB도 Mock도 없이:

```java
@Test
void 할인_유형이_없으면_금액을_건드리지_않는다() {
    var factor = factor(Set.of());               // DISCOUNT 안 켬
    var ctx = context(factor);

    new DiscountCalculator().calculate(factor, ctx);

    assertThat(ctx.getDiscountAmt()).isZero();   // 가드 동작 자체를 검증
}
```

## 어디서 또 보나 — supports-execute 관용구

"내가 이 입력을 처리할 수 있나?"를 먼저 묻고 실행하는 2단 인터페이스는 Spring 전반에 깔려 있다:

| Spring 인터페이스 | supports | execute |
|---|---|---|
| `Validator` | `supports(Class)` | `validate(target, errors)` |
| `HandlerMethodArgumentResolver` | `supportsParameter()` | `resolveArgument()` |
| `HandlerAdapter` | `supports(handler)` | `handle(...)` |

이 관용구의 등록 방식이 두 갈래인 것도 기억해둘 것 — **목록에서 첫 매치를 고르는 쪽**(PolicyFinder처럼 → Strategy 선택)과 **전부 순회하며 매치되는 것마다 실행하는 쪽**(파이프라인의 supports 가드처럼).

## ⚠️ 함정 / 메커니즘

- **단계 간 의존이 Context 뒤에 숨는다.** B단계가 A단계가 써둔 값을 읽는데, 시그니처엔 안 드러난다. 순서를 바꾸면 컴파일은 되고 **값만 조용히 틀어진다.** → 순서를 Pipeline 클래스 한 곳에만 두고, "왜 이 순서인지"를 주석/커밋에 남긴다.
- **supports 스킵은 무음(無音)이다.** 유형 플래그가 빠지면 계산기가 아무 일도 안 하고 통과하는데, 에러가 없으니 "왜 금액이 안 바뀌지?"로 디버깅하게 된다. → "유형이 없으면 건드리지 않는다"는 가드 동작 자체를 테스트로 박아둔다.
- **Context는 스레드 안전하지 않다.** 요청당 1회용으로 만들고 버릴 것. 공유 빈에 담으면 안 된다.
- **Factor 필드 증식.** 입력을 다 담다 보면 `of(...)` 인자가 15개까지 가고, `withXxx()` 복사 메서드마다 전체 필드를 나열하게 된다. Lombok `@With`나 필드 그룹화(하위 값 객체로 묶기)로 완화.
- **조회를 계산 안으로 들이는 순간 무너진다.** 계산기 안에서 mapper/repository를 부르기 시작하면 "순수 계산 + 단위 테스트 가능"이라는 존재 이유가 사라진다. 부족한 입력은 Factor에 추가하는 게 맞다.
- **enum 스위치의 case 누락.** 유형별 `switch`에서 새 enum 값의 case를 빠뜨리면 default로 떨어져 조용히 0/무시가 된다. 컴파일러가 못 잡아주는 자리(👉 default에서 예외를 던지거나, 유형별 테스트를 enum 전수로).

## 💡 판단 기준 (관점)

구체 케이스: 급여 계산 로직 V1은 mutable DTO 하나를 서비스와 유틸 빈들이 setter로 돌려쓰고, 계산 도중에도 DB를 조회했다 → 단계 순서가 코드 곳곳에 암묵적으로 흩어지고, 단위 테스트가 사실상 불가능했다. V2에서 이 구조(불변 Factor + 정책 + 파이프라인 + Context)로 재작성하니 계산기마다 순수 단위 테스트가 붙고, "근태 반영" 같은 옵션이 유형 Set 하나로 표현됐다.

- **도입 신호**: "계산 단계 3개 이상 + 순서 의존 + 일부 단계가 옵션 + 조회가 무겁다"가 겹치면 이 구조. 하나라도 빠지면 통합 서비스 메서드부터.
- **핵심 규율은 구조보다 경계**: "조회는 서비스에서 다 끝내고, 계산 코어에는 I/O를 들이지 않는다." 이 경계만 지켜도 절반은 성공 — 파이프라인/정책은 그 다음 문제다.
- **순서가 스펙이면, 순서를 한 곳에 모아라.** 이 구조의 최대 리스크(숨은 순서 의존)는 Pipeline 클래스 하나에 순서를 못박는 것으로 관리한다.

관련 노트: [변환 계층 (Factory/Mapper/Assembler + Command)](./transform-layers.md) — Factor를 만드는 쪽(FactorFactory)의 이야기. [도메인 검증 위치](./domain-validation.md) — "규칙=도메인·조회=인프라" 경계와 같은 원칙.

## 참고

- GoF, *Design Patterns* — Template Method, Strategy, Chain of Responsibility
- POSA Vol.1 / EIP — Pipes and Filters: https://www.enterpriseintegrationpatterns.com/patterns/messaging/PipesAndFilters.html
- Kent Beck, *Smalltalk Best Practice Patterns* — Collecting Parameter (c2 wiki: https://wiki.c2.com/?CollectingParameter)
- Martin Fowler, *Refactoring* — Introduce Parameter Object: https://refactoring.com/catalog/introduceParameterObject.html
- Spring `Validator` javadoc (supports/validate 2단 구조): https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/validation/Validator.html

---
학습 날짜: 2026-08-04
계기: 회사 급여 계산 V2 모듈 코드 파악 중 "이 구조를 뭐라고 부르나" 정리. V1(mutable DTO + 서비스 계산) → V2(불변 Factor + 파이프라인) 재작성 비교에서 도입 신호를 뽑음.

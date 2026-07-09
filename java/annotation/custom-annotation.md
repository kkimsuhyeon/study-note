# 커스텀 어노테이션 - @interface · 메타 어노테이션 · AOP 처리

> **한 줄 요약**: 어노테이션은 그 자체로는 아무것도 안 하는 **마킹(메타데이터)**이고, 동작은 그걸 읽는 **처리기**(리플렉션/AOP/컴파일러)가 만든다. 핵심 함정: `@Retention(RUNTIME)`이 아니면 런타임에 안 보이고, Spring AOP 처리라면 [프록시 함정](../spring/transactional.md)을 그대로 물려받는다.

## 언제 쓰나

- **횡단 관심사(cross-cutting concern)를 선언적으로 켜고 싶을 때**: 분산 락, 실행 시간 로깅, 권한 체크, 재시도, 캐싱, rate limit 등 — "비즈니스 코드에 보일러플레이트를 섞고 싶지 않다"가 신호
- **메타데이터 마킹**: 직렬화 필드 지정(Jackson), 테스트 그룹핑(`@Tag`), 검증 규칙(Bean Validation 커스텀 제약)
- 스프링이 제공하는 `@Transactional`, `@Cacheable`이 전부 이 패턴이다 — 우리가 만드는 것도 같은 구조

## 사용 예시 (선언 문법)

```java
@Target(ElementType.METHOD)          // 어디에 붙일 수 있나
@Retention(RetentionPolicy.RUNTIME)  // 언제까지 살아있나 ← AOP/리플렉션 처리면 필수
public @interface DistributedLock {
    String key();                              // 필수 요소 (default 없음)
    String value() default "";                 // 선택 요소 (default 있음)
    long waitTime() default 5L;
    long leaseTime() default 10L;
    TimeUnit timeUnit() default TimeUnit.SECONDS;
}
```

- 요소(element)는 메서드 선언 형태로 정의. **허용 타입 제한**: 기본형, `String`, `Class`, enum, 어노테이션, 그리고 이들의 1차원 배열만. (일반 객체·`null` 불가 — "값 없음"은 `""`나 빈 배열 같은 센티널로 표현)
- 요소가 `value` 하나뿐이면 `@Anno("x")`처럼 이름 생략 가능
- 붙일 때 상수만 사용 가능 (컴파일 타임에 값이 박제됨) — 동적 값이 필요하면 **SpEL 문자열**로 받고 처리기에서 평가하는 게 관용 패턴 (아래 AOP 절)

## 메타 어노테이션 (어노테이션에 붙이는 어노테이션)

| 메타 어노테이션 | 역할 | 비고 |
|---|---|---|
| `@Target` | 부착 위치 제한 (`METHOD`, `TYPE`, `FIELD`, `PARAMETER`...) | 생략하면 거의 모든 곳 허용 |
| `@Retention` | 유지 범위 | `SOURCE`(컴파일 후 소멸, Lombok류) / `CLASS`(기본값, class파일까지·런타임 리플렉션 불가) / `RUNTIME`(리플렉션으로 읽기 가능) |
| `@Inherited` | **클래스에 붙인** 어노테이션을 자식 클래스가 상속 | 메서드/인터페이스엔 효과 없음 ⚠️ |
| `@Repeatable` | 같은 어노테이션 여러 번 부착 허용 | 컨테이너 어노테이션 필요 |
| `@Documented` | Javadoc에 표시 | |

⚠️ `@Retention` **기본값은 `RUNTIME`이 아니라 `CLASS`** — 안 붙이고 리플렉션/AOP로 읽으면 "어노테이션이 없는 것처럼" 동작한다. 조용히 무시되므로 원인 찾기 어려운 유형.

## 처리 방식 3가지 (어노테이션은 처리기가 있어야 동작한다)

| 방식 | 시점 | 예 | 특징 |
|---|---|---|---|
| ① 리플렉션 직접 | 런타임 | `method.getAnnotation(X.class)` | 프레임워크 없이 가능, 호출 지점을 직접 제어해야 함 |
| ② Spring AOP | 런타임(프록시) | `@Around("@annotation(x)")` | 선언만 하면 자동 적용. 프록시 제약 있음 |
| ③ 컴파일 타임 | 컴파일 | Lombok, MapStruct, APT(Annotation Processor) | 코드 생성. 런타임 비용 0, 작성 난이도 높음 |

실무에서 "메서드에 붙여서 부가 동작"은 대부분 ②.

## Spring AOP로 처리하기 (핵심 패턴)

```java
@Aspect
@Order(Ordered.HIGHEST_PRECEDENCE + 1)   // @Transactional(기본 LOWEST)보다 먼저 → 락 안에서 트랜잭션 시작/커밋
@Component
@RequiredArgsConstructor
public class DistributedLockAdvice {
    private final DistributedLockTemplate lockTemplate;

    @Around("@annotation(distributedLock)")   // 파라미터 바인딩: 어노테이션 인스턴스를 직접 받는다
    public Object around(ProceedingJoinPoint joinPoint, DistributedLock distributedLock) throws Throwable {
        String lockKey = buildLockKey(distributedLock, joinPoint);  // "key:value" 조합
        return lockTemplate.executeWithThrowable(lockKey,
                distributedLock.waitTime(), distributedLock.leaseTime(), distributedLock.timeUnit(),
                joinPoint::proceed);
    }
}
```

사용하는 쪽은 선언만:

```java
@Transactional
@DistributedLock(key = "empl_lock", value = "#param.orgId", waitTime = 3L, leaseTime = 120L)
public String addEmpl(EmplAdd param) { ... }
```

### 동적 값은 SpEL로

어노테이션 요소는 상수만 가능하므로, `"#param.orgId"` 같은 **SpEL 문자열**을 받아 Aspect에서 평가한다:

```java
Expression expr = parser.parseExpression(spelString);          // 파싱은 비싸므로 ConcurrentHashMap에 캐시
EvaluationContext context = new StandardEvaluationContext();   // 컨텍스트는 매 호출 새로 (스레드 안전)
String[] paramNames = ((MethodSignature) joinPoint.getSignature()).getParameterNames();
Object[] args = joinPoint.getArgs();
for (int i = 0; i < paramNames.length; i++) context.setVariable(paramNames[i], args[i]);
String value = expr.getValue(context, String.class);           // #param.orgId → 실제 orgId
```

## 함정 / 메커니즘 (⚠️)

- ⚠️ **프록시 기반이라 self-invocation에 안 먹힌다**: 같은 클래스 안에서 `this.addEmpl(...)`로 호출하면 프록시를 안 거치므로 락이 안 걸린다. `@Transactional`과 동일한 메커니즘 — 상세는 [@Transactional 노트](../spring/transactional.md)의 프록시 함정 참고. private 메서드도 같은 이유로 불가.
- ⚠️ **`getParameterNames()`가 null일 수 있다**: 파라미터 이름은 기본적으로 바이트코드에 안 남는다. `-parameters` 컴파일 옵션(스프링 부트 그래들/메이븐 플러그인은 기본 켜줌)이 없으면 SpEL `#파라미터명` 바인딩이 깨진다. 빌드 환경 바뀌고 나서야 터지는 유형.
- ⚠️ **Aspect 간 순서는 `@Order`로 명시**: 락과 트랜잭션이 붙은 메서드라면 "락 획득 → 트랜잭션 시작 → 커밋 → 락 해제" 순서여야 정합성이 보장된다. `@Order`를 안 주면 트랜잭션이 락 밖에서 커밋될 수 있다(락 해제 후 커밋 전 틈에 다른 요청이 옛 데이터를 읽음). `@Transactional`의 기본 order는 `LOWEST_PRECEDENCE`이므로 그보다 작은 값을 주면 먼저 실행된다.
- ⚠️ **인터페이스 메서드에 붙인 어노테이션은 구현체로 상속되지 않는다**: `@Inherited`는 클래스 상속에만 적용. 서비스 인터페이스가 있는 구조면 **구현 클래스 메서드**에 붙여야 안전 (스프링 공식 문서도 `@Transactional`을 구체 클래스에 붙이라고 권고).
- ⚠️ 붙이기만 하고 처리기(Aspect 빈)가 등록 안 되면 **조용히 무시된다** — 어노테이션은 컴파일 에러를 못 낸다. 동작 테스트(락이 실제로 잡히는지)로 확인해야 한다.

## 💡 판단 기준

- **케이스**: 사원 등록(`addEmpl`)과 데이터 마이그레이션의 사원 일괄 등록이 같은 기관에 동시 실행되면 사번 채번이 중복될 수 있었다. try-lock/해제 보일러플레이트를 메서드마다 쓰는 대신, `@DistributedLock(key=..., value="#param.orgId")` 어노테이션 + AOP로 선언화하고 `@Order`로 트랜잭션보다 먼저 잡게 했다.
- **판단**: 같은 부가 동작이 **여러 메서드에 반복 + "트랜잭션보다 먼저" 같은 순서 제어가 필요**하면 커스텀 어노테이션 + AOP. 반대로 **한두 곳**이면 템플릿 직접 호출(`lockTemplate.execute(key, () -> ...)`)이 더 단순하고 프록시 함정도 없다 — "어노테이션부터 만들고 보자"는 과설계 신호.
- 분산 락 자체(왜 Redis 락인지, synchronized로 안 되는 이유)는 [락 개념 종합](../concurrency/locks.md) 참고.

## 참고

- [JLS §9.6 Annotation Interfaces](https://docs.oracle.com/javase/specs/jls/se17/html/jls-9.html#jls-9.6) — 요소 허용 타입, 기본 retention
- [Java Tutorial - Annotations](https://docs.oracle.com/javase/tutorial/java/annotations/)
- [Spring AOP - @annotation pointcut & 파라미터 바인딩](https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/pointcuts.html)
- [Spring - AOP 프록시 이해 (self-invocation)](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)

---
학습 날짜: 2026-07-09
계기: 회사 프로젝트에서 `addEmpl`에 붙은 `@DistributedLock`(Redisson 분산 락 커스텀 어노테이션)의 동작 원리를 분석하면서 — 어노테이션 선언·SpEL 키 생성·`@Order`로 트랜잭션과의 순서 제어까지 한 세트로 정리

# OSIV(Open Session In View) — 커넥션을 언제까지 쥐고 있을 것인가

> **한 줄 요약**: OSIV는 **영속성 컨텍스트(와 DB 커넥션 리소스)를 트랜잭션 밖 — API 응답/뷰 렌더링이 끝날 때까지 — 열어두는 전략**이다. 켜면(기본값) 컨트롤러·뷰에서도 지연 로딩이 되는 대신 요청 내내 커넥션을 소비하고, 끄면 커넥션은 아끼지만 모든 지연 로딩을 트랜잭션 안에서 끝내야 한다.

관련 노트: [영속성 컨텍스트·flush](./persistence-context.md) · [프록시와 지연 로딩](./proxy.md) · [@Transactional](../spring/transactional.md) · [락 개념 종합](../concurrency/locks.md)

---

## 0. 시작 로그의 warn, 그 정체

스프링 부트 앱을 띄우면 이런 경고가 뜬다:

```
spring.jpa.open-in-view is enabled by default. Therefore, database queries
may be performed during view rendering. Explicitly configure
spring.jpa.open-in-view to disable this warning
```

- `spring.jpa.open-in-view`: **기본값 true**. 기본값인데도 굳이 warn을 찍는 건, **이 설정이 장애로 이어질 수 있는 트레이드오프**라서 "알고 켜라"는 뜻이다.
- 이름 유래: Open **Session** In View(하이버네이트) / Open **EntityManager** In View(JPA). 관례상 OSIV라 부른다.

---

## 1. OSIV ON — 영속성 컨텍스트를 응답 끝까지 유지

```
요청 → [Filter/Interceptor] → Controller → Service(@Transactional) → Repository
        ├──────────────── 영속성 컨텍스트 생존 범위 ────────────────┤
                                          ├──── 트랜잭션 범위 ────┤
```

- **최초 DB 커넥션을 얻는 시점부터 API 응답이 끝날 때까지** 영속성 컨텍스트와 커넥션 리소스를 유지한다.
- 그래서 **트랜잭션이 끝난 컨트롤러/View Template에서도 지연 로딩이 동작**한다. 지연 로딩은 영속성 컨텍스트가 살아 있어야 가능하고, 영속성 컨텍스트는 기본적으로 커넥션을 필요로 하기 때문. 이것 자체가 큰 장점.
- 단, 트랜잭션 밖에서는 **영속 상태여도 "조회만" 가능** — 값을 바꿔도 flush할 트랜잭션이 없으므로 변경 감지는 트랜잭션 범위 안에서만 의미가 있다. (flush 메커니즘 → [persistence-context.md](./persistence-context.md))

### 대가: 커넥션 점유 시간 = 요청 처리 시간 전체

- 이 전략은 **너무 오랜 시간 커넥션 리소스를 사용**한다. 예: 컨트롤러에서 외부 API를 3초 호출하면, **그 3초 동안 커넥션을 반환하지 못하고 유지**해야 한다.
- 실시간 트래픽이 중요한 애플리케이션에서는 **커넥션이 모자라게 되고, 이는 결국 장애로 이어진다.**
- 커넥션 풀 고갈이 "무관한 기능까지 마비"시키는 메커니즘은 기존 노트에 정리돼 있음 → [transactional.md의 커넥션 고갈](../spring/transactional.md) · [locks.md의 커넥션 풀 고갈](../concurrency/locks.md)

> 참고(세부): Hibernate 5.2+ 기본 커넥션 해제 모드(`DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION`)에서는 트랜잭션이 끝나면 커넥션을 일단 반환하고, 뷰에서 지연 로딩이 일어나면 다시 획득하는 동작도 있다(설정 따라 다름 — 세부는 확인 필요). 어느 쪽이든 **"응답이 끝날 때까지 요청이 커넥션 리소스를 계속 소비할 수 있는 구조"** 라는 본질과 고갈 위험은 같다.

---

## 2. OSIV OFF — 트랜잭션 종료 = 영속성 컨텍스트 종료

```yaml
spring:
  jpa:
    open-in-view: false
```

- 트랜잭션을 종료할 때 **영속성 컨텍스트를 닫고 커넥션도 반환**한다. 커넥션 리소스를 낭비하지 않는다.
- 대신 **모든 지연 로딩을 트랜잭션 안에서 끝내야 한다.** 트랜잭션 밖(컨트롤러/뷰)에서 프록시를 건드리면 `LazyInitializationException` → [proxy.md](./proxy.md)
- 기존에 컨트롤러/뷰에서 하던 지연 로딩 코드를 전부 트랜잭션 안으로 넣어야 하는 단점이 있다. 결국 **트랜잭션이 끝나기 전에 지연 로딩을 강제로 호출해 둬야** 한다.

### 대응 패턴: 커맨드와 쿼리 분리 (Command-Query Separation)

OSIV를 끈 상태로 복잡성을 관리하는 실무 방법 — 화면용 지연 로딩을 **읽기 전용 트랜잭션 안으로** 옮긴다:

```java
// 핵심 비즈니스 로직 (커맨드)
@Service @Transactional
public class OrderService { ... }

// 화면/API 맞춤 조회 (쿼리) — 지연 로딩을 여기(트랜잭션 안)서 끝낸다
@Service @Transactional(readOnly = true)
public class OrderQueryService {
    public List<OrderDto> ordersV3() {
        return orderRepository.findAllWithItem().stream()
                .map(OrderDto::new)   // DTO 변환하며 지연 로딩 전부 발생
                .toList();
    }
}
```

- 보통 비즈니스 로직(커맨드)은 엔티티 몇 개 등록·수정이라 성능 문제가 크지 않다. 반면 **복잡한 화면용 쿼리는 성능 최적화가 중요**하지만 핵심 비즈니스엔 영향이 적다 → 크고 복잡한 앱일수록 이 둘의 관심사 분리가 유지보수 관점에서 의미 있다.
- 두 서비스 모두 트랜잭션을 유지하므로 지연 로딩을 자유롭게 쓸 수 있다.

---

## 3. ON vs OFF 비교

| | OSIV ON (기본) | OSIV OFF |
|---|---|---|
| 영속성 컨텍스트 생존 범위 | 요청 시작 ~ **응답 완료** | **트랜잭션 범위**와 동일 |
| 커넥션 점유 | 최초 획득 ~ 응답 완료 (길다) | 트랜잭션 종료 시 반환 (짧다) |
| 컨트롤러/뷰에서 지연 로딩 | 가능 | **불가** (`LazyInitializationException`) |
| 코드 편의성 | 높음 (아무 데서나 `.get~()`) | 트랜잭션 안에서 로딩 완료 필요 (쿼리 서비스 분리 등) |
| 트래픽 많은 실시간 API | **커넥션 고갈 위험** | 안전 |

---

## 4. ⚠️ 함정

- **"기본값이니 안전하겠지"가 함정.** 기본 ON은 편의를 위한 것이고, 스프링 스스로 warn을 찍을 만큼 대가가 있다. 켜둔 채 컨트롤러에서 외부 API·파일 IO 같은 느린 작업을 하면 그 시간만큼 커넥션을 못 돌려준다.
- **OSIV OFF로 바꾸는 순간 기존 코드가 터진다.** 컨트롤러/뷰 여기저기서 잘 돌던 지연 로딩이 전부 `LazyInitializationException` 후보가 된다. 끄는 건 설정 한 줄이지만, 지연 로딩을 트랜잭션 안으로 옮기는 리팩토링이 본체다.
- **OSIV ON이어도 트랜잭션 밖에선 수정 불가.** "영속 상태 = 변경 감지"로 착각하기 쉬운데, flush를 일으킬 트랜잭션이 없으면 setter는 그냥 자바 객체 변경일 뿐이다.

---

## 5. 💡 판단 기준

> **고객 서비스의 실시간 API는 OSIV를 끄고, ADMIN처럼 커넥션을 많이 사용하지 않는 곳에서는 OSIV를 켠다.** (김영한 기준 그대로)

- 판단 축은 **"커넥션 여유가 있는가"** 하나다. 트래픽이 몰리는 외부 노출 API → OFF + 쿼리 서비스 분리. 내부 관리자 화면·배치성 조회 → ON의 편의를 그냥 누린다.
- OFF로 갈 때의 비용(지연 로딩 재배치)은 **커맨드/쿼리 서비스 분리**로 흡수하는 게 정석이다.

---

## 6. 참고

- 실전! 스프링 부트와 JPA 활용2 — 5장 "OSIV와 성능 최적화"
- 자바 ORM 표준 JPA 프로그래밍 13장 "웹 애플리케이션과 영속성 관리" (OSIV 심화)
- [Baeldung - A Guide to Spring's Open Session in View](https://www.baeldung.com/spring-open-session-in-view)
- [Spring Boot - JPA properties (`spring.jpa.open-in-view`)](https://docs.spring.io/spring-boot/appendix/application-properties/index.html)

---

**학습 날짜**: 2026-08-12
**계기**: 김영한 JPA 활용2 5장 수강 — 앱 시작 때마다 보던 open-in-view warn 로그의 정체와, 지연 로딩 편의 뒤에 커넥션 고갈 트레이드오프가 있음을 정리

# Spring Cache — @Cacheable 3형제와 저장소 추상화

> **한 줄 요약**: Spring Cache는 **어노테이션(정책: 언제 캐시하나)**과 **CacheManager(저장소: 어디에 넣나)**를 분리한 추상화. `@Cacheable`=없으면 실행 후 저장, `@CachePut`=무조건 실행 후 덮어쓰기, `@CacheEvict`=삭제. 핵심 함정: **`@EnableCaching` 없으면 에러도 없이 조용히 무시**되고, 프록시 기반이라 자기 호출은 캐시를 우회한다.

## 언제 쓰나

- 자주 안 바뀌는데 비싼 조회 — 외부 API 응답, 참조 설정, 공통코드 등
- 손수 `Map` 필드 + 만료 체크로 캐시를 만들고 싶어질 때 → TTL·동시성·멀티 인스턴스 동기화를 직접 푸는 대신 선언 한 줄

## 사용 예시 — 캐시 전용 컴포넌트 패턴

```java
@Component
@RequiredArgsConstructor
public class ReferenceConfigCache {
    public static final String CACHE_NAME = "referenceConfigs";
    private static final String CACHE_KEY = "'all'";      // SpEL 문자열 리터럴 (따옴표 이중!)

    private final ExternalApiRequester requester;

    @Cacheable(value = CACHE_NAME, key = CACHE_KEY)       // 캐시에 있으면 본문 실행 안 함
    public List<ConfigResponse> getAll() {
        return requester.getConfigs();                     // miss일 때만 실행 → 결과 저장
    }

    @CachePut(value = CACHE_NAME, key = CACHE_KEY)        // 무조건 실행 + 캐시 덮어쓰기
    public List<ConfigResponse> refresh() {
        return requester.getConfigs();
    }

    @CacheEvict(value = CACHE_NAME, key = CACHE_KEY)      // 캐시 삭제
    public void evict() { }                                // 본문이 빈 이유: 어노테이션 부수효과가 목적
}
```

- `value` = 캐시 이름(네임스페이스), `key` = 항목 키. **key는 SpEL** — 고정 문자열은 `"'all'"`처럼 안에 따옴표, 파라미터 키는 `"#orgId"`, 조합은 `"#orgId + ':' + #usrId"`
- 저장 키 실제 모양(Redis): `referenceConfigs::all`

## 세 어노테이션 비교

| 어노테이션 | 캐시 확인 | 본문 실행 | 캐시 반영 | 용도 |
|---|---|---|---|---|
| `@Cacheable` | O (있으면 반환하고 끝) | miss일 때만 | miss 시 저장 | 읽기 (read-through) |
| `@CachePut` | X | **항상** | 결과로 덮어쓰기 | 강제 갱신 |
| `@CacheEvict` | X | 실행됨(주로 빈 본문) | 삭제 | 무효화 |

## 두 층 구조 — 필수 스위치와 선택 설정

```
@Cacheable만                → 아무 일도 안 일어남 (조용히 무시 — 함정!)
+ @EnableCaching            → 동작. 저장소는 Boot가 클래스패스 보고 자동 선택
+ CacheManager 빈 직접 정의  → 저장소·TTL·직렬화를 내 뜻대로
```

- **`@EnableCaching`이 프록시를 만드는 스위치** — 없으면 어노테이션은 장식. 에러가 안 나서 "캐시 붙였는데 왜 매번 나가지?"로 헤매는 단골 함정
- CacheManager 자동 선택: 캐시 라이브러리 없음 → `ConcurrentMapCacheManager` / Caffeine 의존성 → Caffeine / Redis 의존성 → `RedisCacheManager`
- **커스텀 설정이 필요한 이유**: Boot 자동구성 RedisCacheManager는 **TTL 없음(영원히)** + **JDK 직렬화**(바이너리, 클래스 버전 취약)가 기본 → `entryTtl(Duration.ofMinutes(30))` + `GenericJackson2JsonRedisSerializer`(JSON)로 교체하는 식

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .serializeKeysWith(...new StringRedisSerializer())
                .serializeValuesWith(...new GenericJackson2JsonRedisSerializer())
                .entryTtl(Duration.ofMinutes(30L));
        return RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(factory)
                .cacheDefaults(config).build();
    }
}
```

## 저장소 3종 비교 — "인스턴스별 사본 허용?"

| | ConcurrentMap (기본) | Caffeine | Redis |
|---|---|---|---|
| 저장 위치 | JVM 힙 | JVM 힙 | 별도 서버 (중앙 1개) |
| TTL / 최대크기 | ❌ / ❌ (누수 위험) | ✅ / ✅ (TinyLFU 퇴출) | ✅ / Redis 정책 |
| 속도 | 최고 | 최고 | 네트워크 왕복 |
| 인스턴스 간 공유 | ❌ 각자 사본 | ❌ 각자 사본 | ✅ 전체 공유 |
| 용도 | 개발·테스트 | 사본 허용되는 데이터 | 스케일아웃 공유 캐시 |

**"인스턴스별 사본 허용"의 뜻**: 로컬 캐시는 JVM마다 하나씩 생기므로 서버 3대 = 캐시 3개. ① 각자 따로 채우고(원본 호출 최대 3번) ② 사본끼리 어긋날 수 있다 — A는 TTL 만료로 v2를 받았는데 B는 20분간 v1 유지 → 같은 사용자가 새로고침마다 다른 값을 볼 수 있음. 이걸 업무가 견디면 "허용".

- 허용 예: 참조 설정, 공통코드 (30분 stale 무해) → Caffeine으로 충분, 더 빠름
- 불허 예: 재고, 차단/권한, 결제 상태 ("A서버선 차단, B서버선 통과"는 사고) → Redis 또는 캐시 금지
- **evict 일관성이 결정타**: 로컬 캐시의 `@CacheEvict`는 **요청받은 인스턴스 한 대만** 지워짐. Redis는 중앙에 원본 하나라 evict 한 번 = 전체 즉시 반영
- Caffeine = Guava Cache 후속의 로컬 캐시 라이브러리. `spring.cache.caffeine.spec: maximumSize=500,expireAfterWrite=30m` 한 줄로 붙음
- 대규모에선 2단 조합도 씀: 1차 Caffeine(초고속) → miss 시 2차 Redis → miss 시 원본

## ⚠️ 함정/메커니즘

- **동작 원리 = AOP 프록시** ([@Transactional](./transactional.md)과 동일 메커니즘): 빈을 프록시로 감싸 밖에서 오는 호출을 가로챔 → **자기 호출(`this.getAll()`)은 프록시 우회 = 캐시 미동작**. 해법이 위의 **캐시 전용 컴포넌트 분리 패턴** — 서비스가 주입받아 호출하면 반드시 프록시를 거침
- `@EnableCaching` 누락 = 조용한 무시 (위)
- **캐시에 넣는 객체는 직렬화 가능해야** — Redis+JSON이면 Jackson 역직렬화 가능(기본 생성자 등), JDK 직렬화면 `Serializable`
- `@Cacheable` 메서드가 `null` 반환 시 **null도 캐시됨** (기본) — `unless = "#result == null"`로 제외 가능
- 같은 이름 다른 TTL이 필요하면 캐시 이름별 설정(`withCacheConfiguration`)으로 분리

## 💡 판단 기준

- 코드리뷰에서 "@Cacheable 쓰라"는 지적을 받고 정리 — 손수 Map 캐시를 만들려던 상황: **"이 메서드 결과를 캐시해줘"는 선언(어노테이션)으로, 어디에·얼마나는 설정(CacheManager)으로** 분리하는 게 Spring Cache의 본질. 본문 로직은 캐시를 모른 채 순수하게 유지된다
- 저장소 선택 질문은 하나: **"서버 여러 대가 같은 캐시를 봐야 하는가?"** = "이 데이터, 서버마다 잠깐 달라도 되나?" — 돼야 하면 Redis, 아니면 Caffeine이 더 빠르다
- 캐시가 안 먹는 것 같으면 순서대로 의심: ① `@EnableCaching` 있나 ② 자기 호출 아닌가 ③ 프록시 거치는 public 메서드인가

## 참고

- Spring Framework 공식 문서 — Cache Abstraction: https://docs.spring.io/spring-framework/reference/integration/cache.html
- Spring Boot 공식 문서 — Caching (자동구성·provider 순서): https://docs.spring.io/spring-boot/reference/io/caching.html
- Caffeine GitHub: https://github.com/ben-manes/caffeine
- 관련 노트: [Redis 기초](../../infra/redis/redis-basics.md), [스케일 아웃](../../infra/scaling.md), [@Transactional 프록시](./transactional.md)
- 학습 날짜: 2026-08-03
- 계기: 실무 코드의 `~Cache` 전용 컴포넌트(@Cacheable/@CachePut/@CacheEvict + Redis 30분 TTL)를 따라가며, "Redis에 진짜 넣는 건가?"부터 자동구성·로컬 캐시 사본 문제까지 정리

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

## 캐시 항목의 모양 — 무엇이 key, 무엇이 value인가

```
@Cacheable(value = "summary", key = "#orgId")
           └ 캐시 이름        └ SpEL → "평가 결과"가 키
                         ↓
Redis:   summary::101   →   "{...return 값 JSON...}"
         (캐시 이름 + "::" + key)    (return 값 하나만)
```

- **key = 캐시 이름 + key 평가 결과.** `key`에 적은 문자열이 그대로 들어가는 게 아니라 **SpEL을 평가한 값**이 들어간다 — `"#orgId"`면 문자 `#orgId`가 아니라 `101`. 구분자 `::`는 Redis 쪽 기본 규칙(`CacheKeyPrefix.simple()`, `computePrefixWith`로 변경 가능)
- **value = return 값 하나뿐.** 프록시가 볼 수 있는 건 **들어오는 파라미터(키 재료)와 나가는 반환값(저장 대상)**뿐이라, 메서드 안에서 한 조회·API 호출은 저장되지 않는다. hit이면 본문 자체가 실행되지 않으니 안쪽 조회를 저장할 이유도 없다

```java
@Cacheable(value = "summary", key = "#orgId")
public SummaryDto getSummary(Long orgId) {
    var a = repoA.findAll(orgId);    // 캐시 X
    var b = apiClient.fetch(orgId);  // 캐시 X
    return calc(a, b);               // ← 이 결과만 summary::101 에 저장
}
```

- 안쪽에서 **다른 빈의 `@Cacheable` 메서드**를 부르면 그 호출은 그쪽 프록시를 거쳐 **따로** 캐시된다 (같은 클래스 메서드면 자기 호출 → 캐시 X)
- JPA 1차 캐시(영속성 컨텍스트)는 트랜잭션 범위의 별개 장치 — Spring Cache와 무관 ([영속성 컨텍스트](../jpa/persistence-context.md))

### 조회 흐름

```
@Cacheable 호출
① key 계산 (SpEL, 없으면 파라미터로 자동 생성)
② cache.get(key)
   ├─ hit  → 저장된 값 반환, 메서드 실행 X
   └─ miss → 메서드 실행 → cache.put(key, 결과) → 결과 반환
```

`@CachePut`·`@CacheEvict`도 **같은 캐시 이름 + 같은 key**를 계산해야 같은 항목을 덮어쓰거나 지운다 — 위 예시의 세 메서드가 `CACHE_KEY` 상수를 공유하는 이유.

### key를 안 적으면 — `SimpleKeyGenerator`

| 파라미터 | 생성되는 키 | Redis에서 보이는 모양 |
|---|---|---|
| 없음 | `SimpleKey.EMPTY` | `이름::SimpleKey []` |
| 1개 (null·배열 아님) | 그 값 자체 | `이름::101` |
| 여러 개 (또는 null·배열 1개) | `new SimpleKey(a, b)` | `이름::SimpleKey [a, b]` |

**메서드 이름은 키에 들어가지 않는다** — 파라미터만 본다 (→ ⚠️ 키 충돌).

### 저장소별로 value가 들어가는 방식

| | ConcurrentMap / Caffeine (JVM) | Redis |
|---|---|---|
| 구조 | 캐시 이름마다 `Map<key, 객체>` | 항목 하나 = Redis String 키 하나 (TTL 있으면 만료 함께 설정) |
| 저장하는 것 | **객체 참조 그대로** (ConcurrentMap `storeByValue` 기본 `false`) | **직렬화한 바이트** (JSON 등) |
| 꺼낼 때 | 매번 **같은 인스턴스** | 매번 역직렬화한 **새 객체** |
| 키 비교 | `equals`/`hashCode` | 문자열로 변환한 키 |

```
> SCAN 0 MATCH referenceConfigs::*      # 운영 Redis에선 KEYS 대신 SCAN
> GET referenceConfigs::all
"[\"java.util.ArrayList\",[{\"@class\":\"com.example.ConfigResponse\", ...}]]"
> TTL referenceConfigs::all
(integer) 1742
```

`@class`·`"java.util.ArrayList"`는 `GenericJackson2JsonRedisSerializer`가 **역직렬화할 타입을 알기 위해** 같이 넣는 타입 정보.

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
- **`@Cacheable`의 `value`는 저장되는 값이 아니다** — `cacheNames`의 별칭(= 캐시 이름). 저장되는 건 return 값. 헷갈리면 `cacheNames = ...`로 쓰는 게 명확
- **로컬 캐시(ConcurrentMap·Caffeine)는 참조를 저장한다** — 꺼낸 `List`에 `add()`·`set()`을 하면 **캐시 안의 원본이 바뀌어** 다음 호출자 전부가 오염된 값을 받는다. Redis는 매번 새 복사본이라 이 문제가 안 드러나서, **개발(ConcurrentMap)과 운영(Redis)의 동작이 달라진다.** 캐시 반환값은 읽기 전용으로 다루고(불변 컬렉션 `List.copyOf`·record), 꼭 필요하면 `ConcurrentMapCacheManager.setStoreByValue(true)`로 복사 저장(직렬화 필요)
- **파라미터 없는 메서드끼리 같은 캐시 이름을 쓰면 키 충돌** — 둘 다 키가 `SimpleKey.EMPTY`라 한 칸을 공유 → 서로 덮어쓰거나 엉뚱한 타입을 돌려받는다(`ClassCastException`). 위 예시처럼 `key = "'all'"`을 명시하거나 캐시 이름을 분리
- **Redis에서 객체 파라미터를 키로 쓰면 문자열 변환이 필요** — `RedisCache.convertKey`는 String이면 그대로, ConversionService로 변환 가능하면 변환, 아니면 **`toString()`을 오버라이드했는지** 보고, 없으면 `IllegalStateException`. `Object.toString()`에는 identity 해시가 들어가 같은 내용도 다른 키가 되기 때문. record는 `toString`이 자동 생성돼서 그대로 쓸 수 있다
- **같은 키로 동시에 miss가 나면 전부 본문을 실행한다(캐시 스탬피드)** — 기본은 락이 없다. `@Cacheable(sync = true)`면 한 스레드만 계산하고 나머지는 기다린다. 단 **Redis는 `RedisCacheManager` 기본 writer가 `nonLockingRedisCacheWriter`라 sync를 줘도 락이 안 걸린다** → 필요하면 `RedisCacheWriter.lockingRedisCacheWriter(factory)`. 이 락은 키 단위가 아니라 **캐시 이름 단위(`이름~lock` 키)**라, 그 캐시의 miss 전체가 한 줄로 선다. (Spring Data Redis main 브랜치 소스 기준. 구버전은 `RedisCache.get(key, loader)`가 `synchronized`라 JVM 안에서만 막았던 것으로 보임 — 쓰는 버전 확인 필요)

## 💡 판단 기준

- 코드리뷰에서 "@Cacheable 쓰라"는 지적을 받고 정리 — 손수 Map 캐시를 만들려던 상황: **"이 메서드 결과를 캐시해줘"는 선언(어노테이션)으로, 어디에·얼마나는 설정(CacheManager)으로** 분리하는 게 Spring Cache의 본질. 본문 로직은 캐시를 모른 채 순수하게 유지된다
- 저장소 선택 질문은 하나: **"서버 여러 대가 같은 캐시를 봐야 하는가?"** = "이 데이터, 서버마다 잠깐 달라도 되나?" — 돼야 하면 Redis, 아니면 Caffeine이 더 빠르다
- 캐시가 안 먹는 것 같으면 순서대로 의심: ① `@EnableCaching` 있나 ② 자기 호출 아닌가 ③ 프록시 거치는 public 메서드인가
- "계산 메서드 안의 조회들도 캐시되나?"에서 막혔다 → **프록시는 입구(파라미터)와 출구(반환값)만 본다.** 캐시 항목은 `(캐시 이름, key) → return 값` 한 칸이라, "무엇이 키를 바꾸나(파라미터)"와 "무엇이 저장되나(반환값)" 두 가지만 보면 캐시 동작이 전부 설명된다. 안쪽 조회를 따로 캐시하고 싶으면 그 조회를 **별도 빈의 `@Cacheable` 메서드**로 빼야 한다

## 참고

- Spring Framework 공식 문서 — Cache Abstraction: https://docs.spring.io/spring-framework/reference/integration/cache.html
- Spring Boot 공식 문서 — Caching (자동구성·provider 순서): https://docs.spring.io/spring-boot/reference/io/caching.html
- Caffeine GitHub: https://github.com/ben-manes/caffeine
- Spring Framework 공식 문서 — Default Key Generation · Synchronized Caching: https://docs.spring.io/spring-framework/reference/integration/cache/annotations.html
- 소스 — `SimpleKeyGenerator`·`SimpleKey`·`ConcurrentMapCacheManager`(storeByValue): https://github.com/spring-projects/spring-framework/tree/main/spring-context/src/main/java/org/springframework/cache
- 소스 — `RedisCache.convertKey`·`CacheKeyPrefix`·`DefaultRedisCacheWriter`(`~lock`)·`RedisCacheManager`(nonLocking 기본): https://github.com/spring-projects/spring-data-redis/tree/main/src/main/java/org/springframework/data/redis/cache
- 관련 노트: [Redis 기초](../../infra/redis/redis-basics.md), [스케일 아웃](../../infra/scaling.md), [@Transactional 프록시](./transactional.md), [영속성 컨텍스트](../jpa/persistence-context.md)
- 학습 날짜: 2026-08-03 (보강 2026-09-30)
- 계기: 실무 코드의 `~Cache` 전용 컴포넌트(@Cacheable/@CachePut/@CacheEvict + Redis 30분 TTL)를 따라가며, "Redis에 진짜 넣는 건가?"부터 자동구성·로컬 캐시 사본 문제까지 정리
- 보강 계기: "캐시되는 건 return 값뿐인가? 안쪽 조회들은? 나중엔 어떻게 꺼내나?" — 항목 모양(캐시 이름+key → return 값)·기본 키 규칙·참조 저장 함정·sync와 Redis 락까지 소스로 확인해 추가

# JPA Repository 테스트 (@DataJpaTest) — 사용법

> **한 줄 요약**: `@DataJpaTest`는 JPA 관련 빈만 부팅하는 **슬라이스 테스트** + 각 테스트를 **트랜잭션으로 감싸 롤백** + `TestEntityManager`·Spring Data 레포·임베디드 DB를 자동 구성한다. 핵심 함정은 **persist ≠ INSERT**(flush 전엔 SQL 안 나감) — `em.flush()`/`em.clear()`로 실제 왕복을 강제한다.

관련 노트: [영속성 컨텍스트·flush](../jpa/persistence-context.md) · [테스트 픽스처](./test-fixtures.md) · [JUnit 5 라이프사이클](./junit-lifecycle.md) · [Criteria·Specification·Pageable·Page](../jpa/spring-data-query.md) · [@Lock 실무 패턴(동시성 테스트)](../jpa/lock-practical.md)

---

## 1. 기본 형태

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Import(UserRepositoryAdapter.class)
class UserRepositoryAdapterTest {

    @Autowired private UserRepository userRepository;
    @Autowired private TestEntityManager em;
}
```

---

## 2. `@DataJpaTest`가 실제로 하는 일 (이론)

`@SpringBootTest`처럼 전체 컨텍스트를 띄우지 않고 **JPA 슬라이스**만 부팅한다. 구체적으로:

| 자동 구성 | 내용 |
|---|---|
| **스캔 범위** | `@Entity` 클래스 + Spring Data `Repository`만. `@Service`/`@Component`/`@RestController`는 **안 올라옴** |
| **DB** | 기본적으로 인메모리 임베디드 DB로 **교체**(H2/HSQLDB/Derby 클래스패스 필요) |
| **트랜잭션** | 각 테스트 메서드를 트랜잭션으로 감싸고 **끝나면 롤백** (테스트 간 격리) |
| **제공 빈** | `TestEntityManager`, `EntityManager`, `DataSource`, Hibernate, Spring Data 레포 |
| **SQL 로그** | `spring.jpa.show-sql` 켜면 실행 SQL 확인 가능 |

> 어댑터(`@Repository` 컴포넌트)는 슬라이스 스캔 대상이 아니므로 **`@Import`로 직접 등록**해야 주입된다.

---

## 3. `@AutoConfigureTestDatabase(replace = ...)` 옵션

`@DataJpaTest`는 기본으로 datasource를 임베디드로 **교체**한다. 이 동작을 제어:

| `replace` 값 | 동작 |
|---|---|
| `Replace.ANY` (기본) | 설정된 datasource를 **무조건 임베디드로 교체** |
| `Replace.AUTO_CONFIGURED` | 자동 구성된 datasource만 교체 |
| **`Replace.NONE`** | **교체 안 함** → `application.yml`의 datasource(H2/MySQL) 그대로 사용 |

> 이 프로젝트는 테스트 `application.yml`에 H2를 직접 설정 → **`Replace.NONE`** 으로 그 H2를 쓴다. Testcontainers MySQL을 붙일 때도 `NONE`(교체를 꺼야 컨테이너 DB로 연결됨).

---

## 4. `TestEntityManager` API

JPA `EntityManager`를 테스트용으로 감싼 것. flush/clear 같은 영속성 컨텍스트 제어를 직접 할 수 있다.

| 메서드 | 하는 일 |
|---|---|
| `persist(entity)` | 영속화 (INSERT는 flush 때) |
| `persistAndFlush(entity)` | 영속화 + **즉시 flush** → 엔티티 반환 |
| `persistFlushFind(entity)` | persist + flush + find까지 한 번에 |
| `flush()` | 쌓인 SQL을 DB로 발사 |
| `clear()` | 1차 캐시(영속성 컨텍스트) 비움 → 다음 조회는 진짜 SELECT |
| `find(Class, id)` | PK로 조회 |
| `merge` / `remove` / `detach` / `refresh` | 병합 / 삭제 / 준영속화 / DB값 재로딩 |
| `getId(entity)` | 엔티티의 식별자 반환 |
| `getEntityManager()` | 원본 `EntityManager` 꺼내기 |

```java
UserEntity saved = em.persistAndFlush(UserEntity.create(user)); // INSERT 발사됨
em.clear();                                                     // 캐시 비움
UserEntity found = em.find(UserEntity.class, saved.getId());    // 진짜 SELECT
```

---

## 5. ⭐ persist ≠ INSERT (가장 큰 함정)

```java
User saved = userRepository.save(user);   // 내부 em.persist()
assertThat(saved.getId()).isNotBlank();   // ✅ 통과... 근데 INSERT는 안 나갔을 수 있음
```

- `persist`는 **id(UUID)를 메모리에서 즉시 생성** + INSERT를 쓰기 지연 큐에 **적재만** 한다.
- `getId()`가 차는 건 메모리 UUID 생성 덕분 — **DB에 안 갔는데도** id는 생김.
- `@DataJpaTest`는 **롤백**(commit 없음) + 조회 쿼리도 없으면 → **INSERT가 끝내 발사 안 됨.**

**INSERT가 실제로 나가는 시점 = flush** (① commit ② JPQL 쿼리 직전 auto-flush ③ `em.flush()`). 메커니즘 상세 → [영속성 컨텍스트·flush §2~3](../jpa/persistence-context.md).

> 생성 전략 차이(이론): `GenerationType.UUID`는 **앱이 id를 만들어** persist 시 즉시 부여하고 INSERT는 미룬다. `GenerationType.IDENTITY`(auto_increment)는 **DB가 id를 만들어야** 해서 persist가 INSERT를 즉시 보낸다.

---

## 6. 왕복 검증: save → flush → clear → 조회

```java
@Test
void findByEmail_found() {
    em.persistAndFlush(UserEntity.create(UserFixture.withEmail("a@test.com"))); // flush까지
    em.clear();                                                                 // 캐시 비움

    Optional<User> found = userRepository.findByEmail("a@test.com");

    assertThat(found).isPresent();
    assertThat(found.get().getEmail()).isEqualTo("a@test.com");  // 존재만 X, 매핑까지
}
```

| 단계 | 검증/효과 |
|---|---|
| `flush` | INSERT 발사 → 제약조건·매핑 오류 여기서 터짐 |
| `clear` | 1차 캐시 비움 → 조회가 **진짜 SELECT** (안 하면 넣은 객체 그대로 반환) |
| 조회 | DB→Entity→`toModel()` **역방향 매핑**까지 확인 |

> ⚠️ `clear()` 빼먹으면 "왕복 검증"이 아니라 "방금 만든 객체 재확인". `isPresent()`만 보지 말고 **필드까지** 단언.

---

## 6-1. 💡 셋업 격리 — `em.persistAndFlush` vs `save()` (관점)

조회 테스트의 데이터 셋업을 어댑터 `save()`로 하면, 그 테스트가 `save()` 정상 동작에 **묶인다**(save가 깨지면 조회 테스트도 같이 빨개져 실패 지점이 번짐).

```java
em.persistAndFlush(UserEntity.create(UserFixture.withEmail(email)));  // 격리 ↑ : JPA로 직접 적재 (어댑터 우회)
userRepository.save(...);                                             // 결합 ↑ : save()에 의존
```

- 통합 테스트는 셋업에 영속화가 **불가피**하다(쿼리 검증하려면 데이터가 DB에 있어야 함).
- 하지만 **경로 선택**으로 격리는 가능 → 원칙대로면 `em.persistAndFlush`. 단위 테스트에서 상태를 production 메서드(`addBalance`) 대신 `of`로 직접 주입한 것과 같은 맥락. → [테스트 픽스처 §5-1](./test-fixtures.md)
- 편의 우선이면 `save()`도 용인(로직 단순 + 자기 테스트 있음). 학습/엄밀히는 `persistAndFlush`.

---

## 6-2. update 검증 — 더티 체킹이라 까다롭다

JPA `update`는 보통 `save()`를 안 부르고 **영속 엔티티의 필드만 바꿔놓는다**(더티 체킹 → flush 때 UPDATE 발사). 그래서 검증이 `save`/`find`와 다르다.

```java
@Test
void update_success() {
    User saved = persistedUser("a@test.com");                       // ① 먼저 심어 id 확보
    User changed = User.of(saved.getId(), saved.getEmail(),         // ② 같은 id + 바뀐 필드
                           saved.getPassword(), BigDecimal.valueOf(5000), saved.getRole());

    userRepository.update(changed);
    em.flush();   // ③ 더티 체킹 UPDATE 발사
    em.clear();   // ④ 캐시 비움

    User found = userRepository.findById(saved.getId()).orElseThrow();  // ⑤ 재조회로 검증
    assertThat(found.getBalance()).isEqualByComparingTo(BigDecimal.valueOf(5000));
}
```

| 함정 | 내용 |
|---|---|
| **return값으로 검증 금지** | `update()`가 돌려준 객체는 **메모리에서 바뀐 것** → DB에 안 나가도 통과. 반드시 **재조회**로 확인 |
| **`flush` → `clear` 순서** | `clear`를 먼저 하면 미반영 변경이 **폐기**돼 재조회 시 옛값 → 실패. flush로 UPDATE 발사 후 clear |
| **id 없으면 NOT_FOUND** | 없는 id로 update → `BusinessException(NOT_FOUND)` 케이스도 (errorCode까지 검증) |
| **시작값 ≠ 목표값** | 시작 잔액과 update 목표가 같으면 **update가 아무것도 안 해도 통과**(거짓 양성). "FROM 다른 값 → TO 목표값"이 드러나게 시작값을 명시적으로 다르게 둔다 |

> 💡 **update는 "바꾼 게 진짜 DB에 나갔나"가 핵심.** 호출 결과(return)를 믿지 말고 `flush → clear → 재조회`로 DB를 직접 확인한다. (더티 체킹 메커니즘 → [영속성 컨텍스트·flush](../jpa/persistence-context.md))

---

## 7. Testcontainers (실제 MySQL) — 어노테이션 사용법

H2로 안 되는 것(`FOR UPDATE` 락 시맨틱·MySQL 방언·동시성)만 실제 DB로. **어노테이션 2줄 추가**:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers                                  // ① JUnit5 확장 — 컨테이너 생명주기 관리
@Import(UserRepositoryAdapter.class)
class UserLockRepositoryTest {

    @Container                                   // ② 컨테이너 필드 표시
    @ServiceConnection                           // ③ 접속정보 자동 주입 (Boot 3.1+)
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");
}
```

| 어노테이션 | 역할 |
|---|---|
| `@Testcontainers` | 컨테이너 start/stop을 JUnit 생명주기에 연결 |
| `@Container` | **static 필드** → 클래스당 1번 기동(공유, 빠름) / **인스턴스 필드** → 테스트마다 기동(느림) |
| `@ServiceConnection` | 컨테이너의 url/username/password를 스프링에 자동 연결 |

### `@ServiceConnection` 없이 (구버전·수동)

```java
@DynamicPropertySource
static void props(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", mysql::getJdbcUrl);
    registry.add("spring.datasource.username", mysql::getUsername);
    registry.add("spring.datasource.password", mysql::getPassword);
}
```

> `@ServiceConnection`(Boot 3.1+)이 위 `@DynamicPropertySource` 보일러플레이트를 대체한다.

---

## 8. 무엇을 테스트하나 (가치 기준)

| 가치 | 대상 |
|---|---|
| 높음 | 커스텀 쿼리(`@Query`, 파생 쿼리, Specification), `Entity↔Model` 매핑, unique/nullable 제약, Auditing |
| 낮음 | 순수 `save()`로 id 생기나 (하이버네이트가 보장 = 프레임워크 테스트) |

> 💡 **Spring Data 기본 메서드여도 어댑터가 매핑을 얹으면 테스트 대상.** `findById`는 `JpaRepository` 기본 메서드라 그 자체는 프레임워크지만, 어댑터가 `.map(UserEntity::toModel)`를 얹으면 그 **매핑은 내 코드** → `findByEmail`과 동일하게 "매핑 왕복 + not-found"를 검증한다. 순수 위임이면 스킵, 변환·분기가 붙으면 대상. (조회할 id는 `persistAndFlush`가 돌려준 엔티티의 `getId()`로 얻는다.)

---

## 9. 함정 모음

- **persist≠INSERT** → flush 직접 호출 (§5).
- **`clear()` 누락** → 캐시라 SELECT 검증 안 됨 (§6).
- **unique 컬럼 여러 건** → email `@Column(unique=true)`인데 같은 값 2건 적재 시 flush에서 위반. 픽스처를 `withEmail(email)`로 다르게. → [테스트 픽스처](./test-fixtures.md)
- **테스트 파일 위치** → 대상 클래스와 **같은 패키지**(표준). 어댑터가 `adapter.out.persistence`면 테스트도 거기.
- **Specification이 없는 컬럼 참조** → `likeIgnoreCase("name", ...)`인데 엔티티에 `name` 없으면 쿼리 빌드 시 `IllegalArgumentException`. (Specification/Pageable 개념 자체 → [Criteria·Specification·Pageable·Page](../jpa/spring-data-query.md))

---

## 10. 참고
- [Spring Boot - TestEntityManager](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/test/autoconfigure/orm/jpa/TestEntityManager.html)
- [Testcontainers for Java - MySQL](https://java.testcontainers.org/modules/databases/mysql/)
- 관련 노트: [영속성 컨텍스트·flush](../jpa/persistence-context.md) · [테스트 픽스처](./test-fixtures.md) · [Criteria·Specification·Pageable·Page](../jpa/spring-data-query.md)
- 상세 방법론: server-java `docs/TEST_GUIDE.md` (§4, §4.5)

---

**학습 날짜**: 2026-06-05
**계기**: server-java `UserRepositoryAdapterTest`를 만들며 "save하면 INSERT가 언제 나가나"에서 출발 → `@DataJpaTest` 동작·`TestEntityManager` API·`replace` 옵션·Testcontainers 어노테이션 사용법을 정리.

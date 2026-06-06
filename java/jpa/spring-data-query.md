# Criteria · Specification · Pageable · Page — 출처와 사용법

> **한 줄 요약**: 동적 조회·페이징에 쓰는 도구들인데 **출처가 제각각**이다. `Pageable`/`Page`는 **Spring Data**(JPA 아님), `Specification`은 **Spring Data JPA**, 그 안의 `root`/`criteriaBuilder`는 **JPA 표준 Criteria API**, `UserCriteria`는 **프로젝트가 만든 DTO**. 가장 큰 함정은 **"Criteria"가 두 가지**(JPA Criteria API vs 자작 필터 DTO)라는 것.

관련 노트: [영속성 컨텍스트·flush](./persistence-context.md) · [JPA repository 테스트](../test/jpa-repository-test.md)

---

## 1. ⭐ "JPA 관련인가?" — 출처부터 (핵심)

| 이름 | 패키지 | 소속 | JPA냐? |
|---|---|---|---|
| `Pageable`, `Page`, `PageRequest`, `Sort` | `org.springframework.data.domain` | **Spring Data Commons** | ❌ JPA 아님 (MongoDB 등에도 씀) |
| `Specification`, `JpaSpecificationExecutor` | `org.springframework.data.jpa.*` | **Spring Data JPA** | △ JPA 위에 얹은 Spring 것 |
| `CriteriaBuilder`, `CriteriaQuery`, `Root`, `Predicate` | `jakarta.persistence.criteria` | **JPA(Jakarta Persistence) 표준** | ✅ JPA 그 자체 |
| `UserCriteria` | 프로젝트 패키지 | **내가 만든 DTO** | ❌ 프레임워크 아님 |

> 즉 "이거 다 JPA냐?" → **아니다.** 페이징은 Spring Data, Specification은 Spring Data JPA, Criteria API만 순수 JPA. 한 줄에 섞여 보여서 헷갈릴 뿐이다.

---

## 2. ⚠️ "Criteria"는 두 개다 (가장 큰 혼동)

| | JPA **Criteria API** | 프로젝트 `UserCriteria` |
|---|---|---|
| 정체 | JPA 표준 — 쿼리를 자바 코드로 타입세이프하게 조립 | 그냥 **필터 조건을 담는 DTO** |
| 구성 | `CriteriaBuilder`, `Root`, `Predicate` ... | `id`, `name`, `email` 필드 |
| 누가 만듦 | JPA(Jakarta) | 내가 직접 |

```java
// 프로젝트 UserCriteria — 검색 조건을 담는 평범한 DTO일 뿐
@Getter @Builder
public class UserCriteria {
    private String id;
    private String name;
    private String email;
}
```

> 이름이 같아서 "Criteria = JPA 그거?" 싶지만, `UserCriteria`는 **JPA와 무관한 자작 DTO**다. JPA Criteria API는 `Specification` *내부*에서 쓰인다(§4).

---

## 3. Pageable / Page — 페이징·정렬 (Spring Data)

### `Pageable` — "몇 페이지, 몇 개, 어떤 정렬" 요청

```java
Pageable pageable = PageRequest.of(0, 20, Sort.by("email").ascending());
//                                 ↑page ↑size  ↑정렬
```
> ⚠️ **페이지 번호는 0-based.** 첫 페이지가 `0`.

### `Page<T>` — 결과 + 메타데이터 반환

```java
Page<User> page = repository.findAll(pageable);

page.getContent();        // List<User> — 이번 페이지 데이터
page.getTotalElements();  // 전체 개수
page.getTotalPages();     // 전체 페이지 수
page.hasNext();           // 다음 페이지 있나
```
> ⚠️ `Page`는 **count 쿼리를 추가로** 날린다(전체 개수 계산). 개수가 필요 없으면 `Slice`(다음 페이지 유무만, count 안 함)가 가볍다.

---

## 4. Specification — 동적 조건 (Spring Data JPA)

조건을 **코드로 조립**해 "if문 달린 WHERE"를 만든다. 람다 `(root, query, criteriaBuilder)`가 곧 **JPA Criteria API**.

```java
public static <T> Specification<T> likeIgnoreCase(String field, String value) {
    return (root, query, criteriaBuilder) -> {       // ← 이 3개가 JPA Criteria API
        if (!StringUtils.hasText(value)) return null; // 값 없으면 조건 제외 (null = 조건 무시)
        return criteriaBuilder.like(
            criteriaBuilder.lower(root.get(field)),
            "%" + value.toLowerCase() + "%");
    };
}
```

- `root.get("email")` → 엔티티의 email 컬럼 참조
- `criteriaBuilder.like(...)` → `LIKE` 술어(Predicate) 생성
- **`return null` → 그 조건은 빠짐** → 입력이 있을 때만 WHERE에 붙는 "동적 쿼리" 핵심

### 사용하려면 레포가 `JpaSpecificationExecutor` 상속

```java
public interface UserJpaRepository
    extends JpaRepository<UserEntity, String>, JpaSpecificationExecutor<UserEntity> {
}
// → findAll(Specification, Pageable), findOne(Specification), count(Specification) 등이 생김
```

---

## 5. 어떻게 조립되나 (데이터 흐름)

이 프로젝트의 실제 연결:

```
UserCriteria (필터 DTO: id/name/email)
   │  UserSpecification.withCriteria(criteria)
   ▼
Specification<UserEntity>           ← 조건을 코드로 조립 (내부는 JPA Criteria API)
   │  jpaRepository.findAll(spec, pageable)   ← JpaSpecificationExecutor 제공
   ▼
Page<UserEntity>                    ← 페이징 결과 (+ count)
   │  .map(UserEntity::toModel)
   ▼
Page<User>                          ← 어댑터가 도메인으로 변환해 반환
```

```java
// 어댑터
public Page<User> findAllByCriteria(UserCriteria criteria, Pageable pageable) {
    Specification<UserEntity> spec = UserSpecification.withCriteria(criteria);
    return jpaRepository.findAll(spec, pageable).map(UserEntity::toModel);
}
```

---

## 6. ⚠️ 함정 모음

- **"Criteria" 두 의미** — JPA Criteria API ≠ 자작 `UserCriteria`(§2).
- **PageRequest 0-based** — 첫 페이지 `0`.
- **`Page`는 count 쿼리 추가** — 개수 불필요하면 `Slice`.
- **Specification `return null`** — 조건 제외를 의미. 값 없을 때 null 반환하면 그 조건이 WHERE에서 빠짐(동적 쿼리).
- **존재하지 않는 필드 참조** — `root.get("name")`인데 엔티티에 `name`이 없으면 **쿼리 빌드 시 `IllegalArgumentException`**. (← 이 프로젝트 `UserSpecification`이 실제로 가진 버그)

---

## 7. 💡 판단 기준 — 동적 조회, 뭘로?

| 상황 | 선택 |
|---|---|
| 조건이 **고정** (항상 email로) | 파생 쿼리 `findByEmail` / `@Query` |
| 조건이 **선택적·조합** (있으면 필터, 없으면 무시) | **`Specification`** (동적) |
| 복잡·타입세이프·대규모 동적 | QueryDSL (별도 도입) |

> 한 줄: **`Specification`은 "조건이 들어올 때만 WHERE에 붙이는" 동적 쿼리용.** 조건이 고정이면 그냥 파생 쿼리/`@Query`가 단순하다. 페이징(`Pageable`/`Page`)은 그 위에 얹는 별개 축(Spring Data).

---

## 7-1. 다른 선택지 — 동적 조회 / 페이징은 이것만 있는 게 아니다

### 동적 조회 (Specification 외)

| 방법 | 출처 | 특징 |
|---|---|---|
| **QueryDSL** | 서드파티 | **타입세이프 + 가독성** 최고. `BooleanBuilder`/`BooleanExpression`로 동적 조립. 복잡 동적쿼리 사실상 표준. Q-클래스 생성 셋업 필요 |
| **Query by Example (QBE)** | Spring Data | probe 엔티티 + `ExampleMatcher`로 간단 일치. **범위·OR 불가** |
| **@Query 동적 트릭** | Spring Data JPA | `:p IS NULL OR col = :p` 식. 간단하지만 지저분 |
| JPA Criteria API 직접 | JPA 표준 | Specification이 감싼 저수준 |

### 페이징 (Pageable/Page 외)

| 방법 | 출처 | 특징 |
|---|---|---|
| **Slice** | Spring Data | count 안 함 → "다음 있나"만. "더보기"에 가벼움 |
| **ScrollPosition / Window** (keyset) | Spring Data **3.1+** | 커서/keyset: `WHERE id > :last LIMIT n`. 대용량·무한스크롤, OFFSET 성능 회피 |
| **Limit** | Spring Data **3.2+** | `Limit.of(n)` 단순 개수 제한 |
| Top/First 파생 | Spring Data | `findTop10ByOrderByCreatedAtDesc(...)` |

> ⚠️ **OFFSET 페이징의 함정**: `OFFSET 10000`처럼 깊어지면 DB가 그만큼 건너뛰며 읽어 느려진다 → 대용량은 keyset(`Window`).

> 💡 판단: **동적 조회** — 단순 1~2조건은 파생/`@Query`/QBE, 복잡 조합·타입세이프는 **QueryDSL**(또는 Specification). **페이징** — 총개수 필요하면 `Page`, 불필요하면 `Slice`, 대용량·딥페이지는 keyset(`Window`).

---

## 8. 참고
- [Spring Data JPA - Specifications](https://docs.spring.io/spring-data/jpa/reference/jpa/specifications.html)
- [Spring Data - Paging and Sorting](https://docs.spring.io/spring-data/jpa/reference/repositories/core-concepts.html)
- 관련 노트: [JPA repository 테스트](../test/jpa-repository-test.md) · [영속성 컨텍스트·flush](./persistence-context.md)

---

**학습 날짜**: 2026-06-06
**계기**: `findAllByCriteria` 테스트를 짜려다 `Criteria`/`Page`/`Pageable`/`Specification`이 각각 뭔지·JPA인지 헷갈려서 정리. 핵심은 "출처가 다 다르다"(Spring Data vs Spring Data JPA vs JPA 표준 vs 자작 DTO) + "Criteria가 두 의미".

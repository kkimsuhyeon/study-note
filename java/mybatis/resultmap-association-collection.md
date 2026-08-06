# MyBatis resultMap — association·collection 중첩 매핑과 `<id>`

> **한 줄 요약**: JOIN 쿼리 한 방의 결과를 부모+연관 객체 그래프로 조립하는 도구. `<association>`=단수 객체, `<collection>`=1:N 리스트. 핵심 함정 — `<id>`는 단순 컬럼 매핑이 아니라 **"어떤 row들이 같은 객체인가"를 판별하는 정체성 기준**이라, 빼먹으면 전체 컬럼 비교로 동작해 느려지고 그룹핑이 오동작할 수 있다.

## 언제 쓰나

- **이미 JOIN하고 있는 쿼리에서 연관 entity가 필요할 때** — 연관 테이블 컬럼 몇 개만 SELECT에 추가하고 매핑을 붙이면 **추가 쿼리 0회**로 연관 객체를 손에 쥔다 (조회 → 상태 전이 → update 흐름에 유용)
- 부모 1건 + 자식 N건을 한 번에 조립할 때 (예: 게시글 + 댓글 목록)
- JPA 없이 MyBatis만으로 객체 그래프를 만들어야 할 때

## 사용 예시

```xml
<resultMap id="BlogMap" type="Blog">
    <id property="id" column="blog_id"/>                  <!-- 부모 정체성 (그룹핑 기준) -->
    <result property="title" column="blog_title"/>
    <association property="author" javaType="Author">     <!-- 단수: 1건 -->
        <id property="id" column="author_id"/>            <!-- 중첩 객체의 정체성 -->
        <result property="username" column="author_username"/>
    </association>
    <collection property="posts" ofType="Post">           <!-- 복수: 리스트 -->
        <id property="id" column="post_id"/>
        <result property="subject" column="post_subject"/>
    </collection>
</resultMap>

<select id="selectBlog" resultMap="BlogMap">
    SELECT B.id       AS blog_id,
           B.title    AS blog_title,
           A.id       AS author_id,
           A.username AS author_username,
           P.id       AS post_id,
           P.subject  AS post_subject
    FROM blog B
    JOIN author A ON A.id = B.author_id
    LEFT JOIN post P ON P.blog_id = B.id
    WHERE B.id = #{id}
</select>
```

- `association`은 `javaType`, `collection`은 **`ofType`**(리스트가 담는 원소 타입)을 쓴다.
- 같은 구조를 재사용하려면 `<association resultMap="authorResult" columnPrefix="co_">`처럼 외부 resultMap 참조 + 접두사 매핑도 가능.

## 종류/옵션 비교

| 방식 | 동작 | 특징 |
|---|---|---|
| **중첩 결과(nested results)** — 위 예시 | JOIN 한 방 → MyBatis가 row들을 객체 그래프로 분해 | 쿼리 1회. `<id>` 기준 그룹핑 |
| **중첩 select** — `<association column="author_id" select="selectAuthor"/>` | 부모 조회 후 property마다 **별도 쿼리 실행** | 구현 단순하지만 **N+1**. `fetchType="lazy"`로 지연 가능 |

- `notNullColumn`: 기본은 "매핑된 컬럼이 하나라도 non-null이면 자식 객체 생성". LEFT JOIN 미매칭 row에서 자식이 null이 되는 기준을 조정할 때 사용.

## ⚠️ 함정/메커니즘

### 1. `<id>`는 성능 힌트가 아니라 정체성 선언이다

공식 문서: *"id will flag the result as an identifier property to be used when comparing object instances. This helps to improve general performance, but especially performance of caching and nested result mapping (join mapping)."*

- `<id>`가 없으면 **매핑된 전체 컬럼을 비교**해서 "같은 객체인지" 판별한다 — 느리고, NULL이 섞이면 같은 부모가 두 객체로 쪼개지는 오동작 여지.
- `<collection>` 그룹핑(부모 1 + 자식 N 합치기)이 이 판별에 의존하므로 **collection을 쓰면 부모 `<id>`는 사실상 필수**. association도 단일 row 쿼리면 없어도 동작하지만 명시가 안전.

### 2. collection + LIMIT 페이징은 부모 수가 아니라 row 수를 자른다

JOIN 결과는 자식 수만큼 row가 늘어나 있으므로, `LIMIT 10`은 "부모 10건"이 아니라 "조인된 row 10줄"이다 — 마지막 부모의 자식이 잘려 나가거나 부모 수가 모자란다. 페이징이 필요하면 부모만 먼저 잘라서(서브쿼리/별도 조회) 조인할 것.

### 3. 부모·자식의 동명 컬럼은 alias 필수

부모와 자식 테이블에 같은 이름 컬럼(FK 등)이 있으면 alias 없이 `column="promotion_id"`로는 어느 쪽 값인지 보장되지 않는다. `pub.promotion_cd AS pub_promotion_cd`처럼 접두사 alias를 붙이는 게 관례 (`columnPrefix`도 같은 문제의 해법).

### 4. 기타 메커니즘

- setter가 없어도 **필드 직접 접근**으로 매핑된다 (`@Getter`만 있는 불변형 entity 가능). protected 기본 생성자도 리플렉션으로 생성.
- resultMap 정의가 참조하는 select보다 **문서상 뒤에 있어도 된다** — 파서가 resultMap을 statement보다 먼저 처리.
- 중첩 select(`select=`) 방식은 JPA의 N+1과 동일한 문제 — [N+1과 fetch 전략](../jpa/n-plus-one-fetch.md) 참고.

## 💡 판단 기준

- **"연관 데이터가 필요한데 그 테이블을 이미 JOIN하고 있다면, 새 쿼리를 만들지 말고 컬럼+매핑을 추가한다."** — 실제 케이스: `FOR UPDATE` 락 조회가 발행 코드 테이블을 이미 JOIN하고 있었는데, 상태 변경용 entity가 필요해서 별도 조회를 고민하다가 SELECT에 컬럼 2개 + `<association>` 추가로 해결. 잠긴 row가 그대로 entity로 로드되니 "조회 → 인스턴스 메서드로 상태 전이 → 범용 update" 흐름이 추가 쿼리 0회로 완성됐다. (락 관점은 [SELECT FOR UPDATE](../../database/select-for-update.md))
- **단수/복수는 테이블 관계가 아니라 "이 쿼리가 뭘 반환하나"로 정한다.** — 1:N 테이블이라도 특정 자식을 코드로 찍어 조회하는 쿼리는 결과가 항상 1건 → `<collection>`으로 받아 `get(0)` 하지 말고 `<association>` 단수 필드로. List는 "전체 목록"이라는 오해를 부른다.

## 참고

- [MyBatis 공식: Mapper XML — Result Maps](https://mybatis.org/mybatis-3/sqlmap-xml.html#Result_Maps) — id&result, Nested Results for Association/Collection

학습 날짜: 2026-08-06 · 계기: 프로모션 코드 등록 작업에서 락 조회 쿼리에 발행 코드 row를 `<association>`으로 매핑하며 — `<id>`의 역할("id는 필요 없나?"), association vs collection 선택("collection으로 묶으면 편할 것 같은데")을 검토

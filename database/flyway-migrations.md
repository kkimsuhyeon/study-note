# Flyway — SQL 실행 순서와 적용 이력을 관리하기

> **한 줄 요약:** SQL은 개발자가 준비하고, Flyway는 어떤 DB에 어떤 버전을 적용했는지 기록해 미적용 변경을 실행하고 이미 적용한 파일의 변경을 감지한다.

## 1. SQL을 직접 실행하는 것과 무엇이 다른가?

테이블에 컬럼을 추가하는 SQL 자체는 같아도 실행 책임이 달라진다.

| 방법 | 개발자가 직접 관리하는 부분 |
| --- | --- |
| DB 도구에서 SQL 실행 | 어느 환경에 실행했는지, 순서, 중복 실행, 빠진 변경 |
| Flyway 마이그레이션 | 변경 SQL과 적용 정책. 도구가 버전 순서·적용 이력을 관리 |

**Flyway는 DB 서버가 아니다.** 연결한 DB 안에 기본적으로 `flyway_schema_history`라는 이력 테이블을 사용한다. [공식 이력 테이블 설명](https://documentation.red-gate.com/fd/flyway-schema-history-table-273973417.html)

## 2. 가장 작은 사용 예시

Spring Boot에서 Flyway 의존성·해당 DB 모듈·DataSource 구성이 준비돼 있다고 가정한다. 기본 검색 경로는 `src/main/resources/db/migration`이다.

```text
db/migration/
  V1__create_products.sql
  V2__add_product_description.sql
```

```sql
-- V1__create_products.sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);
```

```sql
-- V2__add_product_description.sql
ALTER TABLE products ADD COLUMN description VARCHAR(500);
```

`V2`는 버전이며, 버전과 설명 사이 구분자는 밑줄 **두 개**다. 버전 마이그레이션은 버전 순으로 각 대상 DB에 한 번 적용한다. dev에 적용했다고 운영 DB에 적용된 것은 아니다. [Versioned migrations](https://documentation.red-gate.com/fd/versioned-migrations-273973333.html)

설정 예시:

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
  jpa:
    hibernate:
      ddl-auto: validate
```

Flyway가 테이블 변경을 맡고, Hibernate의 `validate`는 Entity와 테이블 구조가 맞는지 검사한다. **JPA 조회·저장·변경 감지는 그대로 사용**한다. 테이블 구조 변경(DDL)과 데이터 수정(DML)은 별개다. [Spring Boot — 데이터 초기화](https://docs.spring.io/spring-boot/how-to/data-initialization.html)

## 3. 언제 실행하며 새 파일은 어떻게 아나?

Boot 자동 구성을 사용하면 애플리케이션 시작 시 `migrate()`를 호출한다. CLI나 배포 파이프라인에서 명시적으로 실행하도록 구성할 수도 있다. 실행 중인 서버가 폴더를 계속 감시하는 기능은 아니다.

```text
시작 또는 migrate 호출
  → 설정된 경로의 마이그레이션 탐색
  → 해당 DB의 이력과 비교·검증
  → 아직 적용되지 않은 버전 실행
  → 실행 결과를 이력에 기록
```

예를 들어 DB 이력에 V1만 있고 배포 파일에 V1·V2가 있다면, 검증이 정상일 때 V2를 적용한다. 단순히 `./gradlew build`로 JAR를 만드는 것만으로 운영 DB에 적용되는 것은 아니다. 테스트가 애플리케이션 컨텍스트를 띄운다면 테스트 DB에는 실행될 수 있다.

## 4. 파일을 바꿨다는 것은 어떻게 아나?

Flyway는 실행한 SQL 마이그레이션의 **체크섬(checksum)**을 이력에 기록한다. 이후 현재 파일로 계산한 값과 비교해 변경을 발견한다. 파일 수정 시각만 보는 방식이 아니다. SQL 마이그레이션의 체크섬은 CRC32 기반이며, 보안을 위한 [HMAC](../java/security/hmac-and-hashing.md)과 목적이 다르다. [Validate](https://documentation.red-gate.com/flyway/reference/commands/validate)

| 상태 | 일반적인 처리 |
| --- | --- |
| 적용한 V1이 그대로임 | 다시 실행하지 않음 |
| 미적용 V2가 추가됨 | 다음 migrate에서 적용 |
| 적용한 V1의 내용이 변경됨 | 체크섬 불일치로 검증 실패 가능 |

이 검사는 **실제 테이블 구조의 모든 수동 변경을 감시하는 검사**가 아니다. 누군가 DB에서 컬럼을 직접 바꿔도 SQL 파일·이력 비교만으로 모든 차이를 찾아내지는 못한다.

## 5. ⚠️ 적용된 SQL은 어떻게 고치나?

공유 dev·운영 등에 적용한 V1을 고치지 않고 **새 V3로 변경을 이어서 적용**한다. 개인 로컬의 버려도 되는 DB는 데이터를 지우고 처음부터 재구축하는 선택이 가능하지만, 데이터 삭제를 감수할 때의 이야기다. 로컬이라는 이유만으로 적용 이력과 파일의 불일치가 사라지지는 않는다.

`repair`는 이력을 고치는 명령이다. **수정된 SQL을 다시 실행해서 실제 테이블까지 맞추는 명령이 아니다.** 검증 오류를 지우려고 무조건 실행하면 파일과 DB의 차이를 숨길 수 있다. [Repair](https://documentation.red-gate.com/flyway/reference/commands/repair)

- ⚠️ **브랜치마다 같은 다음 번호를 만들면 충돌한다.** 두 사람이 각자 `V3__...`를 만들면 병합 때 같은 버전이 둘이 된다. 한쪽이 먼저 운영에 적용돼 `V4`까지 갔는데 늦게 합쳐진 `V3`가 들어오면, 기본값(`outOfOrder=false`)에서는 그 V3를 적용하지 않는다(ignored로 남고, 기본 검증 설정이면 migrate가 오류로 멈춘다 — 확인 필요). `outOfOrder=true`(Spring Boot `spring.flyway.out-of-order`)는 적용을 허용하지만 실행 순서가 환경마다 달라진다. 번호를 타임스탬프(`V20260922_1030__...`)로 쓰면 충돌 자체가 줄어든다. [outOfOrder 설정](https://documentation.red-gate.com/fd/flyway-out-of-order-setting-277579015.html)
- ⚠️ **실패한 마이그레이션의 뒷정리.** PostgreSQL은 DDL도 트랜잭션에 묶여 실패한 마이그레이션이 통째로 롤백된다. 반면 MySQL처럼 DDL이 즉시 커밋되는 DB에서는 절반만 적용된 채 실패 이력이 남는다. `repair`는 이 실패 이력을 지우지만, 이미 만들어진 테이블·컬럼은 **직접 치워야** 한다(공식 문서: "User objects left behind must still be cleaned up manually").

위 설명은 `V...` **버전 마이그레이션** 기준이다. 체크섬 변경 때 다시 적용하는 `R__...` 반복 마이그레이션은 별도 개념이다. 보통 테이블 변경을 배우는 단계에서는 V 파일의 순서와 이력부터 이해한다.

## 6. SQL은 자동 생성되나?

일반적인 SQL 마이그레이션에서는 개발자가 파일을 작성·검토한다. 도구로 변경 스크립트를 생성하는 흐름도 있지만, **Flyway를 추가하면 Entity 수정으로 올바른 SQL이 자동 생성된다고 생각하면 안 된다.** SQL 생성 도구의 존재와 Flyway의 적용 이력 관리 기능은 구분한다. [마이그레이션 기반 접근](https://documentation.red-gate.com/fd/migrations-based-approach-168984769.html)

## 7. 이미 JPA `ddl-auto`로 돌던 프로젝트에 붙일 때

**의존성.** Spring Boot 3.4 문서 기준, 메모리·파일 DB(H2 등)는 `org.flywaydb:flyway-core`로 충분하지만 그 밖의 DB는 **DB별 모듈이 따로 필요하다** — MySQL은 `org.flywaydb:flyway-mysql`, PostgreSQL은 `org.flywaydb:flyway-database-postgresql`. 빠뜨리면 MySQL에 붙는 순간 지원하지 않는 DB라는 오류가 난다.

**첫 파일(V1)은 지금 테이블 전체.** 이미 엔티티로 테이블을 만들어 오던 프로젝트라면 `V1__init.sql`에 현재 스키마 전체를 적고, 그다음 변경부터 `V2__...`로 이어 간다. JPA 표준 속성(`jakarta.persistence.schema-generation.scripts.*`)으로 엔티티 기준 DDL을 뽑아 출발점으로 삼을 수 있지만, 타입·인덱스·제약을 사람이 검토해서 다듬는다(6절 — 자동 생성이 정답은 아니다). 이후에는 `ddl-auto`를 `validate`로 두어 엔티티와 SQL이 어긋나면 기동 시점에 알게 한다.

**H2로 테스트하고 MySQL로 운영하면 SQL 문법이 갈린다.** MySQL용으로 쓴 마이그레이션이 H2에서 실패할 수 있다.

| 방법 | 장점 | 비용 |
|---|---|---|
| 테스트도 실제 MySQL(Testcontainers) | 스키마 출처가 SQL 하나, 운영과 같은 DB로 락·제약까지 검증 | 테스트가 느려지고 Docker 필요 |
| H2 프로필에선 Flyway 끄고 `ddl-auto` 유지 | 빠르고 간단 | 스키마 출처가 둘(엔티티와 SQL)이라 어긋날 수 있음 — `validate`가 MySQL 프로필에서만 잡아 줌 |
| `classpath:db/migration/{vendor}`로 DB별 스크립트 | 둘 다 SQL로 관리 | 같은 변경을 두 번 써야 함 |

동시성·락 검증이 중요한 프로젝트라면 첫 번째가 맞다 — H2의 락 동작은 MySQL InnoDB와 다르다.

## 참고·학습 기록

- 위 절별 공식 문서 링크 참고. 학습일: 2026-09-22 (2026-10-02 브랜치 번호 충돌·실패 뒷정리 보강).
- 2026-10-21 보강: `ddl-auto`로 돌던 프로젝트에 붙일 때 — DB별 모듈(`flyway-mysql`, [Spring Boot 3.4 How-to: Execute Flyway Database Migrations on Startup](https://docs.spring.io/spring-boot/3.4/how-to/data-initialization.html)에서 확인), V1 기준선, H2 테스트와 MySQL 문법 차이 선택지. 계기: 다른 프로젝트에서 본 "쿼리 버전 관리"를 H2 테스트·MySQL dev 구성의 프로젝트에 적용할 수 있는지 질문.
- 계기: SQL 직접 실행과 Flyway의 차이, 새 파일·변경 파일 감지, 로컬과 운영의 마이그레이션 처리 질문. 예시 SQL·설정은 학습용으로 일반화했으며 실제 DB에는 실행하지 않았다.

💡 **여러 환경에 같은 구조 변경을 반복 적용해야 한다면, SQL 내용뿐 아니라 실행 이력도 관리한다. 이미 공유된 버전은 새 마이그레이션으로 이어 간다.**

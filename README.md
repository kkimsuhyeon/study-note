# Study Note

개발하면서 학습한 내용을 정리하는 공간

## Java

### Concurrency (동시성)
- [락 개념 종합 - 낙관적/비관적/분산 락](./java/concurrency/locks.md)
- [JVM 동시성 도구 - synchronized/ReentrantLock/Semaphore/Latch/Barrier 등](./java/concurrency/jvm-concurrency-tools.md)
- [데드락(교착 상태) - Coffman 4조건/예방/DB 자동 감지](./java/concurrency/deadlock.md)

### JPA
- [영속성 컨텍스트 · flush · 더티 체킹 - "커밋 시점"의 정체](./java/jpa/persistence-context.md)
- [@Lock 기본 - 락 어노테이션 (언제·종류·사용·주의)](./java/jpa/lock.md)
- [@Lock 심화 개념 - @Version·공유/배타·FORCE_INCREMENT](./java/jpa/lock-concepts.md)
- [@Lock 실무 패턴 - 프록시·네이밍·테스트·재시도·벌크/조건부 UPDATE](./java/jpa/lock-practical.md)
- [Read-Modify-Write와 트랜잭션 경계 - 쓰기 서비스가 조회 서비스에 의존하면 안 되는 이유](./java/jpa/read-modify-write.md)

### Spring
- [@Transactional - 선언적 트랜잭션·전파(propagation)·롤백 규칙·프록시 함정](./java/spring/transactional.md)
- [예제로 보는 트랜잭션 전파·롤백 - a→b→c 워크스루](./java/spring/transaction-rollback-example.md)

### BigDecimal
- [BigDecimal - 돈·정밀 계산, equals vs compareTo, scale, 반올림](./java/bigdecimal/bigdecimal.md)

### Test (테스트 도구 사용법)
- [AssertJ 사용법 - assertThat / isEqualByComparingTo / assertThatThrownBy](./java/test/assertj.md)
- [테스트 방법론 핵심 요약 - 판단 공식 + 막힌 케이스 누적 (상세는 프로젝트 가이드)](./java/test/test-writing-guide.md)

### Jackson
- [Jackson 어노테이션 종합 정리](./java/jackson/annotations.md)

## Database
- [SQL 케이스 쿡북 - 상황별 해법 (날짜 범위 조회, 인덱스/sargable 등)](./database/sql-cookbook.md)
- [PostgreSQL 날짜 함수 - to_date/make_date/EXTRACT/date_trunc](./database/postgresql-date-functions.md)

## Infra / 분산 환경
- [스케일 아웃 & 배포 모델 - 1 JVM/인스턴스 복제/로드밸런서 vs 오토스케일러/무상태](./infra/scaling.md)

---

## 작성 규칙
- 한 어노테이션/개념당 한 파일
- 파일명은 kebab-case (`lock.md`, `json-property.md`)
- 새 글 추가 시 이 README에 링크 등록 (안 그러면 잊어버림)

## 글 템플릿
1. 한 줄 요약
2. 언제 쓰나
3. 사용 예시
4. 종류/옵션 비교
5. 주의사항
6. 참고 링크 + 학습 날짜

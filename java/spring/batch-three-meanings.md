# "배치"의 세 층위 — 쿼리 배치화 / Spring Batch / JDBC 쓰기 배치

> **한 줄 요약**: "배치(batch)"는 문맥에 따라 ① 쿼리 묶기(코딩 기법) ② Spring Batch(일괄 잡 실행 프레임워크) ③ JDBC 쓰기 묶음(드라이버 레벨)을 다 가리킨다. 핵심 오해 두 가지: **Spring Batch에 스케줄러는 없다**(트리거는 밖에서), 그리고 **쿼리 묶기도 Spring Batch의 본체가 아니다**(그건 어느 쪽이든 개발자 몫).

## 언제 쓰나 (층위 구분)

| 층위 | 정체 | 해결하는 문제 |
|---|---|---|
| ① 쿼리 배치화 | 코딩 기법 (프레임워크 무관) | 루프 안 N+1 쿼리 → IN/범위 조회 1번 |
| ② Spring Batch | 프레임워크 | 대량 일괄 작업을 **안전하게 완주** (재시작·이력·skip) |
| ③ JDBC/MyBatis 배치 | 드라이버/실행기 옵션 | 대량 INSERT/UPDATE 왕복 횟수 줄이기 |

## ① 쿼리 배치화 — N+1 을 "넓게 1번 + 메모리 분배"로

```java
// before — 직원 루프마다 쿼리 (N+1)
for (String empNo : empNos) {
    List<Leave> leaves = mapper.selectByEmpNo(orgId, empNo);   // 500명 = 500쿼리
}

// after — IN 조회 1번 → 키별 Map → 루프는 Map 조회
Map<String, List<Leave>> leavesByEmpNo = mapper.selectByEmpNoList(orgId, empNos).stream()
        .collect(Collectors.groupingBy(Leave::getEmpNo));      // 1쿼리
for (String empNo : empNos) {
    List<Leave> leaves = leavesByEmpNo.getOrDefault(empNo, List.of());
}
```

- 패턴은 항상 같다: **넓게 1번 조회 → `groupingBy`/`toMap` → 루프는 Map만 조회.**
- 날짜 축에도 동일 적용: per-day 조회 22번 → 월 범위 조회 1번 + `Map<LocalDate, ...>`.
- **캐시와의 구분**: 캐시는 lazy("물어본 것만 그때 채움" — `computeIfAbsent`), 배치는 eager("미리 다 퍼와서 분배"). 반복 대상(날짜들/직원들)을 미리 알면 배치, 뭐가 올지 모르면(어떤 근로제가 나올지) 캐시. → [Map 주요 메서드](../collections/map-methods.md)

## ② Spring Batch — "언제"가 아니라 "안전한 완주"의 프레임워크

```
Job ─ Step ─ Chunk(read → process → write, 청크 단위 트랜잭션)
        └ Tasklet(단일 작업)
```

```java
@Bean
public Step settleStep(JobRepository jobRepository, PlatformTransactionManager tx) {
    return new StepBuilder("settleStep", jobRepository)
            .<Attendance, Settlement>chunk(1000, tx)   // 1,000건 단위 커밋
            .reader(attendanceReader())                 // 페이징/커서로 읽기
            .processor(settleProcessor())               // 가공
            .writer(settlementWriter())                 // 쓰기 (내부에서 ③ 쓰기 배치 흔함)
            .build();
}
```

본체는 **실행 모델 + 운영성**:
- **청크 트랜잭션**: 100만 건을 1,000건씩 끊어 커밋 — 중간 실패해도 커밋된 청크는 보존
- **메타테이블**(BATCH_JOB_EXECUTION 등): 실행 이력·처리 건수 기록
- **재시작**: 실패한 Job을 실패 지점(청크)부터 재개
- **skip/retry**: 불량 데이터 건너뛰기, 일시 오류 재시도

**포함 안 되는 것**:
- **스케줄러 없음** — "매일 새벽 2시"는 `@Scheduled`/cron/Quartz/Jenkins 등 바깥 트리거 몫. 실무에서 늘 같이 쓰여서 내장으로 오해하기 쉽다.
- **쿼리 최적화 없음** — reader/writer 안에서 쿼리를 어떻게 날릴지는 여전히 개발자 코드. ①은 Spring Batch 를 쓰든 안 쓰든 별도로 해야 한다.

## ③ JDBC / MyBatis 쓰기 배치 — 왕복 줄이기

```java
// JDBC
PreparedStatement ps = conn.prepareStatement(sql);
for (Row r : rows) { ps.setString(1, ...); ps.addBatch(); }
ps.executeBatch();   // 여러 문장을 한 번에 전송

// MyBatis
SqlSession session = factory.openSession(ExecutorType.BATCH);
```

- INSERT/UPDATE N번의 네트워크 왕복을 묶는다. (MySQL 은 `rewriteBatchedStatements=true` 여야 진짜 multi-value 로 재작성)
- ①이 **읽기** 묶기라면 이건 **쓰기** 묶기.

## ⚠️ 함정

- **"요청 안 배치"의 한계** — HTTP 요청 하나가 수백 명 × 수십 일을 루프 도는 구조는 ①로 쿼리를 줄여도 요청 타임아웃·메모리 한계가 남는다. 처리량이 커지면 "요청 시 계산"을 "미리 계산해 적재(② 잡) + 요청은 조회만"으로 바꾸는 게 다음 단계.
- **MyBatis `ExecutorType.BATCH` 중 SELECT 주의** — 배치로 쌓아둔 문장이 SELECT 시점에 flush 된다. 읽기·쓰기가 섞인 로직에서 예상 못 한 실행 순서가 나올 수 있다.
- **Spring Batch 는 동일 파라미터 Job 재실행을 막는다** — JobParameters 가 같으면 "이미 완료된 잡"으로 거부. 재실행하려면 타임스탬프 등 식별 파라미터를 바꿔야 한다 (완주 보장의 뒷면).
- 청크 크기 = 트랜잭션 크기 — 너무 크면 롤백 비용·락 유지가 커지고, 너무 작으면 커밋 오버헤드.

## 💡 판단 기준

- **루프 안에서 쿼리가 보이면 무조건 ①부터** — 프레임워크 도입 이전의 기본기. (근태 마감 rewrite: 직원 루프의 per-day 로더 쿼리 ~130개/인 → 월 범위 조회 3~4개/인으로 배치화 설계. Spring Batch 없이 mapper 쿼리 추가만으로 해결)
- **②는 "요청 시간 안에 못 끝나는 일괄 작업"이 생겼을 때** — 마감 계산이 느려져 "미리 계산해두자"가 되는 순간이 도입 시점. 그 전에 도입하면 인프라만 무겁다.
- 단어 "배치"가 나오면 **어느 층위인지 먼저 확인** — "마감 batch"(일괄 계산 작업, ②적 의미)와 "쿼리 배치화"(①)가 같은 대화에 섞이면 서로 다른 걸 말하게 된다.

## 참고

- [Spring Batch Reference — Domain Language (Job/Step/Chunk)](https://docs.spring.io/spring-batch/reference/domain.html)
- [MyBatis — SqlSession ExecutorType](https://mybatis.org/mybatis-3/java-api.html)
- 관련 노트: [Map 주요 메서드 (computeIfAbsent 캐시)](../collections/map-methods.md), [스케일 아웃 & 배포 모델](../../infra/scaling.md)

---
학습 날짜: 2026-07-03
계기: 근태 마감 per-day 쿼리 배치화(⑥) 논의 중 "배치화가 캐시랑 다른가? Spring Batch랑 다른가?"에서 출발

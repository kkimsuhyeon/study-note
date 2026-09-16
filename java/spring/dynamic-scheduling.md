# 동적 스케줄링 — `ThreadPoolTaskScheduler` + `CronTrigger`로 DB 정의 스케줄 등록

> **한 줄 요약**: `@Scheduled(cron = "...")`은 크론이 **컴파일 시점에 고정**된다. 실행 시각·순서·on/off를 DB나 설정에서 읽어 바꾸려면 스프링 코어의 `TaskScheduler.schedule(Runnable, Trigger)`를 직접 호출한다. Spring Batch도 Quartz도 필요 없다. 단 **"동적"인 것은 언제·어떤 순서로 돌리느냐까지**이고, 잡의 종류(어떤 클래스가 있는가)는 여전히 코드·배포의 몫이다.

관련 노트: [ApplicationContext](./application-context.md) · ["배치"의 세 층위](./batch-three-meanings.md) · [스케일 아웃](../../infra/scaling.md)

---

## 1. 세 가지 선택지

| | `@Scheduled` | `TaskScheduler` + `CronTrigger` | Quartz (+ JDBC JobStore) |
|---|---|---|---|
| 크론 변경 | 재빌드·재배포 | **런타임 (재등록)** | 런타임 (API) |
| 크론 출처 | 소스코드 (`${...}` 플레이스홀더로 설정값은 가능) | 아무 데서나 (DB·API) | DB (Quartz 테이블) |
| 실행 이력·미스파이어 복구 | 없음 | 없음 (직접 구현) | 있음 |
| 다중 인스턴스 중복 방지 | 없음 (ShedLock 등 별도) | 없음 (별도) | 클러스터 모드 내장 |
| 의존성 | 스프링 코어 | 스프링 코어 | quartz + 테이블 11개 |
| 복잡도 | 최저 | 낮음 | 높음 |

> `@Scheduled(cron = "${batch.cron}")`처럼 **설정값을 크론에 넣는 것**까지는 `@Scheduled`로 된다. 다만 기동 시 한 번 읽는 것이라 "운영 중 바꾸기"는 안 된다. 그게 필요해지는 순간이 이 노트의 방식으로 넘어가는 시점.

---

## 2. 사용법 — 스프링 코어만으로

```java
ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
scheduler.setPoolSize(taskCount);        // 동시에 돌 수 있는 스케줄 수
scheduler.setThreadNamePrefix("batch-");
scheduler.initialize();                  // ← 이걸 호출해야 내부 Executor가 만들어진다

ScheduledFuture<?> future = scheduler.schedule(
        () -> runJobs(jobList),          // Runnable
        new CronTrigger("0 0 10 * * ?")  // Trigger — 다음 실행 시각을 계산하는 객체
);

future.cancel(false);                    // 하나만 해제
scheduler.shutdown();                    // 전부 종료 (실행 중 작업을 기다리지 않음 — 기본값)
```

- `Trigger`는 "다음 실행 시각을 알려주는" 인터페이스. `CronTrigger`(크론), `PeriodicTrigger`(고정 간격)가 기본 제공이고, 직접 구현하면 "DB에서 다음 시각을 읽는 트리거"도 가능하다.
- **재등록 패턴 두 가지**: ⓐ 전부 `shutdown()` 후 스케줄러를 새로 만들어 다시 등록(단순, pharos_batch 방식) ⓑ `ScheduledFuture`를 보관해 두고 바뀐 것만 `cancel` → 재`schedule`(세밀, 실행 중 작업 보호).

---

## 3. 사례 — pharos_batch: 테이블 3개 + 클래스 1개

**테이블 역할**

| 테이블 | 역할 | 코드가 |
|---|---|---|
| `cbm_schd` | 스케줄 정의: `schd_cd`(번호) · `schd_cron` · `schd_nm` | 읽음 |
| `cbm_schd_job_command_mapp` | 스케줄 ↔ 명령 매핑: `schd_cd` · `job_command`(**빈 이름**) · `step`(순서, 0=비활성) · `retry` | 읽음 |
| `cbm_job_command` | 명령 카탈로그 (이름·설명) | **안 읽음** — 문서용 |
| `cbm_job_run_hst` | 실행 이력: 명령 1회 = 1행, 성공 여부·에러 | 씀 |

**기동 흐름** (`DynamicBatch.init`, `@PostConstruct`)

```java
List<Schd> schds = schdMapper.selectAll();                        // ① 스케줄 전부
List<SchdJobCommandMapp> mapps = schdJobCommandMappMapper.selectAll();  // ② 매핑 전부

for (Schd schd : schds) {
    List<CommandInfo> jobs = mapps.stream()
            .filter(m -> m.getSchdCd() == schd.getSchdCd() && m.getStep() > 0)   // step 0 = 사용 안 함
            .map(m -> {
                AbstactCommand cmd = ctx.getBean(m.getJobCommand(), AbstactCommand.class);  // ③ 빈 이름으로 조회
                cmd.maxRetry = m.getRetry();
                return CommandInfo.create(m, cmd);
            })
            .sorted(comparingInt(CommandInfo::getStep))                            // ④ step 순
            .collect(toList());
    tasks.add(Task.of(schd, jobs));
}

scheduler.setPoolSize(tasks.size());                                               // 스케줄 1개 = 스레드 1개
for (Task t : tasks) {
    scheduler.schedule(() -> t.getJobs().forEach(this::startJob),                  // ⑤ 같은 스케줄의 명령은 순차
                       new CronTrigger(t.getSchdCron()));
}
```

- `startJob`은 명령마다 `cbm_job_run_hst`에 한 줄 insert → 실행 → 성공/실패 update. 실패 시 `retry`만큼 재시도.
- **재로딩**: `PATCH /api/v1/mgmt/batch/init` → `stopBatch()`(shutdown) → `init()` 다시. DB만 바꾸면 반영 안 되고 이 호출이나 재기동이 필요하다.
- 새 잡 = `AbstactCommand`를 상속한 `@Component` **코드 작성·배포** + `cbm_schd_job_command_mapp`에 행 1개. 크론이 새로 필요하면 `cbm_schd`에도 1행.

**무엇이 정적이고 무엇이 동적인가**

| | 정해지는 곳 | 바꾸려면 |
|---|---|---|
| 어떤 명령(클래스)이 존재하는가 | 코드 | 빌드·배포 |
| 언제·어떤 순서·재시도·on/off | DB | SQL + `/init` |

---

## 4. 크론 표기 — 스프링은 6자리, `?`·`L`·`W`·`#` 지원 (5.3+)

```
┌ 초 (0-59)
│ ┌ 분 (0-59)
│ │ ┌ 시 (0-23)
│ │ │ ┌ 일 (1-31)          L = 말일, W = 가장 가까운 평일, ? = 상관없음
│ │ │ │ ┌ 월 (1-12 / JAN-DEC)
│ │ │ │ │ ┌ 요일 (0-7 / MON-SUN, 0과 7 = SUN)   L = 마지막 X요일, # = n번째 X요일
│ │ │ │ │ │
0 0 10 * * ?      매일 10:00:00
0 */5 * * * ?     5분마다
0 0 2 L * ?       매월 말일 02:00
0 0 7 ? * MON#1   매월 첫 월요일 07:00
@daily / @hourly  매크로도 됨 (5.3+)
```

- **초 필드가 있다** — 유닉스 crontab(5자리)을 그대로 붙이면 한 칸씩 밀려 엉뚱한 시각이 된다.
- Quartz 문법(`?`, `L`, `W`, `#`, `1/1`)은 Spring 5.3의 `CronExpression`부터 받아준다. 그 이전(`CronSequenceGenerator`)은 `?`를 `*`로만 다루고 `L`/`W`/`#`은 미지원. Quartz의 7번째 연도 필드는 스프링에 없다.
- 도구: `CronExpression.parse(expr).next(LocalDateTime.now())`로 다음 실행 시각을 테스트에서 확인할 수 있다.

---

## ⚠️ 함정

- **빈 이름 오타 한 건이 스케줄 전체를 죽인다** — `getBean`은 없으면 예외이고, 위 코드처럼 `for` 전체가 하나의 `try`에 있으면 첫 실패에서 init이 끝나 **아무 스케줄도 등록되지 않는다.** 이름 규칙은 `Introspector.decapitalize`(클래스명 첫 글자 소문자, 단 앞 두 글자가 대문자면 그대로) → [ApplicationContext §7](./application-context.md). 해법은 `Map<String, AbstactCommand>` 주입으로 "없으면 null → 경고 후 건너뛰기".
- **step=0 같은 "삭제 대신 끄기" 관례** — 코드에 `if (step > 0)` 한 줄로만 존재한다. 문서화하지 않으면 다음 사람이 "왜 이 행은 안 도나"를 코드까지 파고 들어야 한다. 비활성 행에는 잘못된 빈 이름이 숨어 있어도 티가 안 나다가 활성화 순간 터진다.
- **CronTrigger는 이전 실행이 끝난 시각 기준으로 다음 시각을 계산한다** — 같은 스케줄은 겹쳐 돌지 않지만, 명령 하나가 오래 걸리면 다음 회차가 밀리거나 건너뛴다(5분 주기인데 7분 걸리면 다음은 10분 시점). 순차 실행이 안전장치이자 지연의 원인.
- **`shutdown()`은 실행 중 작업을 기다리지 않는다** — `waitForTasksToCompleteOnShutdown`이 기본 false. `/init`을 작업 도중 호출하면 그 작업이 중간에 끊긴다. 재로딩 API는 "지금 도는 게 없을 때"만 부르거나, 재등록 패턴 ⓑ(변경분만 cancel)로 바꿔야 한다.
- **인스턴스가 2대면 2번 돈다** — 스케일 아웃하면 모든 인스턴스가 같은 크론을 등록한다. 단일 배치 인스턴스로 두거나, ShedLock(DB 락으로 한 인스턴스만 실행), Quartz 클러스터 모드가 필요하다 → [스케일 아웃](../../infra/scaling.md).
- **미스파이어 복구가 없다** — 서버가 꺼져 있던 시각의 스케줄은 그냥 사라진다. "매월 1일 정산"이 배포 시각과 겹치면 그 달 정산이 안 돈다. 이력 테이블(`cbm_job_run_hst`)로 "돌았는지"는 확인되지만 "안 돌았으니 지금 돌린다"는 없다. 필요하면 Quartz의 misfire 정책이나 수동 실행 엔드포인트(pharos_batch는 `*CommandTest` GET들이 그 역할).

---

## 💡 판단 기준

- **`@Scheduled`로 시작한다.** 운영이 "시각을 바꿔 달라"고 두 번째 말하는 순간이 DB 정의로 넘어가는 시점. 그 전에 만들면 테이블 3개와 재로딩 API가 그냥 짐이다.
- **"동적"의 범위를 정확히 말한다** — DB 정의 스케줄이라도 잡의 **종류** 추가는 배포다. 기획·운영에게 "새 배치 추가는 DB만 넣으면 되죠?"라는 기대를 만들지 않도록 표(§3 마지막)로 설명한다.
- **이름을 DB에 저장하면 그 이름은 계약이다** — 클래스명 리팩토링이 곧 장애다. 클래스 주석에 "이름 변경 금지, DB `job_command`와 계약"을 박거나, 빈 이름 대신 명령이 스스로 선언하는 코드(enum·`getCommandCode()`)를 키로 쓴다.
- **클러스터·미스파이어·이력 조회 셋 중 둘 이상이 요구되면 Quartz** — 직접 구현하면 결국 Quartz 테이블을 다시 만들게 된다.
- 구체 케이스: pharos_batch는 단일 배치 서버·수십 개 잡·"운영이 시각을 바꾼다" 요구에 이 방식이 잘 맞는다. 고칠 건 두 가지 — `Map` 주입으로 부분 실패 격리, `/init`의 실행 중 작업 보호.

---

## 참고

- [Spring Framework Reference — Task Execution and Scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html) (`TaskScheduler`, `Trigger`, `@Scheduled`)
- [Javadoc — CronExpression](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/support/CronExpression.html) (6필드 문법, `L`/`W`/`#`/`?`, 매크로) · [CronTrigger](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/support/CronTrigger.html) · [ThreadPoolTaskScheduler](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/concurrent/ThreadPoolTaskScheduler.html)
- [Spring Blog — Cron Expressions in Spring 5.3](https://spring.io/blog/2020/11/10/new-in-spring-5-3-improved-cron-expressions)
- [Quartz — JDBC JobStore 설정](https://www.quartz-scheduler.org/documentation/quartz-2.3.0/configuration/ConfigJobStoreTX.html) · [ShedLock](https://github.com/lukas-krecan/ShedLock)
- 관련 노트: [ApplicationContext](./application-context.md) · ["배치"의 세 층위](./batch-three-meanings.md) · [스케일 아웃](../../infra/scaling.md)

---

**학습 날짜**: 2026-09-16
**계기**: DB에 `cbm_job_command`·`cbm_schd_job_command_mapp`가 있는데 "코드에서 스케줄을 어떻게 만드는 건가"에서 출발. pharos_batch `DynamicBatch`를 읽으며 Quartz·Spring Batch 없이 스프링 코어만으로 DB 정의 스케줄을 도는 구조, 그리고 빈 이름이 DB와의 계약이 되는 함정을 정리. `AppContextProvider`가 뭔지는 [ApplicationContext](./application-context.md)로 분리.

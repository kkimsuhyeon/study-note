# 테스트 방법론 — 핵심 요약 + 막힌 케이스 누적

> **한 줄 요약**: "무엇을 / 몇 개 테스트할지"의 핵심 공식만 모은 짧은 체크리스트. 이 레포는 **사용법(API) 레퍼런스**가 중심이라, 상세 방법론·구조는 [AssertJ 사용법](./assertj.md) 같은 도구 문서와 프로젝트 가이드에 맡기고, 여기선 **자주 까먹는 판단 공식 + 내가 막혔던 케이스**만 둔다.

관련 노트: [AssertJ 사용법](./assertj.md) · [테스트 픽스처(Object Mother)](./test-fixtures.md) · [JPA repository 테스트](./jpa-repository-test.md) · [Mockito 서비스 테스트](./mockito-service-test.md) · [BigDecimal](../bigdecimal/bigdecimal.md) · [@Lock 실무 패턴(동시성 테스트)](../jpa/lock-practical.md)

> 📎 상세 방법론(테스트 종류 선택, @DataJpaTest/Mockito/Testcontainers 구조, 프로젝트 컨벤션)은 **server-java `docs/TEST_GUIDE.md`** 에 잘 정리돼 있음 — 거길 본다. 이 문서는 그 핵심만 추린 요약.

---

## 1. 핵심 판단 공식 (이것만 외우면 됨)

| 질문 | 답 |
|------|-----|
| **테스트할까 말까?** | 메서드 안에 **`if`/예외/계산/상태변경**이 있나? → 있으면 한다, 없으면(값만 옮기면) 스킵 |
| **케이스 몇 개?** | **성공 경로 1개 + `if`/예외마다 1개** (+ 여유 되면 경계값 1개) |
| **then에 뭘 적지?** | 그 메서드가 **바꾼 것을 전부** (필드 3개 바꾸면 3개 다 검증) |
| **검증 종류는?** | 반환값 / 상태변화 / 예외 / 상호작용(Mock) 중 해당하는 것 |

> 구조는 **given-when-then**, `when`은 한 행위, `then`엔 반드시 단언. 예외는 `errorCode`/메시지까지 검증.

---

## 2. 빠뜨리기 쉬운 케이스 체크리스트

정상 1개만 짜는 게 가장 흔한 실수. **경계·예외가 진짜 버그를 잡는다.**

- [ ] **0 / 음수** — 허용? 거부? ([BigDecimal `compareTo(ZERO)`](../bigdecimal/bigdecimal.md))
- [ ] **경계 바로 위/아래** — 잔액과 *정확히* 같은 금액 차감, 1 모자란 값
- [ ] **null / 빈 값** — null 인자, 빈 컬렉션/문자열
- [ ] **부족·초과** — 잔액보다 큰 차감 → 예외? (핵심 도메인 규칙)
- [ ] **없는 대상** — 없는 id 조회 → 빈 Optional? 예외?
- [ ] **예외 후 상태 불변** — 실패 시 값이 안 바뀌었는지 (원자성)
- [ ] **(해당 시) 동시성** — 같은 자원 동시 요청 → 1개만 성공 ([동시성 테스트](../jpa/lock-practical.md))

---

## 3. 📌 내가 막혔던 케이스 (계속 추가)

> 테스트 짜다 막힌 상황을 여기에 기록 — 다음에 같은 데서 안 막히게. (틀: **상황 / 막힌 점 / 해결**)

### (1) repository `save()` 테스트가 통과하는데 INSERT가 안 나감 (2026-06-05)
- **상황**: `@DataJpaTest`에서 `save()` 후 `getId().isNotBlank()`만 검증 → 통과. 근데 SQL 로그에 insert가 안 보임.
- **막힌 점**: persist는 INSERT를 즉시 안 보낸다. UUID는 메모리에서 생성돼 `getId()`는 차지만, `@DataJpaTest`는 롤백이라 commit flush도 없어 INSERT가 끝내 안 나감 → "UUID 생성기"만 검증한 꼴.
- **해결**: `em.flush()`로 INSERT 강제 + `em.clear()` 후 조회해 왕복 검증. 상세 → [JPA repository 테스트 §2~3](./jpa-repository-test.md).

### (2) 여러 명 심으려는데 unique 제약 위반 (2026-06-05)
- **상황**: 필터 테스트용으로 유저 여러 명 적재하려 했더니 flush에서 제약 위반 예외.
- **막힌 점**: 픽스처가 email을 `"test@test.com"`으로 고정 → email이 `unique` 컬럼이라 2명째에서 충돌.
- **해결**: 픽스처를 `withEmail(email)`처럼 **변하는 값(email)을 받게** 오버로드. → [테스트 픽스처 §2, §7](./test-fixtures.md).

---

## 4. 참고
- 관련 노트: [AssertJ 사용법](./assertj.md) · [테스트 픽스처](./test-fixtures.md) · [JPA repository 테스트](./jpa-repository-test.md) · [BigDecimal](../bigdecimal/bigdecimal.md) · [@Lock 실무 패턴(동시성 테스트)](../jpa/lock-practical.md)
- 상세 방법론: server-java `docs/TEST_GUIDE.md`

---

**학습 날짜**: 2026-05-28 (2026-06-03 사용법 중심 레포 성격에 맞춰 방법론은 핵심 공식만 남기고 슬림화)
**계기**: study-note는 "도구 사용법" 레퍼런스가 본질에 맞다고 판단 → 방법론 상세는 프로젝트 TEST_GUIDE에 맡기고, 자주 까먹는 판단 공식과 막힌 케이스 누적만 유지

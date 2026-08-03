# LEFT JOIN — 자식 조건을 ON에 두나 WHERE에 두나

> **한 줄 요약**: LEFT JOIN에서 **자식(오른쪽) 테이블 조건**을 ON에 두면 "매칭만 제한"(부모 유지), WHERE에 두면 "행 자체가 탈락"(부모도 사라짐). 핵심 함정: 자식 없는 부모의 자식 컬럼은 NULL인데 `NULL = '값'`은 참이 못 되므로, WHERE에 자식 조건을 쓰는 순간 **LEFT JOIN이 조용히 INNER JOIN으로 변한다**.

## 언제 문제되나

- LEFT JOIN 쿼리에 자식 테이블 필터를 추가할 때 — 특히 `deleted_yn = 'N'` 같은 **soft delete 조건을 습관적으로 WHERE에** 붙일 때
- "목록은 다 나와야 하는데 일부 부모가 증발했다"는 버그의 단골 원인 (에러 없이 조용히 발생)

## 사용 예시

데이터 — 방: 매출분석(1)·세금문의(2)·새채팅(3) / 질문: 방1=TAX·GENERAL, 방2=REPORT, 방3=없음
요구사항: **"방 목록을 다 보여주되, TAX 질문이 있으면 옆에 붙여줘"**

```sql
-- ✅ ON에: 매칭만 제한 → 방 3개 전부 유지
SELECT cr.chat_room_name, q.mode
FROM chat_room cr
LEFT JOIN user_question q
    ON  q.chat_room_id = cr.chat_room_id
    AND q.mode = 'TAX'
-- 결과: 매출분석-TAX / 세금문의-NULL / 새채팅-NULL (3줄)
```

```sql
-- ❌ WHERE에: NULL 행이 걸러짐 → 사실상 INNER JOIN
SELECT cr.chat_room_name, q.mode
FROM chat_room cr
LEFT JOIN user_question q
    ON  q.chat_room_id = cr.chat_room_id
WHERE q.mode = 'TAX'
-- 결과: 매출분석-TAX (1줄) — 세금문의·새채팅 증발!
```

메커니즘은 실행(논리 처리) 순서다: **JOIN(ON)이 먼저 완료 → 그 결과에 WHERE 적용**.
1. 조인 결과: 방1-TAX, 방1-GENERAL, 방2-REPORT, 방3-**NULL**
2. `WHERE q.mode = 'TAX'`: 방3의 `NULL = 'TAX'`는 **NULL(unknown)** → WHERE는 TRUE만 통과 → 방3 탈락. 방2도 REPORT라 탈락

## ON vs WHERE 비교

| 자리 | 역할 | 자식 조건을 두면 | 비유 |
|---|---|---|---|
| `ON` | 어떤 자식이 붙을 **자격**이 있나 | 자격 없으면 NULL만 붙음 — 부모를 지울 권한 없음 | **매칭 규칙** (부모 안전) |
| `WHERE` | 조인 끝난 결과에서 행 **생존** 판정 | NULL 행 탈락 → 자식 없는 부모 증발 | **생존 규칙** (부모도 죽음) |

## ⚠️ 함정/메커니즘

- **실무 단골 사고 — soft delete**: `LEFT JOIN q ... WHERE q.deleted_yn = 'N'` → 자식이 0건인 부모가 목록에서 증발. 자식 조건이니 `ON ... AND q.deleted_yn = 'N'`이 맞는 자리
- **WHERE로 살리는 우회도 있긴 하다**: `WHERE (q.mode = 'TAX' OR q.pk IS NULL)` — 동작은 하지만 의도가 안 읽히고 조건 늘수록 지저분. ON으로 옮기는 게 정석
- **INNER JOIN에서는 이 구분이 없다**: 매칭 안 되면 어차피 행이 없으므로 ON이든 WHERE든 결과 동일. 이 함정은 **OUTER(LEFT/RIGHT) JOIN + 자식 조건** 조합에서만 발생
- **부모(왼쪽) 테이블 조건은 반대로 WHERE가 맞다**: `WHERE cr.usr_id = :usrId`는 목록 자체를 거르는 의도이므로 WHERE. 이걸 ON에 넣으면 다른 사용자의 방도 (자식만 NULL인 채로) 살아남는 반대 방향 버그가 된다
- **LATERAL과의 연결**: [LATERAL 조인](./lateral-join-top-n-per-group.md)의 서브쿼리 안 WHERE는 여기서 말하는 "ON 위치"에 해당(매칭만 제한, 바깥 행은 `LEFT ... ON TRUE`가 지킴) — 그래서 LATERAL 패턴에선 이 함정이 구조적으로 잘 안 생긴다

## 💡 판단 기준

- LEFT JOIN 목록 쿼리에서 일부 부모가 증발하는 버그를 추적하다 WHERE의 자식 조건이 범인임을 확인 → **"ON = 매칭 규칙(부모 안전), WHERE = 생존 규칙(부모도 죽음)"**으로 기억
- 조건을 추가할 때 자문: **이 조건은 누구 것인가?** 부모(왼쪽) 조건 → WHERE / 자식(오른쪽) 조건 → ON
- 자식 조건으로 부모까지 거르는 게 **진짜 의도**라면 → LEFT + WHERE 조합으로 우연히 INNER가 되게 두지 말고 **처음부터 INNER JOIN**으로 써서 의도를 문법으로 드러낸다

## 참고

- PostgreSQL 공식 문서 — Joined Tables (outer join 의미론): https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-JOIN
- 학습 날짜: 2026-08-03
- 계기: LATERAL 조인의 `ON TRUE`가 왜 필요한지 따라가다 "그럼 일반 LEFT JOIN에선 조건 위치가 왜 문제인가"로 이어져 정리

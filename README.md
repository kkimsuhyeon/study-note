# Study Note

개발하면서 학습한 내용을 정리하는 공간

## Java

### Concurrency (동시성)
- [락 개념 종합 - 낙관적/비관적/분산 락](./java/concurrency/locks.md)
- [데드락(교착 상태) - Coffman 4조건/예방/DB 자동 감지](./java/concurrency/deadlock.md)

### JPA
- [영속성 컨텍스트 · flush · 더티 체킹 - "커밋 시점"의 정체](./java/jpa/persistence-context.md)
- [@Lock - 비관적 락 어노테이션](./java/jpa/lock.md)

### Jackson
- [Jackson 어노테이션 종합 정리](./java/jackson/annotations.md)

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

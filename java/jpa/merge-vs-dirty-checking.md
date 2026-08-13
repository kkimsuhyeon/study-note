# 준영속 엔티티 수정 — 변경 감지 vs merge

> **한 줄 요약**: 준영속 엔티티(DB 식별자를 가졌지만 영속성 컨텍스트가 관리하지 않는 객체)를 수정하는 방법은 ① **변경 감지**(트랜잭션 안에서 조회 후 값 변경 — 권장)와 ② **merge** 두 가지. merge의 함정은 **모든 필드를 통째로 교체**한다는 것 — 폼/DTO에 없는 필드는 **null로 덮어쓴다.** 그래서 실무 답은 "컨트롤러에서 엔티티를 만들지 말고, **id + 변경 데이터만 서비스로 넘겨 변경 감지**"다. (김영한: "정말 중요, 완벽 이해 필수")

관련 노트: [영속성 컨텍스트 · 더티 체킹](./persistence-context.md) · [변환 계층](../design/transform-layers.md)

---

## 1. 언제 만나는 문제인가 — 수정 폼 저장

```java
@PostMapping("items/{id}/edit")
String updateItem(@PathVariable Long id, @ModelAttribute BookForm form) {
    Book book = new Book();          // ⚠️ new로 만들었지만
    book.setId(form.getId());        //    DB 식별자를 갖고 있다 = "준영속 엔티티"
    book.setName(form.getName());
    book.setPrice(form.getPrice());
    itemService.saveItem(book);      // 이걸 어떻게 DB에 반영하지?
}
```

- **준영속 엔티티** = 이미 DB에 저장된 적 있는 식별자를 가진, 그러나 지금 영속성 컨텍스트가 관리하지 않는 객체. 트랜잭션이 끝나 detach된 것뿐 아니라 **위처럼 임의로 new한 객체도 포함**된다.
- 준영속이므로 [더티 체킹](./persistence-context.md) 대상이 아니다 — 값을 바꿔도 UPDATE가 저절로 안 나간다. 그래서 두 가지 방법이 등장한다.

## 2. 방법 ① 변경 감지 (권장)

```java
@Transactional
public void updateItem(Long itemId, String name, int price, int stockQuantity) {
    Item findItem = itemRepository.findOne(itemId);  // 영속 상태로 조회
    findItem.setName(name);                          // 값만 변경 (원하는 필드만!)
    findItem.setPrice(price);
    findItem.setStockQuantity(stockQuantity);
}   // 트랜잭션 commit → flush → 더티 체킹 → UPDATE
```

트랜잭션 안에서 **영속 엔티티를 조회해서 직접 값을 변경** — 커밋 시점에 변경 감지가 UPDATE를 만든다. **원하는 속성만 선택해서** 바꿀 수 있다.

## 3. 방법 ② merge — 동작과 함정

```java
@Transactional
public void update(Item itemParam) {       // itemParam: 준영속
    Item merged = em.merge(itemParam);     // 반환된 merged가 영속, itemParam은 여전히 준영속
}
```

merge의 내부 동작:

```
① 파라미터의 식별자로 1차 캐시/DB에서 영속 엔티티 조회
② 조회한 영속 엔티티에 준영속 객체의 값을 "전부" 밀어넣음 (모든 필드 교체)
③ 커밋 시 변경 감지로 UPDATE
```

### ⚠️ 핵심 함정 — 없는 필드는 null로 덮어쓴다

**merge는 부분 수정이 아니라 전체 교체다.** 수정 폼에 `price`가 없어서 준영속 객체의 price가 null이면 — **DB의 price가 null로 UPDATE된다.** "폼에 안 넣은 필드가 사라졌어요"류 사고의 정체. 변경 감지는 바꾼 필드만 반영하므로 이 위험이 없다.

### 곁가지 — save()의 정체

Spring Data JPA의 `save()`는 사실 이 분기다:

```java
if (entity.getId() == null) em.persist(entity);   // 식별자 없음 = 신규
else                        em.merge(entity);      // 식별자 있음 = merge!
```

`@GeneratedValue` 전제의 관례 — **"save = 저장"이 아니라 "식별자 있으면 merge"**라서, 수정 의도로 save를 부르면 위의 null 덮어쓰기 함정을 그대로 밟는다. (식별자를 직접 할당하는 엔티티면 이 분기 자체가 어긋난다)

## 4. 💡 권장 패턴 — 컨트롤러에서 엔티티를 만들지 마라

```java
// ❌ 컨트롤러에서 어설픈 엔티티 생성 → merge/save
Book book = new Book(); book.setId(id); ...; service.saveItem(book);

// ✅ id + 변경 데이터만 전달 → 서비스가 조회 후 변경 감지
itemService.updateItem(id, form.getName(), form.getPrice(), form.getStockQuantity());
// 파라미터가 많으면 UpdateItemDto로 묶는다
```

> **엔티티를 변경할 때는 항상 변경 감지를 사용하라** (강의 결론). merge는 "전 필드가 완비된 객체"가 보장될 때만 안전한데, 그 보장을 지키는 비용이 변경 감지보다 비싸다. 컨트롤러는 [변환 계층](../design/transform-layers.md)대로 DTO만 다루고, 엔티티의 생성·수정은 트랜잭션이 있는 서비스 계층 안에서.

---

## 5. 참고
- 김영한, 실전! 스프링 부트와 JPA 활용1 — 7장 "변경 감지와 병합(merge)"
- 관련 노트: [영속성 컨텍스트](./persistence-context.md) (준영속·더티 체킹 메커니즘)

---

**학습 날짜**: 2026-08-13
**계기**: 활용1 7장 — "폼에 없는 필드가 null로 저장되는" 사고의 원인이 merge의 전체 교체 동작임을 이해하고, save()가 내부적으로 merge 분기라는 것까지 연결해서 정리.

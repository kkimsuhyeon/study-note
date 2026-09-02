# 책임 경계 — 클래스·함수를 어디서 나누나 (리뷰에서 "얘의 역할이 뭘까요?"가 나오는 이유)

> **한 줄 요약**: 코드가 돌아가는데 리뷰에 걸리는 건 대부분 **경계를 나눈 기준을 한 문장으로 말할 수 없을 때**다. (2026-09-02 보강: §3-1 — 재료를 Command로 감싸도 **번역이 없으면 봉투**라 같은 질문을 다시 듣는다.) 계층(Controller/Service/Mapper — 위아래)이 아니라 같은 계층 안에서 옆으로 나누는 **책임(responsibility)** 이야기. 나누는 기준은 정답이 없지만(테이블·유스케이스·트랜잭션…) **기준이 있어야** 하고, 기준이 없다는 신호는 세 가지 — ① 한 사실에 대한 **결정이 두 파일에 갈라져** 있다 ② 파라미터로 **완성품**이 넘어와 받는 쪽이 저장밖에 못 한다 ③ 클래스를 설명하면 **"그리고"**가 들어간다.

관련 노트: [변환 계층](./transform-layers.md)(Command가 뭔지) · [도메인 검증 위치 §4·§5-1](./domain-validation.md)(규칙 vs 조회 / Command에 검증 필드 안티패턴) · [애그리거트 소유권](./aggregate-ownership.md)(사실 하나당 결정·저장하는 곳 하나 — 같은 원리를 도메인 *사이*에 적용한 것. 이 노트는 한 도메인 *안*의 클래스 사이) · [계산 파이프라인](./calculation-pipeline.md)(조회는 밖에서, 코어에 I/O 금지 — 경계 기준의 다른 예)

---

## 0. 문제 상황 — 돌아가는 코드에 "이 클래스 역할이 뭐죠?"

회사 등록 API. 본체 테이블(`core_company`·`core_user`)과 확장 테이블(`ext_company`)에 같이 써야 한다. 처음 짠 모양:

```java
// 서비스
public Response register(Request request, Admin admin) {
    Company company = Company.create(request, admin);   // ① 본체 엔티티를 서비스가 조립
    helper.save(company, request, admin);               // 완성품 + 재료를 같이 넘김
    return Response.from(company);
}

// Helper
public void save(Company company, Request request, Admin admin) {
    companyMapper.insert(company);                       // ② 본체 저장
    userMapper.insert(newMasterUser(company, admin));    // ③ 본체 사용자 생성+저장
    extMapper.insert(ExtCompany.of(company, request));   // ④ 확장 테이블 생성+저장
}
```

테스트 통과, 기능 정상. 리뷰 코멘트 두 줄:

> "이 Helper 역할이 뭘까요?"
> "Command에 완성된 엔티티가 들어오는데, 그러면 받는 쪽은 뭘 결정하나요?"

Helper를 한 문장으로 쓰면 — *"본체 회사를 저장하고, **그리고** 사용자를 만들어 저장하고, **그리고** 확장 테이블도 저장한다"*. 그리고가 두 번. 회사를 **어떤 값으로 만들지**는 서비스(①)가 정하고 **저장**은 Helper(②)가 한다. 결정 하나가 두 파일에 갈라졌다.

---

## 1. 용어 — 계층 · 책임 · 응집도 · SRP

| 용어 | 뜻 | 방향 |
|---|---|---|
| **계층(layer)** | 기술 역할로 자른 것. Controller / Service / Repository. 의존이 한 방향(위→아래) | 위아래 |
| **책임(responsibility)** | "이 객체가 **알아야 하는 것**과 **해야 하는 것**" (Wirfs-Brock, Responsibility-Driven Design). 같은 계층 안에서 클래스를 가르는 단위 | 옆 |
| **응집도(cohesion)** | 한 모듈 안 요소들이 **같은 목적**에 얼마나 붙어 있나. 낮으면 설명에 "그리고"가 들어간다 | — |
| **SRP** | Martin: *"클래스는 변경 이유가 하나여야 한다"* → 2014년 재서술: *"같은 이유로 바뀌는 것은 모으고, 다른 이유로 바뀌는 것은 나눈다"* | — |

📖 **SRP를 "한 가지 일만 한다"로 읽으면 안 된다.** 그렇게 읽으면 메서드 하나짜리 클래스가 수십 개 생긴다. 기준은 **변경 이유(누가·왜 바꾸라고 하나)**다. 회사 등록 Helper가 "본체 테이블 정책이 바뀌면" 수정되고, 서비스가 "확장 기능 정책이 바뀌면" 수정된다면 — 그게 경계다.

이 노트는 전부 **"옆으로 나누기"** 얘기다. 위아래(계층)는 프레임워크가 거의 정해주지만, 옆은 매번 내가 정해야 해서 리뷰에 걸린다.

---

## 2. 사례 ① — 결정이 두 파일에 갈라진 Helper → 테이블 소유 기준으로

§0의 코드를 **"어느 테이블을 누가 소유하나"** 한 문장으로 다시 나눴다:

```java
// 서비스 — ext_* 만 책임
public Response register(Request request, Admin admin) {
    Company company = helper.save(request, admin);          // 재료만 넘김
    extMapper.insert(ExtCompany.create(company.getId(), request, admin));
    return Response.from(company);
}

// Helper — core_* 한 채를 "생성부터 저장까지"
public Company save(Request request, Admin owner) {
    Company company = Company.create(request, owner);       // 어떤 값으로 만들지도 여기서
    companyMapper.insert(company);
    userMapper.insert(newMasterUser(company.getId(), owner));
    return company;
}
```

| | 전 | 후 |
|---|---|---|
| Helper 한 문장 | 본체 저장, 그리고 사용자 생성·저장, 그리고 확장 저장 | **본체에 회사 한 채를 세운다** |
| `Company`를 만드는 곳 / 저장하는 곳 | 서비스 / Helper (갈라짐) | Helper / Helper |
| 나눈 기준 | 없음 (그때그때) | **`core_*`는 Helper, `ext_*`는 서비스** |

**기준은 이것 말고도 여러 개가 가능하다** — 유스케이스 기준(등록 흐름 전체를 한 클래스), 트랜잭션 기준(같이 롤백돼야 하는 단위), 외부 시스템 기준(본체 API 호출은 어댑터). 어느 걸 골라도 코드는 돌아간다. **리뷰어가 문제 삼은 건 고른 기준이 아니라, 고른 기준이 없다는 것**이었다. 전 버전에서 "왜 `Company`는 서비스가 만들고 `User`는 Helper가 만드나요?"에 답이 없었다.

---

## 3. 사례 ② — 파라미터로 완성품을 넘기면 받는 쪽은 저장밖에 못 한다

§0의 `helper.save(company, request, admin)`. 여기서 `company`는 **완성품**(이미 값이 다 정해진 엔티티), `request`·`admin`은 **재료**다. 둘을 같이 넘기면:

- Helper는 `company`에 대해 **결정할 게 없다**. 저장 호출 한 줄만 남는다.
- `Company.create(request, admin)`을 호출한 서비스는 **저장을 모른다**. 만들었는데 어디 갔는지 모르는 객체.
- 둘 다 반쪽이라 어느 쪽도 "회사 생성"을 책임진다고 말할 수 없다.

**재료를 넘기면 받는 쪽이 결정을 갖는다.** `helper.save(request, owner)` — 넘기는 건 "회사를 만드는 데 필요한 것"뿐이고, 그걸로 뭘 만들지는 Helper 안에서 정한다.

이건 GRASP의 **Creator**(생성에 필요한 정보를 가진 쪽이 만든다)와 **Information Expert**(판단에 필요한 정보를 가진 객체에 책임을 준다)의 다른 표현이다.

### 3-1. ⚠️ 재료를 Command로 감싸고 싶어질 때 — 번역이 없으면 봉투다

파라미터 2~3개가 거슬려서 Command로 묶고 싶어진다. Fowler의 *Introduce Parameter Object*도 "파라미터 3~4개면 묶어라"라고 한다. 그런데 **Request를 통째로 필드에 담은 Command**는 이렇게 생긴다:

```java
// ❌ 봉투 — 이름만 Command이고 실제로는 Pair<Request, Admin>
@AllArgsConstructor(staticName = "of") @Getter
public class SaveCompanyCommand {
    private Request request;
    private Admin owner;
}
// 받는 쪽: command.getRequest().getBrn() ... 결국 Request 구조를 다 안다
```

Command의 존재 이유는 **번역**이다. web의 언어를 도메인의 언어로 바꿔서 도메인이 화면 사정을 모르게 하는 것. Request를 그대로 품으면 번역이 없어서, 받는 쪽은 여전히 Request를 알아야 하고 Command가 한 일은 파라미터를 하나로 묶은 것뿐이다. *"이게 `save(request, owner)`보다 뭐가 낫나요?"*에 답이 없으면 봉투다.

```java
// ✅ 번역이 있는 평탄 Command — 필드명·타입이 도메인 언어로 바뀐다
public class SaveCompanyCommand {
    private String companyName;   // request.getName()
    private LocalDate openDate;   // LocalDate.parse(request.getOpenedAt())
    private String ownerNo;       // admin.getNo()
}
// Company.create(command) 는 Request 를 import 하지 않는다 → FE 가 필드명을 바꿔도 Factory 한 곳만 수정
```

| 형태 | 코드 | 판정 |
|---|---|---|
| 완성품 | `helper.save(company, request, admin)` | ❌ 받는 쪽 결정권 없음 |
| 봉투 Command | `helper.save(Command.of(request, owner))` | △ 재료는 맞지만 번역이 없음 |
| **Request 직접** | `helper.save(request, owner)` | ✅ 기본값 |
| **평탄 Command + Factory** | `helper.save(factory.create(request, owner))` | ✅ 번역이 필요할 때 |

**평탄 Command로 올라가는 신호 셋** — ① 화면 필드명·타입이 도메인과 다르다 ② 입력 출처가 둘 이상이다(같은 팩토리를 화면 등록과 복제 양쪽에서 부른다) ③ 입력 단계에서 합치거나 변환할 게 있다. 셋 다 없으면 Request를 그대로 넘긴다. 실제로 내가 일하는 코드베이스도 Command 176개 중 Request를 품은 건 6개뿐이었고, 나머지는 전부 평탄 Command + Factory였다.

⚠️ 봉투를 만들고 나면 **리뷰에서 "이 Command 역할이 뭐죠?"를 다시 듣는다** — §0에서 Helper에게 들었던 것과 똑같은 문장. 경계를 못 만든 자리에는 이름을 붙여도 같은 질문이 온다.

📖 **"Command"라는 이름은 세 가지가 있다 — 여기선 첫째.**
| 이름 | 뜻 | 출처 |
|---|---|---|
| Command (입력 객체) | 쓰기 요청의 입력 DTO. `CreateOrderCommand`. **이 노트·이 볼트의 Command** | CQRS 관용구 → [변환 계층 §5](./transform-layers.md) |
| Command 패턴 | 요청을 **실행 가능한 객체**로 캡슐화. `execute()`를 가진다 | GoF |
| Replace Function with Command | 함수 하나를 객체로 바꿔 undo·단계 분리 등을 얻는 리팩터링 | Fowler, Refactoring 2판 |

⚠️ Command에 **검증 결과나 조회한 값**을 미리 채워 넘기는 것도 같은 문제(받는 쪽 결정 박탈) → [도메인 검증 위치 §5-1](./domain-validation.md).

---

## 4. 사례 ③ — 함수 추출 기준은 호출처 수가 아니라 "상위 함수가 읽히는 높이"

`save()` 안에 `newMasterUser(...)`(호출 1회)와 `newRepresentative(...)`(호출 1회)가 있었다. "한 번만 쓰는데 왜 뺐나" 싶어 `newRepresentative`만 인라인했더니 리뷰: *"둘이 같은 거 같은데요?"* — 하나는 함수, 하나는 필드 세팅 5줄이 `save()` 본문에 박힌 비대칭.

두 가지 기준이 있고 **동기가 다르다**:

| 기준 | 동기 | 언제 |
|---|---|---|
| **Rule of Three** (Don Roberts, Fowler 인용) | **중복 제거**. 처음엔 그냥 쓰고, 두 번째는 참고, 세 번째 중복에서 추출 | 같은 코드가 반복될 때 |
| **의도와 구현의 분리** (Fowler, *Extract Function* 첫 동기) | *"코드 조각을 보고 **무엇을 하는지 파악하는 데 노력이 들면** 추출하고, 함수 이름을 그 '무엇'으로 짓는다"* | **호출처가 1개라도** |

`save()`는 "회사 한 채를 세운다"는 높이에서 읽혀야 한다. 그 안에 `setId/setName/setEmail/...` 5줄이 있으면 읽는 사람이 "이게 뭐 만드는 거지"를 한 번 멈춰 파악해야 한다 → 추출 대상. 호출처가 1개라는 건 이유가 안 된다.

⚠️ **내가 실패한 지점**: 기준을 *"이름만 보고 넘어갈 수 있나"*로 들고 갔는데, 그 기준으로는 `newMasterUser`와 `newRepresentative`를 **가를 수 없다**(둘 다 이름만 보면 되니까). 가를 수 없는 두 개는 **같이 처리**해야 한다 — 둘 다 빼거나 둘 다 넣거나. 같은 높이의 재료는 같은 모양.

반대 방향의 예 — 삭제·게시 API 둘이 같은 코드를 썼다:
```java
List<Template> templates = mapper.selectAllByIds(ids);
if (templates.size() != ids.size()) throw new BizException("NOT_FOUND");
```
호출처 2곳(Rule of Three 미달)이지만 상위 함수 관점에서 "**검증된 목록을 얻는다**"가 한 관심사라 `findAllByIds(ids)`로 뺐다. 두 기준이 갈릴 때는 **상위 함수의 높이**가 우선.

---

## 5. 사례 ④ — 공통부가 고유부보다 작으면 합치지 않는다

대표자 이름 변경: 기존 행이 있으면 **복사 후 이름만 바꿔** 재삽입, 없으면 **새로 만든다**. 둘의 공통부(`setName`, `setUserId` 2줄)를 뽑아 `fill(target, orgId, userId, name)`로 합쳤다.

```java
// 실패한 시도
private void fill(Representative r, String orgId, String userId, String name) {
    r.setOrgId(orgId);     // 복사 경로에선 이미 채워져 있음 — 이쪽엔 불필요
    r.setUserId(userId);
    r.setName(name);
}
```

문제 두 가지:
- **공통 2줄 < 고유부**(복사 생성자 vs `new` + 필드 3개). 합쳐서 줄어든 게 없고 호출 형태만 늘었다.
- `fill`이 받는 `orgId`는 **한쪽 경로에서만 의미 있다**. 파라미터가 한쪽 사정을 드러낸다 — "이 함수는 두 경로에 걸쳐 있다"는 자백.

되돌렸다. Fowler의 *Duplicated Code* 냄새는 **같은 구조가 반복**될 때다. 표면상 같은 두 줄은 중복이 아니라 **우연의 일치**일 수 있고, 우연을 함수로 묶으면 나중에 한쪽만 바뀔 때 `if`가 생긴다.

---

## 6. 사례 ⑤ — 구현 수단으로 묶은 클래스는 기준이 아니다

인증 쪽 Helper 하나가 **세션 토큰 관리**와 **활성 회사 캐시**를 같이 갖고 있었다. 이유는 하나 — *둘 다 Redis를 쓴다*. 리뷰: *"얘의 역할이 뭘까요?"* (§0과 같은 문장이 다른 MR에서 또 나왔다.)

- 세션 정책(TTL·로그아웃 흔적)이 바뀌는 이유와 캐시 정책(무효화·히트 시 갱신 여부)이 바뀌는 이유는 **다르다** → SRP의 "변경 이유" 정의에 정확히 걸린다.
- `SessionStore` / `ActiveOrgResolver`로 나눴다. 둘 다 Redis를 쓰지만 그건 **어떻게(how)**이고 역할은 **무엇(what)**이다.

**같은 기술을 쓴다 · 같은 외부 시스템을 부른다 · 같은 어노테이션이 붙는다** — 이런 걸로 묶으면 "그리고"가 생긴다. 구현 수단은 경계 기준이 될 수 없다.

---

## 7. ⚠️ 함정 모음

- **이름을 바꿔서 풀려고 한다.** §0의 Helper는 `Saver` → `Helper` → `CoreCompanyHelper`로 이름을 세 번 바꿨지만 **경계가 바뀌기 전까지 지적은 같았다.** 이름은 경계의 *결과*다. 이름이 안 지어지면 경계가 없는 것.
- **어중간하게 옮긴다.** 필터에서 인증 판단 일부만 `Authenticator`로 옮기고 일부(SSE 제외·헤더 검증)는 남겼더니 *"어중간하게 남기지 말고 다 옮기세요"*. 반쯤 옮기면 **두 곳을 다 봐야** 해서 옮기기 전보다 나쁘다.
- **완성품 넘기기가 편해 보인다.** 파라미터가 줄어드니까. 하지만 줄어든 건 개수이고 늘어난 건 "만든 곳과 저장한 곳이 다르다"는 추적 비용(§3).
- **"한 가지 일만"으로 SRP를 읽는다.** → 클래스 파편화. 기준은 변경 이유(§1).
- **혼자 명확한 이름 vs 팀 컨벤션.** `DemoSessionStore`가 더 명확했지만 옆의 두 스택이 `XxxAuthenticationHelperService`라서 되돌렸다. 세 모듈이 같은 이름 체계를 쓰는 값이 한 클래스의 명확성보다 크다 — **경계는 내 코드 안에서만 아니라 옆 모듈과의 대구(對句)로도 읽힌다.**

---

## 8. 💡 판단 기준

**코드 쓰고 나서 던지는 셀프 질문 두 개** (리뷰어가 물을 것을 먼저 묻기):
1. **이 클래스를 한 문장으로 쓰면?** — "그리고"가 들어가면 의심. 문장이 안 나오면 경계가 없는 것.
2. **이 파라미터는 어디서 만들어져 오나?** — 완성품이면 받는 쪽은 저장밖에 못 한다. 재료인가 확인.

**기준 다섯 개** (실패한 자리와 함께):
| # | 기준 | 어디서 실패했나 |
|---|---|---|
| ① | 함수 추출은 **호출처 수가 아니라 상위 함수가 읽히는 높이**로. Rule of Three는 중복 제거용, 추출의 첫 동기는 의도/구현 분리 | §4 `newRepresentative` 비대칭 인라인 |
| ② | **공통부 < 고유부면 합치지 않는다.** 표면상 같은 두 줄은 우연일 수 있다 | §5 `fill()` |
| ③ | 한 사실의 **결정과 저장이 갈라지면 어느 쪽도 책임 못 진다** → 한 곳에 | §0·§2 Helper |
| ④ | **나눈 기준을 한 문장으로** 말할 수 있어야 한다. 구현 수단(Redis·HTTP)은 기준이 아니다 | §2 "core_*는 Helper, ext_*는 서비스" / §6 Redis Helper |
| ⑤ | **완성품을 넘기면 받는 쪽은 저장밖에 못 한다 — 재료를 넘긴다.** 재료는 Request 그대로여도 된다. Command 껍질은 **번역이 있을 때만** | §3 완성품 넘기기 / §3-1 봉투 Command |

**공부 방법** — 이 주제는 책보다 **리뷰 받고 "왜"를 끝까지 파는 것**이 효과가 크다. 책의 사례는 남의 코드라 내 코드에 안 붙는다. 굳이 하나 고르면 Fowler 『리팩터링』의 **냄새 카탈로그** — 원칙이 아니라 "이 모양이 보이면 의심"이라 바로 쓸 수 있다. 클린 아키텍처·DDD는 구체 경험이 쌓인 뒤가 낫다(먼저 읽으면 격언집이 된다 — README 트랙 6 주석과 같은 얘기).

---

## 9. 참고

- Martin Fowler, *Refactoring* 2nd ed. — [Extract Function](https://refactoring.com/catalog/extractFunction.html) (동기: intention vs implementation) · [Introduce Parameter Object](https://refactoring.com/catalog/introduceParameterObject.html) · Duplicated Code · Rule of Three (Don Roberts)
- Robert C. Martin, [The Single Responsibility Principle](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html) — "변경 이유" 재서술
- Craig Larman, *Applying UML and Patterns* — [GRASP](https://en.wikipedia.org/wiki/GRASP_(object-oriented_design)): Information Expert · Creator
- Rebecca Wirfs-Brock, *Object Design: Roles, Responsibilities, and Collaborations* — 책임 주도 설계(RDD)에서의 "책임" 정의
- Martin Fowler, [Tell Don't Ask](https://martinfowler.com/bliki/TellDontAsk.html) — 데이터를 꺼내 밖에서 결정하지 말고 객체에게 시킨다 (§3 "재료를 넘긴다"의 뿌리)

**학습 날짜**: 2026-09-02 · **계기**: 회사 프로젝트 MR 두 건에서 연속으로 *"이 클래스 역할이 뭘까요?"*를 받음 — 인증 Helper(세션+캐시, Redis라서 묶음)와 등록 Helper(엔티티 생성은 서비스·저장은 Helper로 갈라짐). 같은 날 함수 추출·인라인 기준을 놓고 두 번 되돌린 뒤(비대칭 인라인 · `fill` 공통 추출) 실패 지점을 기준 다섯 개로 정리. 사례 코드는 실제 구조를 단순화한 것(`core_*`/`ext_*`).

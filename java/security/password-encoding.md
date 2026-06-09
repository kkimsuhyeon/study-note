# 비밀번호 — PasswordEncoder(단방향 해시) vs AttributeConverter(양방향 암호화)

> **한 줄 요약**: 비밀번호는 **`PasswordEncoder`로 단방향 해시 + `matches()`로 검증**이 정답이고, JPA `AttributeConverter`는 **비번엔 부적합**하다. 컨버터는 "저장 시 변환 ↔ 로드 시 역변환"의 **양방향**이라 *다시 읽어와야 하는 민감정보(PII) 암호화*용. 갈림길은 **"복호화할 거냐"** — 비번은 되돌릴 일이 없으니(단방향) 해시, 전화번호·주민번호처럼 다시 보여줘야 하면(양방향) 암호화.

관련 노트: [도메인 검증 위치 §4-1 규칙 vs 메커니즘](../design/domain-validation.md) (인코딩=메커니즘→앱 계층)

---

## 1. 둘은 푸는 문제가 다르다

| | `PasswordEncoder` (Spring Security) | `AttributeConverter` (JPA) |
|---|---|---|
| 방향 | **단방향 해시** (복호화 불가) | **양방향** (저장 시 암호화 ↔ 로드 시 복호화) |
| 용도 | **비밀번호** | 다시 읽어와야 하는 민감정보(주민번호·전화·계좌) **암호화** |
| 검증 | `matches(raw, hash)` | 복호화해서 평문 비교 |
| 저장값 | bcrypt/argon2 해시 (매번 salt 다름) | AES 등 암호문 (키로 복원) |

---

## 2. 비밀번호 — `PasswordEncoder` (표준 패턴)

```java
// 1) 설정 — DelegatingPasswordEncoder 권장
@Bean
PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    // 저장 포맷에 {bcrypt}$2a$... 처럼 알고리즘 태그가 붙음
}

// 2) 가입 — 앱 서비스에서 명시적으로 encode
String encoded = passwordEncoder.encode(command.getPassword());
User user = User.create(email, encoded);
userRepository.save(user);

// 3) 로그인 — matches로 검증 (다시 encode해서 비교 ❌)
if (!passwordEncoder.matches(raw, user.getPassword())) {
    throw new BusinessException(INCORRECT_PASSWORD);
}
```

- **`DelegatingPasswordEncoder`**: 저장값이 `{bcrypt}$2a$10$...`처럼 알고리즘 접두사를 가져, 나중에 argon2 등으로 **점진 마이그레이션**이 가능(기존 해시도 그대로 검증). 요즘 스프링 시큐리티 기본 권장.
- **위치 = 앱 서비스**(`AuthService`). 인코딩은 보안 *메커니즘*이라 도메인(엔티티/도메인서비스)에 넣지 않는다 — `PasswordEncoder`는 스프링 시큐리티 인프라라 도메인이 알면 순수성이 깨진다. (→ [domain-validation §4-1](../design/domain-validation.md): 메커니즘은 앱/인프라)

---

## 3. ⚠️ 왜 AttributeConverter는 비번에 쓰면 안 되나 (핵심 3가지)

1. **단방향이라 round-trip 계약이 깨진다.** 컨버터는 `convertToDatabaseColumn`(저장)과 `convertToEntityAttribute`(로드)가 **역연산 쌍**이어야 하는데, 해시는 되돌릴 수 없어 로드 시 평문 복원이 불가능 → 설계 자체가 성립 안 함.
2. **검증을 못 한다.** bcrypt/argon2는 **매번 salt가 달라** `encode("1234")`가 호출마다 다른 값. 그래서 검증은 "다시 encode해서 문자열 비교"가 아니라 **`matches(raw, storedHash)`**(저장 해시에서 salt 추출해 비교)여야 하는데, 컨버터엔 이 API가 없다.
3. **이중 해시 사고.** 컨버터가 저장마다 해시하면, 로드 후 재저장 시 **해시를 또 해시**(hash of hash) → 로그인 영구 불가. 인코딩이 암묵적·은닉돼 흐름도 안 보임.

---

## 4. AttributeConverter는 *언제* 쓰나 — PII 양방향 암호화

```java
@Converter
public class CryptoConverter implements AttributeConverter<String, String> {
    public String convertToDatabaseColumn(String plain) { return aes.encrypt(plain); } // 저장
    public String convertToEntityAttribute(String enc)  { return aes.decrypt(enc); }   // 로드(복원 가능)
}
// 전화번호·주민번호·계좌 등 — 화면에 다시 보여줘야 하니 복호화 필요 → 양방향이 맞음
```
- "값을 다시 봐야 함 + 역연산 가능" 케이스에 딱.
- ⚠️ 단점: 암호문으로 저장돼 **WHERE 부분검색·정렬·인덱스가 깨짐**, 키 관리 필요. "꼭 암호화 + 다시 읽어야" 하는 필드에만.

---

## 5. 💡 판단 기준

| 상황 | 선택 |
|---|---|
| 비밀번호 (되돌릴 일 없음) | **`PasswordEncoder`** + `matches()`, 앱 계층에서 |
| 알고리즘 교체 대비 | `DelegatingPasswordEncoder` (`{bcrypt}` 태그) |
| 다시 읽어야 하는 PII (전화·주민번호) | **`AttributeConverter`** (양방향 암호화) |
| 검색·정렬해야 하는 필드 | 암호화 지양 (인덱스 깨짐) — 토큰화/부분 마스킹 검토 |

> 한 줄: **"복호화할 거냐"가 갈림길.** 안 되돌림(비번)=단방향 해시(`PasswordEncoder`+`matches`), 다시 읽음(PII)=양방향 암호화(`AttributeConverter`). 비번을 컨버터로 하면 검증 불가·이중해시로 깨진다. 인코딩 호출 위치는 **앱 서비스**(메커니즘이므로 도메인 밖).

---

## 6. 참고
- [Spring Security - PasswordEncoder](https://docs.spring.io/spring-security/reference/features/authentication/password-storage.html)
- 관련 노트: [도메인 검증 위치 §4-1](../design/domain-validation.md)

---

**학습 날짜**: 2026-06-09
**계기**: `UserRegistration`/`AuthService`에서 비번 인코딩을 어디서·어떻게 할지 보다가 — "직접 `PasswordEncoder` 호출 vs 예전에 쓰던 `AttributeConverter` 중 뭐가 맞나" 의문. 결론: 비번=단방향 해시(`PasswordEncoder`), 컨버터는 양방향 암호화(PII)용이라 비번엔 부적합(검증·이중해시 문제).

# Jackson 어노테이션 종합 정리

> Jackson은 Java 객체 ↔ JSON 변환 라이브러리. Spring Boot에서 `@RequestBody`, `@ResponseBody`(=`@RestController`) 동작 시 내부적으로 사용됨.

## 목차
1. [기본 개념](#1-기본-개념)
2. [@JsonProperty - 필드명 매핑](#2-jsonproperty)
3. [@JsonIgnore / @JsonIgnoreProperties - 필드 제외](#3-jsonignore--jsonignoreproperties)
4. [@JsonInclude - null/empty 처리](#4-jsoninclude)
5. [@JsonFormat - 날짜/숫자 포맷](#5-jsonformat)
6. [@JsonCreator - 역직렬화 생성자](#6-jsoncreator)
7. [@JsonAlias - 여러 키 허용](#7-jsonalias)
8. [@JsonNaming - 네이밍 전략](#8-jsonnaming)
9. [@JsonUnwrapped - 중첩 평탄화](#9-jsonunwrapped)
10. [@JsonSerialize / @JsonDeserialize - 커스텀 변환](#10-jsonserialize--jsondeserialize)
11. [@JsonTypeInfo / @JsonSubTypes - 다형성](#11-jsontypeinfo--jsonsubtypes)
12. [실전 예시 - Spring DTO](#12-실전-예시---spring-dto)

---

## 1. 기본 개념

### 직렬화(Serialize) vs 역직렬화(Deserialize)
- **직렬화**: Java 객체 → JSON (응답 보낼 때)
- **역직렬화**: JSON → Java 객체 (요청 받을 때)

### 기본 동작
- Jackson은 기본적으로 **getter/setter** 또는 **public 필드**를 기준으로 매핑
- 기본 생성자(no-arg constructor) 필요 (또는 `@JsonCreator` 사용)
- 필드명 = JSON 키 (camelCase)

```java
public class User {
    private String name;
    private int age;
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
}
// → {"name": "kim", "age": 30}
```

---

## 2. @JsonProperty

**용도**: JSON 키 이름과 Java 필드명이 다를 때 매핑.

```java
public class User {
    @JsonProperty("user_name")
    private String name;

    @JsonProperty("user_age")
    private int age;
}
// → {"user_name": "kim", "user_age": 30}
```

### 옵션
| 옵션 | 설명 |
|------|------|
| `value` | JSON 키 이름 |
| `required = true` | 역직렬화 시 해당 키 없으면 예외 |
| `access` | READ_ONLY (직렬화만), WRITE_ONLY (역직렬화만) |

```java
@JsonProperty(value = "password", access = Access.WRITE_ONLY)
private String password;  // JSON으로 출력은 안 되고, 입력만 받음
```

---

## 3. @JsonIgnore / @JsonIgnoreProperties

### @JsonIgnore (필드 단위)
특정 필드를 JSON에서 **완전히 제외** (양방향).

```java
public class User {
    private String name;

    @JsonIgnore
    private String password;  // 직렬화/역직렬화 모두 무시
}
```

**용례**: 응답에서 비밀번호, 내부 ID 등 노출하지 않을 때.

### @JsonIgnoreProperties (클래스 단위)
클래스에서 여러 필드를 한 번에 무시. **알 수 없는 JSON 키 무시**에도 자주 사용.

```java
@JsonIgnoreProperties({"password", "createdAt"})
public class User { ... }

// 또는 알 수 없는 키 모두 무시
@JsonIgnoreProperties(ignoreUnknown = true)
public class User { ... }
```

> `ignoreUnknown = true`는 외부 API 응답을 받을 때 매우 자주 사용. 키가 추가되어도 역직렬화 깨지지 않음.

---

## 4. @JsonInclude

**용도**: 특정 조건(null, empty 등)인 필드는 JSON 출력에서 제외.

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public class User {
    private String name;
    private String email;  // null이면 출력 안 됨
}
```

### Include 옵션
| 값 | 제외 조건 |
|----|-----------|
| `ALWAYS` | (기본) 항상 포함 |
| `NON_NULL` | null이면 제외 |
| `NON_EMPTY` | null, 빈 문자열, 빈 컬렉션 제외 |
| `NON_DEFAULT` | 기본값(0, false 등)이면 제외 |
| `NON_ABSENT` | null + `Optional.empty()` 제외 |

### 필드별 적용도 가능
```java
public class User {
    private String name;

    @JsonInclude(JsonInclude.Include.NON_NULL)
    private String email;
}
```

### 전역 설정 (Spring Boot)
```java
@Configuration
public class JacksonConfig {
    @Bean
    public Jackson2ObjectMapperBuilderCustomizer customizer() {
        return builder -> builder.serializationInclusion(JsonInclude.Include.NON_NULL);
    }
}
```

---

## 5. @JsonFormat

**용도**: 날짜, 숫자 등의 포맷을 지정.

### 날짜 포맷
```java
public class Event {
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Seoul")
    private LocalDateTime startAt;
}
// → "startAt": "2026-05-25 14:30:00"
```

### 숫자 포맷
```java
@JsonFormat(shape = JsonFormat.Shape.STRING)
private long bigNumber;  // 숫자를 문자열로 출력 (JS의 Number 정밀도 문제 회피)
```

### Enum 포맷
```java
public class Order {
    @JsonFormat(shape = JsonFormat.Shape.OBJECT)
    private Status status;  // 기본은 "ACTIVE", OBJECT 지정 시 {"name": "ACTIVE", ...}
}
```

> Spring Boot 2.0+ 기본 설정: `LocalDateTime`은 ISO-8601 형식. 한국 서비스에서는 보통 `@JsonFormat`으로 패턴 지정.

---

## 6. @JsonCreator

**용도**: Jackson이 어떤 생성자로 역직렬화할지 지정. **불변 객체(immutable)** 만들 때 핵심.

```java
public class User {
    private final String name;
    private final int age;

    @JsonCreator
    public User(
        @JsonProperty("name") String name,
        @JsonProperty("age") int age
    ) {
        this.name = name;
        this.age = age;
    }
}
```

→ 기본 생성자 없어도 역직렬화 가능. setter 없어도 됨.

### 정적 팩토리 메서드에도 사용 가능
```java
public class UserId {
    private final long value;

    private UserId(long value) { this.value = value; }

    @JsonCreator
    public static UserId of(long value) {
        return new UserId(value);
    }
}
```

### 단일 값 생성자 (delegating)
```java
public class Money {
    private final long amount;

    @JsonCreator
    public Money(long amount) {  // @JsonProperty 없음 → 단일 값으로 받음
        this.amount = amount;
    }
}
// JSON: 1000 → Money(1000)
```

> **Java record**와 함께 쓰면 Jackson 2.12+에서는 자동 인식 (어노테이션 없이도 동작).

---

## 7. @JsonAlias

**용도**: 역직렬화 시 **여러 키 이름**을 허용. (직렬화는 `@JsonProperty` 이름으로만 됨)

```java
public class User {
    @JsonProperty("name")
    @JsonAlias({"userName", "user_name", "nm"})
    private String name;
}
```

→ JSON에 `name`, `userName`, `user_name`, `nm` 중 무엇이 와도 매핑됨.

**용례**: 외부 API가 키 이름을 바꾸는 경우, 또는 여러 버전 호환.

---

## 8. @JsonNaming

**용도**: 클래스 전체에 네이밍 전략 적용. (snake_case, kebab-case 등)

```java
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
public class User {
    private String firstName;
    private String lastName;
}
// → {"first_name": "...", "last_name": "..."}
```

### 주요 전략
| 전략 | 결과 |
|------|------|
| `SnakeCaseStrategy` | `first_name` |
| `KebabCaseStrategy` | `first-name` |
| `UpperCamelCaseStrategy` | `FirstName` |
| `LowerCaseStrategy` | `firstname` |

### 전역 설정 (Spring Boot application.yml)
```yaml
spring:
  jackson:
    property-naming-strategy: SNAKE_CASE
```

> 백엔드는 camelCase, 외부 API는 snake_case를 쓰는 경우가 많아 자주 사용됨.

---

## 9. @JsonUnwrapped

**용도**: 중첩 객체를 **평탄하게** 출력.

```java
public class User {
    private String name;

    @JsonUnwrapped
    private Address address;
}

public class Address {
    private String city;
    private String zipCode;
}
```

**평소 출력**:
```json
{"name": "kim", "address": {"city": "Seoul", "zipCode": "12345"}}
```

**@JsonUnwrapped 적용**:
```json
{"name": "kim", "city": "Seoul", "zipCode": "12345"}
```

### prefix/suffix 옵션
```java
@JsonUnwrapped(prefix = "addr_")
private Address address;
// → {"name": "...", "addr_city": "...", "addr_zipCode": "..."}
```

---

## 10. @JsonSerialize / @JsonDeserialize

**용도**: 커스텀 (역)직렬화 로직 지정.

### 커스텀 직렬화 예시
```java
public class MoneySerializer extends JsonSerializer<Money> {
    @Override
    public void serialize(Money money, JsonGenerator gen, SerializerProvider provider) throws IOException {
        gen.writeString(money.getAmount() + " KRW");
    }
}

public class Order {
    @JsonSerialize(using = MoneySerializer.class)
    private Money price;
}
// → "price": "10000 KRW"
```

### 커스텀 역직렬화 예시
```java
public class MoneyDeserializer extends JsonDeserializer<Money> {
    @Override
    public Money deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        String text = p.getValueAsString();
        long amount = Long.parseLong(text.replace(" KRW", ""));
        return new Money(amount);
    }
}

public class Order {
    @JsonDeserialize(using = MoneyDeserializer.class)
    private Money price;
}
```

**용례**: 도메인 값 객체(VO)를 외부 표현으로 변환할 때, 암호화/복호화, 마스킹 등.

---

## 11. @JsonTypeInfo / @JsonSubTypes

**용도**: **다형성** 객체 직렬화/역직렬화. 부모 타입으로 받지만 실제 자식 타입을 보존해야 할 때.

```java
@JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME,
    include = JsonTypeInfo.As.PROPERTY,
    property = "type"
)
@JsonSubTypes({
    @JsonSubTypes.Type(value = CreditCard.class, name = "credit"),
    @JsonSubTypes.Type(value = BankTransfer.class, name = "bank")
})
public abstract class PaymentMethod {
    private long amount;
}

public class CreditCard extends PaymentMethod {
    private String cardNumber;
}

public class BankTransfer extends PaymentMethod {
    private String accountNumber;
}
```

**JSON 입력**:
```json
{"type": "credit", "amount": 10000, "cardNumber": "1234-..."}
```
→ 자동으로 `CreditCard` 인스턴스로 역직렬화됨.

**`use` 옵션**:
| 값 | 동작 |
|----|------|
| `NAME` | `@JsonSubTypes`의 name 사용 (위 예시) |
| `CLASS` | 풀패키지 클래스명 (보안 위험, 추천 X) |
| `MINIMAL_CLASS` | 짧은 클래스명 |

> Spring Boot에서 이벤트, 결제 수단 등 다형성 DTO를 다룰 때 필수.

---

## 12. 실전 예시 - Spring DTO

### 요청 DTO (역직렬화)
```java
@JsonIgnoreProperties(ignoreUnknown = true)  // 모르는 키는 무시
public class CreateUserRequest {

    @JsonProperty("user_name")
    @JsonAlias("userName")  // 둘 다 허용
    private String name;

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    private String password;  // 응답에서는 절대 안 나가게

    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate birthDate;

    @JsonCreator
    public CreateUserRequest(
        @JsonProperty("user_name") String name,
        @JsonProperty("password") String password,
        @JsonProperty("birthDate") LocalDate birthDate
    ) {
        this.name = name;
        this.password = password;
        this.birthDate = birthDate;
    }
}
```

### 응답 DTO (직렬화)
```java
@JsonInclude(JsonInclude.Include.NON_NULL)  // null 필드는 응답에서 제외
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)  // snake_case로 응답
public class UserResponse {

    private Long id;
    private String userName;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Seoul")
    private LocalDateTime createdAt;

    @JsonIgnore
    private String internalId;  // 응답에 절대 포함 안 됨
}
```

---

## 자주 만나는 함정

### 1. 기본 생성자 누락
```
JsonMappingException: No default constructor found
```
→ `@JsonCreator` 사용하거나 기본 생성자 추가.

### 2. Lombok과 충돌
`@Builder`만 있고 `@NoArgsConstructor`가 없으면 역직렬화 실패.
→ `@NoArgsConstructor` + `@AllArgsConstructor` 같이 쓰거나 `@JsonCreator` 적용.

### 3. LocalDateTime 직렬화 에러
```
InvalidDefinitionException: Java 8 date/time type not supported
```
→ `jackson-datatype-jsr310` 의존성 추가 (Spring Boot는 기본 포함).

### 4. 알 수 없는 키 에러
```
UnrecognizedPropertyException: Unrecognized field "xxx"
```
→ `@JsonIgnoreProperties(ignoreUnknown = true)` 또는 전역 설정:
```yaml
spring:
  jackson:
    deserialization:
      fail-on-unknown-properties: false
```

---

## 참고
- [Jackson Annotations 공식 GitHub](https://github.com/FasterXML/jackson-annotations)
- [Baeldung - Jackson Annotation Examples](https://www.baeldung.com/jackson-annotations)
- [Jackson Databind 공식 문서](https://github.com/FasterXML/jackson-databind)

---

**학습 날짜**: 2026-05-25
**계기**: 예전에 정리했다가 잃어버린 Jackson 어노테이션 노트를 study-note 레포에서 영구 보관

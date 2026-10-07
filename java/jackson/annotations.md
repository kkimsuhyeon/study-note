# Jackson 어노테이션 종합 정리

> Jackson은 Java 객체 ↔ JSON 변환 라이브러리. Spring Boot에서 `@RequestBody`, `@ResponseBody`(=`@RestController`) 동작 시 내부적으로 사용됨.

> **버전 기준**: 따로 표시하지 않은 절은 **Jackson 2.x(`com.fasterxml.jackson.databind`) + Spring Boot 2·3** 기준이다. Jackson 3(`tools.jackson`, Spring Boot 4)에서 이름이 바뀌는 곳은 `Jackson 3:` 줄로 단다. 어노테이션 패키지(`com.fasterxml.jackson.annotation`)는 Jackson 3에서도 그대로다.

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
- **진입점은 `ObjectMapper`** — 실제 변환(`readValue` / `writeValueAsString`)을 수행하는 객체. Spring Boot가 자동 구성해 `@RequestBody`/`@ResponseBody`가 이걸 쓴다. 아래 전역 설정(네이밍·날짜 등)도 결국 `ObjectMapper`를 구성하는 것.

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

> `ignoreUnknown = true`는 외부 API 응답을 받을 때 매우 자주 사용. 키가 추가되어도 역직렬화 깨지지 않음. 단 Spring Boot가 구성한 mapper와 Jackson 3는 이미 "모르는 키 무시"가 기본이다(→ 함정 4).

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
Jackson 3 / Boot 4: 커스터마이저가 `JsonMapperBuilderCustomizer`로 바뀐다(§5 예시). `Jackson2ObjectMapperBuilderCustomizer`는 Boot 4에서 deprecated된 Jackson 2 지원 모듈(`spring-boot-jackson2`) 쪽 경로다.

---

## 5. @JsonFormat

**용도**: 날짜, 숫자 등의 포맷을 지정.

### 날짜 포맷
```java
public class Event {
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime startAt;
}
// → "startAt": "2026-05-25 14:30:00"
```
⚠️ `timezone` 옵션은 `Date`·`Instant`·`ZonedDateTime`처럼 **순간을 가진 타입**을 지역 시각으로 바꿀 때 쓰인다. `LocalDateTime`은 시간대가 없어서 `timezone = "Asia/Seoul"`을 붙여도 값이 바뀌지 않는다 — Jackson은 포매터에 `withZone`만 걸고, `DateTimeFormatter`는 순간이 없는 값을 변환하지 않는다. "서울 시각으로 내보낸다"는 착각을 만드니 붙이지 않는다(→ [Clock·Instant·LocalDateTime](../basics/java-time-clock-instant-localdatetime.md)).

### LocalTime을 요청에서 직접 받기 — Jackson 3.1.5 기준

```java
import java.time.LocalTime;
import com.fasterxml.jackson.annotation.JsonFormat;
import com.fasterxml.jackson.annotation.OptBoolean;

public record AppointmentRequest(
        @JsonFormat(shape = JsonFormat.Shape.STRING,
                pattern = "HH:mm", lenient = OptBoolean.FALSE)
        LocalTime appointmentTime
) {
}
```

```json
{"appointmentTime":"07:30"}
```

`@RequestBody AppointmentRequest request`로 받으면 Jackson이 문자열을 `LocalTime`으로 변환한다. 서비스에서 다시 `LocalTime.parse()`할 필요가 없다. `HH`는 24시간제 시, `mm`는 분이다. `MM`은 월이므로 구분한다.

**`@JsonFormat`은 `@Valid`를 대신하지 않는다.** 변환은 Jackson이, null 필수 여부 같은 검증은 `@NotNull`과 `@Valid`가 담당한다. “시간 미상이라면 시간은 null이어야 한다” 같은 필드 간 규칙은 별도로 검증한다. [Validation](../spring/validation.md)

⚠️ **`shape = STRING`을 썼다고 모든 비문자열 입력이 거부된다고 단정하지 않는다.** Jackson 3.1.5의 표준 `LocalTimeDeserializer`는 `[7, 30]` 배열도 읽는다. `pattern`은 문자열 파싱 형식이고, `lenient = FALSE`는 문자열의 날짜·시간 해석을 엄격하게 하는 데 쓰인다. “JSON 토큰 종류까지 문자열만 허용”하는 규칙과는 다르다.

클라이언트가 `"07:30"`을 보내도록 약속한 서비스라면 먼저 표준 변환을 사용하면 된다. 외부 계약이 배열 거부까지 요구할 때만 추가 처리를 검토한다. 이 노트의 판단은 Jackson 3.1.5 소스 확인 기준이며 다른 버전의 모든 역직렬화기에 일반화하지 않는다.

### enum의 Java 이름과 JSON 표기를 구분하기

Java에서는 `ONLINE`, JSON에서는 `"online"`처럼 표기를 다르게 할 수 있다. 방법은 목적에 따라 고른다.

| 방법 | 적합한 경우 | 영향 범위 |
| --- | --- | --- |
| enum의 `@JsonValue` | 값별 외부 표기가 정해져 있음 | 해당 enum의 Jackson 표현 |
| mapper의 enum naming strategy | 모두 소문자처럼 공통 규칙이 있음 | 그 mapper가 처리하는 enum 전반 |

Jackson 의존성을 도메인 enum에 두지 않으려면 설정에서 공통 규칙을 적용하는 선택이 있다. Spring Boot 4.1 / Jackson 3.1.5 예시:

```java
import org.springframework.boot.jackson.autoconfigure.JsonMapperBuilderCustomizer;
import org.springframework.context.annotation.Bean;
import tools.jackson.databind.EnumNamingStrategies;
import tools.jackson.databind.cfg.EnumFeature;

@Bean
JsonMapperBuilderCustomizer enumFormat() {
    return builder -> builder
            .enumNamingStrategy(EnumNamingStrategies.LOWER_CASE)
            .enable(EnumFeature.FAIL_ON_NUMBERS_FOR_ENUMS);
}
```

설정 클래스 안에 두는 메서드다. **전체 mapper 설정이므로 다른 enum의 응답 형식도 확인**한다. 기존 문자열이 저장된 JSON을 enum 필드로 읽을 수 있는지, 다시 저장해도 소문자가 유지되는지 테스트한다. 숫자로 enum 순번을 받지 않으려는 설정은 Jackson 3에서 `EnumFeature.FAIL_ON_NUMBERS_FOR_ENUMS`다. Jackson 2의 설정 위치와 혼동하지 않는다.

`Request`에만 시간 포맷을 달아도 `Command`나 도메인 객체의 저장 JSON에 그 어노테이션이 복사되지는 않는다. 별도 응답 DTO에 형식을 지정하거나, 공통 포맷이 필요할 때 해당 mapper의 `LocalTime` 형식을 설정한다. 타입 전달 자체는 [변환 계층 노트](../design/transform-layers.md)를 참고한다.

확인 근거: 설치된 Jackson 3.1.5 소스의 `LocalTimeDeserializer`, `MapperBuilder`, `EnumNamingStrategies`, `EnumFeature`. 보강일: 2026-09-22. 표준 시간 바인딩·enum·저장 JSON 왕복은 프로젝트 테스트로 확인했으며 위 학습용 설정 코드는 별도 실행하지 않았다.

- [Jackson 3.1.5 LocalTimeDeserializer](https://github.com/FasterXML/jackson-databind/blob/jackson-databind-3.1.5/src/main/java/tools/jackson/databind/ext/javatime/deser/LocalTimeDeserializer.java)
- [Jackson 3.1.5 EnumNamingStrategies](https://github.com/FasterXML/jackson-databind/blob/jackson-databind-3.1.5/src/main/java/tools/jackson/databind/EnumNamingStrategies.java)
- [Jackson 3.1.5 EnumFeature](https://github.com/FasterXML/jackson-databind/blob/jackson-databind-3.1.5/src/main/java/tools/jackson/databind/cfg/EnumFeature.java)

💡 **약속한 시간 문자열을 Java 시간 타입으로 받는 일은 Jackson에 맡긴다. 별도 검증은 실제 업무 규칙과 외부 계약에서 요구하는 범위만 추가한다.**

### 숫자 포맷
```java
@JsonFormat(shape = JsonFormat.Shape.STRING)
private long bigNumber;  // 숫자를 문자열로 출력 (JS의 Number 정밀도 문제 회피)
```

### Enum 포맷
```java
// 기본: enum은 이름 문자열로 직렬화 → "status": "ACTIVE"
// Shape.OBJECT: enum을 객체처럼 — 그 enum이 가진 getter/필드를 펼친다
//   (자동으로 {"name":...}이 되는 게 아니라, Status에 있는 프로퍼티가 나온다)
public enum Status {
    ACTIVE("활성"), INACTIVE("비활성");
    private final String label;   // ← OBJECT면 {"label":"활성"} 식으로 나옴 (getter 기준)
}
```
- enum 직렬화 형태를 **모든 곳에서** 바꾸려면 `@JsonFormat`을 **enum 타입 선언부**에 붙인다(필드마다 붙이는 것보다 일관).
- 값 하나(예: `label`)로만 내보내고 싶으면 `Shape.OBJECT`보다 **`@JsonValue`(§6 하단)**가 깔끔하다.

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

    @JsonCreator(mode = JsonCreator.Mode.DELEGATING)  // "값 하나 통째로"를 명시
    public Money(long amount) {
        this.amount = amount;
    }
}
// JSON: 1000 → Money(1000)
```
⚠️ 인자 하나짜리 `@JsonCreator`에서 `mode`를 생략하면 Jackson이 휴리스틱으로 delegating(`1000`)과 properties(`{"amount":1000}`) 중 하나를 고른다. 파라미터 이름 정보(`-parameters` 컴파일 + ParameterNamesModule, Spring Boot 기본)가 있으면 properties로 해석될 수 있으니 `mode`를 명시한다 (버전별 휴리스틱 세부는 확인 필요).

> **Java record**와 함께 쓰면 Jackson 2.12+에서는 자동 인식 (어노테이션 없이도 동작).

### `@JsonValue` — 객체를 값 하나로 직렬화 (`@JsonCreator`의 짝)

`@JsonCreator`가 "값 하나 → 객체"(역직렬화)라면, `@JsonValue`는 "객체 → 값 하나"(직렬화). 한 메서드/필드에 붙이면 그 값으로 통째 직렬화된다.
```java
public enum Status {
    ACTIVE("활성"), INACTIVE("비활성");
    private final String label;
    @JsonValue public String getLabel() { return label; }   // → "활성" (이름 "ACTIVE" 대신)
}
```
- VO를 JSON에서 평범한 값처럼 보이게 할 때 유용: `Money`를 `@JsonValue`로 `amount`만 내보내고 `@JsonCreator`로 다시 받으면 **왕복**이 맞는다.
- ⚠️ 한 타입에 `@JsonValue`는 **하나만**. (enum 포맷도 위 `Shape.OBJECT`보다 이게 깔끔)

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

Jackson 3: `JsonSerializer`→`ValueSerializer`, `JsonDeserializer`→`ValueDeserializer`, `SerializerProvider`→`SerializationContext`. 예외도 전부 unchecked(`JacksonException`)라 `throws IOException`이 빠진다.

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

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")  // LocalDateTime엔 timezone 효과 없음(§5)
    private LocalDateTime createdAt;

    @JsonIgnore
    private String internalId;  // 응답에 절대 포함 안 됨
}
```

---

## 자주 만나는 함정

### 1. 기본 생성자 누락
```
InvalidDefinitionException: Cannot construct instance of `User` (no Creators, like default constructor, exist)
```
→ `@JsonCreator` 사용하거나 기본 생성자 추가. (오래된 글의 "No default constructor found"는 옛 버전 메시지)
Jackson 3: 매핑 예외의 상위 타입이 `JsonMappingException`→`DatabindException`.

### 2. Lombok과 충돌
`@Builder`만 있고 `@NoArgsConstructor`가 없으면 역직렬화 실패.
→ `@NoArgsConstructor` + `@AllArgsConstructor` 같이 쓰거나 `@JsonCreator` 적용. Lombok 1.18.14+면 `@Builder` + **`@Jacksonized`**로 빌더를 역직렬화에 그대로 쓰는 게 가장 깔끔하다(불변 유지).

### 3. LocalDateTime 직렬화 에러
```
InvalidDefinitionException: Java 8 date/time type not supported
```
→ `jackson-datatype-jsr310` 의존성 추가 (Spring Boot는 기본 포함).
Jackson 3: `java.time` 지원이 databind에 내장돼 별도 모듈이 필요 없다.

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
⚠️ 이 에러는 주로 **`new ObjectMapper()`를 직접 만든 Jackson 2 코드**에서 난다. Spring의 `Jackson2ObjectMapperBuilder`(Boot가 구성하는 mapper)는 `FAIL_ON_UNKNOWN_PROPERTIES`를 기본으로 끄고, Jackson 3.0은 기본값 자체가 꺼짐이다. 반대로 오타 난 키가 조용히 무시되는 위험이 있으니, 외부 계약을 엄격히 지켜야 하는 요청 DTO는 켜는 것도 선택지다.

---

## 💡 판단 기준

- **한 DTO만 다르면 어노테이션, 모든 API 공통이면 전역 설정.** 전역 설정(`spring.jackson.*`·커스터마이저)은 다른 응답 형식까지 바꾸니 영향 범위를 먼저 확인한다(§5 enum 예시와 같은 판단).
- **불변 DTO는 `record` 먼저**(2.12+ 자동 인식), 클래스면 `@JsonCreator`(인자 하나면 `mode` 명시), Lombok 빌더면 `@Jacksonized`.
- **Jackson 어노테이션은 web DTO에 붙이고 도메인 객체엔 되도록 붙이지 않는다** — JSON 표기는 화면·외부 계약 사정이라, 도메인에 붙이면 계약 변경이 도메인을 흔든다([변환 계층 §5-1](../design/transform-layers.md)).

---

## 참고
- [Jackson Annotations 공식 GitHub](https://github.com/FasterXML/jackson-annotations)
- [Baeldung - Jackson Annotation Examples](https://www.baeldung.com/jackson-annotations)
- [Jackson Databind 공식 문서](https://github.com/FasterXML/jackson-databind)
- [Migrating to Jackson 3](https://github.com/FasterXML/jackson/blob/main/jackson3/MIGRATING_TO_JACKSON_3.md) — 패키지·클래스 이름 변경, unchecked 예외, 기본값 변경(`FAIL_ON_UNKNOWN_PROPERTIES` 꺼짐)
- [Spring `Jackson2ObjectMapperBuilder` Javadoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/converter/json/Jackson2ObjectMapperBuilder.html) — 기본으로 끄는 기능 목록 · [Spring Boot — JSON (Jackson 3 / Jackson 2 지원)](https://docs.spring.io/spring-boot/reference/features/json.html)
- [Lombok `@Jacksonized`](https://projectlombok.org/features/experimental/Jacksonized) · [`DateTimeFormatter.withZone`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/format/DateTimeFormatter.html#withZone(java.time.ZoneId)) (순간이 없는 값은 날짜·시각을 바꾸지 않음)

---

**학습 날짜**: 2026-05-25
**계기**: 예전에 정리했다가 잃어버린 Jackson 어노테이션 노트를 study-note 레포에서 영구 보관
**보강(2026-10-02)**: Jackson 2 기준 표기와 절마다 Jackson 3 대응 이름 추가, `LocalDateTime`의 `timezone` 무효, 인자 하나 `@JsonCreator`의 `mode` 명시, `@Jacksonized`, Spring·Jackson 3의 모르는 키 기본값, 💡 판단 기준 추가.

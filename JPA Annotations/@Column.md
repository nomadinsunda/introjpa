# 🧩 JPA `@Column` 애노테이션

> `@Column`은 JPA에서 **엔티티의 필드(멤버 변수)를 데이터베이스의 실제 컬럼과 매핑하는 핵심 애노테이션**입니다. 필드에 대한 구체적인 컬럼 특성을 지정할 수 있어, 데이터 정합성 및 테이블 구조 설계의 정밀도를 높입니다.

---

## ✅ 기본 개념

```java
@Column(name = "first_name")
private String firstName;
```

* 이 코드는 `firstName` 필드를 **DB 컬럼 `first_name`과 매핑**합니다.
* `@Column`을 생략하면, 기본적으로 **필드 이름과 동일한 컬럼 이름**이 사용됩니다 (`firstName` → `firstName`).

---

## ⚙️ 주요 속성 정리

| 속성                 | 타입        | 기본값    | 설명                                   |
| ------------------ | --------- | ------ | ------------------------------------ |
| `name`             | `String`  | 필드명    | 매핑할 컬럼 이름                            |
| `nullable`         | `boolean` | `true` | `NOT NULL` 여부                        |
| `insertable`       | `boolean` | `true` | `INSERT` 시 포함 여부                     |
| `updatable`        | `boolean` | `true` | `UPDATE` 시 포함 여부                     |
| `length`           | `int`     | `255`  | 문자열 컬럼 길이 (`VARCHAR`, `CHAR` 전용)     |
| `precision`        | `int`     | 0      | `DECIMAL`/`NUMERIC` 컬럼의 정밀도 (전체 자릿수) |
| `scale`            | `int`     | 0      | `DECIMAL`/`NUMERIC` 컬럼의 소수점 이하 자릿수   |
| `columnDefinition` | `String`  | 없음     | DDL 생성 시 사용될 SQL 조각                  |
| `table`            | `String`  | 주 테이블  | 필드를 다른 테이블과 매핑할 때 사용                 |

---

## 🧪 실습 예제: 모든 속성 사용

```java
@Column(
    name = "salary",
    nullable = false,
    insertable = true,
    updatable = false,
    precision = 10,
    scale = 2,
    columnDefinition = "DECIMAL(10,2) DEFAULT 1000.00"
)
private BigDecimal salary;
```

| 목적               | 설명                             |
| ---------------- | ------------------------------ |
| 컬럼 이름            | `salary`로 지정                   |
| NULL 허용 안 함      | `NOT NULL` 제약 추가               |
| 수정 불가            | UPDATE 문에서 제외                  |
| 정밀도              | 총 10자리, 소수점 이하 2자리             |
| columnDefinition | 생성되는 DDL에 `DEFAULT 1000.00` 포함 |

---

## 🔍 속성별 상세 설명

### 1. `name`

* DB 컬럼명과 자바 필드명을 일치시키지 않을 때 명시
* Snake-case로 바꾸는 데 유용 (`userName` → `user_name`)

### 2. `nullable`

* `false`이면 DDL 생성 시 `NOT NULL` 제약 조건 추가됨
* 실행 시점에 `null` 값이 들어가면 `ConstraintViolationException` 발생 가능

### 3. `insertable`, `updatable`

* **감사 정보, 시스템 필드 등 변경 금지 컬럼 처리**에 유용

```java
@Column(insertable = false, updatable = false)
private LocalDateTime createdAt;
```

→ DB에서 기본값 설정하고, Java에서는 손대지 않도록 함

### 4. `length`

* `VARCHAR`, `CHAR` 전용
* DB마다 기본값은 다르지만, 일반적으로 `255`

```java
@Column(length = 50)
private String nickName;
```

### 5. `precision` & `scale`

* `BigDecimal` 등 정밀한 숫자 타입에 사용
* `precision`: 전체 자릿수
* `scale`: 소수점 이하 자릿수

```java
@Column(precision = 12, scale = 4)
private BigDecimal exchangeRate; // 최대 99999999.9999
```

### 6. `columnDefinition`

* **DBMS에 직접 의존하는 SQL 조각** 지정
* 벤더 독립성을 포기하는 대신 세밀한 제어 가능

```java
@Column(columnDefinition = "TEXT")
private String biography;
```

### 7. `table`

* 이 필드가 주 테이블이 아닌 **secondary table**에 존재할 때 사용

```java
@Column(name = "profile_picture", table = "user_profile")
private String profilePicture;
```

→ `@SecondaryTable(name = "user_profile")`와 함께 사용해야 함

---

## 🧠 실무 팁

| 상황         | 추천 속성                                              |
| ---------- | -------------------------------------------------- |
| 생성일/수정일 필드 | `insertable=false`, `updatable=false` + DB DEFAULT |
| 대용량 문자열 저장 | `columnDefinition = "TEXT"`                        |
| 정밀한 금액 정보  | `precision`, `scale` 반드시 명시                        |
| 복합 테이블 분리  | `table` 속성 활용 + `@SecondaryTable` 매핑               |

---

## ⚠️ 주의사항

* `columnDefinition`은 DBMS에 따라 완전히 다르게 해석되므로 **이식성이 떨어집니다.**
* `precision/scale`은 `float`, `double`과는 무관하고 **`BigDecimal` 전용**입니다.
* `length`, `precision`, `scale` 등은 **DDL 생성 시점에만 영향**을 줍니다.
  → 즉, DB에 이미 생성된 테이블에는 반영되지 않음.

---

## 📘 정리

| 속성                        | 언제 사용하나?          |
| ------------------------- | ----------------- |
| `name`                    | 컬럼 이름 변경          |
| `nullable`                | NOT NULL 제약 설정    |
| `insertable`, `updatable` | 읽기 전용 필드 지정       |
| `length`                  | 문자열 필드 길이 조정      |
| `precision`, `scale`      | 금액, 정밀도 높은 숫자 필드  |
| `columnDefinition`        | DB에서 직접 DDL 타입 지정 |
| `table`                   | 서브 테이블에 필드 매핑할 때  |

---

## 📚 참고 자료

* [📖 Jakarta Persistence `@Column` 공식 문서](https://jakarta.ee/specifications/persistence/3.1/apidocs/jakarta.persistence/jakarta/persistence/Column.html)
* [📘 Hibernate User Guide: Column Mapping](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#mapping-column)

---

### 🎯 마무리

`@Column`은 단순히 필드명을 컬럼명과 맞추는 도구가 아니라, **정확한 스키마 제약을 명시하고, 데이터 정확성과 유지보수성을 높이기 위한 강력한 도구**입니다.
실제 운영 시스템에서 유효성 검증, 감사 처리, 읽기 전용 필드, 이력 테이블 설계 등에 `@Column`의 속성을 적극 활용하세요.

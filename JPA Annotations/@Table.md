# 🧩 JPA의 `@Table` 애노테이션

> `@Entity`가 객체를 영속성 컨텍스트에 등록하는 선언이라면, `@Table`은 그 객체가 어느 **테이블과 매핑**될지를 명확히 정의하는 맵핑 정보입니다.

---

## 📘 `@Table` 이란?

`@Table`은 JPA에서 **엔티티 클래스와 실제 데이터베이스 테이블 사이의 매핑**을 정의하기 위한 애노테이션입니다.

```java
@Entity
@Table(name = "STUDENT")
public class Student {
   ...
}
```

* `@Entity`는 이 클래스가 영속 엔티티임을 선언하고
* `@Table`은 이 클래스가 어떤 테이블과 연결될지를 명시합니다.

---

## 🧠 내부 구조: `@Table`의 속성

| 속성명                 | 타입                   | 기본값      | 설명                |
| ------------------- | -------------------- | -------- | ----------------- |
| `name`              | `String`             | 클래스명     | 매핑할 테이블 이름        |
| `schema`            | `String`             | (DB 설정값) | 스키마 이름            |
| `catalog`           | `String`             | (DB 설정값) | 카탈로그 이름           |
| `uniqueConstraints` | `UniqueConstraint[]` | 없음       | 테이블 수준의 유니크 제약 조건 |

> 카탈로그(Catalog) = 데이터베이스(Database) = 스키마(Schema)
> MySQL에서는 "스키마(schema)"와 "카탈로그(catalog)"가 사실상 동일하게 취급됩니다.
> 그래서 catalog를 명시적으로 설정하지 않아도, 내부적으로는 현재 선택된 스키마(=카탈로그) 가 자동으로 적용됩니다.

> 🔎 **주의:**
>
> * 이름은 **SQL 예약어**를 피해야 함 (`USER`, `ORDER` 등)
> * `schema`와 `catalog`는 DBMS에 따라 동작 방식이 다름 (Oracle은 schema 중심, MySQL은 catalog 중심)

---

## 🧾 예제: 실전 코드

```java
@Entity
@Table(
    name = "STUDENT",
    schema = "school_schema",
    catalog = "school_catalog",
    uniqueConstraints = {
        @UniqueConstraint(columnNames = {"fistName", "lastName"})
    }
)
public class Student {

    @Id
    @Column(name = "id")
    private int id;

    @Column(name = "fistName")
    private String firstName;

    @Column(name = "lastName")
    private String lastName;

    @Column(name = "marks")
    private int marks;

    // 생성자, Getter/Setter 생략
}
```

---

## 📌 속성별 심층 설명

### ① `name`

* 테이블 이름 지정
* 생략하면 클래스 이름을 그대로 사용

```java
@Table(name = "STUDENT") // 테이블 이름 명시적 지정
```

### ② `schema`

* 스키마 이름 지정 (Oracle, PostgreSQL 등에서 유용)
* `schema.table` 형태로 SQL 생성됨

```sql
SELECT * FROM school_schema.STUDENT;
```

### ③ `catalog`

* MySQL에서 주로 사용하는 **데이터베이스 이름**
* DB 연결이 다중 카탈로그일 때 구분자 역할

### ④ `uniqueConstraints`

* 하나 이상의 컬럼 조합에 대해 테이블 수준 유니크 제약을 설정
* `@Column(unique = true)`는 **컬럼 단위**, `@Table(uniqueConstraints)`는 **복합 유니크 키**에 사용

```java
@Table(uniqueConstraints = @UniqueConstraint(columnNames = {"firstName", "lastName"}))
```

---

## 🧠 JPA에서 `@Table`이 생략되면?

생략된 경우 JPA는 다음 규칙에 따라 테이블 이름을 자동 유추합니다:

| 조건           | 테이블 이름 유추 방식                              |
| ------------ | ----------------------------------------- |
| `@Table` 생략  | 클래스 이름 → 대소문자 구분 없이 적용됨 (DBMS 따라 다름)      |
| 엔티티가 상속받은 경우 | 슈퍼클래스에 `@Entity`가 없다면 자식 클래스 기준으로 테이블 생성됨 |

> 💡 하지만, 실무에서는 **명시적 정의를 권장**합니다. 자동 유추는 DB 이식성, 유지보수 측면에서 문제가 될 수 있습니다.

---

## ⚠️ 실무에서의 실수 사례

| 실수                                               | 설명                              |
| ------------------------------------------------ | ------------------------------- |
| `@Column(unique = true)`과 `@UniqueConstraint` 혼용 | 각자 동작 방식이 다르므로 정확한 이해 필요        |
| 스키마 또는 카탈로그 지정 누락                                | 멀티 테넌시 DB에서 스키마 혼동 가능성          |
| 테이블 이름에 예약어 사용                                   | `user`, `order` 등은 SQL 오류 유발 가능 |

---

## 🛠 실무 팁

### ✔ 테이블 네이밍 전략

* 클래스명과 테이블명을 명시적으로 매핑하되, 소문자+언더스코어 전략 사용 권장

```java
@Entity
@Table(name = "student_record")
public class Student { ... }
```

### ✔ 다중 컬럼 유니크 제약

```java
@Table(uniqueConstraints = {
    @UniqueConstraint(columnNames = {"email", "phone_number"})
})
```

→ 이메일과 전화번호 조합이 동일한 사용자 중복 방지

---

## 📚 정리

| 항목      | 내용                          |
| ------- | --------------------------- |
| 필수 여부   | 선택 사항 (`@Entity`만 있어도 동작함)  |
| 주 역할    | 엔티티 ↔ 테이블 이름, 스키마, 제약 조건 매핑 |
| 실무 권장사항 | 테이블명 명시, 복합 유니크 제약 적극 사용    |
| 기타      | 대소문자, 예약어 문제 대비 필요          |


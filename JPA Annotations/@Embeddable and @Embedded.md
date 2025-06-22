# 🧩 JPA의 `@Embeddable`과 `@Embedded` 어노테이션즈

> 단일 테이블에 여러 필드를 구성할 때, 중복되는 필드 구조를 클래스로 분리하여 코드 재사용성과 가독성을 높이는 JPA의 내장형(Embedded) 매핑 전략을 학습하자.

---

## ✅ 1. 개념 정리

| 애노테이션         | 설명                                                 |
| ------------- | -------------------------------------------------- |
| `@Embeddable` | **독립적인 엔티티가 아닌**, 다른 엔티티 안에 포함되어 함께 저장되는 클래스임을 나타냄 |
| `@Embedded`   | 해당 필드가 `@Embeddable` 클래스의 인스턴스를 포함하고 있음을 나타냄       |

즉, `@Embeddable`은 \*\*값 객체(Value Object)\*\*를 정의하고, `@Embedded`는 그것을 **엔티티에 포함시킵니다.**

---

## 🧠 핵심 개념

* **@Embeddable 클래스는 테이블로 매핑되지 않습니다.**
* 대신, 해당 클래스의 필드들이 **소속된 엔티티 테이블의 컬럼으로 확장**되어 저장됩니다.
* `EntityManager`는 `@Embeddable` 타입을 **직접 조회할 수 없습니다.**

  * 예: `SELECT a FROM Address a` → ❌ 불가능
  * 대신: `SELECT u.address FROM User u` → ✅ 가능

---

## 🧾 실전 예제

### 📘 `StudentAddress.java`

```java
@Embeddable
public class StudentAddress {
    private String city;
    private String state;
    private int zipCode;
    // Getter/Setter 생략
}
```

### 📙 `Student.java`

```java
@Entity
@Table(name = "STUDENT")
public class Student {

    @Id
    private int studentId;
    private String firstName;
    private String lastName;

    @Embedded
    private StudentAddress studentAddress;
}
```

### 💾 결과 테이블

```sql
CREATE TABLE STUDENT (
    STUDENTID INT PRIMARY KEY,
    FIRSTNAME VARCHAR(45),
    LASTNAME VARCHAR(45),
    CITY VARCHAR(45),
    STATE VARCHAR(45),
    ZIPCODE NUMERIC(15)
);
```

> 📌 `StudentAddress`는 테이블로 매핑되지 않지만, 필드들은 `STUDENT` 테이블에 컬럼으로 포함됩니다.

---

## 📦 `@EmbeddedId`와의 차이점

| 항목              | @Embedded              | @EmbeddedId                   |
| --------------- | ---------------------- | ----------------------------- |
| 용도              | 주소, 개인정보 등 서브 객체 필드 구성 | 복합키(Composite Primary Key) 구성 |
| @Id 필요 여부       | ❌ 없음                   | ✅ 반드시 `@Id` 대체                |
| Serializable 구현 | 선택 사항                  | 반드시 필요                        |
| equals/hashCode | 선택 사항                  | 반드시 override                  |

```java
@Embeddable
public class StudentPK implements Serializable {
    private int schoolId;
    private int rollNumber;
}
```

```java
@Entity
public class Student {
    @EmbeddedId
    private StudentPK id;
}
```

---

## 🎯 왜 사용하는가?

| 이유             | 설명                            |
| -------------- | ----------------------------- |
| **가독성 향상**     | 비슷한 필드들을 별도 클래스로 묶어 코드 구조 명확화 |
| **재사용성 향상**    | 동일한 구조를 다른 엔티티에서 재사용 가능       |
| **ORM 명세 일관성** | 복잡한 객체 → 단순 필드로 Flatten 하여 저장 |
| **설계 원칙 부합**   | 객체 지향의 컴포지션 설계를 JPA에서도 실현 가능  |

---

## 🔍 실무 적용 팁

| 상황                      | 전략                                                                 |
| ----------------------- | ------------------------------------------------------------------ |
| 사용자 주소가 여러 테이블에서 공통 사용됨 | `@Embeddable Address` 정의 후 각 엔티티에 `@Embedded`                      |
| 생성일/수정일 공통 필드           | `@Embeddable BaseTime` + `@MappedSuperclass` 또는 `@EntityListeners` |
| 복합키                     | 반드시 `@Embeddable + @EmbeddedId` 조합 사용하고 Serializable 구현            |

---

## ⚠️ 주의사항

* `@Embeddable` 클래스는 **절대 @Entity가 아니며**, `EntityManager`로 직접 조회할 수 없습니다.
* `@Embeddable` 클래스는 **파라미터 없는 기본 생성자 필수**입니다.
* `@Embeddable` 클래스 내 필드에는 `@Column(name = "...")`으로 컬럼 이름 커스터마이징 가능

---

## 🖼️ 시각 구조 (관계도)

```
Student (Entity)
 ├── studentId : int
 ├── firstName : String
 ├── lastName : String
 └── studentAddress : StudentAddress (Embeddable)
         ├── city : String
         ├── state : String
         └── zipCode : int
```

**→ DB에서는 모든 필드가 STUDENT 테이블에 Flatten 되어 존재합니다.**

---

## 📚 참고

* 📘 [Jakarta Persistence API – @Embedded](https://jakarta.ee/specifications/persistence/3.1/apidocs/jakarta.persistence/jakarta/persistence/Embedded.html)
* 📘 [Hibernate User Guide – Embeddables](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#embeddable)

---

## ✅ 마무리 요약

| 항목            | 내용                                                      |
| ------------- | ------------------------------------------------------- |
| `@Embeddable` | 별도 테이블 없이 필드만 포함되는 값 타입 클래스 선언                          |
| `@Embedded`   | `@Embeddable` 클래스를 포함하는 필드                              |
| `@EmbeddedId` | 복합키로 `@Embeddable`을 사용하는 경우                             |
| 장점            | 재사용성, 설계 명확성, 테이블 단순화                                   |
| 주의            | 직접 조회 불가, Serializable, equals/hashCode 필요 (EmbeddedId) |


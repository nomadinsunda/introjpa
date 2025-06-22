# 🧩 JPA `@Transient` 애노테이션

> `@Transient`은 영속성과 무관한 필드를 명확하게 구분하기 위한 선언이다. 데이터베이스에 **저장되지 않는 필드**를 정의할 때 사용하는 강력한 도구다.

---

## ✅ 기본 개념

* `@Transient`은 **JPA가 해당 필드를 무시하고 매핑 대상에서 제외**하게 만듭니다.
* **이 필드는 테이블의 컬럼으로 매핑되지 않으며**, DB에 저장도, 조회도 되지 않습니다.
* `transient` 키워드와는 **다른 의미**입니다 (자바의 직렬화 제외와 혼동 금지).

```java
@Transient
private String fullName;
```

---

## 🧠 언제 사용하나?

| 상황             | 설명                                           |
| -------------- | -------------------------------------------- |
| 비즈니스 계산 결과 저장  | 예: `fullName`, `totalPrice`, `statusMessage` |
| DTO에서만 필요한 필드  | 클라이언트 응답용 데이터 처리                             |
| 일시적인 UI 상태 보관  | 예: 선택 여부, 하이라이트 여부 등                         |
| 엔티티 내 파생 필드 관리 | `@PostLoad`로 설정하는 계산 필드 등                    |

---

## 🧾 실전 예제

### 📘 `Student.java`

```java
@Entity
@Table(name = "STUDENT")
public class Student {

    @Id
    private int id;
    private String firstName;
    private String lastName;
    private int marks;

    @Transient
    private String fullName;

    public String getFullName() {
        if (fullName == null)
            fullName = createFullName();
        return fullName;
    }

    private String createFullName() {
        return capitalize(firstName) + " " + capitalize(lastName);
    }

    private String capitalize(String name) {
        return name.substring(0, 1).toUpperCase() + name.substring(1);
    }
}
```

### 💾 테이블 구조

```sql
CREATE TABLE STUDENT (
   ID INT PRIMARY KEY,
   FIRSTNAME VARCHAR(20),
   LASTNAME VARCHAR(20),
   MARKS INT
);
```

→ `FULLNAME` 컬럼 없음. 하지만 Java 객체에는 존재함.

---

## 🚨 예외 상황

만약 `@Transient` 없이 `fullName` 필드를 선언한다면?

```java
@Column // 또는 생략
private String fullName;
```

→ 다음과 같은 오류 발생 가능:

```
org.hibernate.MappingException: Unknown column 'fullName' in 'field list'
```

> 💡 Hibernate는 `@Column` 없이도 모든 필드를 자동 매핑 대상으로 간주하므로, DB에 존재하지 않는 필드는 반드시 `@Transient` 처리해야 합니다.

---

## 📌 `@Transient` vs `transient` (Java 키워드)

| 항목        | `@Transient` (JPA)  | `transient` (Java) |
| --------- | ------------------- | ------------------ |
| 소속        | `javax.persistence` | Java 키워드           |
| 기능        | JPA 영속성 제외          | 직렬화 대상에서 제외        |
| 적용 대상     | DB 매핑 무시            | 객체 직렬화 무시          |
| 동시에 사용 가능 | ✅ 가능 (서로 다름)        | ✅ 가능               |

```java
@Transient
private transient String fullName;
```

---

## ⚙️ `@Transient` 활용 시나리오

### ✔ 파생된 필드 계산

```java
@Transient
private int totalPrice;

@PostLoad
public void calculateTotal() {
    this.totalPrice = unitPrice * quantity;
}
```

### ✔ 응답 DTO용 임시 필드

```java
@Transient
private boolean selected;
```

### ✔ 유효성 검사용 필드

```java
@Transient
private String confirmPassword;
```

---

## 🛠 실무 팁

| 전략                            | 설명                                             |
| ----------------------------- | ---------------------------------------------- |
| `@PostLoad` + `@Transient` 조합 | DB 조회 직후 계산 필드 세팅                              |
| 읽기 전용 Projection              | `fullName`을 JPQL에서 `SELECT CONCAT(...)`로 추출 가능 |
| `@JsonIgnore`와 함께 사용          | JSON 직렬화 시 제외할 필드는 함께 사용 가능                    |

---

## 📚 참고 자료

* 📘 [Jakarta Persistence – Transient](https://jakarta.ee/specifications/persistence/3.1/apidocs/jakarta.persistence/jakarta/persistence/Transient.html)
* 📘 [Hibernate Docs – Property Mapping](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#annotations-basic)

---

## ✅ 마무리 요약

| 항목                   | 요약 설명                                |
| -------------------- | ------------------------------------ |
| 역할                   | JPA에서 DB 컬럼과 무관한 필드를 명시적으로 제외        |
| 주 사용처                | 파생 필드, 일시적 상태, DTO/비즈니스 연산 등         |
| 주의                   | `@Transient`을 생략하면 자동 매핑되므로 오류 발생 가능 |
| Java `transient`와 차이 | JPA와 직렬화는 완전히 별개의 개념                 |

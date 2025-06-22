# 🧩 `@Entity` 애노테이션 – JPA의 핵심 클래스 정의

> JPA에서 가장 기본이자 중요한 개념, `@Entity`는 단순한 애노테이션이 아닙니다. 객체와 테이블을 매핑하는 출발점이며, 모든 영속 객체의 생명은 여기서부터 시작됩니다.

---

## ✅ `@Entity`란?

`@Entity`는 해당 클래스가 JPA에서 관리하는 **영속(Persistent) 객체**임을 선언합니다. 이 애노테이션이 붙은 클래스는 **데이터베이스의 한 테이블과 매핑되는 클래스**로 인식되며, JPA의 `EntityManager` 또는 JPQL을 통해 조작할 수 있는 대상으로 등록됩니다.

```java
@Entity
public class Student {
    ...
}
```

---

## 🧠 핵심 개념 요약

| 항목    | 설명                                           |
| ----- | -------------------------------------------- |
| 정의 위치 | 클래스 수준                                       |
| 기능    | 클래스가 엔티티임을 JPA에 알림                           |
| 필수 조건 | 기본 생성자 존재, 식별자 필드(`@Id`) 존재                  |
| 선택 속성 | `name` – JPQL에서 사용할 엔티티 이름 지정 (기본값: 클래스명)    |
| 필드 조건 | 엔티티 클래스는 final 클래스가 아니어야 하며, 필드도 final이면 안 됨 |

---

## 📌 주요 특징

### 🔹 독립적인 존재성

`@Entity`가 선언된 클래스는 데이터베이스 테이블의 **행(row)** 을 표현하며, 독립적으로 **쿼리 가능**합니다. 즉, 다른 클래스에 의존하지 않고 단독으로 조회, 삽입, 수정, 삭제가 가능합니다.

### 🔹 기본 생성자 필수

JPA는 리플렉션(Reflection)을 통해 객체를 생성하므로 **public 또는 protected 기본 생성자**가 반드시 있어야 합니다.

```java
public Student() {} // 기본 생성자
```

### 🔹 `@Id` 필수

엔티티는 반드시 하나 이상의 **식별자 필드**를 가져야 하며, 이를 `@Id` 애노테이션으로 지정해야 합니다.

---

## 🧾 예제 – `Student` 클래스

```java
@Entity
@Table(name = "STUDENT")
public class Student {

    @Id
    @Column(name = "id")
    private int id;

    @Column(name = "firstName")
    private String firstName;

    @Column(name = "lastName")
    private String lastName;

    @Column(name = "marks")
    private int marks;

    // 기본 생성자
    public Student() {}

    // 파라미터 생성자
    public Student(int id, String firstName, String lastName, int marks) {
        this.id = id;
        this.firstName = firstName;
        this.lastName = lastName;
        this.marks = marks;
    }

    // Getter/Setter 생략
}
```

---

## 🔤 `@Entity(name = "별칭")` 속성 활용

기본적으로 엔티티 이름은 클래스 이름(`Student`)이지만, `name` 속성을 지정하면 JPQL에서 이 이름으로 참조합니다.

```java
@Entity(name = "Learner")
public class Student {
    ...
}
```

```java
// JPQL 예시
TypedQuery<Student> query = em.createQuery("SELECT l FROM Learner l WHERE l.marks > 80", Student.class);
```

> 🚫 **주의:**
>
> * 이 `name`은 JPQL에서만 쓰입니다.
> * SQL 테이블 이름은 여전히 `@Table(name = "STUDENT")` 기준입니다.

---

## ⚠️ 실무에서 주의할 점

| 항목                    | 주의 사항                                          |
| --------------------- | ---------------------------------------------- |
| final 클래스 ❌           | 엔티티 클래스는 프록시 생성을 위해 `final`이 아니어야 함            |
| 복합키                   | 복합키를 사용할 경우 `@IdClass` 또는 `@EmbeddedId` 필요     |
| 불변 필드 ❌               | 필드는 `private`이어야 하지만, `final`은 안 됨 (수정 필요성 때문) |
| equals/hashCode 오버라이드 | 식별자(`@Id`) 기반으로 재정의하는 것이 바람직함                  |

---

## 🛠 실무 활용 팁

### ✔ 엔티티 이름 컨벤션 통일

* 클래스명과 테이블명, JPQL 이름을 혼동하지 않기 위해 명확한 규칙을 세우세요.
* 예: `@Entity(name = "StudentEntity")`, `@Table(name = "STUDENT_TBL")`

### ✔ Serializable 구현

* JPA 명세는 `Serializable` 구현을 요구하지 않지만, 분산 환경(예: Hibernate Second-level Cache)에서는 유용합니다.

---

## 🧩 마무리

`@Entity`는 단순한 선언 이상의 의미를 갖습니다. 객체를 데이터베이스와 연결해주는 **출발점이자 계약**입니다. 올바르게 이해하고 설계하면, 유지보수성과 확장성을 모두 만족하는 강력한 ORM 기반 애플리케이션을 개발할 수 있습니다.



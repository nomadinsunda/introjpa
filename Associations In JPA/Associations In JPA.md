
# 📌 Association Mapping in JPA: 관계형 DB → 객체 지향 매핑 완전 정복

## 🔍 개요: 왜 Association Mapping이 필요한가?
DB 설계 팀은 "정규화 규칙"에 따라 프로젝트의 테이블을 설계합니다. 
정규화 규칙/양식은 6가지입니다. 두 번째 정규화 규칙은 무결성 제약 조건을 갖는 테이블을 설계해야 한다는 것입니다. 
즉, 테이블은 다대일, 일대다, 일대일, 다대다와 같은 관계를 갖도록 설계되어야 합니다. 테이블이 연관/관계에 있을 때, 
한 테이블의 레코드는 다른 테이블의 레코드를 나타내므로 한 테이블의 데이터를 기반으로 다른 테이블의 데이터에 접근할 수 있습니다. 
DB 팀은 기본 키와 외래 키 제약 조건을 활용하여 관계를 갖는 테이블을 설계합니다.

두 DB 테이블이 관계에 있을 경우, 해당 관계를 지원하는 JPA 영속성 클래스를 설계하고 구성해야 합니다. 
이 작업을 "연관 매핑"이라고 합니다. 관계를 지원하는 영속성 클래스를 설계하면, 해당 클래스의 객체는 실제로 객체 수준의 관계/연관성을 갖습니다.
대표적인 관계는 다음과 같습니다:

| 관계형 설계       | 설명                    |
| ------------ | --------------------- |
| One-to-One   | 한 행이 다른 테이블의 한 행과 연결됨 |
| One-to-Many  | 한 행이 여러 행과 연결됨        |
| Many-to-One  | 여러 행이 한 행과 연결됨        |
| Many-to-Many | 여러 행이 여러 행과 연결됨       |

> 📌 이 관계를 Java 객체에서도 그대로 표현하기 위해 사용하는 것이 **Association Mapping**입니다.

---

## 🧱 관계형 데이터의 주요 예시와 매핑 방식

| 관계                          | 예시                 | 객체 간 방향성                       |
| --------------------------- | ------------------ | ------------------------------ |
| `@OneToOne`                 | Citizen ↔ Passport | 일반적으로 단방향 (Citizen → Passport) |
| `@OneToMany` + `@ManyToOne` | Team → Member | 단방향 or 양방향 가능                  |
| `@ManyToMany`               | User ↔ Authority   | 항상 양방향, 조인 테이블 필요              |

---

## 🎯 객체 수준 연관 매핑 정리

### ✅ Unidirectional vs. Bidirectional

| 용어                 | 정의                       |
| ------------------ | ------------------------ |
| **Unidirectional** | 연관된 객체를 **한 쪽**에서만 접근 가능 |
| **Bidirectional**  | 양쪽 모두 연관 객체를 접근 가능       |

단방향(Unidirectional) 케이스
```java
@Entity
public class Order {

    @Id
    @GeneratedValue
    private Long id;

    
}

@Entity
public class OrderItem {

    @Id
    @GeneratedValue
    private Long id;

    private String productName;
    private int price;

    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;
}
```

양방향 케이스

```
@Entity
public class Order {

    @Id
    @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "order")
    private List<OrderItem> orderItems = new ArrayList<>();

    // 연관관계 편의 메서드 (중요)
    public void addOrderItem(OrderItem item) {
        orderItems.add(item);
        item.setOrder(this); // 주인 쪽에 설정
    }
}

@Entity
public class OrderItem {

    @Id
    @GeneratedValue
    private Long id;

    private String productName;
    private int price;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id") // FK 관리
    private Order order;

    public void setOrder(Order order) {
        this.order = order;
    }
}
```

---

## 🔧 연관 매핑 애노테이션 요약

| 애노테이션         | 관계 유형 | 위치     | 관계 주인                         |
| ------------- | ----- | ------ | ----------------------------- |
| `@OneToOne`   | 1:1   | 양쪽 클래스 | 기본적으로 FK 가진 쪽                 |
| `@OneToMany`  | 1\:N  | 부모(1)  | **mappedBy로 비주인 명시**          |
| `@ManyToOne`  | N:1   | 자식(N)  | **항상 주인**                     |
| `@ManyToMany` | N\:M  | 양쪽     | 조인 테이블 필요. `mappedBy`로 비주인 설정 |

⚠️ 외래 키(FK)를 가진 쪽이 자식 테이블이며, 그 외래 키가 참조하는 테이블이 부모 테이블입니다.
---

## 💡 JPA 매핑에서 주의할 점

| 항목                  | 설명                                             |
| ------------------- | ---------------------------------------------- |
| **연관관계의 주인(owner)** | DB에서 외래키를 관리하는 쪽. 주인만이 연관관계를 수정할 수 있음          |
| **mappedBy**        | 연관 관계의 비주인을 지정함                                |
| **Cascade**         | 부모 엔티티의 영속성 전이를 자식에게 전달 (예: persist, remove 등) |
| **FetchType**       | 즉시 로딩(EAGER) vs 지연 로딩(LAZY). 성능에 큰 영향          |

---

## 📘 실전 예제: OneToMany Unidirectional (User → PhoneNumber)

### 🧩 ERD

```
User (id) --------< PhoneNumber (id, user_id FK)
```

### 📄 User.java

```java
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @OneToMany(cascade = CascadeType.ALL)
    @JoinColumn(name = "user_id") // PhoneNumber 테이블의 외래키
    private List<PhoneNumber> phoneNumbers = new ArrayList<>();
}
```

### 📄 PhoneNumber.java

```java
@Entity
public class PhoneNumber {
    @Id @GeneratedValue
    private Long id;
    private String number;
}
```

> 🔎 이 예제는 PhoneNumber가 User를 알지 못하는 **단방향 1\:N 관계**입니다.

---

## 📘 실전 예제: ManyToMany Bidirectional (Student ↔ Course)

### 🧩 ERD

```
STUDENT (ID) <---- STUDENT_COURSE ----> COURSE (ID)
```

### 📄 Student.java

```java
@Entity
public class Student {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @ManyToMany
    @JoinTable(name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id"))
    private List<Course> courses = new ArrayList<>();
}
```

### 📄 Course.java

```java
@Entity
public class Course {
    @Id @GeneratedValue
    private Long id;

    private String title;

    @ManyToMany(mappedBy = "courses")
    private List<Student> students = new ArrayList<>();
}
```

> 💡 `@JoinTable`은 조인 테이블을 지정하고, `mappedBy`는 관계의 비주인을 정의합니다.

---

## 🎯 관계별 비교표

| 관계 유형      | 단방향 지원 | 양방향 지원 | 필요 테이블 수 |
| ---------- | ------ | ------ | -------- |
| OneToOne   | ✅      | ✅      | 2        |
| OneToMany  | ✅      | ✅      | 2        |
| ManyToOne  | ✅      | ✅      | 2        |
| ManyToMany | ❌ (비추) | ✅      | **3**    |

---

## 🧪 JPA 연관관계 설계 시 체크리스트

* [ ] 관계의 주인은 누구인가?
* [ ] mappedBy 설정이 정확한가?
* [ ] 양방향 관계라면 무한 루프(JSON 등)는 방지했는가?
* [ ] 컬렉션 타입은 `Set` vs `List` 중 어떤 게 적합한가?
* [ ] FetchType은 지연(LAZY)로 설정했는가?
* [ ] Cascade 설정은 필요한가? (`ALL`, `PERSIST`, `REMOVE` 등)


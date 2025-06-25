# 🔄 JPA One-to-Many, Many-to-One 양방향 연관관계 매핑

## 📌 1. 연관관계 개요

JPA에서 양방향 연관관계란, **양쪽 엔티티 모두가 서로를 참조**하고 있는 관계입니다. 예를 들어, 하나의 `Department`는 여러 명의 `Employee`를 가질 수 있고, 각 `Employee`는 자신이 속한 `Department`를 알 수 있습니다.

### ✅ 기본 개념

* `@OneToMany(mappedBy = "...")`: **연관관계의 주인이 아님**, 읽기 전용
* `@ManyToOne`: **연관관계의 주인**, 외래키(FK)를 가진 쪽

> 외래키는 항상 `Many` 쪽에 존재하며, 따라서 **`@ManyToOne`이 연관관계의 주인**입니다.

---

## 🔑 연관관계의 주인이란?

**연관관계의 주인(owning side)** 이란 **데이터베이스 외래 키(Foreign Key)의 위치를 기준으로**, JPA가 **연관관계를 관리하는 주체**를 의미합니다.

### ✅ 즉, 연관관계의 주인은 “누가 외래 키를 가지고 있느냐?”에 따라 결정됩니다.

---

## 🎯 예제로 이해하기

### 📌 연관관계 예: Department(1) ↔ Employee(N)

```java
@Entity
@Table(name = "departments")
public class Department {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL)
    private List<Employee> employees = new ArrayList<>();
}
```

```java
@Entity
@Table(name = "employees")
public class Employee {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne
    @JoinColumn(name = "department_id") // FK
    private Department department;
}
```

* `Employee`이 **외래 키(department_id)를 보유**하므로, JPA 입장에서 `Employee.department`가 **연관관계의 주인**입니다.
* 반대로 `Department.employees`는 \*\*주인이 아닌 inverse side(비주인)\*\*입니다.

---

## 🔧 연관관계의 주인이 중요한 이유

JPA는 연관관계 저장 시, **연관관계의 주인 필드만을 기준으로 외래 키를 갱신**합니다.

예를 들어 아래와 같은 코드:

```java
Department department = new Department();
department.setName("개발부");

Employee employee = new Employee();
employee.setName("홍길동");

department.getEmployees().add(employee); // ❌ 주인이 아님. DB 반영 안 됨

em.persist(department);
em.persist(employee);

```

이 코드는 **DB에 외래 키가 null로 저장**될 수 있습니다.
왜냐하면 `orderItems`는 주인이 아니므로, 아무리 값을 설정해도 JPA는 **무시**하기 때문입니다.

---

## 📌 정리: 연관관계의 주인이란?

| 개념          | 설명                                    |
| ----------- | ------------------------------------- |
| 🔑 연관관계의 주인 | 외래 키를 가진 쪽, JPA가 연관관계 변경 시 기준으로 삼는 필드 |
| 📍 위치       | 항상 `@ManyToOne` 쪽이 주인 (외래 키가 존재하므로)   |
| 🧭 설정 방법    | `@JoinColumn`이 있는 쪽이 주인               |
| 🚫 주인이 아닌 쪽 | `mappedBy` 속성으로 주인을 지정함. DB에 직접 관여 X  |

---

## 💡 추가 팁

* `@OneToOne`, `@ManyToMany`도 주인을 명시적으로 설정해주어야 합니다.
* 실수로 주인이 아닌 쪽만 수정하면, **DB에 반영되지 않거나 예외가 발생**할 수 있습니다.

---


## ✅ 연관관계 편의 메서드란?

**연관관계 편의 메서드**는 JPA의 양방향 연관관계에서 **양쪽 값을 동시에 세팅**해 주는 사용자 정의 메서드입니다.

```java
public void addEmployee(Employee employee) {
    employees.add(employee);           // (1) List에 추가
    employee.setDepartment(this);     // (2) 반대 방향도 세팅
}
```

이 메서드는 `Department` ↔ `Employee` 간의 양방향 관계를 **일관성 있게 유지**해 주는 도우미입니다.

---

## 📌 왜 필요한가?

양방향 연관관계는 두 객체가 서로를 참조하므로, **연관관계를 한쪽만 설정하면 데이터 불일치**가 생길 수 있습니다.

### ❌ 한쪽만 설정할 경우:

```java
Employee emp = new Employee();
emp.setDepartment(dept);      // 반대쪽 설정 안 함

dept.getEmployees().size();   // ➡️ 0 (논리적으로는 포함됐지만, 실제 리스트에는 없음)
```

* `Employee.department`는 설정되었지만,
* `Department.employees`에는 반영되지 않음 → 메모리 상의 불일치

### ✅ 양쪽을 모두 설정할 경우:

```java
dept.addEmployee(emp); // 편의 메서드 사용
```

* `Employee.department`도,
* `Department.employees`도 동시에 연결 → 객체 그래프 일관성 유지

---

## 🧠 JPA 관점에서 중요성

JPA는 연관관계의 주인만 DB의 외래키를 관리하지만, **객체는 쌍방 참조 구조를 유지해야 논리적 오류가 안 생깁니다.**

예를 들어,

```java
order.getOrderItems().add(item); // 주인이 아니므로 DB에 반영 X
```

이런 코드는 **객체 그래프는 연결됐지만**, **DB에는 반영되지 않는 문제**를 유발합니다.

---

## 📝 정리

| 항목          | 설명                                          |
| ----------- | ------------------------------------------- |
| ❓ 정의        | 양방향 연관관계에서 양쪽을 동시에 설정해주는 사용자 메서드            |
| 🎯 목적       | 객체 그래프의 **일관성 유지**, 데이터 불일치 방지              |
| 🛠 사용 방법    | 연관관계 설정 시 `addXxx()` 메서드를 통해 양쪽 필드 설정       |
| 🔐 JPA 주의사항 | JPA는 연관관계의 **주인만 DB 반영**, 하지만 양쪽 설정은 **필수** |

---

## 📌 Best Practice

```java
// 연관관계 주인 쪽만 설정하는 건 ❌
employee.setDepartment(dept);

// 항상 편의 메서드 사용이 권장됨 ✅
dept.addEmployee(employee);  // 내부에서 setDepartment 호출 포함
```

## 📁 2. 예제 프로젝트 구조

* JPA 구현체: Hibernate
* DB: H2 또는 MySQL
* JavaSE 환경에서 JPA API만 사용 (Spring 미사용)

```
src/
 ├── model/
 │    ├── Department.java
 │    └── Employee.java
 ├── persistence.xml
 └── Main.java
```

---

## 🧱 3. Entity 설계

### ✅ Department.java

```java
package model;

import javax.persistence.*;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "departments")
public class Department {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL)
    private List<Employee> employees = new ArrayList<>();

    // === 연관관계 편의 메서드 ===
    public void addEmployee(Employee employee) {
        employees.add(employee);
        employee.setDepartment(this);
    }

    // Getters/Setters
    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public List<Employee> getEmployees() { return employees; }
}
```

---

### ✅ Employee.java

```java
package model;

import javax.persistence.*;

@Entity
@Table(name = "employees")
public class Employee {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne
    @JoinColumn(name = "department_id") // FK
    private Department department;

    // Getters/Setters
    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Department getDepartment() { return department; }
    public void setDepartment(Department department) { this.department = department; }
}
```

---

## ⚙ 4. persistence.xml

```xml
<persistence xmlns="https://jakarta.ee/xml/ns/persistence"
             version="3.0">
    <persistence-unit name="jpa-example">
        <class>model.Department</class>
        <class>model.Employee</class>

        <properties>
            <property name="jakarta.persistence.jdbc.driver" value="org.h2.Driver"/>
            <property name="jakarta.persistence.jdbc.url" value="jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1"/>
            <property name="jakarta.persistence.jdbc.user" value="sa"/>
            <property name="jakarta.persistence.jdbc.password" value=""/>
            
            <property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect"/>
            <property name="hibernate.hbm2ddl.auto" value="create-drop"/>
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
        </properties>
    </persistence-unit>
</persistence>
```

---

## 🧪 5. 테스트 코드

```java
import model.Department;
import model.Employee;

import javax.persistence.*;

public class Main {

    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("jpa-example");
        EntityManager em = emf.createEntityManager();

        em.getTransaction().begin();

        Department dept = new Department();
        dept.setName("Engineering");

        Employee emp1 = new Employee();
        emp1.setName("Alice");

        Employee emp2 = new Employee();
        emp2.setName("Bob");

        // 연관관계 편의 메서드로 양방향 설정
        dept.addEmployee(emp1);
        dept.addEmployee(emp2);

        em.persist(dept); // cascade 덕분에 Employee도 저장됨

        em.getTransaction().commit();

        // 조회 테스트
        Department found = em.find(Department.class, dept.getId());
        System.out.println("Department: " + found.getName());
        found.getEmployees().forEach(e ->
                System.out.println("Employee: " + e.getName()));

        em.close();
        emf.close();
    }
}
```

---

## 📊 6. 출력 로그 분석

```sql
insert into departments (name) values (?)
insert into employees (department_id, name) values (?, ?)
insert into employees (department_id, name) values (?, ?)

select * from departments where id=?
select * from employees where department_id=?
```

---

## 🧠 7. 정리 및 주의 사항

| 항목            | 설명                                     |
| ------------- | -------------------------------------- |
| 연관관계 주인       | `Employee` (외래키 가짐)                    |
| 연관관계 설정 위치    | `Employee.department`에 `@ManyToOne` 사용 |
| `mappedBy` 용도 | `Department.employees`는 읽기 전용 역할       |
| 양방향 설정 방법     | `addEmployee()` 같은 편의 메서드 활용           |
| 성능            | 지연 로딩 (`LAZY`) 기본, 필요시 명시 변경           |



## 📝 결론

JPA의 One-to-Many ↔ Many-to-One 양방향 연관관계는 **객체지향적 모델링과 관계형 데이터베이스 설계의 간극을 메우는 핵심 요소**입니다. 특히 편의 메서드를 통해 양방향 연관관계를 유지하는 것이 핵심이며, 항상 연관관계의 주인을 기준으로 데이터 변경이 이루어지도록 코드를 구성해야 합니다.


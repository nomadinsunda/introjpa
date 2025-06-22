# 🔄 JPA One-to-Many, Many-to-One 양방향 연관관계 매핑

## 📌 1. 연관관계 개요

JPA에서 양방향 연관관계란, **양쪽 엔티티 모두가 서로를 참조**하고 있는 관계입니다. 예를 들어, 하나의 `Department`는 여러 명의 `Employee`를 가질 수 있고, 각 `Employee`는 자신이 속한 `Department`를 알 수 있습니다.

### ✅ 기본 개념

* `@OneToMany(mappedBy = "...")`: **연관관계의 주인이 아님**, 읽기 전용
* `@ManyToOne`: **연관관계의 주인**, 외래키(FK)를 가진 쪽

> 외래키는 항상 `Many` 쪽에 존재하며, 따라서 **`@ManyToOne`이 연관관계의 주인**입니다.

---

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


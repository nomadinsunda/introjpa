# 🧠 JPA Tutorial: 자바 ORM의 핵심, JPA

> “JPA는 ORM을 처음 접하는 개발자에게 꼭 필요한 출발점이다.”

---

## 🟡 1. JPA란 무엇인가?

JPA(Java Persistence API)는 자바 객체와 데이터베이스 테이블 간의 **매핑(Mapping)** 을 담당하는 **ORM(Object-Relational Mapping)** 기술입니다.
다음 이미지는 JPA 아키텍쳐의 주요 컴포넌트들간의 관계를 보여주고 있습니다.

![The diagram below illustrates the relationships between the primary components of the JPA architecture.](https://openjpa.apache.org/builds/1.2.3/apache-openjpa/docs/img/jpa-arch.png)

### 🔍 JPA의 핵심 기능

* 자바 클래스 → DB 테이블 매핑
* 클래스의 필드 → 테이블 컬럼 매핑
* 객체 인스턴스 → 테이블의 한 행(Row)으로 동기화
* DB 조작(Insert, Update, Delete, Select)을 자바 코드로 추상화

```java
@Entity
@Table(name = "STUDENT")
public class Student {
    @Id
    private int id;
    private String firstName;
    private String lastName;
    private int marks;
}
```

---

## 🧩 2. ORM이란?

ORM(Object-Relational Mapping)은 자바 객체와 관계형 데이터베이스의 **상호 매핑**과 **동기화**를 의미합니다.

> 예를 들어, 자바 객체에서 이름을 변경하면 자동으로 DB의 이름 컬럼도 갱신됩니다. 반대의 경우도 마찬가지죠.

### ✨ ORM 기술 예시

* JPA (표준)
* Hibernate (JPA 구현체 + 자체 ORM)
* MyBatis (반쪽 ORM)
* TopLink, iBATIS 등

---

## ✅ 3. 왜 JPA를 써야 할까? (장점 정리)

| 기능                      | JPA | JDBC |
| ----------------------- | --- | ---- |
| POJO 기반 개발              | ✅   | ❌    |
| SQL 독립성 (JPQL 사용)       | ✅   | ❌    |
| 객체 지향적 관계 매핑            | ✅   | ❌    |
| 트랜잭션, 커넥션 풀 내장          | ✅   | ❌    |
| 1차/2차 캐시                | ✅   | ❌    |
| 객체 간 관계 탐색 자동 처리        | ✅   | ❌    |
| 버전 관리 (Optimistic Lock) | ✅   | ❌    |
| 명확한 Table-Object 추적 가능  | ✅   | ❌    |

---

## 🏗️ 4. JPA는 혼자 못 써요. 반드시 구현체가 필요합니다.

JPA는 **스펙**(Specification)일 뿐이며, 이를 구현한 제품이 있어야 동작합니다.

| 역할               | 예시                              |
| ---------------- | ------------------------------- |
| JPA (스펙)         | `javax.persistence.*`           |
| 구현체              | Hibernate, EclipseLink, OpenJPA |
| Hibernate 고유 API | `org.hibernate.*`               |

💡 대부분의 개발자들이 "Hibernate"와 "JPA"를 혼동하지만, JPA는 인터페이스이고 Hibernate는 그 구현체입니다.

---

## 💡 5. 실전 예제: JPA CRUD 구현

### 5.1 `Student` 엔티티 클래스

```java
@Entity
@Table(name = "STUDENT")
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

    // 기본 생성자와 전체 필드 생성자, getter/setter 생략
}
```

### 5.2 `persistence.xml` 설정

```xml
<persistence xmlns="https://jakarta.ee/xml/ns/persistence"
             version="3.0">
    <persistence-unit name="StudentPU">
        <class>com.java4coding.Student</class>
        <properties>
            <property name="jakarta.persistence.jdbc.driver" value="com.mysql.cj.jdbc.Driver"/>
            <property name="jakarta.persistence.jdbc.url" value="jdbc:mysql://localhost:3306/study"/>
            <property name="jakarta.persistence.jdbc.user" value="root"/>
            <property name="jakarta.persistence.jdbc.password" value="password"/>
            <property name="hibernate.hbm2ddl.auto" value="update"/>
            <property name="hibernate.dialect" value="org.hibernate.dialect.MySQL8Dialect"/>
        </properties>
    </persistence-unit>
</persistence>
```

---

## ⚙️ 5.3 `Test.java`로 CRUD 실행

```java
public class Test {
    private static EntityManager em;

    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("StudentPU");
        em = emf.createEntityManager();

        createStudent(1, "Manu", "Manjunatha", 100);
        createStudent(2, "Likitha", "Manjunatha", 98);
        createStudent(3, "Advith", "Tyagraj", 99);
    }

    private static void createStudent(int id, String firstName, String lastName, int marks) {
        em.getTransaction().begin();
        Student student = new Student(id, firstName, lastName, marks);
        em.persist(student);
        em.getTransaction().commit();
    }
}
```

> ✔ `persist()` 메서드로 객체를 저장하고, 트랜잭션으로 커밋합니다.

---

## 📌 마무리하며

JPA는 단순히 ORM 기술 그 이상입니다. 객체 지향 프로그래밍의 강점을 살려 **비즈니스 로직과 데이터베이스 간 간극을 좁히는 도구**입니다.

이제 여러분은 다음을 이해하게 되었습니다:

* ORM과 JPA의 관계
* JPA의 등장 배경과 이점
* JPA와 Hibernate의 차이
* 실습 기반의 JPA 프로젝트 구성

---

## 📎 참고

* JPA 공식 스펙: [https://jakarta.ee/specifications/persistence](https://jakarta.ee/specifications/persistence)
* Hibernate 공식 문서: [https://hibernate.org](https://hibernate.org)
* openjpa : [https://openjpa.apache.org/builds/1.2.3/apache-openjpa/docs/manual.html#introduction]



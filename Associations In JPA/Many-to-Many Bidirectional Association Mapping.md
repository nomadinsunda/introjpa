# 🔗 JPA Many-to-Many 양방향 연관관계 매핑

## ✅ 1. 개념 정리: Many-to-Many란?

> **Many-to-Many (N\:N)** 연관관계란, 한 엔티티 인스턴스가 **다수의 다른 엔티티 인스턴스와 연결**되고, 반대로 그 엔티티도 **다수의 원 엔티티와 연결**되는 관계입니다.

### 예시

* 학생(Student)은 여러 과목(Course)을 수강할 수 있고,
* 하나의 과목(Course)은 여러 학생(Student)이 수강할 수 있다.

이 관계는 **양방향**으로 서로 참조하게 만들 수 있습니다.

---

## 📐 2. 관계 모델링 (ERD)

### 📊 관계형 데이터베이스 모델

Many-to-Many 관계는 **두 테이블만으로는 직접 표현할 수 없고**, \*\*연결 테이블(조인 테이블)\*\*이 필요합니다.

```text
STUDENT           STUDENT_COURSE           COURSE
--------          -----------------        --------
id (PK)     <---  student_id (FK)          id (PK)
name              course_id (FK)   --->    name
```

---

## 🛠️ 3. 엔티티 설계 (양방향 매핑)

### 📦 Student 엔티티

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
    private Set<Course> courses = new HashSet<>();

    public void addCourse(Course course) {
        courses.add(course);
        course.getStudents().add(this); // 양방향 설정
    }

    // getter, setter
}
```

### 📦 Course 엔티티

```java
@Entity
public class Course {

    @Id @GeneratedValue
    private Long id;

    private String name;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();

    // getter, setter
}
```

> 💡 `mappedBy = "courses"`는 `Student`가 연관관계의 주인(owning side)임을 의미합니다.

---

## ⚙️ 4. 주요 구성 요소 정리

| 항목                   | 설명                         |
| -------------------- | -------------------------- |
| `@ManyToMany`        | 다대다 매핑을 선언합니다.             |
| `@JoinTable`         | 중간 테이블 명 및 조인 컬럼 명시        |
| `joinColumns`        | 현재 엔티티(Student)의 외래키 컬럼 지정 |
| `inverseJoinColumns` | 상대 엔티티(Course)의 외래키 컬럼 지정  |
| `mappedBy`           | 연관관계의 주인이 아님을 명시 (비주인 측)   |

---

## 💡 5. 연관관계의 주인 vs 비주인

* `Student`가 **주인**: 조인 테이블에 대한 설정을 담당
* `Course`는 **비주인**: `mappedBy`로 주인 지정

### 🔁 왜 주인을 지정해야 하나?

> JPA에서는 양방향 연관관계가 있을 경우, **누가 조인 테이블을 관리할지**를 명확히 해야 충돌 없이 SQL을 생성할 수 있습니다.

---

## 🧪 6. 실습 예제

```java
EntityManager em = emf.createEntityManager();
em.getTransaction().begin();

Student s1 = new Student(); s1.setName("Alice");
Course c1 = new Course(); c1.setName("Math");
Course c2 = new Course(); c2.setName("Physics");

s1.addCourse(c1);
s1.addCourse(c2);

em.persist(s1); // cascade 없이 Course도 persist 해야 함
em.persist(c1);
em.persist(c2);

em.getTransaction().commit();
```

---

## 📜 7. 생성되는 SQL 예시

```sql
insert into student (name) values ('Alice');
insert into course (name) values ('Math');
insert into course (name) values ('Physics');

insert into student_course (student_id, course_id) values (1, 1);
insert into student_course (student_id, course_id) values (1, 2);
```

---

## 🔎 8. 실무 사용 시 유의점

### ✅ 반드시 연관관계 편의 메서드 사용

```java
public void addCourse(Course course) {
    courses.add(course);
    course.getStudents().add(this); // 중요!
}
```

→ 한쪽만 설정하면 `mappedBy` 측에 반영되지 않아 **양쪽 consistency 불일치** 발생

---

## ⚠️ 9. Cascade vs mappedBy 구분

| 구분         | 의미                                       |
| ---------- | ---------------------------------------- |
| `cascade`  | 연관된 엔티티의 **영속성 상태 전이** 전파 여부 (PERSIST 등) |
| `mappedBy` | 연관관계의 **주인/비주인** 설정                      |

> 서로 **역할이 다름**에 주의해야 함!

---

## 🔥 10. 성능과 확장성 이슈

* `@ManyToMany`는 단순할 수 있으나, **추가 컬럼이 필요한 경우 문제가 됨**
* 예: `수강 신청일`, `성적`, `출석률` 등

### ✅ 해결 방안: **중간 엔티티 사용**

```java
@Entity
public class Enrollment {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne
    private Student student;

    @ManyToOne
    private Course course;

    private LocalDate enrolledDate;
    private Double score;
}
```

→ 이 방식은 `@ManyToMany` 대신, **두 개의 `@ManyToOne`으로 대체**하며 **유연한 모델링 가능**

---
### ✅ 리팩토링: `@ManyToMany` → `@OneToMany` + `@ManyToOne` 구조로 변경

> `Enrollment`가 **연관관계 엔티티**로 `score`, `enrolledDate` 같은 부가 정보를 갖는 핵심 역할을 합니다.

---

### 📌 리팩토링 요약

| 기존 구조(@JoinTable)                | 변경 후 구조                                                          |
| ------------------------------------ | ---------------------------------------------------------------- |
| `Student` ↔ `Course` (`@ManyToMany`) | `Student` → `Enrollment` ← `Course` (`@OneToMany`, `@ManyToOne`) |
| 중간 테이블 없음                            | `Enrollment` 엔티티가 중간 테이블 역할 수행                                   |
| 부가 정보 저장 불가                          | `Enrollment.enrolledDate`, `score` 등 저장 가능                       |

---

### 📁 최종 구성

* `Student` – (1\:N) – `Enrollment`
* `Course` – (1\:N) – `Enrollment`
* `Enrollment` – (N:1) – `Student`
* `Enrollment` – (N:1) – `Course`

---

## 🧾 11. 요약

| 항목     | 요약 설명                                              |
| ------ | -------------------------------------------------- |
| 관계 종류  | 다대다 (N\:N)                                         |
| 필요 테이블 | 3개 (엔티티 2개 + 조인 테이블 1개)                            |
| 주인 지정  | `mappedBy`로 주인 명확히 설정                              |
| 실무 권장  | 단순 매핑만 있을 때 `@ManyToMany`<br>추가 정보 필요 시 중간 엔티티로 설계 |
| 주의     | 편의 메서드 필수, cascade와 mappedBy 혼동 금지                 |

---

## ✅ 결론

JPA의 `Many-to-Many` 양방향 매핑은 설계 자체는 간단하지만, 실무에서는 조인 테이블에 부가 정보를 저장할 필요가 많기 때문에 **중간 엔티티 방식**이 대부분 채택됩니다.

양방향 매핑에서는 반드시 주인/비주인 개념을 명확히 하고, **양쪽 객체 참조 일관성 유지**를 위해 편의 메서드를 잘 활용하는 것이 핵심입니다.

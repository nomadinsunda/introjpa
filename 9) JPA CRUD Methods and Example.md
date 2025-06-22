# 📘 JPA CRUD

> JPA의 `EntityManager`가 제공하는 CRUD 기능을 직접 구현하며, JPA의 핵심 동작 원리를 이해하자.

---

## 🔍 CRUD란?

CRUD는 데이터베이스에서 자주 사용하는 네 가지 기본 연산입니다:

| 연산     | 설명         |
| ------ | ---------- |
| Create | 새로운 데이터 저장 |
| Read   | 기존 데이터 조회  |
| Update | 기존 데이터 수정  |
| Delete | 기존 데이터 삭제  |

JPA에서는 이 모든 작업을 `EntityManager`를 통해 처리하며, 각 연산에 대응하는 표준 메서드가 존재합니다.

---

## 🧩 EntityManager의 핵심 메서드

| 연산     | 메서드                                             | 반환값           | 설명                   |
| ------ | ----------------------------------------------- | ------------- | -------------------- |
| Create | `persist(Object entity)`                        | `void`        | 새 엔티티를 영속 상태로 전환     |
| Read   | `find(Class<T> entityClass, Object primaryKey)` | `T` 또는 `null` | 기본키를 기준으로 엔티티 조회     |
| Update | `merge(Object entity)`                          | 병합된 엔티티       | 준영속 상태의 엔티티를 병합하여 갱신 |
| Delete | `remove(Object entity)`                         | `void`        | 영속 상태의 엔티티 삭제        |

> 💡 **참고:**
>
> * `persist()`는 **새로운 엔티티만** 저장합니다.
> * `merge()`는 **준영속(new or detached)** 객체도 병합하여 저장하거나 수정할 수 있습니다.
> * `find()`는 1차 캐시(EntityManager의 영속성 컨텍스트)를 먼저 조회합니다.

---

## 🗃️ 예제 테이블 스키마

```sql
CREATE TABLE EMPLOYEE (
    empid INT NOT NULL,
    firstname VARCHAR(45),
    lastname VARCHAR(45),
    email VARCHAR(45),
    PRIMARY KEY (empid)
);
```

---

> 🛠 Hibernate + EclipseLink JPA API 조합을 사용 중입니다. Spring Data JPA 환경에서는 Hibernate만으로도 충분합니다.

---

## 🧾 Entity 클래스 – `Employee.java`

```java
@Entity
@Table(name = "EMPLOYEE")
public class Employee implements Serializable {

    @Id
    private int empId;

    private String firstname;
    private String lastname;
    private String email;

    // Getters, Setters, toString 생략
}
```

---

## 🧪 테스트 – CRUD 실행 흐름

```java
public class Test {
    public static void main(String[] args) {

        EntityManagerFactory emf = Persistence.createEntityManagerFactory("JPACRUD");
        EntityManager em = emf.createEntityManager();

        // 1. Create (INSERT)
        Employee employee = new Employee();
        employee.setEmpId(1);
        employee.setFirstname("Manu");
        employee.setLastname("Manjunatha");
        employee.setEmail("manu.m@java4coding.com");

        em.getTransaction().begin();
        em.persist(employee); // INSERT INTO EMPLOYEE ...
        em.getTransaction().commit();

        // 2. Read (SELECT)
        Employee found = em.find(Employee.class, 1);
        System.out.println("Found: " + found);

        // 3. Update (dirty checking)
        em.getTransaction().begin();
        found.setEmail("feedback@java4coding.com");
        em.getTransaction().commit(); // UPDATE 발생

        // 3-1. Update using merge
        Employee updateObj = new Employee();
        updateObj.setEmpId(1); // 반드시 식별자 필요
        updateObj.setEmail("manu.m@java4coding.com");

        em.getTransaction().begin();
        em.merge(updateObj); // 변경 내용 병합 후 UPDATE
        em.getTransaction().commit();

        // 4. Delete (DELETE)
        em.getTransaction().begin();
        em.remove(found); // DELETE FROM EMPLOYEE WHERE empid = 1
        em.getTransaction().commit();

        // 5. Read again
        Employee afterDelete = em.find(Employee.class, 1);
        System.out.println("After Deletion: " + afterDelete); // null

        em.close();
        emf.close();
    }
}
```

---

## 📌 실무 팁

### 🔹 `persist()` vs `merge()` 차이점

| 항목  | `persist()`                     | `merge()`                          |
| --- | ------------------------------- | ---------------------------------- |
| 대상  | 새 엔티티                           | 새 또는 준영속 엔티티                       |
| 리턴  | 없음                              | 병합된 영속 엔티티 리턴                      |
| 식별자 | 없어야 함 (DB에 아직 없음)               | 있어야 함 (DB에 존재하거나 없는 경우 INSERT 시도됨) |
| 주의  | 같은 식별자로 `persist()` 두 번 호출하면 예외 | `merge()`는 유연하지만 혼동 주의             |

### 🔹 `remove()` 사용 시 주의사항

* 삭제 대상 객체는 반드시 **영속 상태**여야 함.
* `em.remove(new Employee())`처럼 **비영속 객체** 삭제 시 오류 발생

```java
Employee ghost = new Employee();
ghost.setEmpId(100); // DB에는 없음

em.remove(ghost); // ❌ IllegalArgumentException 발생 가능
```

> ✅ 해결: 먼저 `find()`로 영속화 후 `remove()`

---

## 📚 정리

| 연산     | 메서드         | 핵심 포인트               |
| ------ | ----------- | -------------------- |
| Create | `persist()` | 새로운 엔티티를 영속 상태로      |
| Read   | `find()`    | 기본키 기준 조회 (1차 캐시 우선) |
| Update | `merge()`   | 준영속 객체 병합 (리턴 객체 주의) |
| Delete | `remove()`  | 영속 객체만 삭제 가능         |

---

## 🎯 마무리

JPA의 CRUD 연산은 단순하지만, **영속성 컨텍스트의 상태 관리**와 **메서드 간 동작 차이**를 명확히 이해해야 합니다. `persist()`는 **새로운 엔티티만**, `merge()`는 **업데이트 또는 새 삽입 가능**, `remove()`는 **영속 객체만 삭제 가능**하다는 점을 명심하세요.


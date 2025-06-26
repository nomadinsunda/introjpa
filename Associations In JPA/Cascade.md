# 🔄 JPA Cascade (영속성 전이)

## ✅ 1. 개념 정의: Cascade란 무엇인가?

> **Cascade**는 JPA에서 **한 엔티티의 상태 변경(persist, merge, remove 등)**이 **연관된 엔티티에도 함께 전파되도록** 설정하는 **전이(전파) 메커니즘**이다.

즉, **연관관계의 주인(Owner)** 엔티티의 상태 변화가 **연관된 비주인(Non-Owner)** 엔티티에게 **자동으로 전파**되게 하여, 관련된 작업을 일일이 명시적으로 처리하지 않아도 되도록 한다.

---

## 🔍 2. 왜 필요한가?

```java
em.persist(dept);
em.persist(emp1);
em.persist(emp2);
```

위 코드는 `Department`와 두 `Employee`를 모두 저장해야 하지만, Cascade 설정이 되어 있다면:

```java
em.persist(dept); // 딱 한 줄로 전체 저장 가능!
```

✅ **코드의 간결성**,
✅ **엔티티 간의 생명주기 연결 유지**,
✅ **트랜잭션 일관성** 유지에 유리

---

## ⚙️ 3. CascadeType 종류별 상세 설명

| CascadeType | 설명                                                      |
| ----------- | ------------------------------------------------------- |
| **PERSIST** | 연관관계 주인을 `persist()`하면 비주인도 `persist()`됨                |
| **MERGE**   | 연관관계 주인을 `merge()`하면 비주인도 `merge()`됨                    |
| **REMOVE**  | 연관관계 주인을 `remove()`하면 비주인도 `remove()`됨                  |
| **REFRESH** | 연관관계 주인을 `refresh()`하면 비주인도 `refresh()`됨 (DB 최신 상태로 갱신) |
| **DETACH**  | 연관관계 주인을 `detach()`하면 비주인도 `detach()`됨 (1차 캐시에서 분리)     |
| **ALL**     | 위 모든 동작 전이                                              |

> 💡 `CascadeType.ALL` = `PERSIST + MERGE + REMOVE + REFRESH + DETACH`

---

## 🧪 4. 실습 예제: CascadeType.ALL 적용

### 📦 엔티티 설계

```java
@Entity
public class Department {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL)
    private List<Employee> employees = new ArrayList<>();

    public void addEmployee(Employee emp) {
        employees.add(emp);
        emp.setDepartment(this); // 연관관계 설정
    }
    // getter/setter 생략
}

@Entity
public class Employee {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @ManyToOne
    @JoinColumn(name = "department_id") // 연관관계 주인
    private Department department;
    // getter/setter 생략
}
```

### 🧪 사용 코드

```java
em.getTransaction().begin();

Department dept = new Department();
dept.setName("Dev");

Employee e1 = new Employee(); e1.setName("Alice");
Employee e2 = new Employee(); e2.setName("Bob");

dept.addEmployee(e1);
dept.addEmployee(e2);

em.persist(dept); // Employee까지 자동 저장됨
em.getTransaction().commit();
```

---

## 🔎 5. Hibernate 내부 동작 (SQL 로그)

Cascade가 없을 경우:

```sql
insert into departments (name) values ('Dev');
-- 오류: employees는 저장되지 않음
```

Cascade가 있을 경우 (`CascadeType.ALL`):

```sql
insert into departments (name) values ('Dev');
insert into employees (name, department_id) values ('Alice', 1);
insert into employees (name, department_id) values ('Bob', 1);
```

Hibernate는 연관관계 주인을 저장할 때, `cascade` 설정을 보고 비주인 엔티티를 함께 저장합니다.

---

## 📘 6. JPA 스펙 관점에서 본 Cascade

JPA 스펙에서는 cascade에 대해 다음과 같이 정의합니다:

> When an operation is cascaded to a target entity, the same operation is invoked on the target entity (persist, merge, remove, refresh, detach). Cascading is not automatic by default and must be explicitly specified.

즉, JPA는 연관된 엔티티의 **상태 변경 이벤트 전파를 위한 명시적 정책**을 제공하는 것입니다.

---

## 🧠 7. 실무에서의 Cascade 사용 전략

| 연관관계 종류               | 주인/비주인 구분     | 전이 설정 추천          | 이유                  |
| --------------------- | ------------- | ----------------- | ------------------- |
| Department - Employee | 주인: Employee  | `CascadeType.ALL` | 부서가 직원 생명주기를 소유     |
| Order - OrderItem     | 주인: OrderItem | `CascadeType.ALL` | 주문이 주문상품을 완전 소유     |
| Member - Address      | 주인: Address   | 없음 또는 `PERSIST`   | 주소는 회원 삭제와 관계없이 유지됨 |
| Company - Employee    | 주인: Employee  | 신중하게 선택           | 직원은 독립적 생명주기를 가짐    |

> ❗ `REMOVE`, `orphanRemoval = true`는 꼭 **연관관계 주인 필드**에 선언해야 정확하게 동작함

---

## ❌ 8. Cascade 설정 시 흔한 실수

### ① 연관관계 설정 안 하고 persist

```java
em.persist(dept); // emp.setDepartment(this) 안 했으면 오류 또는 무시됨
```

### ② 무한 루프 발생 가능

```java
// 양방향 관계에서 toString(), equals() 무한 순환 호출 주의!
```

### ③ orphanRemoval과 cascade 혼동

```java
@OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Child> children;
```

| 설정               | 동작                                     |
| ---------------- | -------------------------------------- |
| `cascade=REMOVE` | 연관관계 주인을 삭제하면 비주인도 함께 삭제됨              |
| `orphanRemoval`  | 연관관계에서 비주인을 제거하면 DB에서도 삭제됨 (detach 아님) |

---

## 🧾 9. 요약 정리

| 메서드 호출      | Cascade 설정       | 동작 설명             |
| ----------- | ---------------- | ----------------- |
| `persist()` | `PERSIST`, `ALL` | 연관된 비주인도 저장됨      |
| `merge()`   | `MERGE`, `ALL`   | 비주인도 병합됨          |
| `remove()`  | `REMOVE`, `ALL`  | 비주인도 삭제됨          |
| `detach()`  | `DETACH`, `ALL`  | 비주인도 1차 캐시에서 분리됨  |
| `refresh()` | `REFRESH`, `ALL` | 비주인도 DB 상태로 새로고침됨 |

---

## 💡 마무리: 언제 Cascade를 쓰고 말아야 할까?

* ✅ 생명주기를 **실제로 함께 관리해야 한다면** cascade를 사용하라.
* ❗ 그렇지 않으면 **비주 엔티티는 직접 persist/merge/remove 하도록** 하라.
* **REMOVE 전이**는 특히 조심해서 설정할 것 – 중요한 데이터가 연쇄 삭제될 수 있음.

> 🎯 요약: cascade는 연관관계 주인/비주인 구분을 이해하고, 생명주기 의존성이 명확할 때만 사용하라!

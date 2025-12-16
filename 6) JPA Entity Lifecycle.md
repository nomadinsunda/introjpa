# 🔄 JPA Entity Lifecycle

> “JPA의 진짜 핵심은 Entity의 생명주기를 이해하는 것이다.”

JPA에서는 Entity 객체가 단순히 생성되고 저장되는 것을 넘어, **엄격한 생명주기(state)** 를 따릅니다.
각 상태는 JPA 내부의 **Persistence Context**와 어떻게 연결되었는지를 기준으로 정의되며, 상태에 따라 JPA가 처리하는 방식도 완전히 달라집니다.

---

## 📌 JPA의 3대 Entity 상태
![Entity lifecycle](https://openjpa.apache.org/builds/1.2.3/apache-openjpa/docs/img/jpa-state-transitions.png)

| 상태                  | 설명                                        |
| ------------------- | ----------------------------------------- |
| **Transient (비영속)** | EntityManager에 등록되지 않은 순수 Java 객체         |
| **Managed (영속)**    | Persistence Context에 포함되어 JPA가 관리하는 객체    |
| **Detached (준영속)**  | 원래 영속 상태였다가, Persistence Context에서 분리된 객체 |



JPA는 이 외에도 "Removed" 상태를 **임시 상태**로 분류하지만, 이는 실제 상태라기보다는 "삭제 예약됨" 상태로 취급됩니다.
---

## 🧭 전체 흐름도

```text
new Student()  → [transient]
em.persist(student) → [managed]
em.detach(student) → [detached]
em.merge(student) → [managed]
em.remove(student) → [removed → detached after commit]
```

---

## 🔍 1. Transient (비영속 상태)

**정의**

* `new` 키워드로 단순히 생성된 객체
* 아직 EntityManager에 등록되지 않음
* **ID(식별자)** 가 없거나 DB와 무관함

```java
Student student = new Student(); // transient
```

**특징**

* DB에 아무런 영향 없음
* 식별자 없음 (`null`)
* Persistence Context에 포함되지 않음

---

## 🟢 2. Managed (영속 상태)

**정의**

* EntityManager가 관리하는 상태
* 객체는 Persistence Context에 포함됨
* JPA는 객체의 변경을 자동으로 추적(Dirty Checking)

```java
em.persist(student); // 상태: managed
```

**특징**

* 식별자(ID) 존재
* DB와 연결된 row와 동기화됨
* 트랜잭션 커밋 시 자동으로 DB에 반영

**예제**
```java
package com.example.jpa.entity;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import lombok.ToString;

@Entity
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
}
```

```java
Student student = new Student();
student.setName("JPA");

em.getTransaction().begin();
em.persist(student);  // 영속 상태 진입
student.setName("Hibernate");  // 자동 감지됨 (dirty checking)
em.getTransaction().commit();  // UPDATE 쿼리 발생
```

---

## 🟡 3. Detached (준영속 상태)

**정의**

* 원래 영속 상태였지만, 더 이상 EntityManager가 관리하지 않는 상태
* 여전히 식별자(ID)는 존재함

```java
em.detach(student);  // 상태: detached
```

**특징**

* Persistence Context에서 제거됨
* 변경사항은 DB에 반영되지 않음
* 다시 영속화하려면 `merge()` 사용

**예제**

```java
Student student = em.find(Student.class, 1L);  // managed

em.detach(student);  // detached
student.setName("Modified");  // 반영 안 됨

em.getTransaction().begin();
em.merge(student);  // 병합 → managed 상태로 전환
em.getTransaction().commit();  // UPDATE 쿼리 실행
```

---

## 🔥 Removed (삭제 예약 상태)

**정의**

* DB 삭제가 예약된 상태
* `em.remove(entity)` 호출 시 발생
* **commit 또는 flush** 시 실제 DB에서 삭제됨
* 이후 **Detached** 상태로 전환됨

```java
em.remove(student);  // 상태: removed (→ detached after commit)
```

---

## 🧪 상태별 API 동작 비교

| 상태            | `persist()` | `remove()`  | `refresh()` | `merge()`           |
| ------------- | ----------- | ----------- | ----------- | ------------------- |
| **Transient** | Managed로 전환 | 무시됨         | 무시됨         | 새로운 Managed 객체 생성   |
| **Managed**   | 무시됨         | Removed 상태로 | DB 값으로 덮어씀  | 무시됨                 |
| **Removed**   | Managed로 전환 | 무시됨         | 무시됨         | 예외 발생               |
| **Detached**  | 예외 발생       | 예외 발생       | 예외 발생       | 병합되어 Managed 객체로 전환 |

---

## 🎯 실전 코드 흐름 예제

```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("StudentPU");
EntityManager em = emf.createEntityManager();

em.getTransaction().begin();

Student student = new Student();  // transient
em.persist(student);              // managed
em.getTransaction().commit();     // committed → detached

Student managed = em.find(Student.class, student.getId());
em.getTransaction().begin();
em.remove(managed);              // removed
em.getTransaction().commit();    // detached

Student detached = new Student();
detached.setId(2L);
detached.setName("Detached");
em.merge(detached);              // merge → managed
```

---

## 🧠 Persistence Context와 상태 전이의 핵심

Persistence Context는 JPA의 **1차 캐시**이자 **Entity 상태 관리자**입니다.

| 상태 변화               | 조건                               |
| ------------------- | -------------------------------- |
| Transient → Managed | `persist()` 또는 `merge()` 호출      |
| Managed → Detached  | `detach()`, `clear()`, `close()` |
| Managed → Removed   | `remove()` 호출                    |
| Removed → Detached  | `commit()` 또는 `flush()`          |
| Detached → Managed  | `merge()` 호출                     |

---

## ⚠️ 실무에서 흔히 겪는 상태 관련 실수

1. **Detached 상태에서 값 변경만 하고 commit**
   → 아무 일도 일어나지 않음 (JPA는 감지 못함)

2. **Managed 객체를 remove() 한 후 다시 사용**
   → IllegalArgumentException 또는 오류 발생 가능

3. **Detached 객체를 persist()로 영속화 시도**
   → 예외 발생 (`EntityExistsException`)

---

## ✅ 마무리 요약

| 개념        | 요약                            |
| --------- | ----------------------------- |
| Transient | JPA가 모르는 순수 객체                |
| Managed   | EntityManager가 감시하는 객체        |
| Detached  | 한때 영속이었으나, 관리에서 벗어난 객체        |
| Removed   | 삭제 예약됨. commit/flush 시 DB 삭제됨 |

**상태 전이는 대부분 `EntityManager`의 메서드를 통해 수행되며,
그 시점과 방법을 명확히 이해하는 것이 실무의 핵심입니다.**




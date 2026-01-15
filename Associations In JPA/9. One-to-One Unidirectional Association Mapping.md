# 📘 JPA One-to-One Unidirectional Association Mapping

> 단방향 일대일 연관관계를 제대로 이해하고, 매핑부터 활용까지 완벽하게 마스터해보자!

---

## 🧠 1. One-to-One 단방향 연관관계란?

**일대일(1:1) 연관관계**란 하나의 엔티티가 **다른 하나의 엔티티에만** 연결되는 관계를 말합니다.
**단방향** 연관관계는 **연관관계의 주인만이 상대 엔티티를 참조**하며, **반대 방향의 참조는 존재하지 않는** 구조입니다.

### 예시: 사람(PERSON) ↔ 여권(PASSPORT)

* 한 사람은 하나의 여권만 가진다.
* 한 여권은 하나의 사람에게만 속한다.
* 사람이 여권 정보를 갖고 있지만, 여권은 사람을 알 필요가 없다. → **Unidirectional**

---

## 🔎 2. 언제 One-to-One 단방향을 사용할까?

* 양쪽이 **1:1 매핑**이고,
* 한쪽만 **상대 엔티티의 정보가 필요**할 때,
* DB 설계 상 외래 키가 한 테이블에만 존재할 때

---

## 🧱 3. 실전 예제: PERSON ↔ PASSPORT

### 3.1 DB 스키마 (MySQL 기준)

```sql
CREATE TABLE passport (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    number VARCHAR(30) NOT NULL
);

CREATE TABLE person (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50),
    passport_id BIGINT UNIQUE,
    CONSTRAINT fk_passport FOREIGN KEY (passport_id)
        REFERENCES passport(id)
);
```

> `passport_id`는 `person` 테이블에 존재하며, **UNIQUE 제약조건**으로 1:1을 보장함.

---

## 🧬 4. 엔티티 매핑 (JPA 표준 API)

### Passport.java

```java
package model;

import jakarta.persistence.*;

@Entity
public class Passport {

    @Id @GeneratedValue
    private Long id;

    private String number;

    // getters and setters
}
```

### Person.java

```java
package model;

import jakarta.persistence.*;

@Entity
public class Person {

    @Id @GeneratedValue
    private Long id;

    private String name;

    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "passport_id", unique = true)
    private Passport passport;

    // getters and setters
}
```

> ✅ 핵심 포인트:
>
> * `@OneToOne` : 단방향 일대일 매핑
> * `@JoinColumn(name = "passport_id")` : 외래 키는 `person` 테이블에 생성됨
> * `unique = true` : 1:1 관계임을 보장

---

## ⚙️ 5. 테스트 코드

```java
package main;

import jakarta.persistence.*;
import model.Passport;
import model.Person;

public class Main {

    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("jpa-example");
        EntityManager em = emf.createEntityManager();

        em.getTransaction().begin();

        Passport passport = new Passport();
        passport.setNumber("KOR123456");

        Person person = new Person();
        person.setName("Kim");
        person.setPassport(passport);

        em.persist(person);  // Cascade 덕분에 passport도 저장됨

        em.getTransaction().commit();

        // 검증
        Person found = em.find(Person.class, person.getId());
        System.out.println("Person: " + found.getName());
        System.out.println("Passport: " + found.getPassport().getNumber());

        em.close();
        emf.close();
    }
}
```

---

## 🧩 6. JPA의 동작 방식

### Hibernate가 생성하는 SQL 예시:

```sql
insert into passport (number) values ('KOR123456');
insert into person (name, passport_id) values ('Kim', 1);
```

* 먼저 `passport`가 저장되고,
* 그 ID를 외래 키로 갖는 `person`이 저장된다.
* 이는 `CascadeType.ALL` 덕분에 `persist()`가 전이됨

---

## 🧪 7. 주의사항과 팁

### 🔹 외래 키는 어느 테이블에?

* 단방향 연관관계에서는 **참조 주인**이 외래 키를 **소유**한다.
* 위 예제에서는 `Person`이 `Passport`를 참조하므로 **외래 키는 `person.passport_id`**

### 🔹 `mappedBy`는 필요 없음

* 단방향 연관관계에서는 `mappedBy` 속성을 쓰지 않습니다.
  그건 **양방향 관계**에서만 사용합니다.

### 🔹 `nullable = false`?

```java
@JoinColumn(name = "passport_id", nullable = false, unique = true)
```

* 여권이 반드시 있어야 한다면 `nullable = false`로 명시 가능

---

## 📚 8. 실무 적용 팁

| 상황                 | 전략                                  |
| ------------------ | ----------------------------------- |
| 외래 키를 소유한 쪽만 참조 필요 | 단방향 OneToOne 사용                     |
| 양쪽 모두 참조 필요        | 양방향 OneToOne 매핑                     |
| 성능을 고려해야 할 경우      | 지연 로딩 설정 (`fetch = FetchType.LAZY`) |
| 생명주기를 함께 관리        | `CascadeType.ALL` 설정                |

---

## ✅ 9. 정리

| 항목        | 설명                                |
| --------- | --------------------------------- |
| 관계 유형     | 1:1 단방향                           |
| 참조 방식     | 한 쪽만 다른 쪽을 참조                     |
| 외래 키 위치   | 참조하는 쪽 테이블에 존재                    |
| JPA 어노테이션 | `@OneToOne`, `@JoinColumn`        |
| 주의사항      | `mappedBy`는 쓰지 않음, UNIQUE 제약조건 필수 |

---

## 🧭 마무리

JPA에서 단방향 One-to-One 관계는 **가장 간단하지만 가장 잘못 쓰이기 쉬운** 매핑입니다.
외래 키의 위치, 생명주기 전이, fetch 전략 등 **DB 설계와 비즈니스 로직**을 고려한 설계가 필요합니다.

> 🎯 **JPA 연관관계 매핑은 객체 관계의 정확한 반영보다도, 영속성 컨텍스트와 데이터베이스의 구조적 특성을 고려한 전략적 설계가 핵심입니다.**

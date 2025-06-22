# 🧩 JPA `@EmbeddedId` 어노테이션

> `@EmbeddedId`는 JPA에서 복합키(Composite Primary Key)를 표현할 때 사용하는 핵심 애노테이션입니다. `@IdClass`보다 객체지향적으로 설계할 수 있으며, 여러 컬럼을 하나의 키로 구성할 때 사용됩니다.

---

## ✅ 기본 개념

* `@EmbeddedId`는 **복합키를 하나의 클래스**(`@Embeddable`)로 묶어, 엔티티의 식별자로 사용하는 방식입니다.
* 해당 클래스는 반드시 `Serializable`을 구현해야 하며, \*\*`equals()`와 `hashCode()`\*\*를 반드시 override 해야 합니다.

```java
@Entity
public class User {
    @EmbeddedId
    private PersonalDetails personalDetails;
}
```

---

## 🧠 `@EmbeddedId` vs `@IdClass` 차이

| 항목    | `@EmbeddedId`           | `@IdClass`                       |
| ----- | ----------------------- | -------------------------------- |
| 사용 방식 | 복합키 클래스 필드를 엔티티에 그대로 포함 | 엔티티의 각 필드에 `@Id` 적용 후 외부 클래스로 지정 |
| 가독성   | 높음 (객체 단위로 키 구성)        | 낮음 (필드 분산)                       |
| 복잡도   | 낮음                      | 높음                               |
| 재사용성  | 좋음                      | 낮음                               |
| 실무 추천 | ✅ 적극 권장                 | 제한적 상황에서만 사용                     |

---

## 🧾 예제 코드

### 📘 1. `PersonalDetails` (복합키)

```java
import java.io.Serializable;
import javax.persistence.Embeddable;

@Getter
@Setter
@Embeddable
public class PersonalDetails implements Serializable {       

        private int userId;

        private String firstName;

        private String lastName;       

        
        @Override
        public int hashCode() {

                final int prime = 31;

                int result = 1;

                result = prime * result

                                + ((firstName == null) ? 0 : firstName.hashCode());

                result = prime * result

                                + ((lastName == null) ? 0 : lastName.hashCode());

                result = prime * result + userId;

                return result;

        }

 

        @Override
        public boolean equals(Object obj) {

                if (this == obj)

                        return true;

                if (obj == null)

                        return false;

                if (getClass() != obj.getClass())

                        return false;

                PersonalDetails other = (PersonalDetails) obj;

                if (firstName == null) {

                        if (other.firstName != null)

                                return false;

                } else if (!firstName.equals(other.firstName))

                        return false;

                if (lastName == null) {

                        if (other.lastName != null)

                                return false;

                } else if (!lastName.equals(other.lastName))

                        return false;

                if (userId != other.userId)

                        return false;

                return true;

        }
}
```

### 📙 2. `AddressDetails` (내장 필드)

```java
@Getter
@Setter
@Embeddable
public class AddressDetails {
    private String city;
    private String state;
    private int zipCode;
}
```

### 📗 3. `User` (엔티티)

```java
@Getter
@Setter
@Entity
@Table(name = "USER")
public class User {

    @EmbeddedId
    private PersonalDetails personalDetails;

    @Embedded
    private AddressDetails addressDetails;
}
```
### 📗 4. Test
```java
import javax.persistence.EntityManager;

import javax.persistence.EntityManagerFactory;

import javax.persistence.Persistence;

 

public class Test {

        public static void main(String[] args) {

 

                // Create EntityManagerFactory

                EntityManagerFactory emf = Persistence.createEntityManagerFactory("UserPU");

 

                PersonalDetails personalDetails = new PersonalDetails();

                personalDetails.setFirstName("Manu");

                personalDetails.setLastName("Manjunatha");

                personalDetails.setUserId(1);

               

                AddressDetails addressDetails = new AddressDetails();

                addressDetails.setCity("Bangalore");

                addressDetails.setState("Karnataka");

                addressDetails.setZipCode(560001);

               

                // Create Entity

                User user = new User();

                user.setPersonalDetails(personalDetails);

                user.setAddressDetails(addressDetails);

               

                // Create EntityManager

                EntityManager em = emf.createEntityManager();

               

                //Remove the record if already exists

                em.getTransaction().begin();

                if (em.find(User.class, personalDetails) != null) {

                        em.remove(em.find(User.class, personalDetails));

                }

                em.getTransaction().commit();

 

                // Persist entity

                em.getTransaction().begin();

                em.persist(user);

                em.getTransaction().commit();

               

                System.out.println("\n~~~~~~~~~~~~~Persisted user record~~~~~~~~~~~~");

        }

}
```
---

## 💾 결과 테이블 구조

```sql
CREATE TABLE USER (
    USERID INT NOT NULL,
    FIRSTNAME VARCHAR(45),
    LASTNAME VARCHAR(45),
    CITY VARCHAR(45),
    STATE VARCHAR(45),
    ZIPCODE NUMERIC(15),
    PRIMARY KEY (USERID, FIRSTNAME, LASTNAME)
);
```

→ `PersonalDetails`의 모든 필드가 PK로 지정됨

---

## 🎓 실무에서 언제 사용하나?

| 사용 시나리오                  | 설명                              |
| ------------------------ | ------------------------------- |
| 여러 컬럼이 조합된 키가 필요한 경우     | 예: 주문 ID + 상품 ID로 구성된 주문 상세 테이블 |
| 동일한 복합키를 여러 엔티티에서 공유할 경우 | 객체 재사용 가능                       |
| 설계 원칙 준수                 | Value Object로 모델링할 때 유리         |

---

## ⚠️ 주의사항

| 항목              | 주의 내용                                              |
| --------------- | -------------------------------------------------- |
| Serializable    | `@EmbeddedId` 클래스는 반드시 `Serializable` 구현 필요        |
| equals/hashCode | 정확히 override하지 않으면 조회/삭제 불가능                       |
| 컬럼 순서           | DB의 PK 순서와 필드 순서가 일치해야 함                           |
| 변경 금지           | 복합키는 불변성이 원칙, **Setter 사용을 피하고 생성자로만 세팅하는 것이 이상적** |

---

## 🧠 고급 팁: @MapsId와 조합

* `@MapsId`를 이용하면 복합키 중 일부를 다른 엔티티와 외래키로 연결할 수 있음

```java
@EmbeddedId
private OrderItemId id;

@ManyToOne
@MapsId("orderId")
@JoinColumn(name = "order_id")
private Order order;
```

---

## 📚 참고 자료

* 📘 [Jakarta Persistence `@EmbeddedId`](https://jakarta.ee/specifications/persistence/3.1/apidocs/jakarta.persistence/jakarta/persistence/EmbeddedId.html)
* 📘 [Hibernate Composite Key Docs](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#identifiers-composite)

---

## ✅ 마무리 요약

| 항목     | 요약                                               |
| ------ | ------------------------------------------------ |
| 핵심 목적  | 복합키를 하나의 객체로 표현하여 @Id 대체                         |
| 필요한 조건 | `@Embeddable`, `Serializable`, `equals/hashCode` |
| 장점     | 재사용성, 캡슐화, 유지보수성 향상                              |
| 실무 권장  | ✅ 매우 권장                                          |


# 🔍 JPA 콜백 메서드와 엔티티 리스너

> JPA의 콜백 메서드(Callback Method)와 엔티티 리스너(Entity Listener)를 마스터하여 엔티티 생명주기를 정밀하게 제어하기.

---

## ✅ 개요: 엔티티 생명주기 이벤트란?

JPA는 엔티티 객체가 생성되고 소멸되는 일련의 생명주기(Lifecycle) 이벤트를 제공합니다. 각 이벤트 단계에서 특정 메서드를 자동으로 호출할 수 있으며, 이를 통해 **감사 로깅, 파생 필드 설정, 데이터 정제 작업** 등을 자동화할 수 있습니다.

JPA는 이러한 작업을 위해 **7개의 표준 콜백 어노테이션**을 제공합니다. 또한, 이 로직을 엔티티에 직접 작성하지 않고 **외부 리스너 클래스**에 위임할 수도 있습니다.

---

## 🔁 생명주기 콜백 어노테이션 정리

| 어노테이션          | 호출 시점            | 설명                      |
| -------------- | ---------------- | ----------------------- |
| `@PrePersist`  | `persist()` 호출 전 | 영속 상태로 전이되기 전에 실행       |
| `@PostPersist` | `persist()` 호출 후 | 영속 상태로 전이된 직후 실행        |
| `@PostLoad`    | DB에서 로딩된 직후      | 모든 EAGER 필드가 초기화된 이후 실행 |
| `@PreUpdate`   | `flush()` 직전     | 변경 사항이 DB에 반영되기 전에 실행   |
| `@PostUpdate`  | 트랜잭션 커밋 후        | 변경 사항이 실제 DB에 반영된 이후 실행 |
| `@PreRemove`   | `remove()` 호출 전  | 삭제되기 전에 실행              |
| `@PostRemove`  | 트랜잭션 커밋 후        | 삭제된 후 실행                |

> ☝ **주의:**
>
> * `PreUpdate`, `PreRemove` 등은 트랜잭션 커밋 이전에 호출되며,
> * `PostUpdate`, `PostRemove`는 커밋 후에 실행됩니다.

---

## 🧪 콜백 메서드 작성 규칙

* **반환값:** `void`여야 함
* **파라미터:** 없어야 함 (엔티티 메서드일 경우)
* **접근 제한자:** `public`, `protected`, `private` 모두 허용
* **중복 지정 가능:** 하나의 메서드에 여러 콜백 어노테이션 부여 가능

---

## 👩‍💻 예제: 콜백 메서드를 사용하는 엔티티

```java
@Entity
public class Employee {

    private String firstName;
    private String lastName;

    @Transient
    private String fullName;

    @ManyToMany
    private List<PhoneNumber> phoneNumber;

    @PostLoad
    public void setFullName() {
        fullName = firstName + " " + lastName;
    }

    @PreRemove
    public void logDeletion() {
        System.out.println("Deleting employee: " + fullName);
    }
}
```

* `@PostLoad`: 데이터베이스로부터 로딩 후 `fullName` 재계산
* `@PreRemove`: 삭제 전 로깅 수행

---

## 🔌 외부 리스너 클래스를 통한 생명주기 분리

엔티티 클래스에 콜백 로직을 직접 작성하는 것은 **관심사의 분리(SOC)** 원칙을 위반할 수 있습니다. 이를 보완하기 위해 JPA는 **Entity Listener**를 지원합니다.

### ☑ 리스너 클래스 요건

* **기본 생성자 필요**
* **콜백 메서드는 `Object` 타입의 아규먼트 하나를 가짐**
* **리턴 타입은 `void`**
* `@PostLoad`, `@PreRemove` 등 동일한 어노테이션 사용

---

## 📦 예제: 외부 엔티티 리스너 사용

### 👤 엔티티 클래스

```java
@Entity
@EntityListeners(EmployeeLogger.class)
public class Employee {

    private String firstName;
    private String lastName;

    @Transient
    private String fullName;

    // ... 생략 ...
}
```

### 🧾 리스너 클래스

```java
public class EmployeeLogger {

    @PostLoad
    public void setFullName(Object obj) {
        if (obj instanceof Employee employee) {
            employee.fullName = employee.firstName + " " + employee.lastName;
        }
    }

    @PreRemove
    public void logDeletion(Object obj) {
        if (obj instanceof Employee employee) {
            System.out.println("Deleting employee: " + employee.fullName);
        }
    }
}
```

> 💡 Java 16 이상이라면 `instanceof` 패턴 매칭 문법을 활용하면 더욱 깔끔하게 처리 가능!

---

## 🎯 실전 팁

| 상황                   | 팁                                |
| -------------------- | -------------------------------- |
| 엔티티 로딩 직후 DTO로 변환 필요 | `@PostLoad`에서 가공 필드 채워주기         |
| 데이터 변경 시 로깅          | `@PreUpdate` 또는 `@PostUpdate` 사용 |
| 삭제 전 감사 기록 남기기       | `@PreRemove` 또는 외부 감사 테이블 연동     |
| 공통 처리 로직이 많은 경우      | 외부 리스너 클래스로 분리하여 재사용성 확보         |

---

## ⚠️ 주의사항

* 콜백 메서드 내부에서 예외가 발생하면 트랜잭션이 롤백될 수 있으므로 반드시 **예외 처리**를 할 것
* **`@Transactional`은 콜백 메서드에서 사용하지 말 것**. 콜백 메서드는 자체 트랜잭션 컨텍스트에서 호출되지 않음
* 콜백은 JPA 내부에서 호출되므로 **Spring AOP가 적용되지 않음**

---

## 📌 정리

| 기능      | 설명                                       |
| ------- | ---------------------------------------- |
| 콜백 메서드  | 엔티티 클래스 내에 생명주기 처리 로직을 작성                |
| 엔티티 리스너 | 관심사를 분리하여 외부에서 생명주기 이벤트 처리               |
| 공통 규칙   | 반환값은 void, 파라미터는 없거나 Object 하나, 어노테이션 필수 |
| 실전 적용   | 로깅, 파생 필드 계산, 삭제 감사, 값 검증 등에 유용          |


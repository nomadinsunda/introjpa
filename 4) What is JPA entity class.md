# 🧬 JPA Entity 클래스란? - 진짜 전문가처럼 이해하기

> JPA의 심장이라 불리는 Entity 클래스는 단순한 POJO 이상입니다.
> 객체와 데이터베이스를 연결하고, 애플리케이션의 영속 계층을 구성하는 핵심입니다.

---

## 📌 1. Entity 클래스란 무엇인가?

**Entity 클래스**는 JPA에서 **데이터베이스의 테이블을 자바 객체로 매핑**하기 위해 정의된 클래스입니다.
이는 흔히 **Persistence 클래스**라고도 부르며, 일반적으로 다음의 특징을 갖습니다:

* **POJO(Plain Old Java Object)** 클래스여야 하며
* `@Entity` 어노테이션이 반드시 선언되어 있어야 합니다
* 클래스의 각 인스턴스는 **DB의 한 행(row) 또는 레코드** 를 의미합니다

```java
@Entity
public class User {
    @Id
    private Long id;

    private String name;
}
```

### 💡 핵심 특징

| 항목            | 설명                                    |
| ------------- | ------------------------------------- |
| 독립적 영속성       | `@Entity`가 붙은 클래스는 **독립적으로 쿼리 수행 가능** |
| 고유 ID 필요      | 각 객체는 **하나의 고유한 식별자(PK)** 를 가져야 함     |
| DB 테이블 1:1 매핑 | 하나의 Entity = 하나의 DB 테이블               |

---

## 🧱 2. Embeddable 클래스와의 차이

JPA는 두 종류의 영속 클래스 유형을 정의합니다:

| 구분         | @Entity           | @Embeddable                |
| ---------- | ----------------- | -------------------------- |
| 독립적 존재 가능  | ✅ 가능              | ❌ 불가능                      |
| 쿼리 독립 수행   | 가능                | 불가능 (다른 Entity에 의존)        |
| 고유 식별자(PK) | 필요함               | 없음                         |
| 저장 방식      | 별도 테이블            | 소유 Entity와 동일 테이블 공유       |
| 예시         | `User`, `Product` | `Address`, `Period` 등 부속정보 |

```java
@Embeddable
public class Address {
    private String city;
    private String street;
}
```

```java
@Entity
public class User {
    @Id private Long id;

    @Embedded
    private Address address;
}
```

> `"SELECT a FROM Address a"` → 결과 없음
> `"SELECT u.address FROM User u"` → 정상 동작

---

## ⚠️ 3. Entity 클래스 설계 시 제약 조건

JPA에서 Entity 클래스는 다음과 같은 제약 조건을 따라야 합니다:

### ✅ 필수 요구사항

* 반드시 **POJO 클래스**일 것
* **디폴트 생성자 (no-arg constructor)** 존재해야 함 (`newInstance()` 사용 시 필요)
* `final class`, `final method`, `static field` → ❌ 허용되지 않음
* **직렬화 필요 없음**, 하지만 Serializable을 구현해도 무방

### 👨‍💻 권장 사항

* 클래스명은 가능하면 **DB 테이블명과 동일**하게 설정
* 필드명은 DB 컬럼명과 일치하도록 네이밍
* `@Table(name="...")` 또는 `@Column(name="...")` 으로 명시 가능

---

## 🔁 4. 버전 관리 필드 (@Version)

멀티 쓰레드 또는 멀티 트랜잭션 환경에서 동일 레코드가 동시에 수정되면 **데이터 손실이나 덮어쓰기**가 발생할 수 있습니다. 이를 방지하기 위해 JPA는 **낙관적 잠금(Optimistic Locking)** 을 지원하며, 그 핵심이 `@Version` 필드입니다.

```java
@Version
private int version;
```

* 엔티티 수정 시 `version` 값이 자동으로 증가
* 동시에 두 트랜잭션이 같은 엔티티를 수정하면 **예외(Exception)** 발생 → 마지막 커밋이 차단됨

---

## 🧬 5. 상속과 Entity 클래스

JPA는 Entity 간 **상속 구조**를 지원합니다. 하지만 몇 가지 중요한 제약도 있습니다:

| 허용되는 경우           | 제한되는 경우                                   |
| ----------------- | ----------------------------------------- |
| Entity → Entity   | Entity → 시스템 클래스 (ex. `Thread`, `Socket`) |
| Entity → POJO 클래스 | 슈퍼클래스 필드는 영속 불가                           |
| 비영속 클래스 → Entity  | ✅ 가능                                      |

```java
@MappedSuperclass
public class BaseEntity {
    private LocalDateTime createdDate;
}
```

```java
@Entity
public class Member extends BaseEntity {
    @Id private Long id;
}
```

모든 상속 구조 내 클래스는 **동일한 식별자 타입**을 가져야 합니다.

---

## 🧮 6. 필드 타입에 따른 JPA 지원 여부

JPA에서 Entity 클래스의 필드는 **불변 타입(immutable)** 과 **가변 타입(mutable)** 로 나뉘며, 각각 다르게 취급됩니다.

### ✅ 불변 타입 (immutable types)

| 타입                                   | 설명                         |
| ------------------------------------ | -------------------------- |
| 기본형 (int, long 등)                    | 변경 불가능                     |
| Wrapper (Integer 등)                  | 변경 불가능                     |
| `String`, `BigDecimal`, `BigInteger` | 불변 객체                      |
| 배열                                   | 배열 자체는 변경 가능하지만, 요소 접근은 주의 |

> 변경하려면 **전체 필드 교체** 방식으로 처리해야 함

---

### ✅ 가변 타입 (mutable types)

| 타입                                           | 설명              |
| -------------------------------------------- | --------------- |
| `java.util.Date`, `Calendar`                 | 내부 상태 수정 가능     |
| `java.sql.Date`, `Time`, `Timestamp`         | JPA에서 자동 지원     |
| `Enum`, `Embeddable`, `Entity`, `Collection` | 관계형 필드 및 열거형 지원 |

> 가변 필드는 객체 내부 상태만 변경해도 DB 반영됨

---

### 📚 컬렉션 필드 지원

JPA는 다음과 같은 **관계 매핑 컬렉션**을 필드로 허용합니다:

* `Collection<T>`
* `List<T>`
* `Set<T>`
* `Map<K, V>`: 엔티티 키-값 쌍 매핑

---

## 📌 마무리: JPA Entity 클래스 설계 체크리스트

| 항목                                  | 체크 |
| ----------------------------------- | -- |
| `@Entity` 선언 여부                     | ✅  |
| 디폴트 생성자 존재 여부                        | ✅  |
| ID 필드 (`@Id`) 정의                    | ✅  |
| 불변 타입/가변 타입 처리 방식 구분                | ✅  |
| 상속 구조 고려                            | ✅  |
| Version 필드로 동시성 제어 필요 시 @Version 사용 | ✅  |
| `@Table`, `@Column` 등으로 명시적 매핑 제어   | ✅  |



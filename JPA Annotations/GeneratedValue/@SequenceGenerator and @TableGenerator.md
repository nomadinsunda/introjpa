# JPA @GeneratedValue, @SequenceGenerator, @TableGenerator

Java Persistence API (JPA)에서는 기본 키(Primary Key)를 자동으로 생성해주는 전략을 제공하며, 이를 위해 `@GeneratedValue`, `@SequenceGenerator`, `@TableGenerator`와 같은 애노테이션을 사용합니다. 이 글에서는 각 애노테이션의 역할과 속성, 전략별 차이점, 그리고 실전 예제를 전문가 관점에서 상세히 설명합니다.

---

## 1. @GeneratedValue

`@GeneratedValue` 애노테이션은 엔티티의 식별자 필드(ID)에 대해 **자동 생성 전략**을 지정합니다.

### 주요 속성

* **strategy**: 어떤 방식으로 기본 키를 생성할지를 정의
* **generator**: 이름 붙은 시퀀스나 테이블 제너레이터를 참조할 때 사용 (GenerationType.SEQUENCE, GenerationType.TABLE에 필수)

```java
@GeneratedValue(strategy = GenerationType.IDENTITY)
```

---

## 2. GenerationType 전략 종류

### 2.1. GenerationType.IDENTITY

* **DB에 위임**: MySQL, PostgreSQL의 `AUTO_INCREMENT`, `SERIAL` 등 DB가 직접 식별자 생성
* **JPA가 식별자 미리 알 수 없음**: `persist()` 시점에 insert SQL이 즉시 실행됨
* **전략 특징**:

  * 트랜잭션 버퍼링 불가
  * 배치 성능 저하 가능

```sql
CREATE TABLE student (
  id INT AUTO_INCREMENT PRIMARY KEY,
  firstname VARCHAR(20),
  lastname VARCHAR(20)
);
```

---

### 2.2. GenerationType.SEQUENCE

* **DB 시퀀스 객체 사용**
* PostgreSQL, Oracle 등에서 사용 가능
* **미리 시퀀스 값을 가져와 할당 가능** → `allocationSize` 활용 가능
* **generator** 파라미터에 시퀀스 이름 지정 필수

```java
@SequenceGenerator(
    name = "student_seq_gen",
    sequenceName = "student_seq",
    initialValue = 1,
    allocationSize = 50
)
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "student_seq_gen")
```

```sql
CREATE SEQUENCE student_seq START 1 INCREMENT 1;
```

---

### 2.3. GenerationType.TABLE

* **JPA가 전용 테이블 생성하여 ID 관리**
* **모든 DB 호환** → 가장 이식성 좋음
* **성능 낮음**: 테이블 조회/갱신 필요

```java
@TableGenerator(
    name = "student_tbl_gen",
    table = "id_generator_table",
    pkColumnName = "gen_name",
    valueColumnName = "gen_val",
    pkColumnValue = "student_id",
    initialValue = 1,
    allocationSize = 1
)
@GeneratedValue(strategy = GenerationType.TABLE, generator = "student_tbl_gen")
```

```sql
CREATE TABLE id_generator_table (
    gen_name VARCHAR(50) PRIMARY KEY,
    gen_val INT
);
INSERT INTO id_generator_table(gen_name, gen_val) VALUES ('student_id', 1);
```

---

### 2.4. GenerationType.AUTO

* **JPA 구현체가 DB 벤더에 따라 자동 결정**

  * MySQL → IDENTITY
  * PostgreSQL → SEQUENCE

```java
@GeneratedValue(strategy = GenerationType.AUTO)
```

---

## 3. @SequenceGenerator 설명

시퀀스를 사용하는 DB에서 JPA가 사용할 시퀀스를 정의합니다.

### 주요 속성

* `name`: 제너레이터의 JPA 내부 이름
* `sequenceName`: DB의 시퀀스 객체 이름
* `initialValue`: 초기 값 (기본 1)
* `allocationSize`: 미리 할당할 ID 수 → 퍼포먼스 향상

JPA는 내부적으로 메모리에 allocationSize 만큼 시퀀스 값을 가져와 캐시해 사용합니다.

---

## 4. @TableGenerator 설명

시퀀스를 지원하지 않는 DB를 위해 JPA가 **테이블을 이용해 시퀀스처럼 동작**하도록 설정합니다.

### 주요 속성

* `table`: 사용할 테이블 이름
* `pkColumnName`: 시퀀스 식별 컬럼 (ex. "GEN\_NAME")
* `valueColumnName`: 현재 값 컬럼 (ex. "GEN\_VAL")
* `pkColumnValue`: 이 테이블에서 사용할 논리 이름 (ex. "student\_id")

---

## 5. @GeneratedValue vs @SequenceGenerator / @TableGenerator

| 항목              | @GeneratedValue           | @SequenceGenerator/@TableGenerator |
| --------------- | ------------------------- | ---------------------------------- |
| 역할              | ID 생성 전략 지정               | 생성 전략의 상세 설정 정의                    |
| generator 필드 사용 | 선택적 (SEQUENCE, TABLE에 필수) | generator 이름 지정 필수                 |

---

## 6. 예제 종합 (Student 엔티티)

```java
@Entity
@Table(name = "student")
@SequenceGenerator(
    name = "student_seq_gen",
    sequenceName = "student_seq",
    initialValue = 1,
    allocationSize = 50
)
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "student_seq_gen")
    private Long id;

    private String firstName;
    private String lastName;
}
```

---

## 7. 실전 고려사항

* **allocationSize 성능**:

  * 적절히 크면 DB 호출 줄어 성능 ↑
  * 너무 크면 ID 간격이 넓어짐 (정책에 따라 문제 될 수 있음)

* **IDENTITY는 배치 처리에 불리**

  * INSERT 시점에 키를 받아야 하므로 1건씩 처리됨

* **SEQUENCE + allocationSize** 조합이 성능, 이식성에서 가장 우수 (지원 DB라면)

---

## 🔚 결론

* JPA에서 ID 생성은 단순히 @GeneratedValue만으로 끝나는 것이 아닙니다.
* 전략을 이해하고 DB에 맞는 설정을 적용해야 퍼포먼스와 이식성을 극대화할 수 있습니다.
* 특히 실무에서는 **allocationSize 설정**을 적극 고려해야 합니다.



# 🔑 JPA `@GeneratedValue` 어노테이: ID 자동 생성 전략의 모든 것

> 객체를 저장할 때마다 자동으로 고유한 ID를 생성해주는 `@GeneratedValue`. 단순한 자동 증가 이상의 전략과 최적화가 숨어 있습니다.

---

## ✅ 1. 개념 정리: `@GeneratedValue`란?

`@GeneratedValue`는 엔티티의 기본 키를 자동으로 생성하도록 지시하는 JPA 표준 애노테이션입니다.

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

* **필수 조건:** 반드시 `@Id`와 함께 사용
* **역할:** ID 값을 "어떻게 얻을지" 전략만 지정
* **실제 값 생성은 DB 또는 시퀀스/테이블/자동 전략이 담당**

---

## ⚙️ 2. 주요 속성

| 속성          | 설명                                  |
| ----------- | ----------------------------------- |
| `strategy`  | 어떤 방식으로 ID를 생성할지 지정                 |
| `generator` | 명명된 시퀀스 또는 테이블 생성기 참조 (필수 전략에서만 사용) |

---

## 🧠 3. 전략별 상세 설명

### ① `GenerationType.AUTO` (기본값)

* **JPA 구현체가 DB에 따라 자동으로 결정**
* MySQL, SQL Server, SQLite → `IDENTITY`
* Oracle, PostgreSQL → `SEQUENCE`

```java
@GeneratedValue(strategy = GenerationType.AUTO)
```

> ✅ **장점:** 이식성 우수
> ❗ **단점:** 제어 불가능, 성능 최적화 제한

---

### ② `GenerationType.IDENTITY`

* **DB의 IDENTITY(자동 증가) 기능 사용**
* MySQL: `AUTO_INCREMENT`
* PostgreSQL: `SERIAL`
* SQL Server: `IDENTITY`

```java
@GeneratedValue(strategy = GenerationType.IDENTITY)
```

> ✅ **장점:** 구현 단순, 성능 우수 (DB에서 바로 증가)
> ❗ **단점:** ID값 알기 전까지 INSERT 불가 → `em.persist()` 후에만 ID 접근 가능

---

### ③ `GenerationType.SEQUENCE` + `@SequenceGenerator`

* **DB의 시퀀스 객체 사용**
* Oracle, PostgreSQL 지원

```java
@SequenceGenerator(name = "student_seq", sequenceName = "STUDENT_SEQ", allocationSize = 10)
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "student_seq")
```

* `allocationSize`: 메모리에서 미리 증가값을 할당받아 DB 접근 최소화

> ✅ **장점:** 성능 우수, 유연한 커스터마이징 가능
> ❗ **단점:** MySQL에서는 사용 불가 (시퀀스 없음)

---

### ④ `GenerationType.TABLE` + `@TableGenerator`

* **시퀀스를 별도 테이블로 시뮬레이션**
* 모든 DB에서 사용 가능 (이식성 최고)

```java
@TableGenerator(
    name = "student_tbl_gen",
    table = "ID_GEN",
    pkColumnName = "GEN_NAME",
    valueColumnName = "GEN_VAL",
    pkColumnValue = "STUDENT_SEQ",
    initialValue = 0,
    allocationSize = 5
)
@GeneratedValue(strategy = GenerationType.TABLE, generator = "student_tbl_gen")
```

> ✅ **장점:** DB 독립적
> ❗ **단점:** 성능 최악 (매 INSERT마다 SELECT + UPDATE)

---

## 🔄 4. 전략별 비교 요약

| 전략         | DB 지원              | 성능    | 이식성   | 특징                               |
| ---------- | ------------------ | ----- | ----- | -------------------------------- |
| `AUTO`     | 모두                 | 중     | 높음    | DB에 따라 자동 선택                     |
| `IDENTITY` | MySQL 등            | 높음    | 낮음    | DB에서 직접 증가                       |
| `SEQUENCE` | Oracle, PostgreSQL | 매우 높음 | 중간    | 시퀀스 객체 사용, `allocationSize`로 최적화 |
| `TABLE`    | 모든 DB              | 낮음    | 매우 높음 | 시퀀스 테이블로 구현, 이식성 우수              |

---

## 📦 5. 실전 예제 코드

### 📌 MySQL용 (IDENTITY 기반)

```java
@Entity
@Table(name = "STUDENT")
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private int id;

    private String firstName;
    private String lastName;
}
```

### 📌 Oracle/PostgreSQL용 (SEQUENCE 기반)

```java
@Entity
@Table(name = "STUDENT")
@SequenceGenerator(
    name = "student_seq_gen",
    sequenceName = "STUDENT_SEQ",
    initialValue = 1,
    allocationSize = 10
)
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "student_seq_gen")
    private int id;
}
```

### 📌 DB 독립적 전략 (TABLE 기반)

```java
@Entity
@Table(name = "STUDENT")
@TableGenerator(
    name = "student_tbl_gen",
    table = "ID_GEN",
    pkColumnName = "GEN_NAME",
    valueColumnName = "GEN_VAL",
    pkColumnValue = "STUDENT_SEQ",
    allocationSize = 5
)
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE, generator = "student_tbl_gen")
    private int id;
}
```

---

## 📚 참고 설정 (DDL)

### ▶ 시퀀스 생성 예 (Oracle)

```sql
CREATE SEQUENCE STUDENT_SEQ START WITH 1 INCREMENT BY 1;
```

### ▶ 시퀀스 테이블 생성 예 (TABLE 전략)

```sql
CREATE TABLE ID_GEN (
  GEN_NAME VARCHAR(50) PRIMARY KEY,
  GEN_VAL INT
);
INSERT INTO ID_GEN (GEN_NAME, GEN_VAL) VALUES ('STUDENT_SEQ', 0);
```

---

## 💡 실무 팁

| 상황                 | 전략 추천                                            |
| ------------------ | ------------------------------------------------ |
| MySQL, SQL Server  | `GenerationType.IDENTITY`                        |
| Oracle, PostgreSQL | `GenerationType.SEQUENCE` + `allocationSize` 최적화 |
| 모든 DB 이식성 필요       | `GenerationType.TABLE`                           |
| 대규모 시스템            | 시퀀스 + `allocationSize` ↑ (예: 100, 1000) 로 성능 개선  |

---

## ✅ 마무리 요약

| 핵심 개념                | 정리                              |
| -------------------- | ------------------------------- |
| `@GeneratedValue`    | ID 자동 생성 방식 선언                  |
| `strategy`           | IDENTITY, SEQUENCE, TABLE, AUTO |
| `generator`          | 명명된 시퀀스 또는 테이블 참조               |
| `@SequenceGenerator` | 시퀀스를 통한 고속 ID 생성                |
| `@TableGenerator`    | 시퀀스 테이블 사용, 완전 이식성 보장           |
| 권장 방식                | DBMS에 맞는 전략 선택, 성능 튜닝 필요        |

---



PostgreSQL의 \*\*시퀀스(Sequence)\*\*는 **자동 증가하는 정수 값을 생성하는 객체**입니다. 주로 기본 키를 자동으로 생성하기 위해 사용되며, `AUTO_INCREMENT`를 직접 지원하지 않는 PostgreSQL에서 이를 대신하여 사용됩니다.

---

## 🧱 1. 시퀀스란?

* 시퀀스는 독립적인 **카운터 객체**입니다.
* 테이블과는 별도로 존재하며, 다음 번호를 **자동으로 생성**하는 데 사용됩니다.
* 주로 **기본 키 값 생성용**으로 활용됩니다.

---

## 🛠️ 2. 시퀀스 생성

```sql
CREATE SEQUENCE user_id_seq;
```

옵션을 포함한 전체 문법:

```sql
CREATE SEQUENCE sequence_name
  [ INCREMENT BY n ]
  [ MINVALUE m ]
  [ MAXVALUE x ]
  [ START WITH s ]
  [ CACHE c ]
  [ CYCLE | NO CYCLE ];
```

### 주요 옵션 설명

| 옵션             | 설명                             | 기본값      |
| -------------- | ------------------------------ | -------- |
| `INCREMENT BY` | 증가 값 설정                        | 1        |
| `START WITH`   | 시작값 설정                         | 1        |
| `MINVALUE`     | 최소값 설정                         | 1        |
| `MAXVALUE`     | 최대값 설정                         | 없음       |
| `CACHE`        | 메모리에서 미리 할당할 수치 수              | 1        |
| `CYCLE`        | 최대값 도달 시 다시 MINVALUE부터 시작할지 여부 | NO CYCLE |

---

## 🚀 3. 다음 값 가져오기

```sql
SELECT nextval('user_id_seq');
```

* 시퀀스를 1만큼 증가시키고 반환
* 일반적으로 `INSERT`와 함께 사용됨

```sql
INSERT INTO users (id, name)
VALUES (nextval('user_id_seq'), 'Alice');
```

---

## 🔁 현재 값 보기

```sql
SELECT currval('user_id_seq');
```

* **현재 세션 내에서 가장 최근 호출된 nextval의 결과**를 반환
* nextval 호출 없이 사용하면 오류 발생

---

## ↩️ 이전 값 재사용

```sql
SELECT setval('user_id_seq', 100);
```

* 시퀀스를 **임의의 값으로 수동 설정**

옵션 사용 예:

```sql
SELECT setval('user_id_seq', 100, false);
```

* 다음 `nextval()` 호출 시 **100을 반환**

---

## ⚙️ 4. 테이블에 자동 연결: `SERIAL` vs `GENERATED`

PostgreSQL은 시퀀스를 간편하게 설정하는 **축약 문법**을 제공합니다.

### ✅ `SERIAL`

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100)
);
```

* 내부적으로 아래처럼 처리됨:

```sql
CREATE SEQUENCE users_id_seq;
ALTER TABLE users ALTER COLUMN id SET DEFAULT nextval('users_id_seq');
```

| 타입          | 설명                             |
| ----------- | ------------------------------ |
| `SERIAL`    | `INTEGER` + sequence + default |
| `BIGSERIAL` | `BIGINT` + sequence + default  |

> ⚠️ `SERIAL`은 시퀀스를 자동으로 생성하지만, JPA에서는 내부 시퀀스 명칭을 알기 어렵기 때문에 추천하지 않습니다.

---

### ✅ `GENERATED ALWAYS AS IDENTITY` (SQL 표준)

PostgreSQL 10 이상:

```sql
CREATE TABLE users (
  id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name VARCHAR(100)
);
```

* 내부적으로 시퀀스를 생성하지만 자동 관리되며 `SERIAL`보다 표준 SQL에 더 가깝습니다.
* `OVERRIDING` 키워드를 통해 수동 입력도 가능

---

## 🔍 5. 시퀀스 조회 및 삭제

### 목록 조회

```sql
\dS  -- 시퀀스 목록 (psql)
```

또는

```sql
SELECT * FROM pg_sequences;
```

### 삭제

```sql
DROP SEQUENCE user_id_seq;
```

또는 `CASCADE`로 의존 객체까지 함께 삭제:

```sql
DROP SEQUENCE user_id_seq CASCADE;
```

---

## 🧠 6. 시퀀스의 캐시(CACHE) 성능 최적화

```sql
CREATE SEQUENCE fast_seq CACHE 100;
```

* 메모리 내에서 시퀀스 값을 100개 미리 가져다 놓음
* 성능 향상되지만 **서버 중단 시 중복 또는 손실 가능성 존재**

---

## 🆚 7. JPA와 연동

PostgreSQL에서 JPA의 `GenerationType.SEQUENCE`를 사용할 경우, 직접 시퀀스를 생성하고 `@SequenceGenerator`를 명시해야 함.

```java
@Entity
@SequenceGenerator(
    name = "user_seq_gen",
    sequenceName = "user_id_seq",  // 실제 PostgreSQL 시퀀스 이름
    initialValue = 1,
    allocationSize = 1
)
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq_gen")
    private Long id;
}
```

Hibernate는 `allocationSize`만큼 미리 시퀀스 값을 가져오므로 DB 접근 수를 줄일 수 있습니다.

---

## 📊 8. 시퀀스 vs AUTO\_INCREMENT (MySQL)

| 항목        | PostgreSQL        | MySQL           |
| --------- | ----------------- | --------------- |
| 자동 증가 기능  | 시퀀스 (SEQUENCE)    | AUTO\_INCREMENT |
| SQL 표준 준수 | O (IDENTITY 지원)   | X               |
| 독립 객체     | O (테이블과 별개)       | X (테이블 내에 존재)   |
| 재사용 가능성   | O (여러 테이블이 공유 가능) | X               |
| 시퀀스 이름 설정 | 가능                | 불가능             |
| JPA 호환성   | SEQUENCE 전략 필요    | IDENTITY 전략 필요  |

---

## ✅ 요약

| 항목     | 설명                                                  |
| ------ | --------------------------------------------------- |
| 시퀀스란   | 자동으로 증가하는 수를 생성하는 객체                                |
| 사용 목적  | 주로 기본 키 ID 생성용                                      |
| 생성 방식  | `CREATE SEQUENCE`, 또는 `SERIAL`, `IDENTITY`          |
| 주요 함수  | `nextval()`, `currval()`, `setval()`                |
| JPA 연동 | `GenerationType.SEQUENCE` + `@SequenceGenerator` 필요 |
| 캐시 활용  | 성능 최적화를 위한 `CACHE` 옵션 제공                            |

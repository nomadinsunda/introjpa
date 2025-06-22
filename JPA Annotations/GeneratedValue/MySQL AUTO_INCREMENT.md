MySQL에서 `AUTO_INCREMENT`는 **테이블의 특정 열에 대해 자동으로 증가하는 고유 값을 생성**하도록 지원하는 속성입니다. 주로 \*\*기본 키(primary key)\*\*로 사용되는 정수형 컬럼에 설정되며, 새 레코드가 삽입될 때마다 이전 값보다 **1씩 증가한 고유 숫자**가 자동으로 할당됩니다.

---

## 🔍 1. 기본 문법

```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  username VARCHAR(50) NOT NULL
);
```

* `id` 열은 자동으로 증가하며, `PRIMARY KEY`로 설정되어 중복 없이 고유한 값을 갖습니다.

---

## ⚙️ 2. 동작 방식

1. 테이블 생성 시 `AUTO_INCREMENT` 열이 설정되면, MySQL은 **내부적으로 마지막으로 삽입된 값을 기억**합니다.
2. 새로운 행이 `INSERT`될 때 해당 컬럼(Primary Key)을 생략하면 자동으로 다음 숫자가 채워집니다.
3. 디폴트로 **1부터 시작**하여 **1씩 증가**합니다 (이는 변경 가능함).

```sql
INSERT INTO users (username) VALUES ('Alice');  -- id = 1
INSERT INTO users (username) VALUES ('Bob');    -- id = 2
```

---

## 🔄 3. 현재 AUTO\_INCREMENT 값 확인 및 변경

### ✅ 확인

```sql
SHOW TABLE STATUS LIKE 'users';
```

* `Auto_increment` 컬럼에 다음 삽입될 값이 나옵니다.

### 🔧 변경

```sql
ALTER TABLE users AUTO_INCREMENT = 1000;
```

* 이후 삽입되는 행은 `id = 1000`부터 시작됩니다.

---

## 🔁 4. 삭제 시 재사용 여부

* 기본적으로 **삭제된 ID는 재사용되지 않습니다**.
* 예외: **ROLLBACK된 트랜잭션** 혹은 **세션 실패** 등은 ID 누락을 유발할 수 있습니다.

---

## 🧮 5. 증분 값 변경 (기본 1씩 증가)

* `auto_increment_increment`: 증가 간격 (기본값 1)
* `auto_increment_offset`: 시작 위치

```sql
SET @@auto_increment_increment = 5;
SET @@auto_increment_offset = 2;
```

예시: 2, 7, 12, 17, ...

> ⚠️ 이 설정은 주로 **Replication 환경에서 충돌 방지** 용도로 사용됩니다.

---

## 🧪 6. 테이블 여러 개에서 AUTO\_INCREMENT 공유는 안 됨

MySQL에서는 `AUTO_INCREMENT`는 **테이블 단위로 관리**되므로, **여러 테이블 간에 값을 공유하거나 조정하는 기능은 없습니다**. 만약 그런 요구가 있다면 아래 방법을 사용해야 합니다:

* `SEQUENCE` 대신 사용할 별도 테이블 생성 (예: `sequence_table`)
* 어플리케이션 레벨에서 값 관리
* 트리거 및 저장 프로시저 사용

---

## 🚨 7. 주의사항

| 항목         | 설명                                     |
| ---------- | -------------------------------------- |
| 트랜잭션과 무관   | ID는 `INSERT` 실행 시점에 증가함. 롤백해도 숫자는 소비됨. |
| 중복 방지 보장 X | `UNIQUE` 제약 조건과 함께 사용해야 함.             |
| ID gap 가능성 | 중간에 실패하거나 삭제되면 ID에 공백 발생 가능.           |

---

## 🆚 JPA와의 관계

* JPA의 `GenerationType.IDENTITY` 전략은 MySQL의 `AUTO_INCREMENT`와 정확히 대응됩니다.

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

* Hibernate는 `IDENTITY` 전략을 사용하면 **JDBC 레벨에서 insert 이후에 생성된 키를 조회**합니다.
* 주의: IDENTITY 전략은 **batch insert, insert-before-flush** 최적화를 방해합니다.

---

## ✅ 요약

| 항목       | 설명                            |
| -------- | ----------------------------- |
| 용도       | 자동 증가하는 고유 식별자 생성             |
| 기본값      | 1부터 1씩 증가                     |
| 설정 방법    | `AUTO_INCREMENT` 키워드 사용       |
| 시점       | `INSERT` 실행 시 ID 할당           |
| 변경 가능 항목 | 시작값, 증가 간격                    |
| JPA 전략   | `GenerationType.IDENTITY`와 대응 |
| 제약       | gap 발생 가능성, 멀티 테이블 공유 불가      |


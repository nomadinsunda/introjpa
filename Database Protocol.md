**Database Protocol**은 데이터베이스 서버와 클라이언트(애플리케이션)가 서로 데이터를 주고받기 위해 사용하는 **통신 규약**을 말합니다.


---


# 🚀 DB 프로토콜(Database Protocol)
대부분의 DB 프로토콜은 네트워크 참조 모델인 OSI 7계층 중 최상단인 **응용 계층(Application Layer)**에 해당하며, 하부 전송 계층으로 주로 **TCP/IP**를 사용합니다.

### ― 클라이언트와 데이터베이스가 실제로 어떻게 통신하는가?

대부분의 개발자는 SQL을 작성하고 `executeQuery()` 또는 ORM(JPA/Hibernate)을 사용하면 **알아서 데이터베이스와 통신되고 결과가 온다**고만 알고 있습니다.
그러나 **네트워크 레벨에서 실제로 무슨 일이 벌어지는지**,
**어떤 프로토콜로 패킷이 오가며**,
**JDBC 드라이버가 SQL 문자열을 어떻게 패킷 프레임으로 변환하는지**
까지 이해하시는 분들은 많지 않습니다.

---

## 1️⃣ DB 프로토콜이란 무엇인가?

데이터베이스와 클라이언트는 모두 네트워크 위에서 데이터 패킷을 교환합니다.
이때 사용되는 규칙/스펙이 **Database Protocol**입니다.

📌 **정의**

> DB 프로토콜은 데이터베이스 클라이언트(JDBC, CLI, 드라이버 등)와 DB 서버(MySQL, MariaDB, PostgreSQL)가
> **인증 → 쿼리 전송 → 결과 반환 → 세션 종료**
> 전 과정에서 서로 이해할 수 있는 패킷 포맷을 정의한 통신 규약입니다.

* **L4 (TCP):** 데이터가 유실되지 않고 순서대로 전달되는 것을 보장합니다.
* **L7 (DB Protocol):** "로그인하고 싶다", "이 SQL을 실행해라", "결과를 보내달라"와 같은 구체적인 DB 명령어를 처리합니다.


즉, SQL 문자열 자체는 전송되지 않고,
**SQL을 구조화한 바이너리 패킷으로 변환되어 전송**됩니다.

---

## 2️⃣ 왜 DB 프로토콜을 알아야 하는가? (핵심 가치)

| 이해 요소                 | 설명                                     |
| --------------------- | -------------------------------------- |
| 🔥 성능 최적화             | 쿼리 실행 전 준비/파싱/바인딩 과정을 정확히 이해할 수 있음     |
| 🧵 Connection Pool 튜닝 | 세션 유지 비용, 프로토콜 handshake 이해            |
| 🔐 보안                 | SSL/TLS handshake, 인증 방식               |
| 🐞 디버깅                | JDBC/Hibernate에서 “쿼리가 이상하게 날아간다” 문제 파악 |

예:

> “Statement.RETURN_GENERATED_KEYS는 insert SQL에 안 붙는데 어떻게 서버가 키를 돌려주는가?”

이 질문도 **DB 프로토콜을 이해하면 1초 만에 해결**됩니다.
결론부터 말하면:
✔️ **SQL 문자열에 추가되는 것이 아니라**,
✔️ **패킷의 헤더에 플래그(flag)로 세팅되어 전송됩니다.**

---

## 3️⃣ DB 프로토콜의 전체 Flow

MySQL, PostgreSQL, Oracle 모두 구조는 거의 동일합니다.

```
[1] TCP 연결
      ↓
[2] Handshake(서버 능력 교환, 인증 방식 결정)
      ↓
[3] 인증(Authentication)
      ↓
[4] Command 패킷으로 SQL 실행
      ↓
[5] Result 패킷으로 응답 수신
      ↓
[6] 세션 종료(QUIT 패킷)
```

---

## 4️⃣ JDBC → DB 서버까지: 실제 패킷 흐름 🔍

## 📌 4.1 JDBC가 SQL을 패킷으로 변환하는 과정

예를 들어 다음 코드가 있다고 합시다.

```java
PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO Student(name) VALUES (?)",
    Statement.RETURN_GENERATED_KEYS
);
ps.setString(1, "홍길동");
ps.executeUpdate();
```

당신이 보는 것은 SQL 문자열이지만,
DB 서버가 받는 것은 **SQL 문자열이 아니다**.
아래와 같은 **Command Packet**이다.

### ▶️ 프로토콜 상 실제 패킷 구성 (MySQL 예)

```
[Command Type: COM_QUERY]
[Flags: RETURN_GENERATED_KEYS=1]
[SQL Text Length: 38]
[SQL Text: "INSERT INTO Student(name) VALUES (?)"]
[Parameter: "홍길동"]
```

👉 **RETURN_GENERATED_KEYS는 SQL 문자에 붙지 않음**
👉 **패킷 헤더의 플래그**로 서버에 전달됨
👉 서버는 그 플래그를 보고 **auto_increment 값을 같이 반환**

📌 즉, 이것이 바로 사용자께서 궁금해하셨던:

> “insert 쿼리에 Statement.RETURN_GENERATED_KEYS 안 붙는데 왜 동작하지?”

정답은:
**프로토콜 패킷 상단(헤더)에 담겨서 서버가 해석하기 때문입니다.**

---

## 5️⃣ MySQL 프로토콜 분석 🔬

MySQL은 다음 3종의 패킷을 사용합니다.

### ① Handshake Packet

서버 → 클라이언트

* 서버 capabilities
* 인증 방식
* salt(seed)

### ② OK / ERR Packet

클라이언트 요청에 대한 응답 정의

### ③ Command Packet

모든 SQL 실행은 다음과 같이 전송됩니다.

```
0x03  → COM_QUERY
SQL → "SELECT * FROM Student"
```

Auto-increment 반환을 요청하면:

```
capabilities_flags |= CLIENT_RETURN_GENERATED_KEYS
```

이렇게 플래그가 **추가되어 전송됩니다**.

---

## 6️⃣ PostgreSQL 프로토콜 분석 🔬

PostgreSQL은 MySQL보다 구조가 더 명확합니다.

### ▶️ 주요 메시지 타입

| 코드 | 의미               |
| -- | ---------------- |
| Q  | Query            |
| P  | Prepare          |
| B  | Bind             |
| E  | Execute          |
| C  | Command Complete |
| D  | DataRow          |
| Z  | ReadyForQuery    |

특징:

* PreparedStatement 는 실제로 **P → B → E 패킷** 세트로 실행됨
* auto_increment 값 반환은 **RETURNING id** 로 처리 (기본 스펙)

---

## 7️⃣ 프로토콜 상의 PreparedStatement 동작 💡

클라이언트:

```
P (Parse)
B (Bind parameters)
E (Execute)
```

서버:

```
C (CommandComplete)
D (DataRow)
Z (Ready)
```

SQL이 문자열로 전송되지 않고:
✔️ SQL 구조화 정보
✔️ 파라미터 분리
✔️ 타입 정보
가 모두 **별도의 패킷**으로 전송됩니다.

---

## 8️⃣ SSL/TLS 보안 연결 🌐🔐

DB 연결에서 SSL/TLS는 다음 과정을 거칩니다.

```
Client Hello  →  Server Hello  
Certificate   →  Key Exchange  
Finished
```

이후부터 모든 패킷은
✔️ 암호화되어 전송
✔️ SQL 문자열도 패킷도 모두 암호화됨

---

## 9️⃣ 실제 쿼리 전송 예시 (WireShark 캡처 관점)

예를 들어 insert 쿼리를 보내면:

```
Packet 1: COM_QUERY + Flags + SQL
Packet 2: OK Packet + Generated Key
```

Generated Key는:

```
last_insert_id = 101
```

이렇게 서버의 **result packet**에 포함되어 옴.

---

## 🔟 JDBC가 왜 Statement 옵션을 SQL에 붙이지 않는가?

### 이유는 간단합니다.

**SQL을 그대로 DB 서버에 보내는 것이 아니라,
DB 프로토콜 패킷으로 재조합하기 때문입니다.**

따라서:

* Statement 옵션
* ResultSet 타입
* 자동 생성 키 요청
* Fetch size
  이 모든 것은 SQL 문자열이 아닌
  **프로토콜 플래그로 서버에 전달됩니다.**

---

# 🔥 결론 (DB 프로토콜의 본질 요약)

| 질문                            | 정답                    |
| ----------------------------- | --------------------- |
| SQL 문자열이 그대로 전송되는가?           | ❌ 아님. 패킷으로 구조화됨       |
| Statement 옵션은 SQL에 포함되는가?     | ❌ 패킷 헤더에 담김           |
| PreparedStatement는 텍스트 SQL인가? | ❌ P/B/E 패킷으로 분리됨      |
| auto_increment 값은 어떻게 반환되는가?  | 서버가 플래그 보고 같이 패킷으로 반환 |
| ORM/Hibernate는 이걸 모두 감싸는가?    | ✔️ 맞음                 |

---

# 🎯 마무리

DB 프로토콜을 이해하면
**JDBC, Hibernate, MyBatis의 동작 원리**가 100% 투명하게 보입니다.

특히 다음과 같은 상황에서 절대적인 도움이 됩니다.

✔️ 성능 튜닝
✔️ PreparedStatement 캐싱
✔️ connection pool 튜닝
✔️ 서버/클라이언트 사이의 패킷 분석
✔️ generated keys 반환 로직의 이해


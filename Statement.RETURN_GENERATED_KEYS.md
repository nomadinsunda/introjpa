이번 챕터에서는 데이터베이스가 생성한 키(Generated Keys)의 반환 메커니즘을 심층 분석합니다. JDBC 인터페이스에서 시작해 드라이버와 DB 프로토콜을 거쳐, 서버 엔진에 이르는 데이터의 전 여정을 '로우 레벨(Low-level)'에서 추적하며 ResultSet에 도달하는 메커니즘을 완벽히 파헤쳐 보겠습니다 🔍🔥

---

# 0. 용어 정리 먼저 🧩

여기서 말하는 **generated keys**는 주로 이런 것들입니다.

* `AUTO_INCREMENT` / `IDENTITY` 컬럼 (MySQL, MariaDB, SQL Server 등)
* 시퀀스 기반 PK (`nextval('seq')`, `SERIAL`, `BIGSERIAL`, `GenerationType.SEQUENCE` 등)
* 트리거에서 생성하는 PK (드라이버/ORM이 대응할 수 있는 경우)
* JDBC 입장에서는:

  > “**INSERT 후에 DB가 새로 만들어준 값들**을 다시 넘겨받는 메커니즘”

JDBC API 상으로는 이 두 메서드가 핵심입니다:

```java
PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO Student(name) VALUES (?)",
    Statement.RETURN_GENERATED_KEYS // ← 이게 포인트
);

int updated = ps.executeUpdate();
ResultSet rs = ps.getGeneratedKeys();
```

이번 챕터의 핵심 내용은: <br>

> 1️⃣ `Statement.RETURN_GENERATED_KEYS`는 SQL에 붙지도 않는데 <br>
> 2️⃣ DB 서버는 어떻게 “키 돌려줘야 하는지” 알고 <br>
> 3️⃣ 그 키를 다시 JDBC에서 `ResultSet`으로 만들어주는가? <br>

이걸 단계별로 뜯어보겠습니다. 🧵

---

# 1. JDBC 레벨에서 무슨 일이 벌어지나? 🧱

## 1-1. `prepareStatement(..., RETURN_GENERATED_KEYS)` 호출

```java
PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO Student(name) VALUES (?)",
    Statement.RETURN_GENERATED_KEYS
);
```

이 순간 일어나는 일:

1. **JDBC 드라이버 내부에서만** 다음 정보가 저장됩니다.

   * SQL 텍스트: `"INSERT INTO Student(name) VALUES (?)"`
   * 파라미터 정보 (나중에 setString 등으로 채움)
   * **“Generated Keys를 요청한다”는 플래그** ✅

2. 이 플래그는 **SQL 문자열에 추가되지 않습니다.**

   * 즉, DB로 `INSERT ... RETURN_GENERATED_KEYS` 같은 텍스트가 나가는 게 **아닙니다.**
   * 드라이버 내부의 **메타데이터/옵션**으로만 세팅됩니다.

그래서 소스 코드/로그에서 SQL 문자열만 보면 **Statement 옵션이 전혀 안 보이는 것**입니다.

---

# 2. `executeUpdate()` 호출 시: 패킷이 어떻게 만들어지나? 📦

이제 실행:

```java
ps.setString(1, "홍길동");
ps.executeUpdate();
```

실행 시점의 흐름:

1. 드라이버는 준비된 SQL (`INSERT INTO Student(name) VALUES (?)`)과
2. 바인딩된 파라미터 (`"홍길동"`)
3. 그리고 **“generated keys를 돌려받고 싶다”는 내부 플래그**

를 조합해서 **DB 프로토콜 Command 패킷**을 만듭니다.

### ✅ 중요한 포인트

* `Statement.RETURN_GENERATED_KEYS`는
  **SQL 문자열에 붙지 않고**,
  **패킷의 헤더/플래그 영역에 표현**되거나
  아니면 드라이버가 “이 후에 키를 조회해야겠다”라는 **전략 플래그**로만 사용됩니다.
* 이건 DB 종류마다 살짝 다릅니다.

---

# 3. DB 종류별 generated key 반환 전략 ⚙️

## 3-1. MySQL / MariaDB 계열 🐬

MySQL 프로토콜에서 INSERT 실행 후 서버는 **OK 패킷(OK_Packet)**을 돌려줍니다.
OK 패킷 구조 일부:

* affected_rows
* **last_insert_id** ✅
* status_flags
* warning_count
* …

여기서 핵심이 바로 `last_insert_id`입니다.

### ▶ 흐름 정리

1. 클라이언트(드라이버)가 `COM_QUERY` / `COM_STMT_EXECUTE` 패킷으로 INSERT 전송

2. 서버가 INSERT 실행

3. 성공 시 `OK_Packet` 반환

   * 이 패킷 안에:

     * `affected_rows = 1`
     * `last_insert_id = 101` (예: 새로 생성된 PK)

4. **JDBC 드라이버는 이 `last_insert_id`를 읽어서**

   * 내부에 임시 버퍼에 저장해두고
   * 나중에 `getGeneratedKeys()` 호출 시
     **가짜 ResultSet(메모리 기반)을 하나 만들어 줍니다.**

즉:

```java
ResultSet rs = ps.getGeneratedKeys();
```

할 때:

* 실제 DB에 새 쿼리를 날리는 게 아니라,
* **이미 받아둔 last_insert_id**를 기반으로
* 이런 모양의 ResultSet을 만듭니다:

| GENERATED_KEY |
| ------------- |
| 101           |

(컬럼명은 드라이버 구현마다 약간 다르지만 대개 `GENERATED_KEY` 또는 1번 컬럼)

#### 🔥 포인트 요약 (MySQL)

* `RETURN_GENERATED_KEYS` 👉 SQL에 붙지 않는다.
* 서버는 그 옵션을 몰라도, 원래 **auto_increment가 있는 INSERT는 last_insert_id를 OK 패킷에 채워서 보낸다.**
* 드라이버는 그 값을 읽고,
  `getGeneratedKeys()` 호출 시 ResultSet으로 포장해서 리턴한다.

그래서 **네트워크 패킷 입장에서 보면**:

> “generated key는 insert 응답 패킷(OK_Packet) 안에 `last_insert_id` 필드로 들어있다”

라고 보시면 정확합니다.
(“패킷에 플래그처럼 들어간다”는 사용자의 직관이 거의 맞습니다. 👌)

---

## 3-2. PostgreSQL 계열 🐘

PostgreSQL은 `AUTO_INCREMENT` 대신 시퀀스/`SERIAL`/`BIGSERIAL`을 쓰고,
키를 돌려받을 때는 보통 **`INSERT ... RETURNING`** 문을 활용합니다.

JDBC 드라이버는 `prepareStatement(sql, RETURN_GENERATED_KEYS)`가 오면:

1. 내부적으로 SQL을 `INSERT ... RETURNING pk_col`으로 **변환**하거나,
2. 사용자/ORM이 직접 `RETURNING`을 붙여 쓰기도 합니다.

그러면 DB 서버는:

* INSERT 실행 후,
* 반환해야 할 컬럼들을 **일반 SELECT 결과처럼 DataRow 패킷으로 보내줍니다.**

JDBC 드라이버는 이 DataRow를 그대로 `getGeneratedKeys()`의 ResultSet으로 반환합니다.

### 🔎 차이점

* MySQL: OK 패킷의 `last_insert_id` 필드 → 드라이버가 ResultSet으로 감쌈
* PostgreSQL: `INSERT ... RETURNING` 결과 자체가 **그냥 ResultSet** → 그대로 getGeneratedKeys 결과

---

## 3-3. Oracle, SQL Server 등 ⚙️

여기는 DB마다 전략이 다릅니다만 공통 패턴은 이렇습니다.

### 전략 A: 프로토콜/드라이버가 공식 지원

* DB가 “generated keys protocol”을 공식적으로 지원
* INSERT 후 키를 별도 필드/메시지로 전달
* 드라이버가 이걸 ResultSet으로 감싸서 `getGeneratedKeys()`로 반환

### 전략 B: 드라이버가 **추가 쿼리를 몰래 날림**

예를 들어:

* SQL Server: `SELECT SCOPE_IDENTITY()`
* Oracle: `SELECT seq.CURRVAL FROM dual`
* MySQL old driver: `SELECT LAST_INSERT_ID()`

이 경우:

1. `executeUpdate()` 이후
2. 드라이버가 **자동으로 한 번 더 SELECT 쿼리를 날려서**
3. 그 결과를 generated keys로 사용합니다.

개발자 입장에서는:

```java
ps.executeUpdate();
ResultSet rs = ps.getGeneratedKeys();
```

이렇게만 보이지만,
네트워크 레벨에서는:

```
Packet1: INSERT ...
Packet2: OK
Packet3: SELECT LAST_INSERT_ID()
Packet4: ResultSet row (id=101)
```

이렇게 두 번 왕복하고 있을 수 있습니다. ✈️✈️

---

# 4. Hibernate / JPA까지 포함한 내부 흐름 🧬

Hibernate에서 `@GeneratedValue` 전략에 따라 동작 방식이 달라집니다.

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

`IDENTITY` / `AUTO_INCREMENT` 계열에서는:

1. Hibernate가 INSERT SQL 생성
2. JDBC `prepareStatement(sql, RETURN_GENERATED_KEYS)` 사용
3. `executeUpdate()` 호출
4. `ps.getGeneratedKeys()` 호출해서 PK 값 읽음
5. 이 값을 엔티티의 `id` 필드에 세팅

반대로 `SEQUENCE` 전략이면:

1. Hibernate가 먼저 `select nextval('seq')` 같은 **시퀀스 조회 쿼리**를 날려서
2. PK 값을 미리 얻어온 후
3. 그 값을 INSERT에 직접 포함해서 실행하기도 합니다.

   * 이때는 INSERT 후에 generated keys를 받을 필요가 없으므로
     RETURN_GENERATED_KEYS를 안 쓸 수도 있습니다.

즉:

* **IDENTITY 전략** → INSERT 이후에 DB가 키를 생성 → generated keys 필요
* **SEQUENCE 전략** → INSERT 이전에 키를 먼저 얻음 → generated keys 불필요하거나 제한적 사용

---

# 5. Batch Insert + Generated Keys 🧯

배치 insert에서도 generated keys를 받을 수 있습니다.

```java
PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO Student(name) VALUES (?)",
    Statement.RETURN_GENERATED_KEYS
);

for (String name : names) {
    ps.setString(1, name);
    ps.addBatch();
}

ps.executeBatch();
ResultSet rs = ps.getGeneratedKeys();
```

여기서 DB별로 차이가 큽니다.

### MySQL

* `last_insert_id`는 **“첫 번째 행의 id”**만 제공하는 경우가 많습니다.
* 나머지 키는 드라이버가
  “auto_increment가 1씩 증가한다”는 전제를 두고
  `첫 id + (row index)` 식으로 **추측해서** 만들어낼 수도 있고,
  아예 1개만 반환하는 드라이버도 있습니다.

### PostgreSQL (`INSERT ... RETURNING`)

* 각 행마다 키를 반환하므로,
* `getGeneratedKeys()` ResultSet에는 **여러 row가 존재**하고,
  각 row가 한 insert 레코드의 PK를 의미합니다.

---

# 6. 질문 핵심에 대한 직관적인 요약 💡

> ❓ `Statement.RETURN_GENERATED_KEYS`는 SQL에도 안 붙는데,
> ❓ generated keys는 어떻게 반환되는가?
> ❓ 패킷에 플래그 형태로 들어가는가?

정리하면:

1. **JDBC 레벨**

   * `RETURN_GENERATED_KEYS`는 **드라이버 내부 옵션**입니다.
   * SQL 문자열에는 절대 붙지 않습니다.

2. **프로토콜/네트워크 레벨**

   * MySQL 같은 경우: 서버가 OK 패킷에 `last_insert_id` 필드를 채워서 보내줍니다.
   * 이 값은 **패킷의 정해진 위치에 들어가는 “필드 값”**입니다.
     (일종의 플래그/메타데이터처럼 동작)
   * PostgreSQL은 `INSERT ... RETURNING` 결과를 그대로 DataRow 패킷으로 보내줍니다.

3. **드라이버 레벨**

   * 응답 패킷에서 generated key 관련 데이터를 읽어들입니다.
   * 그 데이터를 메모리에 들고 있다가
     `getGeneratedKeys()` 호출 시 **ResultSet 형태로 포장해서 제공**합니다.
   * 일부 DB는 이 데이터를 얻기 위해
     **드라이버가 자동으로 추가 SELECT 쿼리를 몰래 날리기도** 합니다.

그래서 개발자가 보는 것은 단순히:

```java
ps.executeUpdate();
ResultSet rs = ps.getGeneratedKeys();
```

이 두 줄이지만,
그 뒤에서는:

* 패킷 구조
* OK_Packet, CommandComplete
* last_insert_id / RETURNING 결과
* 드라이버의 ResultSet 생성 로직

이 모든 것이 맞물려 돌아가고 있습니다. ⚙️⚙️


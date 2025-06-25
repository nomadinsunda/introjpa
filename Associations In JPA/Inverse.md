# 🔍 JPA 연관관계 "inverse" 개념

## 1️⃣ 용어 정리부터 시작하자

| 용어                    | 의미                                        |
| --------------------- | ----------------------------------------- |
| **연관관계 주인(owner)**    | 외래 키를 실제로 **관리**하고 **변경**하는 쪽             |
| **연관관계 비주인(inverse)** | 외래 키를 직접 관리하지 않으며, **객체 참조 유지를 위한 보조 역할**을 하는 쪽            |
| **`mappedBy`**        | "나는 비주인이며, 주인은 상대 엔티티의 이 필드입니다"라고 JPA에 알림 |

---

## 2️⃣ 진짜 문제는 **양방향 매핑**에서 생긴다

JPA는 객체지향 언어인 Java와 관계형 DB 간의 \*\*"불일치(O/R Impedance Mismatch)"\*\*를 해결하기 위한 기술입니다.

Java에서는 객체 간에 **양방향 참조**가 가능합니다.

```java
member.setTeam(team);
team.getMembers().add(member);
```

하지만 관계형 DB에서는 \*\*단방향 참조(외래 키는 한쪽에만 존재)\*\*만 가능합니다.

> ✅ 결국, JPA는 객체의 양방향 관계를 DB의 단방향 외래 키 제약으로 **해석**해야 합니다.
> 이때, “누가 외래 키를 조작할지”를 **명확히 지정해야 하는데**,
> 바로 그 결정이 `owner` vs `inverse`입니다.

---

## 3️⃣ 예제를 통한 비교 (양방향 연관관계)

### 💡 클래스 구조

```java
@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @OneToMany(mappedBy = "team")  // ⬅ inverse
    private List<Member> members = new ArrayList<>();
}
```

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @ManyToOne   // ⬅ owner
    @JoinColumn(name = "team_id")  // 외래 키!
    private Team team;
}
```

### 🔄 여기서 무슨 일이 일어나는가?

* `@JoinColumn(name = "team_id")` → 이 필드가 **DB에 외래 키를 만든다.**
* `mappedBy = "team"` → “이쪽은 단지 연관관계의 객체 참조 유지를 위한 보조 역할일 뿐뿐, DB에 영향을 주지 말라”는 의미

---

## 4️⃣ inverse 설정 없이 양방향 매핑을 하면?

```java
@Entity
public class Team {
    @OneToMany
    private List<Member> members = new ArrayList<>();
}
```

👉 그러면 어떻게 될까?

JPA는 \*\*Team-Member 사이에 제3의 조인 테이블(team\_member)\*\*을 생성한다.

### 결과

* Team에는 `team_id` 없음
* Member에도 외래 키 없음
* 대신 조인 테이블: `team_id`, `member_id`

이건 우리가 의도한 구조가 아님.

---

## 5️⃣ inverse는 “외래 키 관리를 하지 마라”는 선언이다

### `mappedBy = "team"`이 없다면?

```java
Team team = new Team();
Member member = new Member();
team.getMembers().add(member);
em.persist(team);
em.persist(member);
```

➡ Hibernate는 다음 쿼리를 실행함

```sql
insert into member (id, name, team_id) values (?, ?, null);
```

➡ 외래 키 NULL
왜? 주인(owner) 쪽인 `member.team`이 설정되지 않았기 때문
**inverse는 DB 반영 대상이 아니다!**

---

## 6️⃣ Hibernate의 내부 처리 흐름

Hibernate의 `org.hibernate.engine.spi.ActionQueue`는 Flush 시점에 `InsertAction`, `UpdateAction`, `DeleteAction`을 생성합니다.

### 주인(owner)의 경우:

* 변경 감지(dirty checking) → `UpdateAction` → 외래 키 UPDATE 수행

### inverse의 경우:

* `mappedBy` 설정 시 → **데이터베이스 외래 키 변경 대상 아님**
* 단지 Java 객체 참조만 유지

> 🔧 Hibernate는 오직 **owner의 필드 값**만을 보고 외래 키를 설정한다!

---

## 7️⃣ 연관관계 편의 메서드는 왜 필요한가?

```java
public void addMember(Member member) {
    members.add(member);        // inverse
    member.setTeam(this);       // ✅ owner 설정
}
```

이런 메서드가 없으면 양방향 관계가 **불일치**하게 되어 데이터가 잘못 저장된다.

---

## 8️⃣ 예제: inverse 없이 저장할 경우

```java
team.getMembers().add(member);
em.persist(team);
em.persist(member);
```

**이때 DB에는 어떻게 저장될까?**

```sql
insert into member (id, name, team_id) values (?, ?, null);
```

> ❌ 외래 키는 NULL
> ❌ Team-Member 연결 안 됨
> ❌ 조회 시 `team.getMembers()`는 비어 있거나 이상하게 동작

---

## 9️⃣ 예제: owner만 설정했을 경우

```java
member.setTeam(team);
em.persist(member);
```

이 경우는?

✅ Hibernate는 `member.team`을 참조하여 team\_id를 설정함
✅ 외래 키는 정상 반영됨
❌ 하지만 `team.getMembers()`는 비어 있음

---

## 🔚 정리

| 구분                         | owner (주인)           | inverse (비주인) |
| -------------------------- | -------------------- | ------------- |
| 외래 키 위치                    | ✅ 가짐 (`@JoinColumn`) | ❌ 없음          |
| DB 반영 주체                   | ✅                    | ❌             |
| `mappedBy` 사용              | ❌                    | ✅ 필수          |
| 변경 감지 대상                   | ✅                    | ❌             |
| Hibernate insert/update 대상 | ✅                    | ❌             |
| Java 객체 참조 유지              | 선택사항                 | ✅             |

---

## ✅ 결론

* `inverse`는 단순한 "반대편"이 아니라, **JPA가 외래 키를 제어하지 않도록 명시적으로 선언하는 역할자**입니다.
* `mappedBy`는 JPA에게 다음을 말해줍니다:

  > "외래 키는 내가 아니라, 상대편이 관리합니다. 나는 거울처럼 반영만 합니다."

---

## 📌 실무 팁

1. `양방향 연관관계`에서는 반드시 `mappedBy`를 지정해 **owner/inverse를 명확히 구분**하라
2. 항상 **연관관계 편의 메서드**를 만들어서 양방향 객체 동기화를 유지하라
3. 엔티티의 `equals()`와 `hashCode()`에는 연관관계 필드를 포함하지 마라 (무한 루프 방지)

좋습니다. 지금부터는 **연관관계의 주인(owner)과 비주인(inverse)** 설정에 따라 **Hibernate가 생성하는 SQL 쿼리 로그**가 어떻게 달라지는지 **실제 Java 코드와 함께 비교 분석**해드리겠습니다.

---

# 🧪 Hibernate SQL 로그로 보는 `owner vs inverse` 실습 비교

## ⚙️ 환경 설정 (Hibernate SQL 로그 활성화)

```properties
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="http://jakarta.ee/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="
                http://jakarta.ee/xml/ns/persistence
                http://jakarta.ee/xml/ns/persistence/persistence_3_0.xsd"
             version="3.0">

    <persistence-unit name="myunit" transaction-type="RESOURCE_LOCAL">
        <class>com.example.domain.Team</class>
        <class>com.example.domain.Member</class>

        <properties>
            <!-- Hibernate 설정 -->
            <property name="jakarta.persistence.jdbc.driver" value="com.mysql.cj.jdbc.Driver"/>
            <property name="jakarta.persistence.jdbc.url" value="jdbc:mysql://localhost:3306/mydb"/>
            <property name="jakarta.persistence.jdbc.user" value="root"/>
            <property name="jakarta.persistence.jdbc.password" value="root"/>

            <property name="hibernate.dialect" value="org.hibernate.dialect.MySQL8Dialect"/>

            <!-- SQL 로그 출력 설정 -->
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.use_sql_comments" value="true"/>

            <!-- SQL 파라미터 바인딩 출력 (TRACE 수준 로그 설정 필요) -->
            <property name="hibernate.type.descriptor.sql.BasicBinder.log_level" value="TRACE"/>
        </properties>
    </persistence-unit>
</persistence>
```

---

## 1️⃣ Case 1: **양방향 매핑에서 owner만 설정** (inverse는 설정 안 함)

### ✅ 엔티티

```java
@Entity
public class Team {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @OneToMany  // ❌ mappedBy 없음 → 이쪽도 owner로 간주
    private List<Member> members = new ArrayList<>();
}

@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @ManyToOne
    @JoinColumn(name = "team_id") // ✅ 외래 키 위치
    private Team team;
}
```

### ✅ 실행 코드

```java
Team team = new Team();
team.setName("Dev Team");

Member member = new Member();
member.setName("Alice");

team.getMembers().add(member);  // inverse만 설정
// member.setTeam(team); // ❌ owner 설정 안함

em.persist(team);
em.persist(member);
```

### 🧾 Hibernate SQL 로그

```sql
insert into team (id, name) values (1, 'Dev Team');

insert into member (id, name, team_id) values (2, 'Alice', null);  -- ❗ team_id = null
```

### ❌ 문제

* 외래 키가 null로 들어감
* `mappedBy` 없이 `@OneToMany`만 쓰면 **Hibernate는 양쪽을 owner로 간주**
* 하지만 `member.setTeam(team)`을 안 했기 때문에 **외래 키가 반영되지 않음**

---

## 2️⃣ Case 2: **양방향 매핑에서 inverse를 명확히 지정 (`mappedBy`)**

### ✅ 수정된 엔티티

```java
@Entity
public class Team {
    @OneToMany(mappedBy = "team")  // ✅ inverse 설정
    private List<Member> members = new ArrayList<>();
}
```

```java
@Entity
public class Member {
    @ManyToOne
    @JoinColumn(name = "team_id")  // ✅ 여전히 owner
    private Team team;
}
```

### ✅ 실행 코드 (이번에는 주인만 설정)

```java
Team team = new Team();
team.setName("Dev Team");

Member member = new Member();
member.setName("Alice");

member.setTeam(team); // ✅ owner만 설정

em.persist(team);
em.persist(member);
```

### 🧾 Hibernate SQL 로그

```sql
insert into team (id, name) values (1, 'Dev Team');

insert into member (id, name, team_id) values (2, 'Alice', 1);  -- ✅ team_id 제대로 반영됨
```

### ✅ 올바른 결과

* 연관관계의 주인(owner)만 설정해도 외래 키가 정확히 반영됨
* `team.getMembers().add(member)`를 하지 않아도 DB에는 이상 없음

---

## 3️⃣ Case 3: **owner와 inverse 모두 설정 (편의 메서드 사용)**

### ✅ 연관관계 편의 메서드

```java
public void addMember(Member member) {
    members.add(member);
    member.setTeam(this);  // ✅ owner도 함께 설정
}
```

### ✅ 실행 코드

```java
Team team = new Team();
team.setName("Dev Team");

Member member = new Member();
member.setName("Alice");

team.addMember(member);  // ✅ owner + inverse 모두 설정

em.persist(team);
em.persist(member);
```

### 🧾 Hibernate SQL 로그

```sql
insert into team (id, name) values (1, 'Dev Team');

insert into member (id, name, team_id) values (2, 'Alice', 1);  -- ✅ 외래 키 완벽하게 반영됨
```

---

## 🔍 비교 요약

| 시나리오               | 설정                                  | 외래 키 반영          | 비고                   |
| ------------------ | ----------------------------------- | ---------------- | -------------------- |
| owner 설정 안함        | `team.getMembers().add(member)`만 수행 | ❌ team\_id는 NULL | Hibernate는 owner만 참조 |
| owner만 설정          | `member.setTeam(team)`만 수행          | ✅ team\_id = 1   | 정상 반영                |
| owner + inverse 설정 | `addMember()`로 양쪽 모두 설정             | ✅ team\_id = 1   | 객체-DB 일관성까지 완벽       |

---

## 🧠 전문가 팁

* Hibernate는 **flush() 시점에 연관관계의 주인만을 기준으로** SQL을 생성함
* inverse만 설정해도 Java 객체 간에는 참조가 유지되지만, **DB에는 아무 영향 없음**
* 실수로 inverse만 설정하면 **silent bug** 발생 → DB에는 연결되지 않음

---

## ✅ 결론

`mappedBy`를 사용한 inverse 설정은 JPA에게 말해주는 선언입니다:

> “이 inverse 필드는 외래 키를 직접 관리하지 않으며, 단지 객체 간의 연관관계를 표현하는 역할만 수행합니다.”

Hibernate는 이를 철저히 지키며, 오직 **주인의 필드 값**만 참조해서 SQL을 생성합니다.



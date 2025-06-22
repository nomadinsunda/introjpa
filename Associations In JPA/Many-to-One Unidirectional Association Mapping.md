# 🧭 JPA에서 Many-to-One 단방향 연관관계 매핑

## 🔍 개요

실무에서 가장 빈번하게 사용되는 연관관계 중 하나는 **Many-to-One** 관계입니다. 특히 **단방향**(`unidirectional`) 방식은 설정이 간단하고, 조회가 한 방향으로만 필요할 때 유용합니다. 이 글에서는 JPA 표준 API만을 사용하여 Many-to-One 단방향 연관관계를 어떻게 정의하고, 어떤 방식으로 동작하는지 상세히 살펴보겠습니다.

---

## 🧱 관계 정의 예: `Member`와 `Team`

### 관계 설명

* **하나의 Team에는 여러 명의 Member가 소속**됩니다.
* 하지만 **Member만 Team을 알고**, Team은 Member에 대해 알지 못하는 **단방향 관계**입니다.

---

## 🗃️ 데이터베이스 스키마 (MySQL)

```sql
CREATE TABLE TEAM (
    ID BIGINT PRIMARY KEY AUTO_INCREMENT,
    NAME VARCHAR(100)
);

CREATE TABLE MEMBER (
    ID BIGINT PRIMARY KEY AUTO_INCREMENT,
    NAME VARCHAR(100),
    TEAM_ID BIGINT,
    CONSTRAINT FK_MEMBER_TEAM FOREIGN KEY (TEAM_ID) REFERENCES TEAM(ID)
);
```

---

## 🧩 JPA 엔티티 클래스 정의

### `Team.java`

```java
import jakarta.persistence.*;

@Entity
@Table(name = "TEAM")
public class Team {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // 기본 생성자
    public Team() {}

    public Team(String name) {
        this.name = name;
    }

    // Getter/Setter
    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

### `Member.java`

```java
import jakarta.persistence.*;

@Entity
@Table(name = "MEMBER")
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // 단방향 Many-to-One 관계
    @ManyToOne
    @JoinColumn(name = "TEAM_ID")  // 외래키 컬럼명 지정
    private Team team;

    public Member() {}

    public Member(String name, Team team) {
        this.name = name;
        this.team = team;
    }

    // Getter/Setter
    public Long getId() { return id; }
    public String getName() { return name; }
    public Team getTeam() { return team; }
    public void setName(String name) { this.name = name; }
    public void setTeam(Team team) { this.team = team; }
}
```

---

## 🧪 실습 코드 (JPA 표준 API만 사용)

```java
import jakarta.persistence.*;

public class JpaManyToOneUnidirectionalExample {

    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("examplePU");
        EntityManager em = emf.createEntityManager();

        try {
            em.getTransaction().begin();

            // 팀 생성
            Team team = new Team("Backend Team");
            em.persist(team);

            // 멤버 생성 및 팀 지정
            Member member1 = new Member("Alice", team);
            Member member2 = new Member("Bob", team);

            em.persist(member1);
            em.persist(member2);

            em.getTransaction().commit();

            System.out.println("== 저장된 멤버 조회 ==");
            Member found = em.find(Member.class, member1.getId());
            System.out.println("Member: " + found.getName());
            System.out.println("Team: " + found.getTeam().getName());

        } finally {
            em.close();
            emf.close();
        }
    }
}
```

---

## 🧠 주요 개념 해설

### 1. `@ManyToOne`과 `@JoinColumn`

* `@ManyToOne`은 JPA에서 가장 기본적인 연관관계이며, `FetchType.LAZY`가 기본입니다.
* `@JoinColumn(name = "TEAM_ID")`는 외래키 컬럼명을 지정하며, 생략 시 기본으로 `team_id`가 생성됩니다.

### 2. 단방향과 양방향

* 이 예제는 단방향: `Member → Team`만 참조합니다.
* 만약 `Team`에서도 `List<Member>`를 참조하도록 하면 양방향이 됩니다. 그러나 이 경우 `mappedBy` 속성을 반드시 지정해야 합니다.

### 3. 연관관계 주인

* 연관관계의 주인은 \*\*외래키를 가진 엔티티(`Member`)\*\*입니다.
* 즉, `Member.setTeam(...)` 호출로 관계가 관리됩니다.

---

## 🔍 Hibernate SQL 로그 (중요 분석 포인트)

```sql
insert into TEAM (NAME) values ('Backend Team');
insert into MEMBER (NAME, TEAM_ID) values ('Alice', 1);
insert into MEMBER (NAME, TEAM_ID) values ('Bob', 1);
```

* `TEAM`이 먼저 저장된 후, `TEAM_ID`를 참조하여 `MEMBER`가 저장됩니다.

---

## 🧱 퍼시스턴스 설정 (persistence.xml)

```xml
<persistence xmlns="https://jakarta.ee/xml/ns/persistence" version="3.0">
  <persistence-unit name="examplePU">
    <class>com.example.Member</class>
    <class>com.example.Team</class>
    <properties>
      <property name="jakarta.persistence.jdbc.driver" value="com.mysql.cj.jdbc.Driver" />
      <property name="jakarta.persistence.jdbc.url" value="jdbc:mysql://localhost:3306/jpa_example" />
      <property name="jakarta.persistence.jdbc.user" value="root" />
      <property name="jakarta.persistence.jdbc.password" value="root" />

      <property name="hibernate.hbm2ddl.auto" value="update" />
      <property name="hibernate.show_sql" value="true" />
      <property name="hibernate.format_sql" value="true" />
    </properties>
  </persistence-unit>
</persistence>
```

---

## 🚨 주의사항

| 항목                 | 설명                                                                     |
| ------------------ | ---------------------------------------------------------------------- |
| 💡 지연 로딩           | `@ManyToOne`은 기본으로 `LAZY` 이지만, 일부 구현체(Hibernate 5 이하)에서는 `EAGER`일 수 있음 |
| 🧹 영속성 전이(Cascade) | 필요 시 `cascade = CascadeType.PERSIST` 추가                                |
| 🔗 무결성             | 외래키 제약조건은 항상 DB 레벨에서 생성 권장 (`REFERENCES` 제약)                           |

---

## ✅ 결론

`Many-to-One` 단방향 매핑은 **JPA 연관관계의 기초이자 실무의 핵심**입니다. 이 구조를 정확히 이해해야 더 복잡한 양방향 매핑이나 `cascade`, `orphanRemoval`, `fetch join` 등도 자연스럽게 습득할 수 있습니다.

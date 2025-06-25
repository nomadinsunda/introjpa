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
    ID BIGINT PRIMARY KEY AUTO_INCREM트입니다.
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
      <property name="jakarta.persistence.jdbc.user" value="트로 `LAZY` 이지만, 일부 구현체(Hibernate 5 이하)에서는 `EAGER`일 수 있음 |
| 🧹 영속성 전이(Cascade) | 필요 시 `cascade = CascadeType.PERSIST` 추가                                |
| 🔗 무결성             | 외래키 제약조건은 항상 DB 레벨에서 생성 권장 (`REFERENCES` 제약)                           |

---

## ✅ 결론

`Many-to-One` 단방향 매핑은 **JPA 연관관계의 기초이자 실무의 핵심**입니다. 이 구조를 정확히 이해해야 더 복잡한 양방향 매핑이나 `cascade`, `orphanRemoval`, `fetch join` 등도 자연스럽게 습득할 수 있습니다.

# 🔍 JPA 연관관계 "inverse" 개념

## 1️⃣ 용어 정리부터 시작하자

| 용어                    | 의미                                        |
| --------------------- | ----------------------------------------- |
| **연관관계 주인(owner)**    | 외래 키를 실제로 **관리**하고 **변경**하는 쪽             |
| **연관관계 비주인(inverse)** | 외래 키를 갖지 않고, **거울처럼 반영만 하는 쪽**            |
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
* `mappedBy = "team"` → “이쪽은 단지 연관관계를 거울처럼 반영하는 것이며, DB에 영향을 주지 말라”는 의미

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



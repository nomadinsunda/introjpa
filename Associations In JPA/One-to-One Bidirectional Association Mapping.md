# 🔁 JPA One-to-One Bidirectional Association Mapping

## ✅ 1. 개념 정의: One-to-One 양방향 연관관계란?

**One-to-One 양방향 연관관계**는 두 엔티티가 1:1 관계를 맺고 있으며, **양쪽 엔티티 모두 서로를 참조**할 수 있는 관계를 말합니다. 즉,

* 객체 A에서 객체 B로 접근 가능
* 동시에 객체 B에서 객체 A로도 접근 가능

> 예시: 하나의 `Citizen`은 하나의 `Passport`를 갖고, 그 `Passport` 역시 그 `Citizen`에 속함

---

## 🔍 2. 데이터베이스 설계 관점

테이블 관계는 다음 두 가지 방식 중 하나로 설계할 수 있습니다:

* `Passport` 테이블에 `citizen_id` 외래키 (보통 이 방식 사용)
* `Citizen` 테이블에 `passport_id` 외래키

> 외래키가 존재하는 테이블이 JPA 연관관계의 "주인"이 됩니다.

---

## ⚙️ 3. 엔티티 설계: 연관관계의 주인 vs 비주인

* 연관관계의 **주인(owning side)**: 외래 키를 갖고 있는 엔티티 (SQL 실행에 영향을 줌)
* 연관관계의 **비주인(inverse side)**: `mappedBy`를 사용하는 쪽 (DB 연산 영향 없음)

```java
@Entity
public class Citizen {

    @Id @GeneratedValue
    private Long id;

    private String name;

    @OneToOne(mappedBy = "citizen", cascade = CascadeType.ALL)
    private Passport passport; // 비주인

    // getter/setter
}

@Entity
public class Passport {

    @Id @GeneratedValue
    private Long id;

    private String number;

    @OneToOne
    @JoinColumn(name = "citizen_id")
    private Citizen citizen; // 연관관계의 주인 (외래 키 소유)

    // getter/setter
}
```

> `mappedBy = "citizen"`: `Passport`의 `citizen` 필드가 연관관계의 주인이라는 뜻

---

## 🧪 4. 실습 예제

### 💾 엔티티 저장

```java
Citizen citizen = new Citizen();
citizen.setName("John Doe");

Passport passport = new Passport();
passport.setNumber("P123456");
passport.setCitizen(citizen);

citizen.setPassport(passport);

em.persist(citizen); // CascadeType.ALL 덕분에 passport도 함께 저장
```

### 📤 SQL 로그

```sql
insert into citizen (name) values ('John Doe');
insert into passport (number, citizen_id) values ('P123456', 1);
```

---

## 🔁 5. 연관관계 관리 팁: 편의 메서드

연관관계가 양방향이면 반드시 편의 메서드를 통해 **양쪽 모두 설정**해야 합니다.

```java
public void setPassport(Passport passport) {
    this.passport = passport;
    passport.setCitizen(this); // 반대편도 설정!
}
```

> **주의:** 한쪽만 설정하면 영속성 전이 또는 조회 시 객체 상태가 일치하지 않을 수 있음

---

## ⚠️ 6. mappedBy 실수 사례

### ❌ mappedBy 잘못 지정

```java
@OneToOne(mappedBy = "passport") // ❌ 잘못됨!
```

* `mappedBy` 값은 연관관계 주인의 **필드명**이어야 하며, 대상 테이블의 컬럼명이 아님
* 주인이 누구인지 정확히 파악하고 `mappedBy` 설정

---

## 🔒 7. 제약 조건 & 성능 이슈

* **외래 키 제약**: One-to-One 관계에서는 외래 키에 `UNIQUE` 제약이 반드시 필요
* **지연 로딩(LAZY)**: `@OneToOne`은 기본이 EAGER → 원하지 않으면 명시적으로 LAZY 설정 필요

  ```java
  @OneToOne(fetch = FetchType.LAZY)
  ```

---

## 🧠 8. 실무에서의 적용 전략

| 요구 사항                        | 설계 전략                         |
| ---------------------------- | ----------------------------- |
| 두 엔티티 모두 상대 정보를 자주 조회해야 함    | 양방향 매핑 사용 (`mappedBy` 명확히 지정) |
| 한쪽만 상대 정보 필요                 | 단방향 매핑 사용 (주인만 설정)            |
| 데이터 모델링 상 외래키 방향이 확정되어 있는 경우 | 그 테이블이 연관관계의 주인이 됨            |

---

## ✅ 9. 요약

| 용어       | 의미                                |
| -------- | --------------------------------- |
| 연관관계의 주인 | 외래 키를 가진 쪽. 실제 SQL에 영향을 미침        |
| mappedBy | 연관관계의 비주인에서 주인의 필드명을 명시하는 속성      |
| 양방향 연관관계 | 양쪽 모두 상대방 엔티티에 대한 참조를 가짐          |
| 편의 메서드   | 양방향 관계에서 양쪽 참조를 동시에 세팅해주는 사용자 메서드 |
| cascade  | 한쪽 연산이 상대에도 전파되도록 하는 설정           |

---

## 🧾 부록: 테이블 구조

```sql
create table citizen (
    id bigint not null auto_increment,
    name varchar(255),
    primary key (id)
);

create table passport (
    id bigint not null auto_increment,
    number varchar(255),
    citizen_id bigint unique,
    primary key (id),
    foreign key (citizen_id) references citizen(id)
);
```

---

## 📝 마무리

One-to-One 양방향 매핑은 설계가 간단해 보이지만, 연관관계의 주인 개념, 외래 키 위치, cascade 설정, fetch 전략 등 고려할 요소가 많습니다. 객체의 관계와 데이터베이스의 관계를 **정확히 일치시키는 설계**를 한다면 JPA의 강력한 ORM 기능을 최대한 활용할 수 있습니다.

# 🔍 JPQL

## 📌 목차

1. JPQL이란 무엇인가?
2. JPQL과 SQL의 차이
3. JPQL의 구문 구조
4. JPQL 내부 동작 원리
5. 실전 JPQL 예제
6. JPQL의 고급 기능
7. 성능 최적화 및 튜닝 전략
8. 오류 분석 및 트러블슈팅
9. QueryDSL과 비교
10. 마무리 및 실무 적용 팁

---

## 1. 🧠 JPQL이란 무엇인가?

\*\*JPQL(Java Persistence Query Language)\*\*은 JPA(Java Persistence API)에서 제공하는 객체지향 쿼리 언어입니다. **테이블이 아닌 엔티티 객체를 대상으로 쿼리**를 수행한다는 점에서 전통적인 SQL과 차별화됩니다.

> 📘 JPQL은 결국 SQL로 변환되어 실행되지만, **엔티티와 필드명**을 사용하여 JPA의 영속성 컨텍스트와 밀접하게 통합됩니다.

예:

```java
TypedQuery<Member> query =
    em.createQuery("SELECT m FROM Member m WHERE m.age > :age", Member.class);
query.setParameter("age", 20).getResultList();
```

---

## 2. ⚖ JPQL vs SQL: 결정적 차이점

| 항목      | JPQL                            | SQL                    |
| ------- | ------------------------------- | ---------------------- |
| 대상      | Entity 객체                       | 데이터베이스 테이블             |
| 필드 접근   | Java 필드명                        | 컬럼명                    |
| 실행 시점   | 컴파일 시 문법 체크 (정적), 런타임 시 SQL 생성  | 정적 쿼리                  |
| 결과 타입   | Entity or DTO                   | ResultSet (row/column) |
| 연관관계 탐색 | 객체 그래프 탐색 가능 (Join 불필요한 경우도 있음) | 반드시 Join 필요            |

> JPQL은 객체 모델 중심의 비즈니스 로직 구현에 최적화되어 있으며, 연관관계 매핑을 그대로 활용할 수 있습니다.

---

## 3. 📐 JPQL의 기본 구문 구조

```sql
SELECT e FROM Entity e WHERE e.field = :param
```

### 🔸 SELECT

```sql
SELECT e FROM Member e           -- 전체 엔티티
SELECT e.name FROM Member e     -- 단일 필드
SELECT new com.dto.MemberDTO(e.name, e.age) FROM Member e  -- DTO 매핑
```

### 🔸 WHERE, AND, OR

```sql
SELECT m FROM Member m WHERE m.age >= :age AND m.name LIKE :name
```

### 🔸 JOIN (내부조인, 외부조인)

```sql
SELECT o FROM Order o JOIN o.member m WHERE m.name = :name
SELECT o FROM Order o LEFT JOIN o.delivery d
```

### 🔸 IN, BETWEEN, IS NULL

```sql
SELECT m FROM Member m WHERE m.age BETWEEN 20 AND 30
SELECT m FROM Member m WHERE m.name IS NULL
SELECT m FROM Member m WHERE m.team.name IN :teamNames
```

---

## 4. ⚙ JPQL 내부 동작 원리

### 📌 엔티티 중심 → SQL 변환 → DB 실행 → 결과 매핑

1. `EntityManager.createQuery()`로 JPQL 작성
2. JPQL 파서가 문법 분석 → **AST(Abstract Syntax Tree)** 생성
3. JPA 구현체(Hibernate)가 JPQL → SQL로 변환
4. SQL 실행 → ResultSet 반환
5. 엔티티 매핑 및 영속성 컨텍스트에 반영

> 📌 중요한 점은 **JPQL은 JPA의 영속성 컨텍스트를 우선 조회**한다는 것입니다. 즉, DB에서 데이터를 가져오기 전에 \*\*1차 캐시(영속성 컨텍스트)\*\*에서 먼저 검색합니다.

---

## 5. 💡 실전 JPQL 예제

### ✅ 1. 기본 조회

```java
String jpql = "SELECT m FROM Member m WHERE m.username = :name";
List<Member> members = em.createQuery(jpql, Member.class)
                         .setParameter("name", "홍길동")
                         .getResultList();
```

### ✅ 2. DTO 직접 매핑

```java
String jpql = "SELECT new com.example.MemberDTO(m.username, m.age) FROM Member m";
List<MemberDTO> dtos = em.createQuery(jpql, MemberDTO.class).getResultList();
```

### ✅ 3. JOIN + 조건

```java
String jpql = "SELECT o FROM Order o JOIN o.member m WHERE m.name = :name";
```

---

## 6. 🎯 JPQL 고급 기능

### ✅ 1. 서브쿼리

```sql
SELECT m FROM Member m WHERE m.age > (SELECT AVG(m2.age) FROM Member m2)
```

> FROM 절에는 서브쿼리를 사용할 수 없습니다 (JPQL 제한사항).

### ✅ 2. CASE 식

```sql
SELECT CASE 
           WHEN m.age <= 10 THEN '학생'
           WHEN m.age >= 60 THEN '노인'
           ELSE '성인'
       END
FROM Member m
```

### ✅ 3. 페이징 처리

```java
List<Member> result = em.createQuery("SELECT m FROM Member m ORDER BY m.age DESC", Member.class)
                        .setFirstResult(0)
                        .setMaxResults(10)
                        .getResultList();
```

---

## 7. 🛠 성능 최적화 및 튜닝 전략

| 전략                 | 설명                                               |
| ------------------ | ------------------------------------------------ |
| **지연 로딩 vs 즉시 로딩** | JPQL 사용 시 FetchType.LAZY가 기본. 필요 시 FETCH JOIN 사용 |
| **FETCH JOIN 사용**  | 연관된 엔티티를 1쿼리로 조회 (`JOIN FETCH`)                  |
| **DTO 직접 매핑**      | 불필요한 엔티티 로딩 방지                                   |
| **Batch Size 설정**  | N+1 문제 완화 (`hibernate.default_batch_fetch_size`) |
| **쿼리 캐시**          | 읽기 전용 쿼리 캐싱 (2차 캐시 설정)                           |

### ✅ N+1 문제 예시

```java
List<Team> teams = em.createQuery("SELECT t FROM Team t", Team.class).getResultList();
for (Team team : teams) {
    System.out.println(team.getMembers().size()); // N번 추가 쿼리 발생
}
```

### ✅ 해결: FETCH JOIN

```java
String jpql = "SELECT t FROM Team t JOIN FETCH t.members";
```

---

## 8. 🧩 오류 분석 및 트러블슈팅

| 에러 메시지                                                                               | 원인                              |
| ------------------------------------------------------------------------------------ | ------------------------------- |
| `org.hibernate.hql.internal.ast.QuerySyntaxException: unexpected token`              | 잘못된 JPQL 문법                     |
| `IllegalArgumentException: org.hibernate.QueryException: could not resolve property` | 필드명이 엔티티에 없음                    |
| `javax.persistence.NonUniqueResultException`                                         | `getSingleResult()`에서 2개 이상 반환됨 |
| `NoResultException`                                                                  | `getSingleResult()`에서 0개일 때 발생  |

### 실전 팁

* `getSingleResult()` 대신 `getResultList()`로 받고 size 체크하는 것이 안전
* DTO 매핑 시 패키지 경로 포함 클래스명 정확히 지정해야 함

---

## 9. 🆚 JPQL vs QueryDSL

| 항목     | JPQL   | QueryDSL                |
| ------ | ------ | ----------------------- |
| 문법     | 문자열 기반 | 타입 안전한 Java 코드          |
| IDE 지원 | 낮음     | 자동완성 및 리팩토링 가능          |
| 동적 쿼리  | 복잡함    | `BooleanBuilder` 등으로 유연 |
| 유지보수   | 어렵다    | 뛰어남                     |

```java
// QueryDSL 예시
QMember m = QMember.member;
List<Member> result = queryFactory.selectFrom(m)
                                  .where(m.age.gt(20).and(m.username.eq("홍길동")))
                                  .fetch();
```

---

## 🔚 마무리: 실무 적용 팁

1. **JPQL은 영속성 계층을 추상화**하며 객체 모델 중심 개발을 가능하게 함
2. 단순 조회는 JPQL, 복잡하고 동적인 쿼리는 QueryDSL로 분리
3. DTO 매핑은 `new` 키워드 기반으로 직접 생성자를 명시
4. 연관 관계는 반드시 `JOIN FETCH`로 필요 시 최적화
5. 쿼리 튜닝, BatchSize, 1차 캐시 활용으로 성능을 극대화



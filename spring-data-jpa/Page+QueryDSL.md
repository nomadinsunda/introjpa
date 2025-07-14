## ✅ 1. 기본 개념 요약

* **Querydsl**: 동적 조건 쿼리를 자바 코드로 타입 안전하게 작성 가능.
* **Page<T>**: Spring Data JPA가 제공하는 페이징 결과 포맷.
* **통합 목적**: Querydsl로 작성한 복잡한 조건 쿼리 결과를 `Page<T>`로 리턴.

---

## ✅ 2. 전제 조건

### 2.1 Gradle/Maven 의존성

```groovy
// Gradle 예시
implementation "com.querydsl:querydsl-jpa"
annotationProcessor "com.querydsl:querydsl-apt"
```

---

### 2.2 기본 엔티티 예시

```java
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;

    private String name;
    private int age;

    // getters, setters
}
```

---

### 2.3 Q 클래스 생성

빌드 후 자동 생성된 클래스: `QUser`

```java
QUser user = QUser.user;
```

---

## ✅ 3. Querydsl과 `Page<T>` 통합 핵심 코드

### 3.1 Custom Repository 인터페이스 정의

```java
public interface UserRepositoryCustom {
    Page<User> search(String name, int age, Pageable pageable);
}
```

---

### 3.2 구현 클래스 작성

```java
public class UserRepositoryImpl implements UserRepositoryCustom {

    private final JPAQueryFactory queryFactory;

    public UserRepositoryImpl(EntityManager em) {
        this.queryFactory = new JPAQueryFactory(em);
    }

    @Override
    public Page<User> search(String name, int age, Pageable pageable) {
        QUser user = QUser.user;

        // where 조건 동적 생성
        BooleanBuilder where = new BooleanBuilder();
        if (name != null && !name.isEmpty()) {
            where.and(user.name.containsIgnoreCase(name));
        }
        if (age > 0) {
            where.and(user.age.eq(age));
        }

        // 실제 쿼리: 데이터 목록 조회
        List<User> content = queryFactory
            .selectFrom(user)
            .where(where)
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .orderBy(QuerydslUtils.convertToOrderSpecifier(pageable.getSort(), user))
            .fetch();

        // 전체 개수 조회 쿼리
        long total = queryFactory
            .select(user.count())
            .from(user)
            .where(where)
            .fetchOne();

        return new PageImpl<>(content, pageable, total);
    }
}
```

> 💡 `PageImpl<T>`는 `Page<T>`의 구현체입니다.

---

## ✅ 4. 정렬(Sort) 처리

Querydsl은 `Pageable`의 `Sort` 객체를 그대로 사용할 수 없습니다. 별도로 변환 필요:

### 4.1 유틸리티 클래스 예시

```java
public class QuerydslUtils {
    public static OrderSpecifier<?>[] convertToOrderSpecifier(Sort sort, QUser user) {
        return sort.stream()
                .map(order -> {
                    PathBuilder<User> path = new PathBuilder<>(User.class, user.getMetadata());
                    return new OrderSpecifier(
                        order.isAscending() ? Order.ASC : Order.DESC,
                        path.get(order.getProperty())
                    );
                })
                .toArray(OrderSpecifier[]::new);
    }
}
```

---

## ✅ 5. 통합 Repository 사용

### 5.1 통합 Repository 인터페이스

```java
public interface UserRepository extends JpaRepository<User, Long>, UserRepositoryCustom {}
```

---

### 5.2 서비스 or 컨트롤러 사용

```java
Pageable pageable = PageRequest.of(0, 10, Sort.by("name").descending());
Page<User> result = userRepository.search("kim", 20, pageable);
```

---

## ✅ 6. 결과 Page<User>는 다음과 같은 정보 포함

* `result.getContent()` → 실제 `List<User>` 데이터
* `result.getTotalElements()` → 전체 개수
* `result.getTotalPages()` → 전체 페이지 수
* `result.getNumber()` → 현재 페이지 번호

---

## ✅ 7. 고급 팁: DTO로 반환

```java
List<UserDTO> content = queryFactory
    .select(Projections.constructor(UserDTO.class, user.name, user.age))
    .from(user)
    .where(where)
    .offset(pageable.getOffset())
    .limit(pageable.getPageSize())
    .fetch();

return new PageImpl<>(content, pageable, total);
```

---

## ✅ 8. 전체 구조 요약 다이어그램

```
Controller
   ↓
Pageable (자동 주입)
   ↓
Service
   ↓
Repository (UserRepository + UserRepositoryCustom)
   ↓
UserRepositoryImpl
   ├─ queryFactory.selectFrom(...)
   ├─ .offset(pageable.getOffset())
   ├─ .limit(pageable.getPageSize())
   └─ .orderBy(...).fetch()
       ↓
PageImpl<T>
```

---

## ✅ 9. 정리

| 항목              | 설명                               |
| --------------- | -------------------------------- |
| 페이징 처리는         | `offset()`, `limit()`            |
| 정렬 처리는          | `orderBy(...)`, 수동 변환 필요         |
| 전체 개수 조회는 별도 쿼리 | `.select(count()).fetchOne()`    |
| 반환 타입은          | `PageImpl<T>`                    |
| DTO 변환 가능       | `Projections.constructor(...)` 등 |


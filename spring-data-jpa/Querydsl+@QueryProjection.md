## ✅ 1. 전체 아키텍처 목표

```java
Page<UserDto> result = userRepository.searchUsersByCondition(..., pageable);
```

* **Querydsl**로 복잡한 where 조건을 생성하고
* `.offset().limit().orderBy()`로 **페이징 처리**
* DTO 생성은 `@QueryProjection` 기반으로 **컴파일 타임 타입 체크 가능**
* 결과는 `PageImpl<UserDto>`로 래핑

---

## ✅ 2. 전제 조건

### 2.1 DTO에 @QueryProjection 사용

```java
@Data
public class UserDto {

    private String name;
    private int age;

    @QueryProjection
    public UserDto(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

> 💡 컴파일 시 `QUserDto` 클래스가 자동 생성됨.

---

### 2.2 Gradle 설정 (Querydsl APT)

```groovy
dependencies {
    implementation "com.querydsl:querydsl-jpa"
    annotationProcessor "com.querydsl:querydsl-apt"
}
```

```groovy
def querydslDir = "$buildDir/generated/querydsl"

sourceSets {
    main.java.srcDirs += querydslDir
}

compileJava {
    options.annotationProcessorGeneratedSourcesDirectory = file(querydslDir)
}
```

---

## ✅ 3. Repository 구조

### 3.1 Custom Repository 인터페이스

```java
public interface UserRepositoryCustom {
    Page<UserDto> search(String keyword, Pageable pageable);
}
```

---

### 3.2 Custom Repository 구현

```java
public class UserRepositoryImpl implements UserRepositoryCustom {

    private final JPAQueryFactory queryFactory;

    public UserRepositoryImpl(EntityManager em) {
        this.queryFactory = new JPAQueryFactory(em);
    }

    @Override
    public Page<UserDto> search(String keyword, Pageable pageable) {
        QUser user = QUser.user;

        // 동적 where 조건
        BooleanBuilder builder = new BooleanBuilder();
        if (keyword != null && !keyword.isEmpty()) {
            builder.and(user.name.containsIgnoreCase(keyword));
        }

        // 본문 쿼리: UserDto로 직접 매핑
        List<UserDto> content = queryFactory
                .select(new QUserDto(user.name, user.age))  // Q 클래스 활용
                .from(user)
                .where(builder)
                .offset(pageable.getOffset())
                .limit(pageable.getPageSize())
                .orderBy(QuerydslSortConverter.convert(pageable.getSort(), user))
                .fetch();

        // 전체 개수 쿼리
        long total = queryFactory
                .select(user.count())
                .from(user)
                .where(builder)
                .fetchOne();

        return new PageImpl<>(content, pageable, total);
    }
}
```

---

## ✅ 4. 정렬 처리 유틸리티 (Querydsl용)

```java
public class QuerydslSortConverter {
    public static OrderSpecifier<?>[] convert(Sort sort, QUser user) {
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

## ✅ 5. 최종 Repository 조합

```java
public interface UserRepository extends JpaRepository<User, Long>, UserRepositoryCustom {}
```

---

## ✅ 6. 컨트롤러 사용 예

```java
@GetMapping("/users")
public Page<UserDto> searchUsers(@RequestParam(required = false) String keyword,
                                 Pageable pageable) {
    return userRepository.search(keyword, pageable);
}
```

→ 요청 예시:

```
GET /users?keyword=kim&page=0&size=5&sort=name,desc
```

---

## ✅ 7. 장점 정리

| 기능          | 설명                                      |
| ----------- | --------------------------------------- |
| ✅ 타입 안정성    | `@QueryProjection`을 통해 컴파일 시점에 오류 감지    |
| ✅ 코드 자동 완성  | `QUserDto` 사용 시 IDE 자동 완성 가능            |
| ✅ 페이징 완전 지원 | `PageImpl`로 페이지 메타데이터 완벽 지원             |
| ✅ 정렬 지원     | `Sort` → `OrderSpecifier` 변환으로 동적 정렬 가능 |

---

## ✅ 8. 보완 팁

* @QueryProjection은 `constructor()`보다 타입 안정성은 높지만, **DTO에 QueryDSL 의존성이 생긴다**는 단점 있음
* 이를 피하려면 `Projections.constructor(UserDto.class, ...)`로 대체 가능 (단, 컴파일러 타입 체크 불가)

---

## ✅ 9. 전체 흐름 요약

```plaintext
Controller
   ↓
Pageable + 조건
   ↓
UserRepositoryImpl
   ├─ QUserDto 생성자 기반 select()
   ├─ .offset().limit().orderBy()
   ├─ .fetch()
   └─ PageImpl<> 생성
   ↓
Page<UserDto>
```

---

## ✅ 결론

`Querydsl + @QueryProjection + Page<T>` 조합은 **타입 안정성과 기능 완성도** 면에서 가장 강력한 패턴입니다.
실무에서 복잡한 조건 검색 + 페이징 응답 API를 만들 때 이 조합은 매우 선호됩니다.


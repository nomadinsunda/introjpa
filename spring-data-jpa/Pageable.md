## ✅ 1. Pageable의 정의

```java
public interface Pageable {
    int getPageNumber();       // 현재 페이지 번호 (0부터 시작)
    int getPageSize();         // 페이지 당 데이터 수
    long getOffset();          // SQL에서 OFFSET으로 사용될 값
    Sort getSort();            // 정렬 정보
    Pageable next();           // 다음 페이지
    Pageable previousOrFirst(); // 이전 페이지 또는 첫 페이지
    Pageable first();          // 첫 페이지
    boolean hasPrevious();     // 이전 페이지 존재 여부
}
```

* `Pageable`은 `인터페이스`이며, 주로 `PageRequest` 구현체를 사용합니다.
* `PageRequest`는 불변(immutable) 객체입니다.

---

## ✅ 2. PageRequest.of(...)로 생성

```java
Pageable pageable = PageRequest.of(0, 10); // 0번째 페이지, 10개씩
```

정렬 포함:

```java
Pageable pageable = PageRequest.of(0, 10, Sort.by("createdDate").descending());
```

여러 필드로 정렬:

```java
Pageable pageable = PageRequest.of(0, 10, Sort.by("name").ascending().and(Sort.by("createdDate").descending()));
```

---

## ✅ 3. 실제 SQL 변환 예

```java
Pageable pageable = PageRequest.of(2, 5);  // 3번째 페이지, 5개씩
```

→ 다음과 같은 SQL 구문으로 변환됨 (JPA 기준):

```sql
SELECT * FROM users ORDER BY id LIMIT 5 OFFSET 10
```

* OFFSET = pageNumber × pageSize = 2 × 5 = 10

---

## ✅ 4. 컨트롤러에서 자동 바인딩

Spring MVC는 `Pageable`을 파라미터로 받을 수 있게 지원합니다:

```java
@GetMapping("/users")
public Page<User> getUsers(Pageable pageable) {
    return userRepository.findAll(pageable);
}
```

요청 URL 예시:

```http
GET /users?page=1&size=5&sort=name,desc
```

Spring이 위 URL을 보고 `Pageable` 객체로 자동 변환합니다:

| 파라미터   | 의미                   |
| ------ | -------------------- |
| `page` | 1 → 2번째 페이지 (0부터 시작) |
| `size` | 5개씩                  |
| `sort` | `name` 기준으로 내림차순 정렬  |

---

## ✅ 5. @PageableDefault 사용

기본값을 설정할 수 있습니다:

```java
@GetMapping("/posts")
public Page<Post> list(@PageableDefault(size = 10, sort = "createdAt", direction = DESC) Pageable pageable) {
    return postRepository.findAll(pageable);
}
```

요청 시 `page`, `size`, `sort`가 없어도 위 기본값으로 적용됩니다.

---

## ✅ 6. PageRequest를 직접 생성할 때 주의

* `PageRequest.of(-1, 10)` → `IllegalArgumentException`: 페이지는 0 이상이어야 함
* `PageRequest.of(0, 0)` → `IllegalArgumentException`: size는 1 이상이어야 함

---

## ✅ 7. Pageable vs Sort

| 항목    | Pageable                             | Sort                         |
| ----- | ------------------------------------ | ---------------------------- |
| 기능    | 페이징 + 정렬                             | 정렬만 수행                       |
| 사용 위치 | `Page<T> findAll(Pageable pageable)` | `List<T> findAll(Sort sort)` |
| 목적    | 페이징 처리 (offset + limit)              | 정렬만 필요할 때                    |

---

## ✅ 8. 커스텀 Pageable을 만들고 싶다면

```java
public class MyPageable implements Pageable {
    // getPageNumber(), getPageSize(), getSort(), ... 구현
}
```

하지만 일반적으로는 `PageRequest`를 그대로 사용하는 것이 바람직합니다.

---

## ✅ 9. DTO로 변환 예시

페이징 + DTO 변환:

```java
Page<UserDTO> dtoPage = userRepository.findAll(pageable)
                                      .map(user -> new UserDTO(user));
```

---

## ✅ 10. 정리

| 기능                    | 설명                             |
| --------------------- | ------------------------------ |
| 페이징 처리                | `page`, `size`, `offset` 값을 통해 |
| 정렬 정보 포함 가능           | `Sort`와 조합 가능                  |
| `PageRequest.of(...)` | 가장 많이 쓰는 구현체                   |
| 컨트롤러 자동 바인딩           | Spring MVC가 자동으로 파싱            |
| DTO 변환 가능             | `.map()` 사용                    |



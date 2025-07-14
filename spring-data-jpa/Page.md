## ✅ 1. `Page<T>`란?

`Page<T>`는 Spring Data에서 제공하는 인터페이스로, 다음과 같은 정보를 포함합니다:

* 현재 페이지에 포함된 데이터 목록 (`List<T>` 또는 `getContent()`)
* 전체 요소 수 (`getTotalElements()`)
* 전체 페이지 수 (`getTotalPages()`)
* 현재 페이지 번호 (`getNumber()`)
* 페이지 크기 (`getSize()`)
* 첫 페이지인지 여부 (`isFirst()`)
* 마지막 페이지인지 여부 (`isLast()`)
* 다음 페이지가 있는지 여부 (`hasNext()`)
* 이전 페이지가 있는지 여부 (`hasPrevious()`)

---

## ✅ 2. 상속 구조

```java
public interface Page<T> extends Slice<T> {
    int getTotalPages();
    long getTotalElements();
    <U> Page<U> map(Function<? super T, ? extends U> converter);
}
```

* `Page<T>`는 `Slice<T>`를 상속합니다.
* `Slice<T>`는 단순히 "페이지 슬라이스" (다음 페이지가 있는지 등의 정보)만 담고 있으며, 전체 개수는 포함하지 않습니다.
* `Page<T>`는 `Slice<T>`보다 더 많은 정보를 담고 있어 실제 실무에서 더 자주 사용됩니다.

---

## ✅ 3. 주요 메서드 설명

| 메서드                              | 설명                           |
| -------------------------------- | ---------------------------- |
| `List<T> getContent()`           | 현재 페이지에 포함된 실제 데이터 목록        |
| `int getNumber()`                | 현재 페이지 번호 (0부터 시작)           |
| `int getSize()`                  | 한 페이지에 보여질 데이터 수             |
| `int getTotalPages()`            | 전체 페이지 수                     |
| `long getTotalElements()`        | 전체 데이터 수                     |
| `boolean hasNext()`              | 다음 페이지가 있는지 여부               |
| `boolean isFirst()`              | 첫 페이지인지 여부                   |
| `boolean isLast()`               | 마지막 페이지인지 여부                 |
| `Sort getSort()`                 | 정렬 조건                        |
| `<U> Page<U> map(Function<T,U>)` | 페이징 정보는 그대로 유지하면서 데이터 타입만 변환 |

---

## ✅ 4. `Pageable`과의 관계

`Page<T>`는 보통 `Pageable`을 받아 페이징 쿼리를 실행한 결과입니다.

예제:

```java
Page<User> users = userRepository.findAll(PageRequest.of(1, 10));
```

* `PageRequest.of(1, 10)`은 2번째 페이지(0부터 시작), 10개씩 요청
* 결과는 `Page<User>`로 반환되어, 10개 이하의 `User` 객체와 메타데이터가 포함됨

---

## ✅ 5. 전체 흐름 예제

```java
@GetMapping("/users")
public ResponseEntity<Page<UserDTO>> listUsers(@RequestParam int page, @RequestParam int size) {
    Pageable pageable = PageRequest.of(page, size, Sort.by("createdDate").descending());
    Page<User> usersPage = userRepository.findAll(pageable);

    Page<UserDTO> dtoPage = usersPage.map(user -> new UserDTO(user));
    return ResponseEntity.ok(dtoPage);
}
```

* `PageRequest.of(...)`로 페이징 조건 생성
* `findAll(Pageable)`로 `Page<User>` 획득
* `.map(...)`으로 `User`를 `UserDTO`로 변환

---

## ✅ 6. JSON 응답 예시

```json
{
  "content": [
    { "id": 101, "name": "Alice" },
    { "id": 102, "name": "Bob" }
  ],
  "pageable": {
    "sort": { "sorted": true, "unsorted": false, "empty": false },
    "offset": 0,
    "pageSize": 2,
    "pageNumber": 0,
    "paged": true,
    "unpaged": false
  },
  "totalPages": 5,
  "totalElements": 10,
  "last": false,
  "size": 2,
  "number": 0,
  "sort": { "sorted": true, "unsorted": false, "empty": false },
  "first": true,
  "numberOfElements": 2,
  "empty": false
}
```

---

## ✅ 7. `Page<T>` vs `Slice<T>` vs `List<T>`

| 특성        | `Page<T>`                | `Slice<T>`             | `List<T>` |
| --------- | ------------------------ | ---------------------- | --------- |
| 전체 개수 포함  | ✅ (`getTotalElements()`) | ❌                      | ❌         |
| 페이지 정보 포함 | ✅ (`getNumber()`, ...)   | ✅ (`getNumber()`, ...) | ❌         |
| 다음 페이지 여부 | ✅                        | ✅                      | ❌         |
| 가장 가벼움    | ❌                        | 중간                     | ✅         |

---

## ✅ 8. 커스터마이징 예제 (커스텀 페이지 DTO 생성)

```java
public class CustomPageResponse<T> {
    private List<T> content;
    private int pageNumber;
    private int pageSize;
    private long totalElements;
    private int totalPages;
    private boolean last;
    private boolean first;

    public CustomPageResponse(Page<T> page) {
        this.content = page.getContent();
        this.pageNumber = page.getNumber();
        this.pageSize = page.getSize();
        this.totalElements = page.getTotalElements();
        this.totalPages = page.getTotalPages();
        this.last = page.isLast();
        this.first = page.isFirst();
    }
}
```

---

## ✅ 9. 참고 팁

* `Pageable`은 `Sort`와 같이 사용 가능 → `PageRequest.of(page, size, Sort.by(...))`
* `Page<T>.map()`은 API 응답에서 DTO 변환 시 유용
* Spring Data REST를 사용하면 `Page<T>`를 자동으로 JSON에 직렬화하여 응답

---

## ✅ 10. 실무에서 자주 쓰는 패턴

* Controller에서 `Pageable`을 `@PageableDefault`와 함께 사용:

```java
@GetMapping("/posts")
public Page<PostDTO> getPosts(@PageableDefault(size = 5, sort = "createdDate", direction = DESC) Pageable pageable) {
    return postRepository.findAll(pageable).map(PostDTO::from);
}
```

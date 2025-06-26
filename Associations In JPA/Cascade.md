# 📌 JPA의 Cascade란?

> \*\*Cascade(영속성 전이)\*\*란, **특정 엔티티에 수행한 영속성 작업(persist, merge, remove 등)을 연관된 다른 엔티티로 자동 전파하는 기능**입니다.

즉, 부모 엔티티에 어떤 작업을 하면, 자식 엔티티에게도 그 작업이 **자동으로 전달**되도록 해줍니다.

---

## ✅ Cascade의 적용 대상은?

* **컬렉션(List, Set 등)을 포함하거나, 연관 엔티티를 참조하는 곳에만 설정할 수 있습니다.**
* **연관관계의 주인/비주인 여부와 무관합니다.**

  * ✔️ **중요 포인트**: `cascade`는 \*\*연관관계를 누가 관리하느냐(owner)\*\*와는 전혀 관계가 없습니다.
  * ❗ 반대로 `mappedBy`는 반드시 **연관관계 주인에서만 생략 가능**하며, 연관관계를 어떻게 맵핑할지를 명시합니다.

---

## 💥 예제 코드: `Order` → `OrderItem`

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
private List<OrderItem> orderItems;
```

* `Order`에 `persist` 하면 `OrderItem`도 함께 `persist`됨
* `Order`를 `remove`하면 `OrderItem`도 함께 `remove`됨

즉, `cascade`는 다음처럼 동작합니다:

| 작업                  | 설명                       |
| ------------------- | ------------------------ |
| `em.persist(order)` | `orderItems`도 자동 persist |
| `em.merge(order)`   | `orderItems`도 자동 merge   |
| `em.remove(order)`  | `orderItems`도 자동 remove  |
| `em.detach(order)`  | `orderItems`도 detach     |
| `em.refresh(order)` | `orderItems`도 DB에서 다시 로딩 |

---

## 🔎 Cascade 종류

| 옵션                    | 설명                              |
| --------------------- | ------------------------------- |
| `CascadeType.PERSIST` | 부모가 `persist`될 때 자식도 `persist`  |
| `CascadeType.MERGE`   | 부모가 `merge`될 때 자식도 `merge`      |
| `CascadeType.REMOVE`  | 부모가 `remove`될 때 자식도 `remove`    |
| `CascadeType.DETACH`  | 부모가 `detach`될 때 자식도 `detach`    |
| `CascadeType.REFRESH` | 부모가 `refresh`될 때 자식도 DB에서 다시 조회 |
| `CascadeType.ALL`     | 위의 모든 타입을 포함 (실무에서 가장 자주 사용됨)   |

---

## 🎯 핵심 정리: 주인과 cascade의 관계

| 질문                            | 답변                                                             |
| ----------------------------- | -------------------------------------------------------------- |
| cascade는 연관관계의 주인만 설정할 수 있나요? | ❌ 아닙니다. 주인/비주인 관계와 무관합니다.                                      |
| 그럼 누가 설정하나요?                  | 일반적으로 연관된 엔티티를 **소유하는 쪽**, 즉 **부모(Aggregate Root)** 에 설정합니다.   |
| 왜 부모에 설정하나요?                  | 도메인 모델의 제어권이 **부모에 있기 때문입니다.** 예: `Order`가 `OrderItem`을 포함 관리함 |
| mappedBy와는 다른가요?              | ✔️ `mappedBy`는 **연관관계 주인/비주인을 명시**, `cascade`는 **영속성 전이를 제어**  |

---

## 🧠 그림으로 이해하기

```plaintext
Order (비주인)
 └── OrderItem (주인)

[Order]         →     @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
[OrderItem]     →     @ManyToOne @JoinColumn(name = "order_id")
```

* `Order`가 `persist`되면, 그 안의 `OrderItem`도 자동 `persist`
* 단, `OrderItem`은 주인이므로 `@JoinColumn`으로 외래 키를 설정함

---

## 🧪 테스트 코드 예시

```java
Order order = new Order("고객A");
order.addItem(new OrderItem("커피", 2));
order.addItem(new OrderItem("케이크", 1));

em.persist(order); // orderItems도 자동 persist

em.remove(order);  // orderItems도 자동 remove (orphanRemoval = true면 삭제됨)
```

---

## 🚨 주의할 점

1. **Cascade가 없으면**

   * `em.persist(order)`만으로는 `orderItems`가 DB에 저장되지 않음
   * 각 `orderItem`도 `em.persist(item)`해야 함

2. **orphanRemoval = true**

   * 리스트에서 엔티티를 제거하면 실제 DB에서도 삭제됨
   * **부모와의 관계가 끊어진 자식은 고아가 되므로 삭제**

---

## ✅ 정리 요약

| 요소         | 역할                              |
| ---------- | ------------------------------- |
| `cascade`  | 부모의 작업을 자식에게 **전파**             |
| `mappedBy` | 연관관계의 **소유자(owner)** 를 명시       |
| 설정 위치      | 일반적으로 도메인 제어권이 있는 **부모 쪽**에 설정  |
| 주인과 무관?    | ✔️ 무관. 주인이 아니어도 `cascade` 설정 가능 |



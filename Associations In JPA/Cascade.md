# 📘 1. 부모-자식 관계란 무엇인가?

JPA 도메인에서의 부모-자식은 다음과 같은 관계입니다:

| 개념     | 설명                                                              |
| ------ | --------------------------------------------------------------- |
| 부모     | 자식 엔티티를 **소유하거나 집합으로 보유**하는 엔티티                                 |
| 자식     | 부모 없이는 존재 의미가 없고, **부모에 종속적인 생명주기**를 가짐                         |
| 도메인 예시 | `Order` ↔ `OrderItem`, `Board` ↔ `Comment`, `Team` ↔ `Player` 등 |

이러한 관계는 객체지향 설계상 다음을 의미합니다:

* 자식은 항상 부모의 생명주기를 **함께 따릅니다**
* 자식은 독립적으로 persist/remove/merge되지 않습니다

---

## 🧠 핵심 문제: 어떻게 이 관계를 JPA에 "정확히 반영"할 것인가?

> 답:
>
> 1. **Cascade 설정을 통해 부모의 생명주기를 자식에게 전파**
> 2. **orphanRemoval 설정을 통해 부모로부터 분리된 자식을 자동 삭제**

---

# ✅ 2. 부모(비주인) 쪽에 `cascade` 설정하는 경우

## ✔ 일반적인 방식 (정석)

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> orderItems = new ArrayList<>();
}
```

```java
@Entity
public class OrderItem {
    @ManyToOne
    @JoinColumn(name = "order_id") // 주인
    private Order order;
}
```

## 🔎 이 설정의 의미

* `Order`는 비주인(inverse side)이지만 **도메인 상 Aggregate Root**
* 자식(`OrderItem`)을 직접 제어하지 않고, 부모를 통해서만 조작됨
* `Order`를 `persist()`하면 **orderItems도 자동 persist**
* `Order`에서 `OrderItem`을 `remove()`하면 **DB에서도 삭제됨** (`orphanRemoval=true`)

## 🧪 실행 코드

```java
Order order = new Order("홍길동");
OrderItem item1 = new OrderItem("커피", 2);
OrderItem item2 = new OrderItem("케이크", 1);

order.addItem(item1);
order.addItem(item2);

em.persist(order); // item1, item2 자동 persist됨
```

### ✅ 결과 SQL

```sql
insert into orders (customer) values ('홍길동');
insert into order_item (product, quantity, order_id) values ('커피', 2, 1);
insert into order_item (product, quantity, order_id) values ('케이크', 1, 1);
```

---

## 🧠 왜 이게 정석인가?

* 도메인 설계상 **Order가 중심**이며, OrderItem은 부속 객체
* Order 없이 OrderItem은 존재할 수 없음
* 객체 그래프 단위로 트랜잭션을 제어 → `CascadeType.ALL`과 `orphanRemoval=true`의 조합이 핵심

---

# ✅ 3. 자식(주인) 쪽에 `cascade` 설정하는 경우

## ✔ 비정형적이지만 유효한 상황

```java
@Entity
public class OrderItem {
    @ManyToOne(cascade = CascadeType.PERSIST)
    @JoinColumn(name = "order_id")
    private Order order;
}
```

## 📌 이 설정의 의미

* `OrderItem`이 관계의 주인 (FK 보유)
* 자식이 생성되면서 부모를 설정하고, **자식 persist 시 부모도 함께 persist 하길 원함**

## 🧪 실행 코드

```java
Order order = new Order("홍길동");
OrderItem item = new OrderItem("커피", 2);
item.setOrder(order);

em.persist(item); // order도 함께 저장됨 (cascade=PERSIST 덕분에)
```

### ✅ 결과 SQL

```sql
insert into orders (customer) values ('홍길동');
insert into order_item (product, quantity, order_id) values ('커피', 2, 1);
```

---

## 🧠 언제 이런 구조를 사용하나?

| 사용 시점               | 설명                                                        |
| ------------------- | --------------------------------------------------------- |
| 도메인 흐름이 하위 객체 중심일 때 | 예: 학생(Student)이 반(Classroom)을 생성해서 속하는 흐름                 |
| 단방향 관계일 때           | `@ManyToOne` 단방향 관계에서는 cascade를 **자식(주인)** 쪽에 설정하는 수밖에 없음 |
| 임시 저장용 부모가 필요할 때    | 자식 중심 저장 로직에서 부모까지 전파할 수 있어 유용                            |

---

## ❗ 문제점 및 한계

* 도메인 주도 설계 철학과 어긋날 수 있음 (자식이 부모를 생성하거나 관리함)
* `CascadeType.REMOVE` 사용 시 큰 문제 발생 가능

```java
@ManyToOne(cascade = CascadeType.REMOVE)
private Department department;
```

* 여러 자식이 같은 부모를 참조할 경우, 하나 삭제 시 부모도 삭제됨 → **참조 무결성 위반**

---

# 📊 종합 비교

| 항목         | 부모(비주인)에 cascade 설정                  | 자식(주인)에 cascade 설정      |
| ---------- | ------------------------------------ | ----------------------- |
| 권장도        | ✅ 일반적이며 권장                           | ⚠️ 특정 상황에만 권장           |
| 도메인 흐름     | 부모 중심 (정석)                           | 자식이 흐름 시작점일 때           |
| 관계 방향      | 양방향 or 단방향                           | 주로 단방향                  |
| persist 전파 | 부모 persist 시 자식 전파                   | 자식 persist 시 부모 전파      |
| remove 전파  | 안전함 (단, orphanRemoval=true만 설정해도 충분) | 매우 위험함 (공유 자원 삭제 위험)    |
| 예시         | `Order` ↔ `OrderItem`                | `Student` → `Classroom` |

---

# ✅ 결론

| 질문                                    | 답변                                                                           |
| ------------------------------------- | ---------------------------------------------------------------------------- |
| 부모-자식 관계에서 cascade는 어디에 설정해야 하나요?     | 일반적으로 **부모(비주인)** 쪽에 설정하는 것이 도메인 모델과 객체 생명주기를 가장 잘 반영합니다.                    |
| 주인 쪽에 cascade 설정은 틀린 건가요?             | ❌ 아닙니다. **단방향 관계**나 **도메인 흐름이 자식 중심인 경우**에는 유효합니다. 단, 제한적이며 설계 의도가 뚜렷해야 합니다. |
| `CascadeType.REMOVE`를 주인 쪽에 설정해도 되나요? | ❌ 강력히 비추천. 공유 자원 삭제로 인해 참조 무결성이 깨질 수 있습니다.                                   |


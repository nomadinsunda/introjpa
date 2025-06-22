# 🧭 JPA에서 One-to-Many 단방향 연관관계 매핑

## 🔍 개요

`One-to-Many` 단방향 매핑은 JPA에서 객체의 컬렉션을 통해 **일대다 관계**를 단방향으로 표현하는 방식입니다. 예를 들어, 하나의 **카테고리(Category)** 안에 여러 개의 **상품(Product)** 이 있을 수 있지만, 상품에서는 카테고리를 참조하지 않을 때 사용하는 방식입니다.

> ✅ **주의**: 단방향 One-to-Many는 설계가 직관적이지만, **외래키가 소유자 테이블이 아닌 반대쪽(컬렉션 대상 테이블)에 생성**되므로 성능이나 관리 측면에서 고려할 사항이 있습니다.

---

## 🎯 설계 목표: Category → Product (단방향)

* **관계 설명**: 하나의 `Category`가 여러 개의 `Product`를 가짐
* **단방향**: `Product`는 `Category`에 대한 참조 없음

---

## 🗃️ 데이터베이스 스키마 (MySQL)

```sql
CREATE TABLE CATEGORY (
    ID BIGINT PRIMARY KEY AUTO_INCREMENT,
    NAME VARCHAR(100)
);

CREATE TABLE PRODUCT (
    ID BIGINT PRIMARY KEY AUTO_INCREMENT,
    NAME VARCHAR(100),
    CATEGORY_ID BIGINT,
    CONSTRAINT FK_PRODUCT_CATEGORY FOREIGN KEY (CATEGORY_ID) REFERENCES CATEGORY(ID)
);
```

---

## 🧩 JPA 엔티티 클래스 정의

### `Product.java`

```java
import jakarta.persistence.*;

@Entity
@Table(name = "PRODUCT")
public class Product {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    public Product() {}
    public Product(String name) {
        this.name = name;
    }

    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

### `Category.java`

```java
import jakarta.persistence.*;
import java.util.*;

@Entity
@Table(name = "CATEGORY")
public class Category {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // One-to-Many 단방향
    @OneToMany
    @JoinColumn(name = "CATEGORY_ID") // Product 테이블의 외래키 지정
    private List<Product> products = new ArrayList<>();

    public Category() {}
    public Category(String name) {
        this.name = name;
    }

    public void addProduct(Product product) {
        products.add(product);
    }

    public Long getId() { return id; }
    public String getName() { return name; }
    public List<Product> getProducts() { return products; }

    public void setName(String name) { this.name = name; }
}
```

> 💡 `@JoinColumn(name = "CATEGORY_ID")`은 `Product` 테이블에 `CATEGORY_ID` 외래키를 생성하도록 지시합니다. 단방향 매핑에서는 **관계를 소유하는 쪽이 아님에도 불구하고** 주 테이블에서 관리합니다.

---

## 🧪 실습 코드 (JPA API 사용)

```java
import jakarta.persistence.*;

public class JpaOneToManyUnidirectionalExample {

    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("examplePU");
        EntityManager em = emf.createEntityManager();

        try {
            em.getTransaction().begin();

            // 상품들 생성
            Product p1 = new Product("Laptop");
            Product p2 = new Product("Monitor");
            Product p3 = new Product("Keyboard");

            // 카테고리 생성 및 상품 추가
            Category electronics = new Category("Electronics");
            electronics.addProduct(p1);
            electronics.addProduct(p2);
            electronics.addProduct(p3);

            // 저장
            em.persist(electronics); // Category만 persist해도 연관된 Product도 저장됨
            em.persist(p1);
            em.persist(p2);
            em.persist(p3);

            em.getTransaction().commit();

            // 조회
            Category found = em.find(Category.class, electronics.getId());
            System.out.println("Category: " + found.getName());
            for (Product product : found.getProducts()) {
                System.out.println(" - Product: " + product.getName());
            }

        } finally {
            em.close();
            emf.close();
        }
    }
}
```

---

## ⚙️ 퍼시스턴스 설정 (persistence.xml)

```xml
<persistence xmlns="https://jakarta.ee/xml/ns/persistence" version="3.0">
  <persistence-unit name="examplePU">
    <class>com.example.Category</class>
    <class>com.example.Product</class>
    <properties>
      <property name="jakarta.persistence.jdbc.driver" value="com.mysql.cj.jdbc.Driver" />
      <property name="jakarta.persistence.jdbc.url" value="jdbc:mysql://localhost:3306/jpa_example" />
      <property name="jakarta.persistence.jdbc.user" value="root" />
      <property name="jakarta.persistence.jdbc.password" value="root" />

      <property name="hibernate.hbm2ddl.auto" value="update" />
      <property name="hibernate.show_sql" value="true" />
      <property name="hibernate.format_sql" value="true" />
    </properties>
  </persistence-unit>
</persistence>
```

---

## 🧠 핵심 개념 정리

| 항목         | 설명                                 |
| ---------- | ---------------------------------- |
| 연관관계 방향    | 단방향 (Category → Product)           |
| 외래키 위치     | `Product` 테이블 (CATEGORY\_ID)       |
| 주 테이블      | `Category` (소유자가 아님에도 외래키 관리)      |
| fetch 전략   | 기본 `LAZY`지만 구현체에 따라 다를 수 있음        |
| cascade 옵션 | 명시하지 않았으므로 product를 직접 persist해야 함 |

---

## 🚨 성능 및 설계상 주의사항

* ❗ **단방향 OneToMany는 실무에서 비추천**되는 경우도 많음

  * 이유: 외래키 관리 권한이 실제 외래키를 가진 테이블(Product)에 없기 때문
* ✅ 해결 방안:

  * 양방향 관계로 바꾸고 `mappedBy`로 주인 지정
  * 또는 직접 SQL 쿼리 기반으로 외래키 관리

---

## ✅ 결론

JPA의 `One-to-Many` 단방향 매핑은 간단하지만 **외래키 위치와 주인의 혼동**으로 인해 설계에 주의가 필요합니다. 관계의 흐름이 명확할 때만 사용하는 것이 바람직하며, **실무에서는 대부분 양방향 매핑**으로 전환하거나 `Many-to-One`만으로 모델링하는 것도 고려해볼 수 있습니다.

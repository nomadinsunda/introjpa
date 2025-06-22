# JPA EntityManager

Java Persistence API(JPA)의 핵심 컴포넌트인 `EntityManager`는 영속성 컨텍스트(Persistence Context)를 통해 엔티티의 생명주기, 트랜잭션, 쿼리, 캐시 등을 전반적으로 관리하는 역할을 담당합니다. 이는 마치 객체와 데이터베이스 사이의 브리지 역할을 수행하며, 데이터베이스 작업의 추상화를 제공합니다.

---

## 📌 주요 기능별 분류 및 메서드 상세 설명

### 1. 트랜잭션 관리 (Transaction Association)

| 메서드                                  | 설명                                                                                  |
| ------------------------------------ | ----------------------------------------------------------------------------------- |
| `EntityTransaction getTransaction()` | 리소스 레벨의 트랜잭션 객체를 반환합니다. 이 객체를 통해 트랜잭션을 명시적으로 시작하고 커밋할 수 있습니다. (Java SE 환경에서 주로 사용됨) |

### 2. 엔티티 생명주기 관리 (Entity Lifecycle Management)

| 메서드                           | 설명                                                                               |
| ----------------------------- | -------------------------------------------------------------------------------- |
| `void persist(Object entity)` | 새 엔티티를 영속화(persistent) 상태로 전환하여 식별자를 할당하고, 영속성 컨텍스트에 등록합니다.                      |
| `void remove(Object entity)`  | 현재 영속성 컨텍스트에 관리되는 엔티티를 삭제합니다. 이미 삭제되었거나 비영속 상태이면 예외는 발생하지 않음.                    |
| `void refresh(Object entity)` | 데이터베이스에서 해당 엔티티의 현재 상태를 다시 로딩하여 덮어씌웁니다. 변경 사항은 무시됩니다.                            |
| `<T> T merge(T entity)`       | 준영속(detached) 상태나 새로운 객체의 상태를 병합합니다. 새로운 영속 객체를 반환하며, 원본 객체는 여전히 detached 상태입니다. |

### 3. 엔티티 식별 및 관리 상태 확인 (Entity Identity Management)

| 메서드                                                   | 설명                                                                          |
| ----------------------------------------------------- | --------------------------------------------------------------------------- |
| `<T> T find(Class<T> cls, Object primaryKey)`         | 주어진 식별자(PK)에 해당하는 영속 엔티티를 반환합니다. 없으면 null. 즉시 로딩(eager).                    |
| `<T> T getReference(Class<T> cls, Object primaryKey)` | 프록시 객체를 반환합니다. 실제 데이터는 필요 시(lazy) 조회되며, 없는 경우 `EntityNotFoundException` 발생. |
| `boolean contains(Object entity)`                     | 해당 엔티티가 현재 `EntityManager`에 의해 관리되는 상태인지 확인합니다.                             |

### 4. 캐시 및 동기화 관리 (Persistence Context & Cache Management)

| 메서드                                          | 설명                                                          |
| -------------------------------------------- | ----------------------------------------------------------- |
| `void flush()`                               | 영속성 컨텍스트의 변경 내용을 데이터베이스에 동기화합니다. 단, 트랜잭션 커밋은 아님.            |
| `FlushModeType getFlushMode()`               | 현재 설정된 flush mode를 반환합니다. 기본값은 `AUTO`.                      |
| `void setFlushMode(FlushModeType flushMode)` | flush 시점을 수동(MANUAL) 또는 자동(AUTO)으로 설정합니다.                   |
| `void clear()`                               | 영속성 컨텍스트를 초기화하여 모든 관리 엔티티를 detach 상태로 만듭니다. 미반영 변경 사항은 사라짐. |

### 5. 쿼리 생성 (JPQL/Native SQL Query Factory)

| 메서드                                   | 설명                                                                                    |
| ------------------------------------- | ------------------------------------------------------------------------------------- |
| `Query createQuery(String jpql)`      | JPQL(Java Persistence Query Language) 쿼리를 실행할 Query 객체를 생성합니다.                        |
| `Query createNamedQuery(String name)` | 사전에 정의된 Named Query를 실행할 Query 객체를 생성합니다. NamedQuery는 @NamedQuery 어노테이션이나 XML에 정의됩니다. |

> 💡 *Spring Data JPA에서는 이들 메서드를 직접 사용하기보다는 Repository 추상화를 통해 대부분의 CRUD/쿼리를 처리함.*

### 6. 엔티티 잠금 제어 (Entity Locking)

| 메서드                                           | 설명                                                                    |
| --------------------------------------------- | --------------------------------------------------------------------- |
| `void lock(Object entity, LockModeType mode)` | 엔티티에 비관적 잠금(PESSIMISTIC\_READ/WRITE) 또는 낙관적 잠금(OPTIMISTIC) 설정을 적용합니다. |
| `LockModeType getLockMode(Object entity)`     | 해당 엔티티에 적용된 현재 잠금 모드를 반환합니다.                                          |

### 7. 구성 정보 및 속성 조회 (Properties & Configuration)

| 메서드                                   | 설명                                                     |
| ------------------------------------- | ------------------------------------------------------ |
| `Map<String, Object> getProperties()` | EntityManager에 적용된 설정 속성들을 Map 형태로 반환합니다. 이는 읽기 전용입니다. |

### 8. EntityManager 상태 확인 및 종료 (Lifecycle)

| 메서드                | 설명                                                                                |
| ------------------ | --------------------------------------------------------------------------------- |
| `boolean isOpen()` | EntityManager가 아직 닫히지 않고 유효한 상태인지 여부를 반환합니다.                                      |
| `void close()`     | 명시적으로 EntityManager를 닫습니다. 닫힌 이후 대부분의 메서드 호출은 `IllegalStateException` 예외를 발생시킵니다. |

---

## 📘 고급 팁 (Advanced Tips)

* **flush() 호출 시점**은 다음과 같은 상황에서 자동 발생합니다:

  * 트랜잭션 커밋 직전
  * JPQL/Native 쿼리 실행 직전 (FlushModeType.AUTO일 경우)
* `merge()`는 새로 생성된 객체를 persist하지 않고도 병합해서 insert할 수 있으므로 혼동 주의
* `getReference()`는 지연로딩 프록시이므로, 실체에 접근하기 전까지 DB 접근이 일어나지 않음
* `clear()` 이후에는 `find()`로 다시 조회하지 않는 한, 관리되지 않는 상태가 됨 (영속성 컨텍스트 분리)

---

## ✅ EntityManager vs EntityManagerFactory

| 항목      | EntityManager                  | EntityManagerFactory            |
| ------- | ------------------------------ | ------------------------------- |
| 목적      | 개별 작업 단위 (트랜잭션 단위)             | 애플리케이션 전체에서 공유되는 객체             |
| 생성 비용   | 상대적으로 무거움                      | 비교적 가벼움 (싱글톤 추천)                |
| 스레드 안전성 | **비**스레드 안전                    | **스레드 안전**                      |
| 사용 방법   | per-request 또는 per-transaction | Application Scope에서 하나만 생성 후 공유 |

---

## 💡 예시 코드

```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("my-unit");
EntityManager em = emf.createEntityManager();

em.getTransaction().begin();

MyEntity entity = new MyEntity();
em.persist(entity);

em.getTransaction().commit();
em.close();
```

---

# ✅ JPA의 Persistence Context 내부 구성

`Persistence Context`는 간단히 말해 **EntityManager에 의해 관리되는 1차 캐시**입니다.
하지만 내부적으로는 다음과 같은 **핵심 구성 요소**들이 존재하며, 이들이 유기적으로 동작하여 JPA의 영속성 관리 기능을 제공합니다.

---

## 📦 1. **Entity Map (영속 엔티티 저장소)**

* 구조: `Map<EntityKey, ManagedEntity>`
* `EntityKey = (EntityClass, @Id 값)`으로 구성된 유니크 키
* 각 엔티티 객체는 해당 키로 매핑되어 저장됨
* 역할:

  * 중복 저장 방지 (`em.find()` 재호출 시 동일 인스턴스 반환)
  * Dirty Checking(변경 감지)의 대상
  * `flush()` 시 SQL 생성의 대상

예:

```java
em.find(Member.class, 1L);  // 최초 쿼리 후 Map에 저장
em.find(Member.class, 1L);  // 2회 호출 시 DB 조회 없이 Map에서 리턴
```

---

## 🔄 2. **Snapshot 저장소 (변경 감지용 초기 스냅샷)**

* 구조: `Map<EntityKey, Snapshot>`
* `Snapshot`은 엔티티가 DB에서 로딩되었을 당시의 필드 상태를 보관
* 역할:

  * `flush()` 또는 트랜잭션 커밋 시점에 현재 필드 값과 비교하여 변경 사항 추적
  * 변경된 경우에만 `UPDATE SQL` 생성

예:

```java
member.setUsername("newName"); // Snapshot과 비교해 다르면 UPDATE
```

---

## 🔗 3. **Entity State 트래킹 (엔티티 상태 관리)**

JPA는 각 엔티티 인스턴스에 대해 다음 상태 중 하나를 부여합니다:

| 상태                  | 설명                                                     |
| ------------------- | ------------------------------------------------------ |
| **New (Transient)** | 아직 저장되지 않은 새 객체 (`persist()` 호출 전)                     |
| **Managed**         | Persistence Context에 등록되어 트랜잭션과 동기화됨                   |
| **Detached**        | Persistence Context에서 분리된 상태 (`clear()`, `detach()` 등) |
| **Removed**         | 삭제가 예약된 상태 (`remove()` 호출됨)                            |

* 상태 변경은 내부적으로 `PersistenceContext#manage()`, `#detach()` 등 메서드에 의해 조작됨

---

## 🔐 4. **Entity Lock Map (락 정보 저장소)**

* 구조: `Map<EntityKey, LockModeType>`
* 역할:

  * `em.lock(entity, LockModeType.XXX)` 호출 시 해당 정보 저장
  * 트랜잭션 커밋 전 `SELECT ... FOR UPDATE` 같은 락 SQL 생성
  * 비관적 락 / 낙관적 버전 관리용

---

## 🛠️ 5. **Flush Queue (쓰기 지연 저장소 / Action Queue)**

Hibernate 기반 구현에서는 **쓰기 지연 SQL**을 모아두는 큐 구조 존재:

* `InsertAction`, `UpdateAction`, `DeleteAction` 등으로 분류
* `persist()` → InsertAction 생성
* `remove()` → DeleteAction 생성
* 트랜잭션 커밋 또는 `flush()` 호출 시 순차적으로 실행됨

> 💡 즉, 실제 DB에 SQL이 발행되기 전까지는 이 Queue에 대기 중

---

## ❌ 6. **Removed Entities Set (삭제 예정 엔티티 집합)**

* `remove()`된 엔티티는 별도로 `Set`에 보관
* `flush()` 시 DELETE SQL이 실행되도록 함
* 동시에 Entity Map에서도 제거 예정으로 표시

---

## ❄️ 7. **Detached Entity 관리 (명시적으로는 없음)**

Detached 상태의 엔티티는 Persistence Context에 **존재하지 않음**
→ 하지만 `merge()` 시 다시 managed 상태로 복귀하며, 이전 snapshot과 비교하여 `UPDATE` 가능

---

## 🧠 전체 구조 요약 다이어그램

```
[ Persistence Context ]
 ├── Entity Map:           {EntityKey → Entity Instance}
 ├── Snapshot Map:         {EntityKey → Field Snapshot}
 ├── Lock Map:             {EntityKey → LockModeType}
 ├── Flush Queue:
 │    ├─ Insert Actions
 │    ├─ Update Actions
 │    └─ Delete Actions
 └── Removed Entities:     {Set of EntityKeys}
```

---

## 🔍 실제 동작 시나리오

```java
em.getTransaction().begin();
Member m = em.find(Member.class, 1L);   // 1. DB 조회 후 Entity Map에 등록 + Snapshot 저장
m.setUsername("newName");              // 2. 값 변경 → Snapshot과 비교하여 변경 감지
em.remove(m);                          // 3. Removed Set에 추가
em.getTransaction().commit();          // 4. Flush Queue 처리 → DELETE SQL 실행
```

---

## 🧪 확인 가능한 Hibernate 내부 클래스들

* `PersistenceContext` → Hibernate의 구현체: `StatefulPersistenceContext`
* `EntityEntry`, `ActionQueue`, `DefaultFlushEventListener`
* `Snapshot`은 `EntityEntry` 내부에 `loadedState`로 존재

---

## 📌 결론

| 구성 요소                | 설명                         |
| -------------------- | -------------------------- |
| Entity Map           | 엔티티 인스턴스를 캐싱하여 1차 캐시 기능 제공 |
| Snapshot             | 변경 감지를 위한 최초 상태 저장         |
| Lock Map             | 엔티티에 대한 락 정보 관리            |
| Flush Queue          | 지연된 쓰기 액션을 모아둠             |
| Removed Entities Set | 삭제 예약된 엔티티 추적              |
| 상태 관리                | 각 엔티티의 생명주기 상태 추적          |

---



# ✅ JPA 1차 캐시 동작 로그 예제

## 🎯 목표

* 동일한 엔티티를 두 번 조회할 때 **DB에 접근하지 않고 캐시에서 가져오는지 확인**
* `flush()`와 `clear()`가 1차 캐시에 어떤 영향을 미치는지 확인

---

## 🧱 예제 환경

* JPA + Hibernate + H2 (또는 MySQL)
* `application.properties`:

```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

---

## 🧪 테스트용 엔티티

```java
@Entity
public class Member {

    @Id @GeneratedValue
    private Long id;

    private String name;

    // getters, setters
}
```

---

## 📜 테스트 코드 (1차 캐시 확인)

```java
package com.example;

import jakarta.persistence.*;

public class Main {

    public static void main(String[] args) {

        EntityManagerFactory emf = Persistence.createEntityManagerFactory("myJpaUnit");
        EntityManager em = emf.createEntityManager();

        try {
            em.getTransaction().begin();

            // 초기 저장
            Member member = new Member("OriginalName");
            em.persist(member);

            Long id = member.getId();

            em.getTransaction().commit(); // INSERT 발생

            em.getTransaction().begin();

            System.out.println("== 1. DB 조회");
            Member member1 = em.find(Member.class, id);  // SQL 실행됨

            System.out.println("== 2. 동일 엔티티 재조회");
            Member member2 = em.find(Member.class, id);  // 1차 캐시에서 조회 (SQL 없음)

            System.out.println("== 3. 동일 객체인가?");
            System.out.println(member1 == member2);  // true

            System.out.println("== 4. flush 후 변경");
            member1.setName("ChangedName");
            em.flush();  // UPDATE SQL 발생

            System.out.println("== 5. clear 후 재조회");
            em.clear(); // 1차 캐시 비움

            Member member3 = em.find(Member.class, id);  // SQL 재실행됨

            System.out.println("== 6. 캐시에서 가져왔는가?");
            System.out.println(member1 == member3);  // false

            em.getTransaction().commit();

        } finally {
            em.close();
            emf.close();
        }
    }
}

```

---

## 🧾 Hibernate 로그 분석

### 🔹 Step 1: 첫 조회 → DB 접근

```sql
select m1_0.id, m1_0.name from member m1_0 where m1_0.id=1
```

### 🔹 Step 2: 동일 ID 재조회 → **SQL 없음**

* 로그 출력 없음 → 1차 캐시에서 반환

```java
System.out.println(member1 == member2); // true
```

### 🔹 Step 3: 필드 변경 후 flush

```sql
update member set name='ChangedName' where id=1
```

* dirty checking으로 인해 snapshot과 비교 후 UPDATE SQL 생성

### 🔹 Step 4: clear 후 재조회

```sql
select m1_0.id, m1_0.name from member m1_0 where m1_0.id=1
```

* clear()에 의해 persistence context 초기화
* DB 재조회 발생 (1차 캐시에서 제거됨)

---

## 📌 요약 정리

| 시점                  | 동작                 | SQL 발생 여부 | 캐시 사용 여부 |
| ------------------- | ------------------ | --------- | -------- |
| `em.find(1L)` 최초 조회 | DB에서 가져와 1차 캐시에 저장 | ✅ 있음      | ❌        |
| `em.find(1L)` 재조회   | 1차 캐시에서 반환         | ❌ 없음      | ✅        |
| `flush()`           | 변경 감지 후 UPDATE     | ✅ 있음      | ✅        |
| `clear()` 후 재조회     | 1차 캐시 초기화, DB 재조회  | ✅ 있음      | ❌        |

---

## ✅ 결론

* JPA는 같은 트랜잭션 내에서는 같은 ID의 엔티티에 대해 **DB에서 단 한 번만 조회**
* 모든 변경 감지는 1차 캐시에 있는 스냅샷 기반으로 이루어짐
* `em.clear()`는 1차 캐시 초기화이므로, 이후의 모든 조회는 다시 SQL 실행됨



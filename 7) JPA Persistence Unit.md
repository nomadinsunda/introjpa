# ⚙️ JPA Persistence Unit 완전 정복

**persistence.xml 설정부터 Hibernate 속성까지, 제대로 알고 쓰자!**

> "Persistence Unit은 단순한 설정 파일이 아니라, JPA의 심장이다."

---

## 1️⃣ Persistence Unit이란?

**Persistence Unit**은 JPA에서 **EntityManagerFactory** 생성을 위한 **논리적 설정 단위**입니다.
애플리케이션이 관리하는 모든 Entity 클래스는 하나 이상의 Persistence Unit에 소속되며, 이 설정에 따라 DB 연결, 트랜잭션 처리, 구현체 선택이 이루어집니다.

```java
EntityManagerFactory emf = 
    Persistence.createEntityManagerFactory("StudentPU");
```

위 코드의 `"StudentPU"`는 persistence.xml에서 정의된 Persistence Unit 이름입니다.

---

## 2️⃣ Persistence Unit의 위치

Persistence Unit은 `persistence.xml` 파일 내에서 정의되며, **`META-INF` 디렉토리**에 위치해야 합니다.

### ✅ 패키징에 따른 위치 규칙

| 패키징       | 위치                                          |
| --------- | ------------------------------------------- |
| EJB JAR   | `META-INF/persistence.xml`                  |
| WAR       | `WEB-INF/classes/META-INF/persistence.xml`  |
| 라이브러리 JAR | `WEB-INF/lib/*.jar` (WAR에 포함) 또는 EAR 루트/lib |

---

## 3️⃣ persistence.xml 예제와 전체 구조

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="https://java.sun.com/xml/ns/persistence"
             xmlns:xsi="https://www.w3.org/2001/XMLSchema-instance"
    리

| 항목                 | 요약 설명                                |
| ------------------ | ------------------------------------ |
| `persistence.xml`  | JPA 설정의 시작점                          |
| `Persistence Unit` | 하나의 논리적 EntityManagerFactory 설정 단위   |
| 핵심 속성              | DB 연결, 구현체 지정, 트랜잭션, DDL 전략 등        |
| 확장성                | 멀티 DB, 복수 Unit, 외부 jar 포함 가능         |
| 실무 권장 사항           | 개발/운영 환경 별도 구성, 보안 분리, DDL 도구와 병행 사용 |

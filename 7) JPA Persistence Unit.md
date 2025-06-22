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
             xsi:schemaLocation="https://java.sun.com/xml/ns/persistence
                                 https://java.sun.com/xml/ns/persistence/persistence_2_0.xsd"
             version="2.0">

    <persistence-unit name="StudentPU">
        <provider>org.hibernate.ejb.HibernatePersistence</provider>

        <class>com.example.entity.Student</class>

        <properties>
            <!-- ① DB 접속 정보 -->
            <property name="hibernate.connection.url" value="jdbc:mysql://localhost:3306/study" />
            <property name="hibernate.connection.driver_class" value="com.mysql.cj.jdbc.Driver" />
            <property name="hibernate.connection.username" value="root" />
            <property name="hibernate.connection.password" value="root" />

            <!-- ② 클래스 자동 감지 설정 -->
            <property name="hibernate.archive.autodetection" value="class" />

            <!-- ③ SQL 로그 출력 설정 -->
            <property name="hibernate.show_sql" value="true" />
            <property name="hibernate.format_sql" value="true" />

            <!-- ④ 테이블 생성/수정 전략 -->
            <property name="hibernate.hbm2ddl.auto" value="update" />
        </properties>
    </persistence-unit>

</persistence>
```

---

## 4️⃣ 속성별 상세 설명

### 🔷 DB 연결 관련

| 속성                                           | 설명                                     |
| -------------------------------------------- | -------------------------------------- |
| `hibernate.connection.url`                   | DB 접속 URL (`study`라는 DB 사용)            |
| `hibernate.connection.driver_class`          | MySQL 8 이상: `com.mysql.cj.jdbc.Driver` |
| `hibernate.connection.username` / `password` | DB 인증 정보                               |

> ⚠ 운영 환경에서는 보안을 위해 JNDI 설정 또는 외부 환경 변수로 분리 필요

---

### 🔷 클래스 매핑 감지

```xml
<property name="hibernate.archive.autodetection" value="class" />
```

| 값       | 의미                                 |
| ------- | ---------------------------------- |
| `class` | `@Entity` 등 어노테이션으로 표시된 클래스를 자동 감지 |
| `hbm`   | hbm.xml을 통한 매핑                     |
| `none`  | 자동 감지 비활성화                         |

> ✅ 일반적인 어노테이션 기반 매핑에는 `class`가 가장 적절

---

### 🔷 SQL 로그 출력

```xml
<property name="hibernate.show_sql" value="true" />
<property name="hibernate.format_sql" value="true" />
```

* SQL 문 출력 여부와 포맷팅 설정
* 개발 중 디버깅용으로 유용

> ⚠ 운영 환경에서는 `false` 권장

---

### 🔷 테이블 생성 전략 (DDL 전략)

```xml
<property name="hibernate.hbm2ddl.auto" value="update" />
```

| 값             | 동작                      |
| ------------- | ----------------------- |
| `create`      | 실행 시 기존 테이블 삭제 후 재생성    |
| `update`      | 테이블 없으면 생성, 변경된 필드 반영   |
| `create-drop` | 시작 시 생성, 종료 시 삭제        |
| `validate`    | 스키마 일치 여부만 검증           |
| `none`        | 아무 작업도 하지 않음 (운영 환경 기본) |

> ✅ 개발 시에는 `update`, 운영 시에는 `none` 또는 Flyway/Liquibase 등 도구 활용 권장

---

## 5️⃣ 실무 확장 팁

* **복수 Persistence Unit** 정의로 멀티 DB 구성 가능
* Entity 클래스 명시 (`<class>` 태그)로 정확한 스캔 제어
* `orm.xml`, `mapping-file`, `jar-file` 등을 통해 비어노테이션 기반 매핑도 가능
* Spring Boot 사용 시에는 `persistence.xml` 대신 `application.yml`로 설정

---

## 📦 실전 패키징 예시

### ✅ JAR 라이브러리 형태로 포함될 때

| 구조              | 위치                               |
| --------------- | -------------------------------- |
| `lib/mylib.jar` | `META-INF/persistence.xml` 포함 필수 |
| WAR 구조          | `WEB-INF/lib/` 내부에 포함            |

---

## 🧠 마무리 요약

| 항목                 | 요약 설명                                |
| ------------------ | ------------------------------------ |
| `persistence.xml`  | JPA 설정의 시작점                          |
| `Persistence Unit` | 하나의 논리적 EntityManagerFactory 설정 단위   |
| 핵심 속성              | DB 연결, 구현체 지정, 트랜잭션, DDL 전략 등        |
| 확장성                | 멀티 DB, 복수 Unit, 외부 jar 포함 가능         |
| 실무 권장 사항           | 개발/운영 환경 별도 구성, 보안 분리, DDL 도구와 병행 사용 |

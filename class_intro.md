```java
package com.example.noticeboard.account.profile.domain;

import com.example.noticeboard.account.profile.dto.ProfileRequest;
import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

@Entity
@Getter
@Setter
public class Profile {
	
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private long profileKey;

	private String phone;
	
	@Column(nullable = false)
	private int options;
	
	static public Profile createInitProfileSetting() {
		Profile profile = new Profile();
		profile.setOptions(1);
		return profile;
	}

	public void updateProfile(ProfileRequest profileRequest) {
		this.phone = profileRequest.getPhone();
		this.options = profileRequest.getOptions();
	}
	
}
```


### 📌 클래스 용도 요약

> `Profile` 클래스는 사용자 프로필에 해당하는 **전화번호(`phone`)**, **설정 옵션(`options`)** 등의 정보를 데이터베이스에 저장하고, 이를 생성 및 갱신하는 로직을 포함하는 **도메인 객체(Entity)** 입니다.

---

### 📦 구성 요소 설명

#### ✅ 어노테이션

* `@Entity`
  → 이 클래스가 JPA 엔티티임을 나타냅니다. 즉, DB의 `profile` 테이블과 매핑됩니다.

* `@Id` + `@GeneratedValue(strategy = GenerationType.IDENTITY)`
  → `profileKey`는 이 엔티티의 기본 키(primary key)이며, DB에서 자동으로 값을 증가시킵니다 (AUTO\_INCREMENT).

* `@Column(nullable = false)`
  → `options` 필드는 DB에서 `NOT NULL` 제약조건이 붙습니다.

* `@Getter`, `@Setter` (lombok)
  → 모든 필드에 대한 getter, setter를 자동 생성합니다.

---

### 🧱 필드 역할

| 필드명          | 설명                                             |
| ------------ | ---------------------------------------------- |
| `profileKey` | 각 프로필의 고유 식별자 (기본키)                            |
| `phone`      | 사용자의 전화번호                                      |
| `options`    | 사용자의 기본 설정 값 (예: 알림 설정, 공개 여부 등 비즈니스 로직에서 정의됨) |

---

### 🛠 메서드 설명

* `static Profile createInitProfileSetting()`
  → 기본 `options` 값을 1로 설정한 초기 `Profile` 객체를 생성합니다.
  → 예: 회원가입 시 기본 프로필을 생성할 때 사용.

* `void updateProfile(ProfileRequest profileRequest)`
  → 외부에서 전달된 DTO(`ProfileRequest`)를 기반으로 프로필 정보를 갱신합니다.
  → 예: 사용자가 전화번호나 설정을 수정할 때 사용.

---

### 🧩 관련 클래스 (유추)

* `ProfileRequest`
  → 일반적으로 클라이언트에서 전달된 JSON 데이터를 담는 DTO.
  → 필드 예시: `String phone`, `int options`



---

### ✅ 정리

`Profile` 클래스는 **사용자 정보 중 확장 가능한 프로필 정보(전화번호, 사용자 설정 등)를 따로 분리**하여 저장하기 위한 도메인 모델이며, 사용자별 설정을 관리하기 위한 목적으로 사용됩니다. 이와 함께 `User` 엔티티와 `1:1` 연관 관계로 매핑되어 있을 가능성이 높습니다.

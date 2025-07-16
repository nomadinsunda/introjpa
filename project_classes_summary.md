```java
package com.example.noticeboard.oauth.service;

import com.example.noticeboard.account.user.constant.UserType;
import com.example.noticeboard.account.user.domain.User;
import com.example.noticeboard.account.user.repository.LoginRepository;
import com.example.noticeboard.oauth.dto.UserSession;
import com.example.noticeboard.oauth.support.OAuthAttributes;
import jakarta.servlet.http.HttpSession;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService;
import org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest;
import org.springframework.security.oauth2.client.userinfo.OAuth2UserService;
import org.springframework.security.oauth2.core.OAuth2AuthenticationException;
import org.springframework.security.oauth2.core.OAuth2Error;
import org.springframework.security.oauth2.core.user.DefaultOAuth2User;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDate;
import java.util.Collections;

@Service
@RequiredArgsConstructor
public class CustomOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {

    private final LoginRepository loginRepository;
    private final HttpSession httpSession;

    @Override
    @Transactional
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuthAttributes attributes = createOauthAttributes(userRequest);

        User user = saveOrUpdateUser(attributes);
        isValidAccount(user);

        httpSession.setAttribute("user", UserSession.of(user));

        return new DefaultOAuth2User(Collections.singleton(new SimpleGrantedAuthority("ROLE_USER")),
                attributes.getAttributes(),
                attributes.getNameAttributeKey());
    }

    public void isValidAccount(User user) {
        if(user.isSuspension() && user.getSuspensionDate().compareTo(LocalDate.now()) > 0) {
            throw new OAuth2AuthenticationException(new OAuth2Error("null"), "해당 계정은 " + user.getSuspensionDate() + " 일 까지 정지입니다. \n사유 : " + user.getSuspensionReason());
        }

        if(user.getUserType() == UserType.GENERAL_USER) {
            throw new OAuth2AuthenticationException(new OAuth2Error("null"), "해당 이메일로 일반 계정이 가입되어있습니다.");
        }

        if(user.isDelete()) {
            throw new OAuth2AuthenticationException(new OAuth2Error("null"), "탈퇴한 사용자입니다.");
        }

        user.updateLoginDate();
    }

    public OAuthAttributes createOauthAttributes(OAuth2UserRequest userRequest) {
        OAuth2UserService delegate = new DefaultOAuth2UserService();
        OAuth2User oAuth2User = delegate.loadUser(userRequest);

        String registrationId = userRequest.getClientRegistration().getRegistrationId();
        String userNameAttributeName = userRequest.getClientRegistration().getProviderDetails()
                .getUserInfoEndpoint().getUserNameAttributeName();

        return OAuthAttributes.of(registrationId, userNameAttributeName, oAuth2User.getAttributes());
    }

    public User saveOrUpdateUser(OAuthAttributes attributes) {
        if(attributes.getEmail() == null) {
            throw new OAuth2AuthenticationException(new OAuth2Error("null"), "계정의 이메일을 찾을 수 없거나, 이메일 수집 여부에 동의하지 않았습니다.");
        }

        return loginRepository.findByEmail(attributes.getEmail())
                .orElseGet(() -> createAndSaveUser(attributes.getEmail()));
    }

    public User createAndSaveUser(String email) {
        User createUser = User.createOAuthUser(getIdWithoutEmail(email), email);

        return loginRepository.save(createUser);
    }

    public String getIdWithoutEmail(String email) {
        return email.split("@")[0];
    }
}
```

`CustomOAuth2UserService` 클래스는 **Spring Security OAuth2 인증 과정에서 사용자 정보를 처리하는 핵심 서비스 클래스**입니다. 이 클래스는 **OAuth2 로그인 요청을 처리하고**, 해당 사용자가 이미 존재하는지 확인하며, 존재하지 않으면 새로운 사용자를 생성하고, 이후 **세션에 사용자 정보를 저장**하는 역할을 수행합니다.

---

## ✅ 클래스 용도 요약

> `CustomOAuth2UserService`는 OAuth2 인증을 통해 로그인한 사용자의 정보를 받아서,
>
> * 사용자 존재 여부를 확인하고
> * 필요한 경우 DB에 새로 저장하며
> * 로그인 제한 조건(정지, 탈퇴 등)을 검사하고
> * Spring Security에서 사용할 수 있는 형태로 사용자 정보를 반환하는
>   **사용자 정의 OAuth2 서비스 클래스**입니다.

---

## 📦 구성요소 설명

### ✅ 주요 필드

| 필드명               | 역할                                      |
| ----------------- | --------------------------------------- |
| `LoginRepository` | 사용자 정보를 DB에서 조회하거나 저장                   |
| `HttpSession`     | 로그인한 사용자 정보를 세션에 저장 (ex. `UserSession`) |

---

### ✅ 주요 메서드

#### 1. `loadUser(OAuth2UserRequest userRequest)`

* OAuth2 로그인 요청 처리 진입점.
* `OAuthAttributes` 객체를 생성해서 OAuth2 사용자 정보 처리
* 사용자 계정 검사 (`isValidAccount`)
* 사용자 정보를 세션에 저장 (`UserSession`)
* Spring Security가 인증 객체로 사용할 수 있는 `DefaultOAuth2User` 반환

#### 2. `isValidAccount(User user)`

* 로그인할 수 없는 상태인지 검증:

  * **정지된 계정**인지 확인 (정지 기간 비교)
  * 일반 계정(GENERAL\_USER)으로 이미 가입된 경우
  * 탈퇴한 사용자 여부
* 모든 조건 통과 시 로그인 날짜 갱신

#### 3. `createOauthAttributes(...)`

* `DefaultOAuth2UserService`를 이용해 OAuth 제공자에서 사용자 정보를 가져옴
* `OAuthAttributes` 객체로 변환 (registrationId, nameAttributeKey 포함)

#### 4. `saveOrUpdateUser(...)`

* 이메일을 통해 기존 사용자 조회
* 없다면 새로 생성하여 저장 (`createAndSaveUser`)
* 이메일 누락 시 예외 발생

#### 5. `createAndSaveUser(...)`

* 이메일 기반으로 ID를 생성하고 (`getIdWithoutEmail`)
* `User.createOAuthUser(...)`를 호출하여 새 User 객체 생성 및 저장

#### 6. `getIdWithoutEmail(email)`

* 이메일에서 아이디 추출 (ex: `abc@example.com` → `abc`)

---

## 🧩 관련 클래스들

| 클래스               | 역할                                       |
| ----------------- | ---------------------------------------- |
| `OAuthAttributes` | OAuth 공급자에서 가져온 속성들을 표준화된 형태로 가공         |
| `UserSession`     | 세션에 저장할 사용자 정보 DTO                       |
| `User`            | 사용자 도메인 객체 (정지 여부, 탈퇴 여부, 로그인 일자 등 포함)   |
| `LoginRepository` | 사용자 조회/저장을 위한 Spring Data JPA Repository |

---

## 🔒 보안 및 인증 흐름에서의 역할

1. 사용자가 Google, Naver 등으로 로그인 시도
2. `CustomOAuth2UserService`가 호출되어 사용자 정보 로드
3. 사용자 DB 조회 및 검증
4. 문제가 없으면 세션에 정보 저장 (`httpSession.setAttribute(...)`)
5. `DefaultOAuth2User`를 반환 → SecurityContext에 저장 → 인증 완료

---

## ✅ 정리

`CustomOAuth2UserService`는 **OAuth2 로그인 사용자 처리의 핵심 서비스**로서,

* 사용자 식별 및 저장
* 계정 상태 검증 (정지, 탈퇴, 일반 계정 중복 등)
* 세션 저장
* Spring Security와 호환 가능한 인증 정보 반환

의 역할을 수행합니다.
이 서비스는 **OAuth2 로그인 사용자 처리의 커스터마이징 지점**이라고 할 수 있습니다.

```java
`User` 클래스는 **웹 애플리케이션의 사용자 계정 정보**를 담는 **핵심 도메인 모델(Entity)** 로서,
**JPA 엔티티**, **Spring Security의 `UserDetails` 구현체**, 그리고 **비즈니스 로직 보유 객체**로 사용됩니다.

이 클래스는 일반 회원 또는 OAuth2 회원의 로그인 처리, 게시글 작성자 연관, 정지/탈퇴 처리, 보안 인증 정보 제공 등을 담당합니다.

---

## ✅ 클래스 용도 요약

> `User` 클래스는 회원가입, 로그인, 정지/탈퇴 처리 등 **사용자 계정 관리 기능을 통합적으로 처리하는 엔티티**로서,
> DB 저장, 인증 정보 제공, 게시글/모임 연관 관리까지 포괄하는 **전체 사용자 도메인의 중심 클래스**입니다.

---

## 📦 주요 구성요소 설명

### ✅ 필드 요약

| 필드명                | 설명                                   |
| ------------------ | ------------------------------------ |
| `userKey`          | PK (자동 생성, IDENTITY 전략)              |
| `id`               | 사용자 ID (30자 제한, 유니크)                 |
| `email`            | 이메일 (유니크)                            |
| `password`         | 인코딩된 비밀번호                            |
| `profileImage`     | 프로필 이미지 URL                          |
| `suspensionDate`   | 계정 정지 종료일                            |
| `isSuspension`     | 정지 여부 (`@Convert(BooleanConverter)`) |
| `isDelete`         | 탈퇴 여부 (`@Convert(BooleanConverter)`) |
| `suspensionReason` | 정지 사유                                |
| `lastLoginDate`    | 마지막 로그인 날짜                           |
| `role`             | 사용자 권한 (예: USER, ADMIN 등)            |
| `userType`         | 가입 유형 (일반, OAuth 등)                  |
| `joinDate`         | 가입일 (`@CreationTimestamp`)           |

### ✅ 연관관계

| 관계                                      | 대상                   | 설명                      |
| --------------------------------------- | -------------------- | ----------------------- |
| `@OneToOne`                             | `Profile`            | 사용자 프로필 설정 (전화번호, 옵션 등) |
| `@OneToMany(mappedBy = "writer")`       | `Post`               | 사용자가 작성한 게시글            |
| `@OneToMany(mappedBy = "meetingOwner")` | `Meeting`            | 사용자가 개설한 모임             |
| `@OneToMany(mappedBy = "userList")`     | `MeetingParticipant` | 사용자가 참가한 모임             |

---

## 🔐 `UserDetails` 인터페이스 구현

Spring Security에서 인증 객체로 사용되기 위해 `UserDetails` 인터페이스를 구현하고 있습니다.

| 메서드                      | 설명                                    |
| ------------------------ | ------------------------------------- |
| `getUsername()`          | Spring Security에서 사용자 식별자로 사용 (`id`)  |
| `getPassword()`          | 인코딩된 비밀번호 반환                          |
| `getAuthorities()`       | 사용자의 권한 (`ROLE_USER`, `ROLE_ADMIN` 등) |
| `isAccountNonLocked()` 외 | 계정 상태 플래그 (기본은 모두 `true`)             |

---

## 🛠 주요 메서드

### 🔸 생성 메서드

```java
public static User createGeneralUser(LoginRequest req, String url, String encodedPw)
public static User createOAuthUser(String userId, String email)
```

→ 일반 로그인/소셜 로그인에 따라 각각 다른 방식으로 초기화

### 🔸 계정 상태 관련

* `addSuspensionDate(int, String)`
  → 정지일 연장 및 사유 설정
* `minusSuspensionDate(int)`
  → 정지일 단축 (정지 해제)
* `deleteUser()`
  → 탈퇴 처리
* `updateLoginDate()`
  → 로그인 시 호출됨

### 🔸 정보 수정

* `updatePassword(String)`
* `updateId(String)`
* `updateProfileImage(String)`

### 🔸 비밀번호 검사

```java
public boolean checkPassword(String plainPassword, PasswordEncoder passwordEncoder)
```

→ 평문 비밀번호와 암호화된 비밀번호를 비교

### 🔸 게시글 등록

```java
public void addPost(Post post)
```

→ 연관관계 편의 메서드: 사용자-게시글 양방향 연결

---

## 🧩 기타 설정

* `@JsonIdentityInfo`
  → Jackson 순환 참조 방지 (특히 게시글, 모임 등 양방향 관계를 직렬화할 때)

* `@Convert(BooleanConverter.class)`
  → `isDelete`, `isSuspension` 필드를 DB에서 숫자/문자(예: 0/1 또는 'Y'/'N')로 저장 가능하게 함

---

## ✅ 정리

`User` 클래스는:

1. **DB 저장용 사용자 계정 엔티티**
2. **Spring Security 인증 정보 제공 객체**
3. **계정 관리 비즈니스 로직의 중심**
4. **게시글 및 모임 등 여러 도메인과의 연관을 갖는 루트 엔티티**

로 사용되며, 애플리케이션 전반에서 **계정의 생성, 로그인, 정지/탈퇴, 인증 처리** 등에 핵심적인 역할을 담당합니다.













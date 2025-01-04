---
title: "[Spring/Spring Security] Spring Security를 이용해 로그인 구현하기. - (3) OAuth2 (Kakao)"
date: 2025-01-04 08:22:23 +09:00
categories: [Spring, Spring Security]
tags:
  [
    Spring,
    Spring Boot,
    Spring Security,
    로그인,
    backend,
    로그인 구현,
    OAuth2,
    OAuth,
    kakao,
    카카오 로그인
  ]
---

# 서론

Spring Security에서 [Session](https://longnh214.github.io/posts/Spring-Security-%EB%A1%9C%EA%B7%B8%EC%9D%B8/)과 [JWT](https://longnh214.github.io/posts/Spring-Security-%EB%A1%9C%EA%B7%B8%EC%9D%B8-JWT/) 방식을 통한 로그인을 구현했다. 이에 더 나아가서, Session 방식에 OAuth2 방식으로 카카오 플랫폼을 통한 로그인을 구현해 보고자 한다.

# OAuth란?

OAuth는 Open Authorization의 줄임말로, 권한 인증을 외부 서비스 업체에 위임함으로써 안전하게 액세스 권한을 부여할 수 있는 방법이다. 외부 서비스 업체에서 제공하는 권한 제공 흐름을 통해서 액세스 토큰 기반으로 해당 리소스에 접근할 수 있다.

# OAuth 서비스 흐름

![OAuth](https://github.com/user-attachments/assets/4ef7ce8b-2734-4802-897c-770d7e8d10a4)

1. A 서비스는 B 서비스의 OAuth 인증 방식을 이용하기 위해 B 서비스에 애플리케이션 등록을 하고 사용자가 A 서비스에서 B 서비스 로그인 화면으로 이동한다.
2. 사용자가 B 서비스에 로그인하면 B 서비스는 OAuth 액세스 토큰을 발급하고 이를 리다이렉트를 통해 A 서비스 서버에 전달한다.
3. A 서비스는 해당 액세스 토큰을 저장하거나 클라이언트에 전달하며, 이 토큰을 이용해 B 서비스의 리소스를 접근하거나 사용자의 인증 상태를 확인한다.

# Spring Security 내에서 OAuth (kakao) 구현

먼저 OAuth 서비스를 이용할 카카오에 스프링 프로젝트(애플리케이션)를 등록한다.

## 카카오 OAuth 어플리케이션 등록

[Kakao Developers](https://developers.kakao.com/) 페이지에서 어플리케이션 등록을 해주어야한다.

<img width="467" alt="search_kakao_developers" src="https://github.com/user-attachments/assets/30a955f8-3ebb-4d87-8b7a-7d48980de762" />

<br>
로그인 후 '애플리케이션 추가하기' 버튼을 통해 애플리케이션을 등록해준다.

<img width="673" alt="application_list" src="https://github.com/user-attachments/assets/957bdd94-3b04-4fe8-9be1-5642332b751c" />

<br>
앱 이름은 스프링 프로젝트 이름, 회사명은 개인 별칭, 카테고리는 교육으로 해서 애플리케이션을 추가한다.

<img width="668" alt="add_application" src="https://github.com/user-attachments/assets/9640e3f0-8cfc-4c0a-9a6e-efc5cd3affa5" />

<br>
애플리케이션 페이지에서 '앱 키' 탭에 들어가면 각 플랫폼에서 이용될 키가 주어지는데 이 중에서 'REST API'의 키를 사용한다.

<img width="518" alt="app_key" src="https://github.com/user-attachments/assets/ab60a73e-ae54-4ef9-ab22-0e2f50b6ff87" />

<br>
'플랫폼' 탭에서 플랫폼에 대한 도메인 URI를 등록해준다.

<img width="482" alt="enroll_platform" src="https://github.com/user-attachments/assets/6b86629f-feb0-4970-8280-1300abc75b25" />

<br>

'카카오 로그인' 탭에서 활성화를 하고, 하단에 Redirect URI를 설정해준다. 이 때 기입할 주소는 카카오 로그인 성공 이후 access token을 전달받을 REST API의 주소를 기입하면 된다. **주소는 고정적으로 기입된 주소를 사용해야 한다.**

<img width="364" alt="activate" src="https://github.com/user-attachments/assets/62d7b735-782b-4ed1-8fbb-33fe58afa28f" />

<img width="614" alt="redirect_uri" src="https://github.com/user-attachments/assets/90d15dab-578e-45cb-ad32-f5c578e0f797" />

<br>

카카오 계정의 정보를 이용하기 위해 개인 개발자 비즈 앱을 등록한다.

<img width="579" alt="biz_app_change" src="https://github.com/user-attachments/assets/8a84a53c-f301-480a-adf0-a295da40ce9e" />
<img width="354" alt="biz_app_purpose" src="https://github.com/user-attachments/assets/290481a2-07c9-4123-8dd0-46130daa6aa6" />

<br>

카카오 계정 정보 중 닉네임과 프로필 사진만 이용하기 위해 필수 동의 후 사용함을 설정했다.

<img width="897" alt="personal_info" src="https://github.com/user-attachments/assets/92194f1a-4b7c-421f-b4fb-66b3dbc6a42f" />

아래와 같이 카카오 로그인 페이지로 이동할 경로를 화면에 등록했다.

```html
<a href="/oauth2/authorization/kakao">Kakao Login</a>
```

<br>

서버에서 사용할 클라이언트 코드를 발급 후 `application.yml` 에 기입한다.

<img width="978" alt="client_secret" src="https://github.com/user-attachments/assets/0970a922-2f51-45a4-8633-7befc8e184f2" />

### application.yml

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          kakao:
            client-name: kakao
            authorization-grant-type: authorization_code
            redirect-uri: http://localhost:8080/login/oauth2/code/kakao
            client-id: {발급받은 REST API KEY}
            client-secret: {발급받은 client secret KEY}
            client-authentication-method: client_secret_post
            scope:
              - profile_nickname
              - profile_image
        provider:
          kakao:
            authorization-uri: https://kauth.kakao.com/oauth/authorize
            user-name-attribute: id
            token-uri: https://kauth.kakao.com/oauth/token
            user-info-uri: https://kapi.kakao.com/v2/user/me
```

## build.gradle

```gradle
dependencies{
  //...
  implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
  //...
}
```

## OAuth2UserInfo.java

OAuth 계정 정보를 담을 공통 인터페이스를 선언한다.

```java
public interface OAuth2UserInfo {
    String getProvider();
    String getProviderId();
    String getUserName();
}
```

## KakaoUserInfo

`OAuth2UserInfo` 의 카카오 기반 구현체를 선언한 뒤에 Redirect 후 전달받을 객체의 정보를 담을 `attribute`를 선언 후 메소드를 구현한다.

```java
public class KakaoUserInfo implements OAuth2UserInfo {

    private Map<String, Object> attributes;
    public KakaoUserInfo(Map<String, Object> attributes) {
        this.attributes = attributes;
    }

    @Override
    public String getProvider() {
        return "kakao";
    }

    @Override
    public String getProviderId() {
        return String.valueOf(attributes.get("id"));
    }

    @Override
    public String getUserName() {
        return (String) attributes.get("name");
    }
}
```

## UserDetail.java

Session 기반 UserDetail 객체에 `OAuth2User`를 다중 상속 받아 attribute 기반 UserDetail 생성자를 선언해준다.

```java
@Data
public class UserDetail implements UserDetails, OAuth2User {
    private User user;
    private Map<String, Object> attributes;

    public UserDetail(User user){
        this.user = user;
    }

    public UserDetail(User user, Map<String, Object> attributes) {
        this.user = user;
        this.attributes = attributes;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        Collection<GrantedAuthority> collect = new ArrayList<>();

        collect.add(new SimpleGrantedAuthority(user.getRole()));

        return collect;
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }

    @Override
    public String getName() {
        return null;
    }

    @Override
    public Map<String, Object> getAttributes() {
        return this.attributes;
    }
}
```

## OAuthUserDetailService.java

`DefaultOAuth2UserService`를 구현한 OAuth 기반 UserDetailService를 구현해서 `loadUser` 메소드에 전달받은 attribute 기반으로 계정이 없다면 계정을 생성해서 저장하고 OAuth 기반 UserDetail 객체를 반환한다.

```java
@Service
@RequiredArgsConstructor
@Log4j2
public class OAuthUserDetailService extends DefaultOAuth2UserService {
    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepository;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        log.debug("getClientRegistration = " + userRequest.getClientRegistration()); // registrationId로 어떤 OAuth로 로그인 했는지 확인 가능
        log.debug("getAccessToken = " + userRequest.getAccessToken().getTokenValue());

        OAuth2User oAuth2User = super.loadUser(userRequest);
        // 카카오 로그인 버튼 클릭 -> 카카오 로그인창 -> 로그인을 완료 -> code를 리턴(OAuth2-Client 라이브러리) -> AccessToken 요청
        // userRequest 정보 -> 회원 프로필 받아야함(loadUser함수 호출) -> 카카오로부터 회원프로필 받아준다.
        log.debug("getAttributes = " + oAuth2User.getAttributes());

        OAuth2UserInfo oAuth2UserInfo = null;
        if (userRequest.getClientRegistration().getRegistrationId().equals("kakao")) {
            log.debug("Request Kakao Login");
            oAuth2UserInfo = new KakaoUserInfo(oAuth2User.getAttributes());
        } else {
            log.debug("We only supported kakao.");
        }

        String provider = oAuth2UserInfo.getProvider(); // kakao
        String providerId = oAuth2UserInfo.getProviderId();
        String username = provider + "_" + providerId; // kakao_10021320120
        String password = passwordEncoder.encode("kakaoProviderPassword");
        String role = "ROLE_USER";

        Optional<User> userEntityOptional = userRepository.findByUsername(username);
        User userEntity;

        if (userEntityOptional.isEmpty()) {
            log.debug("Your first Kakao Login.");
            userEntity = User.builder()
                    .username(username)
                    .password(password)
                    .role(role)
                    .provider(provider)
                    .providerId(providerId)
                    .build();
            userRepository.saveAndFlush(userEntity);
        } else {
            log.debug("You already joined this service via kakao.");
            userEntity = userEntityOptional.get();
        }

        // 회원 가입을 강제로 진행해볼 예정
        return new UserDetail(userEntity, oAuth2User.getAttributes());
    }
}
```

## LoginController.java

RestController에 Redirect URI를 처리할 API를 기입한다.

```java
//...
@GetMapping("/login/oauth2/code/kakao")
@ResponseBody
public String kakaoLogin(@AuthenticationPrincipal UserDetail userDetail){
    log.info("attributes : {}", userDetail.getAttributes());

    return "kakao login success";
}
//...
```

## SecurityConfig.java

OAuth 기반 UserDetail Service를 Security Config에 등록해준다.

```java
httpSecurity.oauth2Login(formLogin -> formLogin
                      .userInfoEndpoint(userInfoEndpointConfig -> userInfoEndpointConfig.userService(oAuthUserDetailService))
                      .loginPage("/login")
                      .defaultSuccessUrl("/hello", true)
                );
```

해당 위의 설정을 마무리하면 Kakao 기반 OAuth 로그인을 구현할 수 있다.

### OAuthUserDetailService의 loadUser 동작하지 않는 이슈

구현 중에 카카오 로그인을 해도 `loadUser()`가 동작하지 않는 이슈가 있었다. 카카오 로그인을 성공해도 카카오 로그인 기반 서비스 계정이 생기지 않았다.  
이유는 OAuth Login 시에 동작하는 `OAuth2LoginAuthenticationFilter` 내 기본 Redirect URI가 `/login/oauth2/code/kakao`이기 때문에 기본 값이 아닌 새로운 경로를 설정하려면 `SecurityConfig`에서 OAuth 관련 설정 내에 `loginProcessingUrl()`에 기본값이 아닌 Redirect URI를 기입하면 된다.

```java
public class OAuth2LoginAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
    public static final String DEFAULT_FILTER_PROCESSES_URI = "/login/oauth2/code/*";
    
    //...
}
```

# 결론

> Spring Security 프레임워크를 통해 Session부터 JWT, OAuth 기반 방식의 로그인까지 다뤄보았다. Spring Security 프레임워크 덕분에 로그인 기능을 구현하기에 편해졌다.

## 참고

- [Spring Security를 이용한 통합 OAuth2 소셜 로그인 기능 구현(Kakao, Google, Naver)](https://velog.io/@yso8296/Spring-Security%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-%ED%86%B5%ED%95%A9-OAuth2-%EC%86%8C%EC%85%9C-%EB%A1%9C%EA%B7%B8%EC%9D%B8-%EA%B8%B0%EB%8A%A5-%EA%B5%AC%ED%98%84#%ED%86%B5%ED%95%A9-%EB%A1%9C%EA%B7%B8%EC%9D%B8-%ED%99%98%EA%B2%BD-%EC%A4%80%EB%B9%84)
- [OAuth2 구현 시 loadUser 연결되지 않을 때](https://non-stop.tistory.com/756)
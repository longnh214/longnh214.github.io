---
title: "[Spring/Spring Security] Spring Security를 이용해 로그인 구현하기. - (1) Session"
date: 2024-12-07 00:22:23 +09:00
categories: [Spring, Spring Security]
tags:
  [
    Spring,
    Spring Boot,
    Spring Security,
    로그인,
    backend,
    로그인 구현
  ]
---

# 서론

실제 프로젝트를 생성해서 Spring Security를 통해 로그인을 구현해본다.  
그리고 Spring Security에서 인증(Authentication)과 인가(Authorization)이 어떻게 관리되어지는 지 공부해보고자한다.

## Spring Security란?

Spring Security는 스프링 기반 어플리케이션에서 인증과 인가를 처리하는 프레임워크이다.  
이 프레임워크를 통해 어플리케이션의 보안을 체계적으로 관리할 수 있다.

## 인증과 인가란?

### 인증(Authentication)

> 인증이란 사용자(user)나 장치를 식별하는 절차로, 회원가입하고 로그인하는 로직을 의미하며,  
> **사용자가 사용자와 서비스 시스템 간에 공유되는 합의된 정보를 시스템에 전달해서 자신의 신원을 증명하는 과정**이다.

### 인가(Authorization)

> 인가란 **리소스나 로직(API)에 이 사용자가 접근할 수 있는 사용자인지 권한을 확인하는 프로세스**를 의미한다.

## build.gradle 설정

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.5'
    id 'io.spring.dependency-management' version '1.1.4'
}

//...

dependencies {
  //...
  implementation 'org.springframework.boot:spring-boot-starter-security'
  //...
}
```

실습 프로젝트의 Spring Boot 버전은 3.2.5를 사용했고, 'spring-boot-starter-security' 의존성을 추가해주었다.

<br>

![security_password](https://github.com/user-attachments/assets/d7e77b97-7c90-4a1d-a55a-479e057b6f89)

`build.gradle`에 `spring-security` 관련 의존성을 추가하고 어플리케이션을 실행하면 콘솔에 위와 같은 비밀번호가 임시로 발급되고, `http://localhost:8080` 기본 페이지에서 아래와 같이 기본 로그인 페이지에서 username은 `user`로 로그인이 가능하다.

<br>

![security_base_page](https://github.com/user-attachments/assets/2cd2361c-06fd-40f5-a1e1-3d717a436abf)

![security_login_fail](https://github.com/user-attachments/assets/212e2cb6-24e4-4044-9c17-69997e52cc0f)

바로 위는 로그인이 실패한 경우이다.

아래의 `filterChain` 설정을 진행한 뒤에 Bean으로 아래와 같은 `UserDetailService`를 추가하면, 문자열로 정한 username과 password로 로그인이 가능하다.

```java
@Bean
public UserDetailsService userDetailService() {
    InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
    manager.createUser(User.withUsername("user").password("1234").build());
    return manager;
}
```


## SecurityConfig 설정 파일

Spring Security 프레임워크에 대한 설정에 대한 정보를 파일로 작성해서 적용해야한다.

이전 버전에서는 아래와 같이 `@EnableWebSecurity` 어노테이션을 설정하고, `WebSecurityConfigurerAdapter` 클래스를 상속받은 Configuration 파일을 선언해서 작성하는 방식으로 진행되었다.

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter{
  @Override
    protected void configure(HttpSecurity http) throws Exception {
        //...
    }
}
```

<br>
<br>

하지만 `WebSecurityConfigurerAdapter` 클래스는 Deprecated 되었고, 아래와 같이 SecurityFilterChain을 Bean으로 등록해서 사용한다.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
    return httpSecurity
        .csrf(AbstractHttpConfigurer::disable)
        .authorizeHttpRequests(
            (authorizeRequests) ->
                    authorizeRequests
                            .requestMatchers("/user/add").permitAll()
                            .requestMatchers("/login/**").permitAll()
                            .requestMatchers("/admin/**").access("hasRole('ROLE_ADMIN')")
                            .anyRequest()
                            .authenticated()
        )
        .formLogin(
            (formLogin) 
                    -> formLogin
                            .loginPage("/login")
                            .defaultSuccessUrl("/hello", true)
                            .permitAll()
        )
        .build();
}
```

`HttpSecurity` 객체에 인증과 인가에 대한 설정 정보를 builder 패턴 형태로 담아 `build()` 메소드 호출 후 반환해야 한다.

<br>
<br>

### 각 설정의 흐름에 대해 보면,

```java
httpSecurity.csrf(AbstractHttpConfigurer::disable)
```

CSRF(Cross-Site Request Forgery) 공격에 대한 방지를 설정하는 부분으로, '사이트 간의 요청 위조' 공격이 방지된다.

> CSRF Filter가 적용되며, 해당 Spring 어플리케이션 내의 POST, PUT, DELETE 요청에 대해 HTTP 파라미터에 포함된 무작위의 CSRF 토큰을 서버의 값과 비교해서 검증해서 유효한 요청만 처리되는 방식으로 방어가 이루어진다.

<br>

```java
httpSecurity.authorizeRequests()
```

HTTP 통신에 사용되는 경로에 대한 권한을 부여하기 위한 설정 공간을 나타내는 메소드이다.

<br>

```java
.requestMatchers("/login/**").permitAll()
                            .requestMatchers("/admin/**").access("hasRole('ROLE_ADMIN')")
                            .anyRequest()
                            .authenticated()
```

`requestMatchers()`의 매개변수에는 경로를, 그 뒤에는 `permitAll()`이나, `access()`, `authenticated()`와 같은 메소드가 체이닝 형식으로 호출되는데, 의미대로 '모든 권한을 허용한다.', '특정 권한만 호출할 수 있다.', '인증이 필요하다.'를 나타낸다.

<br>

```java
httpSecurity.formLogin(
  formLogin(
    (formLogin) 
      -> formLogin
        .loginPage("/login")
        .defaultSuccessUrl("/hello", true)
        .permitAll()
  )
)
```

`formLogin()`은 인증을 위한 로그인을 지원하는 Spring Security의 기능으로, Spring Security를 통한 로그인 방식 사용 시의 설정을 하는 메소드이다. `loginPage()` 메소드에는 로그인 페이지의 경로를, `defaultSuccessUrl()`에는 로그인 성공 시 이동할 url과, 이 기능을 항상 작동시키도록 할 지 boolean으로 매개변수를 입력하면 된다.

<br>

## PasswordEncoder

Spring Security에서 제공하는 로그인 기능을 사용하려면, 비밀번호를 관리할 때 평문의 문자열로 관리하면 안되고, 보안상 암호화된 문자열로 비밀번호를 관리해야한다.  
이를 위해 `PasswordEncoder` 클래스의 구현체를 Bean으로 등록해주어야한다.

```java
@Bean
public PasswordEncoder bCryptPasswordEncoder() {
    return new BCryptPasswordEncoder();
}
```

아래와 같이 회원가입을 하는 로직을 작성할 때 `PasswordEncoder.encode()` 메소드로 평문의 비밀번호를 암호화해서 DB에 저장한다.

```java
public void addUser(UserDto userDto){
    User user = User.builder()
                .username(userDto.getUsername())
                .password(passwordEncoder.encode(userDto.getPassword()))
                .role(userDto.getRole())
                .build();
}
```

![encode_password](https://github.com/user-attachments/assets/3c2db75f-b575-4895-ad39-77508b225b12)

회원가입한 이후에 password가 암호화되어 저장되는 것을 볼 수 있다.

## UserDetails와 UserDetailsService

`UserDetails`는 인증된 사용자의 정보를 담는 인터페이스를 의미한다. Spring Security에서 사용자의 정보를 조회하려면 해당 인터페이스를 구체화한 객체를 이용해야한다.  
어플리케이션에서 사용되는 사용자 엔티티를 멤버 변수로 가지고, 해당 사용자의 만료, 잠금, 아이디와 비밀번호 관련된 기본 메소드를 구체화한 객체를 선언한다.

```java
@Data
public class PrincipalDetails implements UserDetails {
    private User user;

    public UserDetail(User user){
        this.user = user;
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
    public Collection<? extends GrantedAuthority> getAuthorities() {
        Collection<GrantedAuthority> collect = new ArrayList<>();

        collect.add(new SimpleGrantedAuthority(user.getRole()));

        return collect;
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
}
```

로그인이 성공된다면 `PrincipalDetails` 객체에 해당 사용자에 대한 정보가 담겨서 Spring Security 안에서 관리된다.

`SimpleGrantedAuthority`객체를 통해 로그인한 계정의 권한을 정하여 `UserDetails` 구현체를 만들 수 있다.

<br>

`filterChain` 메소드 내에 `.loginProcessingUrl("/login")`와 같이 로그인 프로세스의 경로를 설정하게 되면, 해당 경로로 로그인을 요청할 때 `UserDetailsService` 객체의 `loadUserByUsername` 메소드가 자동으로 호출되어 내부적으로 `PasswordEncoder.matches()` password를 검증하게 된다.

```java
@Service
@RequiredArgsConstructor
public class PrincipalDetailsService implements UserDetailsService {
    private final UserRepository userRepository;
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username).orElseThrow(() -> new RuntimeException("User Not Found"));
        return new PrincipalDetail(user);
    }
}
```

Spring Security에서 로그인 기능을 위임해서 처리하고, 로그인이 완료되면 `defaultSuccessUrl()`의 경로로 Redirect된다.

## 결론

Spring Security의 기본적인 Session 방식의 로그인을 알아봤다. 이후에는 Session보다 많이 사용되는 JWT, OAuth2 방식으로 로그인 구현을 주제로 다뤄보고자 한다.

코드는 [Github](https://github.com/longnh214/back_playground/tree/main/spring-security-example)에 정리해서 업로드했다.

## 참고

- [Spring Boot로 만드는 Spring Security 로그인 구현 - Session(1)](https://stir.tistory.com/266)
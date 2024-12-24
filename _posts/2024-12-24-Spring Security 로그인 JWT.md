---
title: "[Spring/Spring Security] Spring Security를 이용해 로그인 구현하기. - (2) JWT"
date: 2024-12-24 14:22:23 +09:00
categories: [Spring, Spring Security]
tags:
  [
    Spring,
    Spring Boot,
    Spring Security,
    로그인,
    backend,
    로그인 구현,
    JWT
  ]
---

# 서론

[지난 포스트](https://longnh214.github.io/posts/Spring-Security-%EB%A1%9C%EA%B7%B8%EC%9D%B8/)에서 Spring Security 프레임워크를 통해 Session 방식을 이용한 로그인 구현을 다뤄보았다. 이번 포스트에서는 JWT 방식으로 로그인을 구현해볼 것이며, Session 방식과 JWT 방식의 차이점을 알아본다.

# JWT란?

JWT는 'JSON Web Token'의 줄임말로, 클라이언트와 서버 사이에 통신할 때 권한에 대한 정보를 전달하기 위한 목적으로 전달되는 토큰이다.
암호화된 토큰으로, 일반적으로 사람이 읽을 수 없는 문자열로 이루어져있다.

## JWT의 구성 요소

![jwt](https://github.com/user-attachments/assets/70da6cb7-d3e6-4a70-8069-be7ff0b6c862)

JWT는 위와 같이 header, payload, signature로 구성되어있다.

- header(헤더) : 어떤 암호화 알고리즘으로 암호화되어 있는지, 어떤 토큰을 사용할 것인지에 대한 정보가 담겨있다.

- payload(정보) : 토큰을 통해 전달하고자 하는 내용이 담겨있다. 예를 들어 계정에 대한 id나 정보가 포함되어있다. payload는 수정이 가능하기 때문에 토큰의 발급일과 만료일자 같은 인증, 인가에 관련된 정보가 담겨있어야한다.

- signature(서명) : payload의 조작을 방지하기 위해 JWT를 발급할 당시에 header의 내용과 서버의 secret key를 기반으로 암호화한 payload 내용이 저장된다. JWT 토큰을 검증할 때 signature의 내용과 payload의 비교로 payload의 조작을 방지한다.

## Session과 JWT

우선, Session과 JWT 방식의 차이점 중 하나는 JWT는 **무상태성(stateless)**을 유지할 수 있다는 것이다. Session은 서버에서 관리되지만, JWT 토큰은 클라이언트에서 관리되고 서버의 HTTP 통신 시에 암호화된 토큰을 복호화해서 이용된다.

> 무상태성(stateless) : 서버가 클라이언트의 상태를 관리하지 않는 특성.

### SSR과 CSR

위와 같은 특징으로, Session은 Controller(API)에서 View(화면)가 결정되기 때문에 Thymeleaf, Mustache과 같은 SSR(Server Side Rendering) 방식의 템플릿 엔진에서 사용된다.

그리고 JWT 방식은 REST API를 통해 서버와 클라이언트(웹, 앱) 통신에서 이용된다.

웹에서는 로그인 시에 localStorage에 JWT 토큰을 보관해서 서버 API 통신 시에 해당 토큰 Request Header 중 `Authorization`에 `Bearer {JWT token}`을 담아 요청함으로서 권한을 검증한다.

<br>
<br>

> 💡 [Github](https://github.com/longnh214/front_playground/tree/main/jwt-login-example)에 JWT를 이용한 로그인 관련 클라이언트 코드를 Vue.js로 작성했다.

아래에는 Spring Security 프레임워크로 JWT 방식 로그인을 구현한 코드를 기록하고자 한다.

> 💡 서버 코드는 [Github](https://github.com/longnh214/back_playground/tree/main/spring-security-example)에 상세히 작성했다.

## build.gradle 설정

```gradle
//...

dependencies {
    //...

    //JWT
    implementation 'io.jsonwebtoken:jjwt-api:0.11.5'
    implementation 'io.jsonwebtoken:jjwt-impl:0.11.5'
    implementation 'io.jsonwebtoken:jjwt-jackson:0.11.5'

    //...
}
```

기존 build.gradle에 `JWT` 관련 의존성을 추가해주었다.

## application.yml

```yaml
jwt:
  secret_key: "base64 encoding string"
  expiration_time: 86400000 # 24 hours = 1 day
```

`applicaton.yml` 에 JWT 암호화를 위한 secret key와 만료 시간을 지정했다.

## JwtTokenUtil.java

```java
@Component
@RequiredArgsConstructor
public class JwtTokenUtil {
    @Value("${jwt.secret_key}")
    private String SECRET_KEY;
    @Value("${jwt.expiration_time}")
    private long EXPIRATION_TIME;
    private final UserDetailsService userDetailsService;

    /**
     * JWT에서 사용자 이름을 추출한다.
     *
     * @param token JWT 토큰
     * @return 추출된 사용자 이름 (토큰의 subject 값)
     */
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    /**
     * JWT에서 특정 클레임을 추출한다.
     *
     * @param token          JWT 토큰
     * @param claimsResolver 클레임을 처리하는 함수
     * @param <T>            클레임의 반환 타입
     * @return 추출된 클레임 값
     */
    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    /**
     * JWT에서 모든 클레임을 추출한다.
     *
     * @param token JWT 토큰
     * @return 토큰에 포함된 모든 클레임
     */
    private Claims extractAllClaims(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(SECRET_KEY) // 비밀 키를 설정
                .build()
                .parseClaimsJws(token) // 토큰 파싱
                .getBody(); // 클레임 반환
    }

    /**
     * 사용자 정보를 기반으로 JWT를 생성한다.
     *
     * @param userDetails 인증된 사용자 정보
     * @return 생성된 JWT 토큰
     */
    public String generateToken(UserDetails userDetails) {
        return Jwts.builder()
                .setSubject(userDetails.getUsername()) // subject에 사용자 이름 설정
                .setIssuedAt(new Date(System.currentTimeMillis())) // 발급 시간 설정
                .setExpiration(new Date(System.currentTimeMillis() + EXPIRATION_TIME)) // 만료 시간 설정
                .signWith(SignatureAlgorithm.HS256, SECRET_KEY) // 서명 알고리즘 및 비밀 키 설정
                .compact(); // 최종적으로 토큰 생성
    }

    /**
     * JWT의 유효성을 검사한다.
     *
     * @param token       JWT 토큰
     * @param userDetails 인증된 사용자 정보
     * @return 토큰이 유효한 경우 true, 그렇지 않으면 false
     */
    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token); // 토큰에서 사용자 이름 추출
        return username.equals(userDetails.getUsername()) // 사용자 이름이 일치하는지 확인
                && !isTokenExpired(token); // 토큰이 만료되지 않았는지 확인
    }

    /**
     * JWT가 만료되었는지 확인한다.
     *
     * @param token JWT 토큰
     * @return 만료된 경우 true, 그렇지 않으면 false
     */
    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    /**
     * JWT 토큰을 기반으로 Authentication 객체를 생성한다.
     *
     * @param token JWT 토큰
     * @return 생성된 Authentication 객체
     */
    public Authentication getAuthentication(String token, UserDetails userDetails) {
        // 토큰에서 사용자 이름 추출
        String username = extractUsername(token);

        // 사용자 이름이 UserDetails의 username과 일치하는지 확인
        if (!username.equals(userDetails.getUsername())) {
            throw new IllegalArgumentException("JWT 토큰의 사용자 정보가 일치하지 않습니다.");
        }

        // UserDetails를 기반으로 Authentication 객체 생성 및 반환
        return new UsernamePasswordAuthenticationToken(
                userDetails,
                null,
                userDetails.getAuthorities() // 권한 정보 포함
        );
    }
}
```

JWT 토큰 생성, 유효성 검사, 만료 확인 Authentication 객체 생성 등 JWT 토큰 관련 메소드를 Util 클래스로 작성했다.

## JwtAuthenticationFilter.java

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtTokenUtil jwtTokenUtil;
    private final UserDetailsService userDetailsService;
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        final String authHeader = request.getHeader("Authorization");
        final String jwtToken;
        final String username;

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        jwtToken = authHeader.substring(7);
        username = jwtTokenUtil.extractUsername(jwtToken);

        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            if (jwtTokenUtil.isTokenValid(jwtToken, userDetails)) {
                var authToken = (UsernamePasswordAuthenticationToken) jwtTokenUtil.getAuthentication(jwtToken, userDetails);
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

Spring의 Filter 기능을 통해 HTTP API 통신 시에 Request Header 중 `Authorization`의 `Bearer {JWT token}`내용 중 토큰을 추출해서 검증하고, username을 얻는다.

## SecurityConfig 설정 파일

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
    //JWT 방식 return
    return httpSecurity
        .csrf(AbstractHttpConfigurer::disable)
        .cors(httpSecurityCorsConfigurer -> httpSecurityCorsConfigurer
                .configurationSource(corsConfigurationSource()))
        .sessionManagement(sessionManagement -> sessionManagement
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS) // 세션 생성 정책 설정
        )
        .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login/**").permitAll() // 인증 없이 접근 허용
                .anyRequest().authenticated() // 나머지는 인증 필요
        )
        .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
        .build();
}
```

`HttpSecurity` 객체에 Session 방식과 마찬가지로 설정에 대한 내용을 builder 패턴 형태로 담아 `build()` 메소드 호출 후 반환했다.

<br>
<br>

### 각 설정의 흐름에 대해 보면,

```java
httpSecurity.cors(httpSecurityCorsConfigurer -> httpSecurityCorsConfigurer
                        .configurationSource(corsConfigurationSource()))
```

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();
    configuration.setAllowCredentials(true);
    configuration.addAllowedOrigin("http://localhost:5173");
    configuration.addAllowedMethod("*");
    configuration.addAllowedHeader("*");
        
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
}
```

서버와 클라이언트가 별도로 구성되어있고, localhost 호스트 통신이기 때문에 CORS 설정을 하고, 그 내용을 SecurityFilterChain에 등록해주었다.  
클라이언트에 대한 Origin을 허용하고, 모든 HTTP 메소드와 header에 대해서 허용을 해주었다.

<br>

```java
httpSecurity.sessionManagement(sessionManagement -> sessionManagement
                        .sessionCreationPolicy(SessionCreationPolicy.STATELESS) // 세션 생성 정책 설정
                )
```

세션 방식을 이용하지 않고, 세션 생성 정책을 무상태성(STATELESS)으로 할 것을 명시해주었다.

<br>

```java
httpSecurity.authorizeHttpRequests(auth -> auth
                        .requestMatchers("/login/**").permitAll() // 인증 없이 접근 허용
                        .anyRequest().authenticated() // 나머지는 인증 필요
                )
```

login 관련 API 엔드포인트는 검증에 상관없이 허용해주었고, 그 이외의 API 엔드포인트 요청에는 권한 검증이 적용되도록 설정했다.

<br>

```java
httpSecurity.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
```

모든 HTTP 요청에 대해 `jwtAuthenticationFilter`가 적용되도록 설정해서, HTTP Request 헤더에 Authorization 값이 존재하는 지 검증하도록 설정했다.

## LoginController.java

```java
@RequiredArgsConstructor
@RestController
@Log4j2
public class LoginController {
    private final AuthenticationManager authenticationManager;
    private final JwtTokenUtil jwtTokenUtil;

    @PostMapping("/login")
    public String login(@RequestBody AuthRequestDto auth) {
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(auth.getUsername(), auth.getPassword())
        );

        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        log.info("id : {}", userDetails.getUsername());
        return jwtTokenUtil.generateToken(userDetails);
    }
}
```

해당 로직을 `LoginService`로 추출했어야 할 것이라고 생각된다. HTTP 통신으로 전달받은 `username`, `password` 기반으로 맞다면 `Authentication` 객체를 반환받고, `UserDetail` 객체를 이용해서 JWT 토큰을 생성 후 클라이언트에 반환한다.

💡 클라이언트에서는 반환받은 JWT 토큰을 아래와 같이 localStorage에 저장 후 만료 시점까지 로그인을 유지할 수 있다.

<img width="753" alt="localStorage" src="https://github.com/user-attachments/assets/fd1b8142-2632-4233-a6c1-bcb9bdae696a" />

### TestController.java

```java
@RestController
public class TestController {
    @GetMapping("/test")
    public String hello(){
        return "Hello.";
    }
}
```

클라이언트에서 로그인 후 username을 보이고, JWT 토큰을 검증받고 TestController를 호출해서 'Hello.' 문자열을 반환받아 화면에 보였다.

<img width="226" alt="dashboard" src="https://github.com/user-attachments/assets/579f246b-5ab0-4e81-84ce-e1b10d884b8a" />

## 결론

Spring Security에서 JWT 방식의 로그인을 알아봤다. 이후에는 OAuth2 방식으로 로그인 구현을 다뤄보고자 한다.

## 참고

- [Spring Boot로 만드는 Spring Security 로그인 구현 - JWT(2)](https://stir.tistory.com/275)
- [JWT](https://velog.io/@hahan/JWT%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9D%B8%EA%B0%80)
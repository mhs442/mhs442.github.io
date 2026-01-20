---
layout: post
title: "Spring Security"
date: 2026-01-11 18:20:00 +0900
categories: [개발]
excerpt: "Spring Security 사용하기"
published: true
---

# Spring Security 사용하기

<br>
<br>

## 목차
1. Spring Security란?
2. Spring Security 동작 방식
3. 간단 구현하기
4. 커스텀

<br>
<br>

## 1. Spring Security란?

> "로그인, 권한, CSRF까지 자동으로 처리해주는 Spring의 보안 전문가"

Spring Security는 Spring Boot 애플리케이션의 인증(Authentication), 인가(Authorization), 보안 보호를 위한 강력한 프레임워크이다.

세상에는 정말 많은 로그인 방식이 존재한다.
그 중, Spring Boot 에서 제공하고 있는 Spring Security를 이용해 구현해보려 한다.


<br>
<br>


## 2. Spring Security 동작 방식

Spring Security의 간단한 구조를 먼저 살펴보자.

<img src="/assets/img/spring-security-arch.png" width="600" height="400">

위 그림을 천천히 보면 어렵지 않게 동작 방식을 알 수 있다.

<br>

### ☝️핵심 포인트

| 구성요소 | 역할                         |
| ------ |----------------------------|
| Filter | 요청 가로채기                    |
| Token | 인증 정보를 담는 그릇               |
| Manager | 적절한 Provider 찾아 위임         |
| Provider | 실제 인증 로직                   |
| Context | 인증 결과 전역 저장                |

<br>
<br>

그래서 아주 간단하게 4가지만 기억하면 된다.

<br>

1. 필터에서 토큰 생성
2. 매니저가 프로바이더를 찾고
3. UserDetailsService가 DB연결
4. 성공하면 SecurityContext에 저장

<br>
<br>

## 3. 간단 구현하기
<br>
- 우선 사용을 위한 build.gradle에 의존성을 추가

```groovy
implementation 'org.springframework.boot:spring-boot-starter-security'
testImplementation 'org.springframework.boot:spring-boot-starter-security-test'
```

<br>

config 패키지 아래에 SecurityConfig 클래스 파일을 만들고 작업


```java
@EnableWebSecurity  // spring security 활성화
@Configuration
public class SecurityConfig {
    @Bean   // 빈으로 등록
    public SecurityFilterChain securityFilterChain(HttpSecurity http) {
        http.authorizeHttpRequests(auth ->
                auth.requestMatchers("/login").permitAll()
                        .anyRequest().authenticated())
                .formLogin(AbstractAuthenticationFilterConfigurer::permitAll);

        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
        manager.createUser(User.withUsername("user").password("1234").roles("USER").build());
        return manager;
    }

    @Bean
    BCryptPasswordEncoder bCryptPasswordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```
<br>

@EnableWebSecurity 어노테이션을 이용해 Spring Security를 활성화 해주고 FilterChain역할을 하는 메서드의 해당 동작들을 빈으로 등록 해주어야 한다.

<br>

SecurityFilter 메서드의 내부에 `auth.requestMatchers("/login")` 부분에 `"/login"` 경로는 필터에 걸지 않겠다는 내용이다.
해당 방법으로 에셋이나 필수 포함 파일들을 필터에서 제외시킬 수 있다. 뒤에 `permitAll()` 메서드를 호출하여 모두 허용 처리.
`formLogin` 메서드를 이용해 `loginProcessingUrl : "/login"` 자동 `permitAll()` 처리까지 해주면 된다. 해당 설정을 넣어주지 않으면 로그인에 필요한 url("/login" GET, POST 등..)
이 필터에 걸려 접근이 불가능해진다.

<br>

`.anyRequest().authenticated()` 를 붙여 그 외 요청들은 모두 인증대상이 되도록 설정한다.

<br>

보통 DB에 연결하여 사용자 정보를 조회해서 인증을 구현하지만 간단 구현이기 때문에 인메모리 방식으로 사용자를 만들어 주기 위해 `userDetailsService()` 를 만들어 빈으로 등록했다.
유저 이름은 `user` 패스워드는 `1234` 실제로는 암호화 한 상태로 넣어주어야 한다.(여기서는 편의를 위해 그냥 1234로 대체) 각 사용자별로 role을 정할 수 있기에 role도 임의로 `USER` 로 등록.

<br>

그리고 암호화를 위한 `bCryptPasswordEncoder()` 메서드까지 빈으로 등록하면 된다.

<br>

이후 서버를 실행시켜보면 아래와 같은 기본 로그인 페이지를 볼 수 있다. 이 페이지는 Spring Security에서 기본으로 제공하는 페이지이다.

<br>

<img src="/assets/img/spring-security-login-page.png" width="600" height="400">

<br>

이후 로그인 하게 되면, 404 에러 페이지가 나오는데 말 그대로 노출할 페이지가 없기 떄문이다. 자, 기본 동작은 확인 해봤으니 이제 커스텀 해보도록 하자.

<br>

<br>

## 4. 커스텀

<br>

우선 내가 원하는 로그인 페이지와 해당 로그인 동작을 수행하는 로직을 만들어 보자. 나는 로그인 로직 자체는 내가 만들고 Spring Security에게 필터 및 인가의 역할만을 맡기려고 한다. 

<br>

```html
<!DOCTYPE html>
<html lang="ko" xmlns="http://www.w3.org/1999/xhtml"
      xmlns:th="http://www.thymeleaf.org"
      class="loading" data-textdirection="ltr">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <title>Title</title>
</head>
<body>
<div class="content">
    <form class="login-form" method="post" action="/login">
        <h2>Please sign in</h2>
        <p>
            <label for="email">Email</label>
            <input type="text" id="email" name="email" placeholder="email" required autofocus>
        </p>
        <p>
            <label for="password">Password</label>
            <input type="password" id="password" name="password" placeholder="password" required>
        </p>
        <button type="submit" class="primary">로그인</button>
    </form>
</div>
</body>
</html>

```

<br>

위 내용은 thymeleaf를 이용한 아주 간단 로그인 페이지이다. 실제 spring security에서는 username 과 password를 받았지만 나는 email을 이용해 로그인을 구현하려 한다.

<br>

```java
@Controller
@RequiredArgsConstructor
public class LoginController {
    private final LoginService loginService;

    @GetMapping("/login")
    public String loginPage(){
        return "login/login-page";
    }

    @PostMapping("/login")
    public String login(Model model, HttpServletRequest request, LoginRequest loginRequest){
        if(loginService.login(loginRequest, request))
            return "redirect:/home";
        else{
            model.addAttribute("message", "로그인에 실패하였습니다. 이메일과 패스워드를 확인해주세요.");
            return "login/login-page";
        }
    }


    @PostMapping("/logout")
    public String logout(HttpServletRequest request, HttpServletResponse response){
        loginService.logout(request, response);

        return "redirect:/login";
    }

}
```

<br>

로그인 로직의 엔드포인트인 LoginController이다 이곳에서 로그인 페이지, 로그인 시도, 그리고 로그아웃까지 처리한다.

<br>

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class LoginService implements UserDetailsService {
    private final LoginRepository loginRepository;
    private final BCryptPasswordEncoder bCryptPasswordEncoder;

    public boolean login(LoginRequest loginRequest, HttpServletRequest request){
        // 로그인 요청이 들어오면 해당 email로 사용자가 있는지 판단.
        UserDTO userDTO = loadUserByUsername(loginRequest.getEmail());

        // 사용자가 있다면 패스워드를 비교하여 검증을 진행한다.
        if(bCryptPasswordEncoder.matches(loginRequest.getPassword(), userDTO.getPassword())){
            // 패스워드가 맞다면 권한을 생성하고
            Authentication auth = new UsernamePasswordAuthenticationToken(
                    userDTO,
                    userDTO.getPassword(),
                    userDTO.getAuthorities()
            );

            // Security Context에 해당 권한 정보 저장
            SecurityContext context = SecurityContextHolder.createEmptyContext();
            context.setAuthentication(auth);
            SecurityContextHolder.setContext(context);

            // 마찬가지로 세션에도 저장하고
            HttpSession session = request.getSession(true);
            session.setAttribute(
                    HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY,
                    context
            );
            // true 반환
            return true;
        }
        else
            // 패스워드가 맞지 않다면 false 반환
            return false;
    }


    @Override
    public UserDTO loadUserByUsername(String email) throws UsernameNotFoundException {  // Spring Security 필수 구현 메서드
        // 이곳에서 사용자 정보를 받아온다.
        User user = loginRepository
                .findByEmail(email)
                .orElseThrow(() -> new UsernameNotFoundException("User Not Found"));

        // UserDTO는 UserDetails를 구현한 객체이다.
        return UserDTO.builder()
                .id(user.getId())
                .email(user.getEmail())
                .password(user.getPassword())
                .username(user.getUsername())
                .age(user.getAge())
                .role(user.getRole())
                .build();
    }

    public void logout(HttpServletRequest request, HttpServletResponse response){
        // SecurityContext 정리
        SecurityContextHolder.clearContext();

        // 세션 무효화
        HttpSession session = request.getSession(false);
        if (session != null) {
            session.invalidate();
        }
    }
}
```

<br>

실제 로그인 로직을 처리하는 LoginService이다. Spring Security의 사용자 처리 인터페이스인 UserDetailsService의 구현체로써 필수 메서드인 loadByUserName 메서드를 구현하고 실제 로그인 로직은 해당 메서드를 호출하고 있는
login() 메서드에서 처리하고 있다.(주석 참고)

<br>

```java
@EnableWebSecurity
@Configuration
@Slf4j
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http, LoginService loginService, BCryptPasswordEncoder bCryptPasswordEncoder) {
        http.csrf(AbstractHttpConfigurer::disable)
                .authorizeHttpRequests(auth ->
                        auth.requestMatchers("/login").permitAll()   // login 페이지만 권한 없이 접근 가능 
                                .anyRequest().authenticated())
                .exceptionHandling(ex ->
                        ex.authenticationEntryPoint(authEntryPoint())   // 로그인 없이 다른 페이지 접근 불가 처리
                                .accessDeniedHandler(accessDeniedHandler()))    // 인가되지 않은 페이지 접근 차단 처리
                .formLogin(AbstractHttpConfigurer::disable) // 로그인 로직 커스텀으로 인해 사용안함
                .logout(AbstractHttpConfigurer::disable); // 로그아웃도 커스텀으로 인해 사용 안함

        return http.build();
    }

    @Bean
    public AuthenticationEntryPoint authEntryPoint() {  // 로그인 없이 접근할 경우의 동작 명시
        return (request, response, authException) -> 
                response.sendRedirect("/login");    // 로그인 페이지로 리다이렉트
    }

    @Bean
    public AccessDeniedHandler accessDeniedHandler() {
        return (request, response, accessDeniedException) -> {  // 인가되지 않은 접근에 대한 동작 명시
            request.setAttribute("exception", "권한이 없습니다.");
            request.getRequestDispatcher("/error/403").forward(request, response);  // 403 페이지 포워드
        };
    }

    @Bean
    BCryptPasswordEncoder bCryptPasswordEncoder() {
        return new BCryptPasswordEncoder(); // 패스워드 암호화 빈
    }
}
```
<br>

위 내용은 Spring Security 설정 파일인데, 사실 Spring Security의 전반적인 기능은 모두 제외했다.

<br>

그저 권한 여부를 판단하는 필터로만 동작을 명시하고 권한 및 인가가 안된 부분에 대해서 처리를 명시했다.

나머지는 전부 내가 직접 만든 로직을 이용해서 구현하였다.

<br>

위와 같이 로그인을 구현하고 실제 동작을 검증하면?

<br>

<img src="/assets/img/spring-security-custom-login-page.png" width="600" height="400">

<br>

실제 이메일 형식의 데이터를 h2 데이터베이스와 연동해서 사용자 정보를 넣어두어서 구현하였다.

<br>

<img src="/assets/img/spring-security-custom-home-page.png" width="600" height="400">

<br>

실제 로그인 하게 되면 임의로 만든 /home 경로를 리다이렉트 하도록 구현하였다.

<br>

이렇게 조금 막장(?) 으로 Spring Security를 만들어 보았는데 사실 그냥 내 욕심으로 어디까지 바꿀 수 있나를 실험해 본 결과인 것 같다.

다음 글에는 조금 더 Spring Security를 적극적으로 활용해서 세션 기반이 아닌, JWT 토큰 방식으로 로그인 하는 방법을 가져오겠다.

<br>
<br>

전체 코드 git : https://github.com/mhs442/spring-security.git
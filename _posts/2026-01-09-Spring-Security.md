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

그런데 내부에서 실제로 어떤 코드가 실행되는지를 알면 훨씬 더 명확하게 이해된다. 하나씩 파헤쳐 보자.

<br>
<br>

### 2-1. DelegatingFilterProxy — 서블릿과 Spring의 다리

Spring Security의 필터들은 사실 순수한 서블릿 필터가 아니다.
서블릿 컨테이너(Tomcat 등)는 Spring의 `ApplicationContext`를 전혀 모른다. 그래서 서블릿 컨테이너와 Spring 사이에 **DelegatingFilterProxy**라는 다리 역할의 서블릿 필터가 존재한다.

<br>

```java
// DelegatingFilterProxy.java (Spring Framework)
public class DelegatingFilterProxy extends GenericFilterBean {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        Filter delegateToUse = this.delegate;
        if (delegateToUse == null) {
            WebApplicationContext wac = findWebApplicationContext();
            // Spring ApplicationContext에서 "springSecurityFilterChain" 빈을 찾아온다
            delegateToUse = initDelegate(wac);
            this.delegate = delegateToUse;
        }
        // 실제 처리는 Spring 빈에게 위임
        invokeDelegate(delegateToUse, request, response, filterChain);
    }

    protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
        String targetBeanName = getTargetBeanName(); // "springSecurityFilterChain"
        // Spring 컨텍스트에서 해당 빈을 꺼내온다
        Filter delegate = wac.getBean(targetBeanName, Filter.class);
        return delegate;
    }
}
```

<br>

서블릿 컨테이너에는 `DelegatingFilterProxy`가 등록되고, 실제 보안 처리는 Spring이 관리하는 `FilterChainProxy` 빈에 위임되는 구조다.

<br>
<br>

### 2-2. FilterChainProxy — 보안 필터들의 지휘자

DelegatingFilterProxy로부터 위임받은 주인공이 바로 **FilterChainProxy**다.
`springSecurityFilterChain`이라는 이름으로 빈에 등록되어 있으며, 등록된 `SecurityFilterChain` 목록 중 요청에 매칭되는 체인을 찾아 실행한다.

<br>

```java
// FilterChainProxy.java (Spring Security)
public class FilterChainProxy extends GenericFilterBean {

    private List<SecurityFilterChain> filterChains;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        doFilterInternal(request, response, chain);
    }

    private void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        // 현재 요청에 매칭되는 SecurityFilterChain의 필터 목록을 가져온다
        List<Filter> filters = getFilters((HttpServletRequest) request);

        if (filters == null || filters.isEmpty()) {
            // 매칭되는 체인이 없으면 그냥 통과
            chain.doFilter(request, response);
            return;
        }

        // 찾은 필터들로 가상의 FilterChain을 만들어 순서대로 실행
        VirtualFilterChain virtualFilterChain = new VirtualFilterChain(request, chain, filters);
        virtualFilterChain.doFilter(request, response);
    }

    private List<Filter> getFilters(HttpServletRequest request) {
        for (SecurityFilterChain chain : this.filterChains) {
            if (chain.matches(request)) {
                return chain.getFilters(); // 매칭된 체인의 필터 목록 반환
            }
        }
        return null;
    }
}
```

<br>

필터 체인 안에는 Spring Security가 기본으로 제공하는 필터들이 **순서대로** 등록되어 있다.

<br>

| 필터 | 역할 |
|------|------|
| SecurityContextHolderFilter | 세션에서 SecurityContext 복원 |
| UsernamePasswordAuthenticationFilter | 폼 로그인 인증 처리 |
| ExceptionTranslationFilter | 인증/인가 예외를 HTTP 응답으로 변환 |
| AuthorizationFilter | 요청에 대한 권한 검사 |

<br>
<br>

### 2-3. UsernamePasswordAuthenticationFilter — 인증의 시작점

`POST /login` 요청이 들어오면 **UsernamePasswordAuthenticationFilter**가 가장 먼저 가로챈다.
부모 클래스인 `AbstractAuthenticationProcessingFilter`의 `doFilter`부터 실행된다.

<br>

```java
// AbstractAuthenticationProcessingFilter.java
public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        if (!requiresAuthentication((HttpServletRequest) request, (HttpServletResponse) response)) {
            // /login 이외의 요청은 그냥 다음 필터로 넘김
            chain.doFilter(request, response);
            return;
        }

        try {
            // 실제 인증 시도 (구현체에서 처리)
            Authentication authenticationResult = attemptAuthentication(
                    (HttpServletRequest) request, (HttpServletResponse) response);

            // 인증 성공 처리
            successfulAuthentication(request, response, chain, authenticationResult);
        } catch (AuthenticationException ex) {
            // 인증 실패 처리
            unsuccessfulAuthentication(request, response, ex);
        }
    }
}

// UsernamePasswordAuthenticationFilter.java
public class UsernamePasswordAuthenticationFilter extends AbstractAuthenticationProcessingFilter {

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
            throws AuthenticationException {

        // 요청에서 username, password 추출
        String username = obtainUsername(request);
        String password = obtainPassword(request);

        // 아직 인증되지 않은 토큰 생성
        UsernamePasswordAuthenticationToken authRequest =
                UsernamePasswordAuthenticationToken.unauthenticated(username, password);

        // AuthenticationManager에게 인증을 위임
        return this.getAuthenticationManager().authenticate(authRequest);
    }
}
```

<br>

여기서 중요한 점은 이 시점에서 만들어지는 `UsernamePasswordAuthenticationToken`이 **미인증 상태**라는 것이다.
`unauthenticated()`로 생성되며, 이 토큰을 `AuthenticationManager`에 넘겨 실제 인증을 요청한다.

<br>
<br>

### 2-4. ProviderManager & DaoAuthenticationProvider — 진짜 인증은 여기서

`AuthenticationManager`의 기본 구현체는 **ProviderManager**다.
여러 `AuthenticationProvider`를 리스트로 갖고 있으며, 넘어온 토큰 타입을 지원하는 Provider를 찾아 인증을 위임한다.

<br>

```java
// ProviderManager.java
public class ProviderManager implements AuthenticationManager {

    private List<AuthenticationProvider> providers;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {

        AuthenticationException lastException = null;
        Authentication result = null;

        for (AuthenticationProvider provider : getProviders()) {
            if (!provider.supports(authentication.getClass())) {
                continue; // 해당 토큰 타입을 지원하지 않는 Provider는 건너뜀
            }

            try {
                result = provider.authenticate(authentication);
                if (result != null) {
                    break; // 인증 성공 시 루프 탈출
                }
            } catch (AuthenticationException ex) {
                lastException = ex;
            }
        }

        if (result == null) {
            throw lastException;
        }
        return result;
    }
}
```

<br>

폼 로그인의 경우 `UsernamePasswordAuthenticationToken`을 처리하는 **DaoAuthenticationProvider**가 선택된다.

<br>

```java
// DaoAuthenticationProvider.java (일부 발췌)
public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {

    @Override
    protected UserDetails retrieveUser(String username, UsernamePasswordAuthenticationToken authentication)
            throws AuthenticationException {
        // 등록해둔 UserDetailsService를 호출해 DB에서 사용자를 조회
        UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
        return loadedUser;
    }

    @Override
    protected void additionalAuthenticationChecks(UserDetails userDetails,
            UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {

        String presentedPassword = authentication.getCredentials().toString();
        // 등록해둔 PasswordEncoder(BCryptPasswordEncoder)로 비밀번호 검증
        if (!this.passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
            throw new BadCredentialsException("Bad credentials");
        }
    }
}
```

<br>

우리가 직접 구현한 `UserDetailsService`의 `loadUserByUsername()`과 `BCryptPasswordEncoder`가 실제로는 여기서 호출된다.

<br>
<br>

### 2-5. SecurityContextHolder — 어디서든 꺼낼 수 있는 이유

인증에 성공하면 `Authentication` 객체는 **SecurityContextHolder**에 저장된다.
서비스 어디서든 인증 정보를 꺼낼 수 있는 이유가 바로 여기에 있다.

<br>

```java
// SecurityContextHolder.java
public class SecurityContextHolder {

    // 기본 전략은 ThreadLocal 방식
    private static SecurityContextHolderStrategy strategy = new ThreadLocalSecurityContextHolderStrategy();

    public static SecurityContext getContext() {
        return strategy.getContext();
    }

    public static void setContext(SecurityContext context) {
        strategy.setContext(context);
    }

    public static void clearContext() {
        strategy.clearContext();
    }
}

// ThreadLocalSecurityContextHolderStrategy.java
final class ThreadLocalSecurityContextHolderStrategy implements SecurityContextHolderStrategy {

    // 스레드마다 독립적인 저장소 (ThreadLocal)
    private static final ThreadLocal<Supplier<SecurityContext>> contextHolder = new ThreadLocal<>();

    @Override
    public SecurityContext getContext() {
        return getDeferredContext().get();
    }

    @Override
    public void setContext(SecurityContext context) {
        contextHolder.set(() -> context);
    }

    @Override
    public void clearContext() {
        contextHolder.remove(); // 스레드 반납 시 반드시 제거
    }
}
```

<br>

내부적으로 `ThreadLocal`을 사용하기 때문에, 같은 요청을 처리하는 스레드라면 컨트롤러든 서비스든 어디서든 `SecurityContextHolder.getContext().getAuthentication()`으로 인증 정보에 접근할 수 있다.

<br>

> ⚠️ 단, `@Async`나 새로운 스레드를 직접 생성하는 경우 ThreadLocal이 공유되지 않아 인증 정보가 `null`이 될 수 있다. 이럴 때는 `SecurityContextHolder`의 전략을 `MODE_INHERITABLETHREADLOCAL`로 변경하거나 수동으로 전달해야 한다.

<br>
<br>

### 2-6. HttpSessionSecurityContextRepository — 요청 간 인증 상태 유지

SecurityContextHolder는 현재 요청 스레드에서만 유효하다.
그렇다면 다음 요청에서는 어떻게 로그인 상태를 유지할까?

바로 **HttpSessionSecurityContextRepository**가 그 역할을 담당한다.
요청이 끝날 때 SecurityContext를 HTTP 세션에 저장하고, 다음 요청이 시작될 때 세션에서 꺼내 다시 ThreadLocal에 복원한다.

<br>

```java
// HttpSessionSecurityContextRepository.java (일부 발췌)
public class HttpSessionSecurityContextRepository implements SecurityContextRepository {

    public static final String SPRING_SECURITY_CONTEXT_KEY = "SPRING_SECURITY_CONTEXT";

    @Override
    public SecurityContext loadDeferredContext(HttpServletRequest request) {
        HttpSession session = request.getSession(false);

        if (session == null) {
            return generateNewContext(); // 세션이 없으면 빈 컨텍스트 반환
        }

        // 세션에 저장된 SecurityContext 복원
        Object contextFromSession = session.getAttribute(SPRING_SECURITY_CONTEXT_KEY);

        if (contextFromSession instanceof SecurityContext) {
            return (SecurityContext) contextFromSession;
        }
        return generateNewContext();
    }

    @Override
    public void saveContext(SecurityContext context, HttpServletRequest request, HttpServletResponse response) {
        HttpSession session = request.getSession(false);

        if (session != null) {
            // 응답이 커밋되기 전에 세션에 SecurityContext를 저장
            session.setAttribute(SPRING_SECURITY_CONTEXT_KEY, context);
        }
    }
}
```

<br>

이제 앞서 커스텀 코드에서 세션에 직접 저장했던 이유가 명확해진다.

<br>

```java
// 커스텀 로그인에서 세션에 직접 저장한 이유:
// formLogin을 disable했기 때문에 UsernamePasswordAuthenticationFilter가 동작하지 않고,
// 결과적으로 HttpSessionSecurityContextRepository의 자동 저장도 동작하지 않는다.
// 그래서 아래 코드처럼 수동으로 세션에 저장해야 다음 요청에서도 인증 상태가 유지된다.
HttpSession session = request.getSession(true);
session.setAttribute(
    HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY,
    context
);
```

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

이후 로그인 하게 되면, 404 에러 페이지가 나오는데 말 그대로 노출할 페이지가 없기 때문이다. 자, 기본 동작은 확인 해봤으니 이제 커스텀 해보도록 하자.

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

위 내용은 thymeleaf를 이용한 아주 간단한 로그인 페이지이다. 실제 spring security에서는 username 과 password를 받았지만 나는 email을 이용해 로그인을 구현하려 한다.

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

로그인 로직의 엔드포인트인 LoginController이다. 이곳에서 로그인 페이지, 로그인 시도, 그리고 로그아웃까지 처리한다.

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

실제 로그인 로직을 처리하는 LoginService이다. Spring Security의 사용자 처리 인터페이스인 UserDetailsService의 구현체로써 필수 메서드인 loadUserByUsername 메서드를 구현하고 실제 로그인 로직은 해당 메서드를 호출하고 있는
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
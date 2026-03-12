---
layout: post
title: "코인자동매매시스템(with. claude) - 4"
date: 2026-03-11 00:00:00 +0900
categories: [개발]
excerpt: "개발자라는 이유 하나로 코인 자동매매 시스템을 만들게 된 이야기"
published: true
---

# 껍데기부터 만들자


[3편 보러가기](https://mhs442.github.io/2026-02-27/coin-auto-trading-03)

<br>

DB 설계 -> 아키텍처(인프라) 구상 및 기술 선정까지 진행했으니, 이젠 웹 서비스의 기본적인 껍데기를 작성하려 한다.

<br>

## 로그인은 뭘로 할까?

<br>

1편에서 정리한 요구사항 중 가장 먼저 손대야 할 건 **사용자 관리** 부분이었다.

<br>

**📌 사용자 관리**
- 로그인을 필수로 해야 한다
- 계정이 없다면 회원가입을 통해 계정을 생성할 수 있어야 한다

<br>

사용자를 관리해야 하니 로그인 기능부터 만들기로 했다.

<br>

로그인은 Spring Security를 사용하고, 우선은 기본적인 세션-쿠키 방식으로 구현하기로 했다. 완성 단계에서는 JWT 방식으로 전환할 생각인데, Redis를 도입하게 되면 거기에 Refresh Token을 저장하고 아니라면 DB에 저장할 생각이다.

<br>

물론 토큰 무효화 방식에 대해선 조금 더 생각해 봐야 할 부분이 있겠지만 이건 나중일이니까. 이 정도만 구상하자.

<br>

그래서 Security 설정을 해주었다.

<br>

``` java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http, LoginService loginService) throws Exception {
    http.csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth ->
                    auth.requestMatchers("/login", "/signup", "/_resources/**").permitAll()
                            .anyRequest().authenticated())  // 그 외는 모두 인증 필요
            .formLogin(formLogin ->
                    formLogin.loginPage("/login")
                            .loginProcessingUrl("/login")
                            .defaultSuccessUrl("/dashboard")
                            .usernameParameter("phoneNumber")
                            .permitAll())
            .logout(logout ->
                    logout.logoutUrl("/logout").permitAll())
            .userDetailsService(loginService);

    return http.build();
}
```

<br>

이전 포스팅에도 말해두었지만 휴대폰 번호 기반 로그인을 구현할 것이고, 로그인 페이지`/login`와 회원가입 페이지`/signup` 그리고 앞단에서 필요로 하는 리소스`/_resources/**` 경로를 제외하고는 로그인이 필요하도록 설정했다.

<br>

로그인 성공 후에는 `/dashboard`로 보내 코인 목록을 뿌려주도록 할 것이다. 

<br>

이후 로그인 페이지를 노출할 `LoginController`와 로그인을 처리할 `LoginService`를 작성한다.

<br>

``` java
@Controller
public class LoginController {
    @GetMapping("/login")
    public String login() {
        return "login/login-page";
    }
}
```

<br>

``` java
@Service
@RequiredArgsConstructor
public class LoginService implements UserDetailsService {
    private final LoginRepository loginRepository;

    /**
     * Spring Security 인증 진입점.
     * 전화번호로 사용자를 조회하여 {@link UserDetails}를 반환한다.
     *
     * @param phoneNumber 로그인 시 입력한 전화번호 (Spring Security의 username 역할)
     * @return 사용자 정보 DTO (id, phoneNumber, username, password 포함)
     * @throws CustomException 사용자를 찾을 수 없는 경우 (USER_NOT_FOUND)
     */
    @Override
    public UserDTO loadUserByUsername(String phoneNumber){
        User user = loginRepository.findByPhoneNumber(phoneNumber)
                .orElseThrow(() -> new CustomException(ExceptionMessage.USER_NOT_FOUND));

        return UserDTO.builder()
                .id(user.getId())
                .phoneNumber(user.getPhoneNumber())
                .username(user.getUsername())
                .password(user.getPassword())
                .build();
    }
}
```

<br>

자 이렇게, DB에서 휴대폰 번호를 이용해 사용자 정보를 가져오는 역할을 한다. 이 메서드를 `Spring Security`내부적으로 호출해서 사용하고 있다.

<br>

`CustomException`이나 `UserDTO`는 뒤에서 설명하겠다.

<br>


``` java
@Repository
public interface LoginRepository extends JpaRepository<User, Long> {
    Optional<User> findByPhoneNumber(String phoneNumber);
}
```

<br>

`Spring Data JPA`를 사용하기로 했으니 위와같이 선언하여 DB에서 사용자 정보를 가져오도록 했다!

<br>

## 나만의 예외처리

<br>

예외처리는 간단하게 가기로 했다. 먼저 예외 메시지를 `enum`으로 정의했다.

<br>

``` java
@AllArgsConstructor
@Getter
public enum ExceptionMessage {
    USER_NOT_FOUND(HttpStatus.BAD_REQUEST, "사용자를 찾을 수 없습니다."),
    SIGNATURE_GENERATION_FAILED(HttpStatus.INTERNAL_SERVER_ERROR, "Bybit 서명에 실패하였습니다."),
    PASSWORD_MISMATCH(HttpStatus.BAD_REQUEST, "비밀번호가 일치하지 않습니다."),
    DUPLICATE_PHONE_NUMBER(HttpStatus.CONFLICT, "이미 등록된 휴대폰 번호입니다."),
    INVALID_API_KEY(HttpStatus.BAD_REQUEST, "유효하지 않은 Bybit API Key입니다."),
    ENCRYPTION_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "암호화에 실패하였습니다."),
    DECRYPTION_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "복호화에 실패하였습니다.");

    private final HttpStatus status;
    private final String message;
}
```

<br>

각 예외 상황에 맞는 `HttpStatus`와 메시지를 한 곳에서 관리할 수 있게 했다. 새로운 예외가 필요하면 여기에 한 줄만 추가하면 된다.

<br>

처음에는 예외마다 별도 클래스를 만드는 방식도 고민했는데, 그러면 예외가 늘어날수록 클래스 파일이 계속 생기니까 enum 하나로 관리하는 게 더 낫겠다 싶었다.

<br>

지금은 이렇게만 만들고(말 그대로 껍데기니까.) 나중에는 ControllerAdvice를 이용해서 front단과 back단에서 던지는 메시지 형태를 다르게 하도록 구성할 예정이다!

<br>

그리고 이 enum을 감싸는 `CustomException`을 만들었다.

<br>

``` java
@Getter
public class CustomException extends RuntimeException {
    private final ExceptionMessage exceptionMessage;

    public CustomException(ExceptionMessage exceptionMessage) {
        super(exceptionMessage.getMessage());
        this.exceptionMessage = exceptionMessage;
    }

    public CustomException(ExceptionMessage exceptionMessage, Throwable cause) {
        super(exceptionMessage.getMessage(), cause);
        this.exceptionMessage = exceptionMessage;
    }
}
```

<br>

`RuntimeException`을 상속받았기 때문에 Unchecked Exception이고, 별도의 `throws` 선언 없이 어디서든 던질 수 있다. 위에서 봤던 `LoginService`처럼 말이다.

<br>

``` java
User user = loginRepository.findByPhoneNumber(phoneNumber)
        .orElseThrow(() -> new CustomException(ExceptionMessage.USER_NOT_FOUND));
```

<br>

이렇게 하면 예외를 던지는 쪽에서는 `ExceptionMessage`만 골라서 넘기면 되니까 코드가 깔끔해진다.

<br>

## 사용자 정보를 조금 더 입맛대로

<br>

`LoginService.loadUserByUsername()` 을 보면 리턴 객체가 `UserDTO`인 것을 알 수 있는데 원래는 `UserDetails`를 반환해야 하는 것이 맞다.
하지만 인터페이스를 그대로 반환할 수 없으니 해당 인터페이스를 구현해서 사용하기로 했다.

<br>

``` java
@Getter
@Builder
public class UserDTO implements UserDetails {
    private Long id;                // 사용자 PK
    private String phoneNumber;     // 휴대폰 번호 (Spring Security username)
    private String password;        // BCrypt 해싱된 비밀번호
    private String username;        // 사용자 이름

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of();
    }

    @Override
    public String getPassword() {
        return this.password;
    }

    @Override
    public String getUsername() {   
        return this.phoneNumber;
    }
}
```

<br>

내가 필요로 하는 정보들만 가져와서 커스텀 해서 사용하기로 했다. 우선은 껍데기만 작성하는 단계이기 때문에 간단한 정보만 담기로 했다.

<br>


## 암호화의 시작

<br>

Bybit와 통신이 필요한 `API_KEY`와 `API_SECRET`을 받고 있으니 이걸 평문으로 DB에 저장할 순 없다.
당연하게도 암호화 해서 저장해야 하니까 암호화를 위한 통로를 만들어두자! 전 회사에서 CSAP 인증 받을 때도 개인정보로 판단되는 모든 정보는 암호화 해야 했었다. 하지만 지금은 `API_KEY`와 `API_SECRET`만 암호화 하고 나중에 추가하던지 해야지....
<br>

``` java
@Component
public class AesEncryptor {

    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private static final int IV_LENGTH = 12;       // GCM 권장 IV 길이 (12byte)
    private static final int TAG_LENGTH = 128;     // 인증 태그 길이 (128bit)

    @Value("${aes.encryption.key}")
    private String encryptionKey;

    /**
     * AES-GCM 암호화
     * - SecureRandom으로 매번 랜덤 IV 생성
     * - 결과: Base64( IV(12byte) + 암호문 + 인증태그 )
     *
     * @param plainText 암호화할 평문
     * @return Base64 인코딩된 암호문 (IV 포함)
     */
    public String encrypt(String plainText) {
        try {
            // 랜덤 IV 생성
            byte[] iv = new byte[IV_LENGTH];
            new SecureRandom().nextBytes(iv);

            SecretKeySpec keySpec = new SecretKeySpec(
                    encryptionKey.getBytes(StandardCharsets.UTF_8), "AES");
            GCMParameterSpec gcmSpec = new GCMParameterSpec(TAG_LENGTH, iv);

            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, keySpec, gcmSpec);
            byte[] encrypted = cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));

            // IV + 암호문을 합쳐서 저장
            ByteBuffer buffer = ByteBuffer.allocate(IV_LENGTH + encrypted.length);
            buffer.put(iv);
            buffer.put(encrypted);
            return Base64.getEncoder().encodeToString(buffer.array());
        } catch (Exception e) {
            throw new CustomException(ExceptionMessage.ENCRYPTION_ERROR, e);
        }
    }

    /**
     * AES-GCM 복호화
     * - 암호문 앞 12byte에서 IV 추출 후 복호화
     * - 인증 태그 검증 실패 시 예외 발생 (변조 감지)
     *
     * @param encryptedText Base64 인코딩된 암호문 (IV 포함)
     * @return 복호화된 평문
     */
    public String decrypt(String encryptedText) {
        try {
            byte[] decoded = Base64.getDecoder().decode(encryptedText);

            // IV 추출
            ByteBuffer buffer = ByteBuffer.wrap(decoded);
            byte[] iv = new byte[IV_LENGTH];
            buffer.get(iv);

            // 암호문 추출
            byte[] cipherText = new byte[buffer.remaining()];
            buffer.get(cipherText);

            SecretKeySpec keySpec = new SecretKeySpec(
                    encryptionKey.getBytes(StandardCharsets.UTF_8), "AES");
            GCMParameterSpec gcmSpec = new GCMParameterSpec(TAG_LENGTH, iv);

            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, keySpec, gcmSpec);
            byte[] decrypted = cipher.doFinal(cipherText);
            return new String(decrypted, StandardCharsets.UTF_8);
        } catch (Exception e) {
            throw new CustomException(ExceptionMessage.DECRYPTION_ERROR, e);
        }
    }
}
```

<br>

암호화 방식을 AES-GCM 방식으로 했다. API_KEY 값이 변조 되었는지 검증까지도 수행하면 좋을것이기에 해당 알고리즘을 채택했다.

<br>

그래서 암호화, 복호화 로직을 만들고 빈으로 등록했다. 처음에는 static으로 관리할까 했는데... `encryptionKey`값을 주입받고 있고, `ALGORITHM`필드도 외부로 주입받는 형태로 변경할 가능성이 있어 그냥 bean으로 관리하기로 했다. 

---

<br>

## 마무리

<br>

이번 편에서는 로그인 기능을 중심으로 Spring Security 설정, 예외처리 구조, 사용자 정보 DTO, 그리고 API Key 암호화를 위한 AES-GCM 유틸까지 웹 서비스의 기본적인 뼈대를 세워봤다.

<br>

다음 편에서는 회원가입 기능을 구현하면서, Bybit API Key 검증과 실제 암호화 저장 흐름까지 다뤄볼 예정이다. 그리고 실제로 Claude에게 어떤식으로 명령을 주어서 작업하는지까지 다뤄볼 예정이다!

<br>

> **쿠키 🍪**
>
> 개발을 진행하면서 사용자를 새롭게 추가하거나 삭제하는 경우가 반복적으로 생겼는데.. 회원가입 기능을 안 넣어둬서 이 작업을 계속 DB에 붙어 쿼리를 날려서 작업해야 했다. (진짜 너무 귀찮다...)
>
> 그래서 회원가입까지 그냥 빠르게 만들기로 했다! 어차피 Claude가 만들고 난 검수만 하면 되니까. 이건 다음 편에서 자세히 다뤄볼 예정이다.

<br>


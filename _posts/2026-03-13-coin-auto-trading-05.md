---
layout: post
title: "코인자동매매시스템(with. claude) - 5"
date: 2026-03-13 00:00:00 +0900
categories: [개발]
excerpt: "개발자라는 이유 하나로 코인 자동매매 시스템을 만들게 된 이야기"
published: true
---

# 드디어 코인이 보인다


[4편 보러가기](https://mhs442.github.io/2026-03-11/coin-auto-trading-04)

<br>

이전편에서 웹 프로젝트의 기본적인 껍데기를 만들었는데 이젠 본격적인 기능 구현을 시작해야한다.

<br>

## 회원가입이 이렇게 쉬울수가!

<br>

사실 이미 회원가입 기능을 구현 해두었다. Claude 에게 부탁하니 작업을 아주 쉽게 해주더라. 우선 SRS.txt(요구사항정의서)와 이에 따른, 엔터티를 먼저 읽어보라고 했다. 그랬더니 필요한 기능들에 대해 먼저 조언을 해줬었는데 확인하고 실제로 미흡하거나 부족한 부분들은 반영해서 구현을 했다.

<br>

> 사실 이미 개발이 어느정도 진행 된 상태이다.
> 
> 대시보드에 코인목록을 출력하고, 실제 코인에 패턴을 입력하는 폼까지는 완성된 상태이다.
> 그래서 Claude와 대화한 캡처를 첨부하고 싶지만 아쉽게도 캡처해둔 내역이 없어서 너무 아쉽다..

<br>

우선 코드로 회원가입의 흐름을 보자.

<br>

``` java
@GetMapping("/signup")
public String signupPage() {
    return "signup/signup-page";
}

@PostMapping("/signup")
public String signup(SignupRequest request, Model model) {
    try {
        signupService.signup(request);
        return "redirect:/login?signupSuccess";
    } catch (CustomException e) {
        model.addAttribute("error", e.getMessage());
        model.addAttribute("request", request);
        return "signup/signup-page";
    }
}
```

<br>

``` java
/**
 * 회원가입을 처리한다.
 * 비밀번호 일치 확인 → 전화번호 중복 확인 → Bybit API Key 유효성 검증 → 사용자 저장 순서로 진행된다.
 * 비밀번호는 BCrypt로 해싱하고, API Key/Secret은 AES-256으로 암호화하여 저장한다.
 *
 * @param request 회원가입 요청 (username, phoneNumber, password, passwordConfirm, apiKey, apiSecret 포함)
 * @throws CustomException 비밀번호 불일치(PASSWORD_MISMATCH), 전화번호 중복(DUPLICATE_PHONE_NUMBER),
 *                         API Key 무효(INVALID_API_KEY) 시 예외 발생
 */
@Transactional
public void signup(SignupRequest request) {
    // 1. 비밀번호 일치 확인
    if (!request.getPassword().equals(request.getPasswordConfirm())) {
        throw new CustomException(ExceptionMessage.PASSWORD_MISMATCH);
    }

    // 2. 휴대폰 번호 중복 확인
    if (loginRepository.findByPhoneNumber(request.getPhoneNumber()).isPresent()) {
        throw new CustomException(ExceptionMessage.DUPLICATE_PHONE_NUMBER);
    }

    // 3. Bybit API Key 유효성 검증
    boolean isValid = bybitApiKeyValidator.validate(request.getApiKey(), request.getApiSecret());
    if (!isValid) {
        throw new CustomException(ExceptionMessage.INVALID_API_KEY);
    }

    // 4. User 저장 (비밀번호 BCrypt 해싱, API Key/Secret AES 암호화)
    User user = new User();
    user.setUsername(request.getUsername());
    user.setPhoneNumber(request.getPhoneNumber());
    user.setPassword(bCryptPasswordEncoder.encode(request.getPassword()));
    user.setEmail(request.getEmail());
    user.setApiKey(aesEncryptor.encrypt(request.getApiKey()));
    user.setApiSecret(aesEncryptor.encrypt(request.getApiSecret()));

    loginRepository.save(user);
}
```

<br>

역시 `CLAUDE.local.md`에 적어둔 내용대로 `JavaDoc`도 잘 작성하고 각 줄마다 주석도 잘 달아준 모습! 그리고 요청, 반환객체도 Convention에 맞게 잘 작성해준다.(편안)

<br>

여기서 알아야 할 건, `@Transactional`을 선언해서 회원가입 과정 중 문제가 생기면 롤백되도록 설정했다. 그리고 거래에 필요한 `API_KEY`와 `API_SECRET`유효성을 검증하기 위해 검증 로직을 따로 두어서 호출해서 사용하기로 했다.

<br>

## Bybit와 손잡기

<br>

그렇다면 검증하는 로직을 안볼순 없겠지.

<br>

```java
@Component
public class BybitApiKeyValidator {

    @Value("${bybit.api.url}")
    private String bybitApiUrl;

    @Value("${bybit.api.recv-window:5000}")
    private String recvWindow;

    private final RestTemplate restTemplate = new RestTemplate();
    private final ObjectMapper objectMapper = new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

    /**
     * 사용자의 API Key/Secret으로 Bybit GET /v5/user/query-api 를 호출하여 유효성 검증.
     * retCode == "0" 이면 유효, 그 외 무효.
     */
    public boolean validate(String userApiKey, String userApiSecret) {
        try {
            String timestamp = String.valueOf(Instant.now().toEpochMilli());
            String str2Sign = timestamp + userApiKey + recvWindow;
            String signature = hmacSha256(userApiSecret, str2Sign);

            HttpHeaders headers = new HttpHeaders();
            headers.set("X-BAPI-API-KEY", userApiKey);
            headers.set("X-BAPI-TIMESTAMP", timestamp);
            headers.set("X-BAPI-SIGN", signature);
            headers.set("X-BAPI-RECV-WINDOW", recvWindow);
            headers.setContentType(MediaType.APPLICATION_JSON);

            HttpEntity<Void> entity = new HttpEntity<>(headers);

            ResponseEntity<String> response = restTemplate.exchange(
                    bybitApiUrl + "/v5/user/query-api",
                    HttpMethod.GET,
                    entity,
                    String.class
            );

            QueryApiKeyResponse apiResponse =
                    objectMapper.readValue(response.getBody(), QueryApiKeyResponse.class);
            return "0".equals(apiResponse.getRetCode());
        } catch (Exception e) {
            return false;
        }
    }

    /**
     * 주어진 데이터에 대해 HMAC-SHA256 서명을 생성한다.
     *
     * @param secret 서명에 사용할 비밀키 (Bybit API Secret)
     * @param data   서명할 데이터 문자열
     * @return 16진수 소문자로 인코딩된 HMAC-SHA256 서명 문자열
     * @throws RuntimeException 서명 생성 중 오류 발생 시
     */
    private String hmacSha256(String secret, String data) {
        try {
            Mac sha256Hmac = Mac.getInstance("HmacSHA256");
            SecretKeySpec secretKey = new SecretKeySpec(
                    secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256");
            sha256Hmac.init(secretKey);
            byte[] bytes = sha256Hmac.doFinal(data.getBytes(StandardCharsets.UTF_8));
            StringBuilder hash = new StringBuilder();
            for (byte b : bytes) {
                hash.append(String.format("%02x", b));
            }
            return hash.toString();
        } catch (Exception e) {
            throw new RuntimeException("HMAC-SHA256 서명 생성 실패", e);
        }
    }
}
```

<br>

참고로 `bybitApiUrl`은 Bybit 거래소의 API 주소인데, Bybit는 **테스트넷**과 **메인넷** 두 가지 환경을 제공한다. 테스트넷은 가상 자금으로 실제와 동일한 API를 테스트할 수 있는 환경이고, 메인넷은 실제 거래가 이루어지는 환경이다. 현재는 개발 테스트 중이라 테스트넷을 사용하고 있고, 실 배포 시에는 메인넷 URL로 변경하면 된다.(테스트넷 데이터 엄청 튀긴 하더라....)

<br>

그리고 Bybit API 문서를 보면, 인증이 필요한 API를 호출할 때 요청 헤더에 다음 정보들을 담아야 한다.

<br>

| 헤더 | 설명 |
|------|------|
| `X-BAPI-API-KEY` | 발급받은 API Key |
| `X-BAPI-TIMESTAMP` | 요청 시점의 타임스탬프 (밀리초) |
| `X-BAPI-SIGN` | HMAC-SHA256 서명값 |
| `X-BAPI-RECV-WINDOW` | 요청 유효 시간 (밀리초) |

<br>

위 코드에서 `HttpHeaders`에 세팅하는 부분이 바로 이 요구사항을 충족하기 위한 것이다. 서명값은 `timestamp + apiKey + recvWindow`를 이어 붙인 문자열을 `API Secret`으로 **HMAC-SHA256** 서명해서 만든다.

<br>

여기서 `recvWindow`는 요청의 유효 시간인데, 서버가 요청을 받았을 때 timestamp와 현재 시간의 차이가 이 값을 넘으면 요청을 거부한다. 쉽게 말해 **"이 요청은 5초 안에 도착해야 유효합니다"**라는 의미다. 재전송 공격(Replay Attack)을 방지하기 위한 장치이다.

<br>

이렇게 헤더를 구성해서 `/v5/user/query-api`에 요청을 보내고, 응답의 `retCode`가 `"0"`이면 유효한 키로 판단한다.

<br>

그리고 이 검증 로직은 `OpenFeign`이 아닌 `RestTemplate`을 사용했는데, 회원가입 시점에는 아직 사용자의 API Key가 시스템에 등록되기 전이라 Feign에 설정해둔 인증 헤더 구성 로직을 태울 수 없기 때문이다. 일회성 검증이니 `RestTemplate`으로 직접 호출하는 게 더 깔끔했다.

<br>

그래서 이러한 과정들을 거쳐서 회원가입 로직을 완성했다. 프론트단 작업은 만진부분은 거의 없고 디자인도 Claude가 짜준대로 사용했는데 나름 괜찮아서 그대로 쓰기로 했다!

<br>

<img src="/assets/img/coin-auto-trading-05/login-page.png">

<br>

<img src="/assets/img/coin-auto-trading-05/signup-page.png">

<br>

내가 좋아하는 파란색이 포인트로 들어가서 아주 흡족스럽다.

<br>

## 코인목록 만들기 - 이게 무슨 코인이지?

<br>

코인 목록을 만들고 나서 느낀건 실제 거래되는 코인명들이 내가 아는 코인 이름과는 달랐다는 점이다. 

<br>

**비트코인** -> **BTC** 이런식이였다! 한국 거래소에서는 한국식 이름으로 바꿔서 출력해주는 것 같던데..(설마 번역까지 해달라는건 아니겠지..?)

<br>

아무튼 간에, Spring Security 설정을 다시 한번 떠올려보면 `.defaultSuccessUrl("/dashboard")`부분에 성공하면 `/dashboard`엔드포인트로 보내도록 설정했다. 이 부분도 Claude가 야무지게 작성해줬다.

<br>

코인 목록을 가져오기 위해 Bybit API를 호출해야 하는데, 여기서 드디어 **OpenFeign**이 등장한다. 3편에서 기술 스택으로 선정해뒀던 그 녀석이다.

<br>

``` java
@FeignClient(name = "marketClient", url = "${bybit.api.url}")
public interface MarketClient {

    /**
     * Bybit GET /v5/market/tickers
     * 카테고리별 전체 종목의 현재가·거래량 등 실시간 티커 정보를 조회한다.
     *
     * @param category 파생상품 카테고리 (예: "linear", "inverse", "spot")
     * @return 티커 목록 응답
     */
    @GetMapping("/v5/market/tickers")
    ResponseEntity<FindTickerResponse> getTickers(@RequestParam("category") String category);
}
```

<br>

이게 끝이다. 인터페이스 하나 선언하고 `@FeignClient`를 붙이면 알아서 구현체를 만들어준다. `RestTemplate`으로 직접 URL 조합하고 헤더 세팅하고 할 필요 없이, 메서드를 호출하기만 하면 된다.

<br>

그리고 이 API는 **인증이 필요 없는 Public API**라서 앞서 회원가입 검증에서 했던 것처럼 서명 헤더를 세팅할 필요가 없다. Bybit API는 시세 조회 같은 공개 데이터는 인증 없이 호출할 수 있도록 열어두고 있다.

<br>

응답 데이터는 `FindTickerResponse`로 매핑했다. 여기서 `@JsonIgnoreProperties(ignoreUnknown = true)`를 붙여줬는데, Bybit API 응답에는 내가 필요로 하지 않는 필드들도 잔뜩 내려온다. 이 어노테이션이 없으면 매핑 객체에 정의하지 않은 필드가 응답에 포함되어 있을 때 역직렬화 에러가 발생하기 때문에 필수로 넣어줘야 한다.

<br>

``` java
@Getter @Setter
@JsonIgnoreProperties(ignoreUnknown = true)
public class FindTickerResponse extends BybitMasterDTO {

    private Result result;      // 티커 조회 결과

    public static class Result {
        private String category;            // 파생상품 카테고리 (예: "linear")
        private List<TickerInfo> list;      // 티커 목록
    }

    public static class TickerInfo {
        private String symbol;              // 종목 심볼 (예: BTCUSDT)
        private String lastPrice;           // 현재가
        private String price24hPcnt;        // 24시간 가격 변동률 (%)
        private String highPrice24h;        // 24시간 최고가
        private String lowPrice24h;         // 24시간 최저가
        private String volume24h;           // 24시간 거래량 (코인 기준)
        private String turnover24h;         // 24시간 거래대금 (USDT 기준)
    }
}
```

<br>

2편에서 Claude에게 지시했던 규칙 중 "API 응답은 Map 대신 매핑 객체를 만들어서 사용하라"는 항목이 있었는데, 그 규칙대로 잘 작성해줬다. 각 필드마다 주석도 꼼꼼하게 달아줬고.

<br>

그 중에 `BybitMasterDTO` 객체를 상속받고 있는데 해당 객체는 다음과 같다.

<br>

```java
@Getter @Setter
public abstract class BybitMasterDTO {
    private String retCode;     // 응답 코드 ("0": 정상)
    private String retMsg;      // 응답 메시지 (예: "OK", 오류 설명)
}
```

<br>

Bybit API 응답 객체는 모두 응답코드(`retCode`)와 응답메시지(`retMsg`)를 함께 반환하고 있어 하나의 파일로 관리하기 위해 마스터 객체를 생성해서 사용하기로 했다.

<br>

이렇게 조합된 데이터를 대시보드에 뿌려주는 컨트롤러는 다음과 같다.

<br>

``` java
@GetMapping("/dashboard")
public String dashBoard(Model model) {
    FindTickerResponse tickerResponse = marketService.getTickers();

    if (tickerResponse != null && tickerResponse.getResult() != null
            && tickerResponse.getResult().getList() != null) {
        model.addAttribute("coins", tickerResponse.getResult().getList());
    }

    return "home/dash-board";
}
```

<br>

`MarketService`에서 코인 목록을 가져와 `Model`에 담고, Thymeleaf 템플릿에서 뿌려주는 구조다. 간단하다!

<br>

그렇게 완성된 대시보드 화면이 이거다!

<br>

<img src="/assets/img/coin-auto-trading-05/dashboard.png">

<br>

드디어 코인이 보인다. 제목 회수 완료.

---

<br>

## 마무리

<br>

이번 편에서는 회원가입 기능과 Bybit API Key 검증, 그리고 OpenFeign을 활용한 코인 목록 조회까지 다뤄봤다. 껍데기만 있던 프로젝트에 실제 데이터가 흘러들어오기 시작하니 슬슬 그럴듯한 모습이 갖춰지고 있다.

<br>

다음 편에서는 코인 상세 차트와 자동매매 패턴 설정 UI를 다뤄볼 예정이다.

<br>

> **쿠키 🍪**
>
> 대시보드에 코인 목록이 뜨는 걸 보고 지인에게 바로 캡처를 보내줬더니 "오 진짜 되네?!" 하면서 신기해했다. 프로토타입 모델의 힘이란 이런 거다. 일단 보여줘야 반응이 온다.

<br>


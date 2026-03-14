---
layout: post
title: "코인자동매매시스템(with. claude) - 6"
date: 2026-03-14 00:00:00 +0900
categories: [개발]
excerpt: "개발자라는 이유 하나로 코인 자동매매 시스템을 만들게 된 이야기"
published: true
---

# 이제 진짜 매매 준비다


[5편 보러가기](https://mhs442.github.io/2026-03-13/coin-auto-trading-05)

<br>

## 매초 갱신, 근데 제한이 있다?

<br>

전편까지 해서 코인 목록을 드디어 출력했다. 이제 해야할 일은 다음 요구사항을 보면 알 수 있다.

**📌 대시보드**
- 코인 목록이 노출되어야 하며, 거래량 기준 내림차순으로 정렬되어야 한다
- 특정 코인을 클릭하면 상세 차트를 확인할 수 있어야 한다

<br>

단순히 코인 목록만 뿌리지 말고, 거래량을 기준으로 내림차순으로 뿌려주어야 한다. 근데 실제 거래소를 확인 해보니 매초마다 코인의 데이터가 바뀌는게 아닌가! 그렇다면, 나도 똑같이 코인 목록을 1초마다 갱신하는 쪽으로 하기로 했다.

<br>

처음에는 단순히 매초마다 API를 호출해서 계속해서 프론트단 데이터를 갈아끼우는 **폴링**방식으로만 구현하려 했다. 그런데, 매초마다 계속 호출하는건 괜찮은데...

<br>

> 사용자가 100명이라면..?

<br>

아 물론 계속해서 언급했듯이 혼자서만 사용할거라고 이야기는 했지만, 그래도 확장 가능성에 대해서 생각은 해두어야 하는게 맞지 않겠는가!

<br>

그래서 생각해봤다. 매 요청마다 새로운 데이터를 요청한다고 생각하면 100명 사용자 기준 1초에 100번의 요청을 Bybit 서버에 요청해야 한다.

<br>

Bybit API를 사용하기 위해서는 트래픽 제한이 있던걸로 기억한다. 실제로 Bybit 공식 문서를 확인해보니 다음과 같은 **Rate Limit**(요청 제한)이 존재했다.

<br>

| 제한 항목 | 내용 |
|----------|------|
| **HTTP 요청** | IP당 5초 내 최대 600건 |
| **WebSocket 연결** | 5분 내 최대 500건, IP당 최대 1,000건 |
| **제한 초과 시** | `403 access too frequent` 에러 발생, **최소 10분간 IP 차단** |

<br>

Rate Limit에 걸리면 자동으로 IP가 차단되고 최소 10분을 기다려야 풀린다. 자동매매 시스템에서 이런 일이 발생하면 그 시간 동안 매매가 멈추게 되니까 꽤 치명적이다.

<br>

그래서 생각한 방식이 **서버에서 1초마다 딱 한 번만 폴링하고, 그 결과를 캐시에 저장**해서 모든 사용자가 캐시된 데이터를 공유하도록 하는 것이었다. 사용자가 100명이든 1,000명이든 Bybit로 나가는 요청은 1초에 1건이면 충분하다.

<br>

그리고 여기서 아까의 요구사항도 함께 처리했다. USDT 마켓 종목만 필터링하고, **거래량 기준 내림차순 정렬**까지 캐시 갱신 시점에 해두면 컨트롤러에서는 캐시만 꺼내 쓰면 된다.

<br>

```java
// REST 폴링 캐시 (대시보드용)
private final AtomicReference<FindTickerResponse> cachedTickers = new AtomicReference<>();

@Scheduled(fixedRate = 1000)
public void refreshTickersCache() {
    FindTickerResponse response = marketClient.getTickers(Category.LINEAR.getCategory()).getBody();

    if (response != null && response.getResult() != null && response.getResult().getList() != null) {
        // USDT 마켓 종목만 필터링
        response.getResult().getList()
                .removeIf(ticker -> !ticker.getSymbol().endsWith("USDT"));
        // 24시간 거래량 기준 내림차순 정렬
        response.getResult().getList()
                .sort((t1, t2) -> Double.compare(
                        Double.parseDouble(t2.getVolume24h()),
                        Double.parseDouble(t1.getVolume24h())));
        cachedTickers.set(response);
    }
}
```

<br>

`@Scheduled(fixedRate = 1000)`으로 1초마다 Bybit API를 호출하고, USDT 종목 필터링 → 거래량 정렬 → `AtomicReference`에 캐시하는 흐름이다. 이렇게 하면 Rate Limit 걱정도 없고, 정렬 로직도 매 요청마다 반복할 필요 없이 캐시 갱신 시 한 번만 수행하면 된다.

<br>

지금은 `AtomicReference`로 단일 서버 메모리에 캐시하고 있지만, 3편에서 언급했듯이 나중에 분산 환경으로 확장하게 되면 이 캐시를 **Redis**로 옮기는 것도 고려하고 있다. 여러 서버 인스턴스가 동일한 캐시 데이터를 공유해야 하는 상황이 오면, Redis가 딱 맞는 선택이 될 것이다.

<br>

## 폴링으로는 부족하다

<br>

자, 그런데 여기까지는 대시보드에 코인 목록을 뿌리기 위한 이야기다. 이 프로젝트의 핵심인 **자동매매**는 다른 방식이 필요했다.

<br>

사용자가 자동매매 패턴을 활성화하면, 해당 코인의 가격 변동을 실시간으로 감지해서 즉시 거래를 트리거해야 한다. 1초마다 폴링하는 캐시 데이터로는 반응 속도가 부족할 수 있다. 여기서 **WebSocket**이 등장한다.

<br>

패턴이 활성화되면 해당 코인의 WebSocket을 구독하고, 가격이 변동될 때마다 `MarketService`의 가격 리스너를 통해 `AutoTradeService`에 콜백이 전달된다.

<br>

```java
// AutoTradeService - 애플리케이션 시작 시 가격 리스너 등록
@PostConstruct
public void init() {
    marketService.addPriceListener(this::onPriceUpdate);
}

// WebSocket에서 가격 변동이 수신되면 호출
public void onPriceUpdate(String symbol, String newPrice) {
    // 해당 심볼의 활성 세션만 필터링하여 매매 로직 처리
    for (AutoTradeSessionDTO session : activeSessions.values()) {
        if (session.getSymbol().equals(symbol)) {
            processSessionWithPrice(session, newPrice);
        }
    }
}
```

<br>

WebSocket으로 가격을 실시간 수신하면서, 직전가 대비 현재가를 비교해 Long/Short 신호를 판별하고 등록된 패턴과 일치하면 즉시 주문을 실행하는 구조다.

<br>

추가로, WebSocket이 끊기는 상황에 대비해서 아까 만들어둔 1초 주기 캐시 데이터를 **fallback**(안전망)으로도 활용하고 있다. WebSocket이 정상이면 실시간 콜백으로 처리하고, 장애가 발생하면 캐시된 REST 데이터로 매매가 이어지도록 한 것이다. 하나의 캐시가 대시보드와 안전망 두 가지 역할을 하는 셈이다.

<br>

정리하면 이렇다.

<br>

| 방식 | 용도 | 동작 |
|------|------|------|
| **REST 폴링 + 캐시** | 대시보드 코인 목록 + fallback | 서버에서 1초마다 1건 호출, USDT 필터링 + 거래량 정렬 후 캐시 |
| **WebSocket** | 자동매매 가격 감지 | 활성화된 코인만 구독, 가격 변동 시 매매 엔진에 콜백 |

<br>

## 차트 없는 거래소가 어디 있어

<br>

데이터를 안정적으로 뿌려주는 로직을 완성했으니, 아까 요구사항에 있던 "특정 코인을 클릭하면 상세 차트를 확인할 수 있어야 한다"를 구현할 차례다. 대시보드에서 코인을 클릭하면 해당 코인의 캔들스틱 차트를 볼 수 있어야 한다. 실제 거래소처럼 **시가(Open), 고가(High), 저가(Low), 종가(Close)** 네 가지 가격 정보를 시각적으로 보여주는 OHLC 차트 말이다.

<br>

솔직히 말하면 차트 라이브러리는 잘 모른다. 그래서 이 부분은 통째로 Claude에게 위임했다. 나는 **"코인 상세 차트 페이지를 만들어줘. 캔들스틱 차트로 시간 간격 선택도 가능하게 해줘."** 라고 지시하고, Claude가 라이브러리 선정부터 구현까지 알아서 해왔다.

<br>

Claude가 선택한 라이브러리는 **ApexCharts**였다. CDN 한 줄이면 바로 사용할 수 있고 캔들스틱 차트 지원이 잘 되어 있다고 했다. 나는 차트가 뜨는지, 데이터가 제대로 찍히는지만 확인했다.

<br>

차트에 뿌릴 데이터는 Bybit의 **Kline(K-라인)** API에서 가져온다. K-라인이란 특정 시간 간격(1분, 5분, 15분 등)마다의 시가·고가·저가·종가·거래량 정보를 말한다.(아직도 코인 도메인은 어렵다니까....)

<br>

5편에서 만들어둔 `MarketClient`에 Kline 조회용 엔드포인트를 추가했다.

<br>

```java
@GetMapping("/v5/market/kline")
ResponseEntity<GetKlineResponse> getKline(
        @RequestParam("category") String category,
        @RequestParam("symbol") String symbol,
        @RequestParam("interval") String interval,
        @RequestParam("limit") int limit);
```

<br>

`interval`에 `"1"`, `"5"`, `"15"`, `"30"`, `"60"` 등을 넘기면 해당 간격의 캔들 데이터를 최대 200개까지 가져올 수 있다. 응답은 `GetKlineResponse`로 받는다.

<br>

```java
@Getter
public class GetKlineResponse extends BybitMasterDTO {
    private Result result;

    @Getter
    public static class Result {
        private String symbol;
        private String category;
        private List<List<String>> list;    // [startTime, open, high, low, close, volume, turnover]
    }
}
```

<br>

`list`가 `List<List<String>>` 형태인 이유는, Bybit API가 캔들 데이터를 배열의 배열로 내려주기 때문이다. 각 배열의 순서가 곧 `[시작시간, 시가, 고가, 저가, 종가, 거래량, 거래대금]`이다.

<br>

`MarketService`에서는 이 API를 호출하는 메서드를 간단하게 작성했다.

<br>

```java
public GetKlineResponse getKline(String symbol, String interval) {
    return marketClient.getKline(Category.LINEAR.getCategory(), symbol, interval, 200).getBody();
}
```

<br>

그리고 컨트롤러에서 차트 페이지와 API 엔드포인트를 노출한다.

<br>

```java
@GetMapping("/chart")
public String chartPage(@RequestParam String symbol, Model model) {
    model.addAttribute("symbol", symbol);
    return "home/chart";
}

@GetMapping("/api/kline")
@ResponseBody
public GetKlineResponse getKlineData(@RequestParam String symbol, @RequestParam String interval) {
    return marketService.getKline(symbol, interval);
}
```

<br>

대시보드에서 코인을 클릭하면 `/chart?symbol=BTCUSDT` 같은 형태로 이동하고, 차트 페이지에서는 `/api/kline`을 호출해서 데이터를 가져오는 구조다. 여기까지가 내가 직접 챙긴 백엔드 영역이다.

<br>

### 프론트는 Claude에게

<br>

프론트 쪽은 Claude가 알아서 구현해왔다. 내가 확인한 핵심 동작만 정리하면 이렇다.

<br>

- Bybit API에서 받아온 Kline 데이터를 ApexCharts가 이해할 수 있는 `{x: Date, y: [open, high, low, close]}` 형태로 변환
- 1분, 5분, 15분, 30분, 1시간 간격을 버튼으로 선택 가능
- **1초마다 차트 데이터를 갱신**하되, 사용자가 줌을 해놓은 상태는 유지
- 한국어 로케일 적용 (툴바, 날짜 포맷 등)

<br>

차트 아래에는 **호가창(Order Book)**도 붙여두었다. 매도 호가는 빨간색, 매수 호가는 초록색으로 표시해서 한눈에 매도·매수 압력을 파악할 수 있게 했다. 호가창 데이터도 Bybit API에서 가져온다.

<br>

```java
public OrderBookResponse getOrderBook(String symbol) {
    return marketClient.getOrderBook(Category.LINEAR.getCategory(), symbol, 50).getBody();
}
```

<br>

매수·매도 각각 상위 50개 호가를 가져오고, 프론트에서는 그 중 가장 좋은 8개씩만 화면에 표시한다.

<br>

Claude가 가져온 결과물을 실행해보니 차트도 잘 뜨고, 시간 간격 변경이나 줌도 정상적으로 동작했다. 데이터가 실제 Bybit 거래소와 일치하는지 교차 확인도 했다.

<br>

그런데 차트만으로 끝이 아니었다. 이 프로젝트의 핵심은 **자동매매**이고, 차트 페이지 옆에는 자동매매를 위한 **패턴 관리 큐**를 등록하는 UI가 필요했다. 이것도 Claude에게 맡겼다.

<br>

내가 설명한 건 이 정도였다.

<br>

> "차트 옆에 자동매매 큐를 등록하는 패널을 만들어줘. Long은 초록색, Short은 빨간색 블록으로 표현하고, 블록들을 조합해서 패턴을 만들고, 패턴들이 모여서 단계가 되고, 단계들이 모여서 하나의 큐가 완성되는 구조야."

<br>

아 물론, 내가 원하는 결과물이 완전히 나올때까지 Claude를 괴롭히긴 했지만...

<br>

어쨌든 Claude가 가져온 결과물은 꽤 괜찮았다. **L**(Long) 블록은 초록색, **S**(Short) 블록은 빨간색으로 시각적으로 구분되고, 조건 블록과 리프 블록을 콜론(`:`)으로 구분해서 패턴을 구성할 수 있게 만들어왔다. 각 패턴에는 금액, 레버리지, 손절률, 익절률까지 설정할 수 있다.

<br>

그리고 큐에는 **트리거** 설정도 달아두었다. 트리거란 자동매매가 시작되는 조건을 말하는데, **기준 시간(초)**과 **상승/하락률(%)**을 설정하면 해당 시간 내에 설정한 비율만큼 가격이 변동했을 때 큐가 동작하는 구조다. 예를 들어 `60초 / 1.00%`로 설정하면, 60초 안에 가격이 1% 이상 움직였을 때 자동매매가 트리거되는 것이다.

<br>

구조를 정리하면 이렇다.

<br>

| 단위 | 설명 |
|------|------|
| **트리거** | 큐가 동작하는 조건 (기준 시간 + 상승/하락률) |
| **블록** | L(Long) 또는 S(Short), 조건 블록은 최대 5개 + 리프 블록 1개 |
| **패턴** | 블록 조합 + 금액/레버리지/손절/익절 설정, 단계당 최대 2개 |
| **단계** | 패턴 1~2개로 구성, 큐당 최대 20개 |
| **큐** | 트리거 조건 + 단계들의 집합 |

<br>

블록 → 패턴 → 단계 → 큐(+트리거), 이렇게 작은 단위부터 쌓아 올리는 구조다. 차트부터 호가창, 큐 등록 UI까지 내가 구조와 요구사항만 설명하면 Claude가 구현해오고 나는 검수하는 흐름이 꽤 잘 맞았다. 솔직히 프론트 쪽은 내가 직접 짰으면 훨씬 오래 걸렸을 것이다. 이럴 때 AI의 도움이 진짜 체감된다.

<br>

그래서 완성된 결과물을 보면 다음과 같다!

<br>

<img src="/assets/img/coin-auto-trading-06/coin-chart.png">

---

<br>

## 마무리

<br>

이번 편에서는 꽤 많은 걸 다뤘다. 대시보드의 코인 목록을 1초마다 갱신하면서도 Rate Limit에 걸리지 않도록 서버 캐시를 도입하고, 자동매매를 위해서는 WebSocket으로 실시간 가격을 감지하는 구조를 잡았다. 그리고 차트와 호가창, 큐 등록 UI까지 Claude에게 맡기면서 프론트 영역도 빠르게 완성할 수 있었다.

<br>

돌이켜보면 이번 편이 "개발자 + AI" 협업의 장점이 가장 잘 드러난 구간이었다. 백엔드 설계와 데이터 흐름은 내가 직접 챙기고, 프론트 구현처럼 내가 약한 영역은 Claude에게 위임하되 결과물을 검수하는 방식. 이게 지금까지 찾은 가장 효율적인 협업 방식인 것 같다.

<br>

다음 편에서는 이렇게 만들어둔 큐가 실제로 어떻게 동작하는지, 자동매매 엔진의 핵심 로직을 다뤄볼 예정이다!

<br>

> **쿠키 🍪**
>
> 차트 페이지를 처음 띄웠을 때 캔들이 하나도 안 그려져서 한참 헤맸는데, 알고 보니 Bybit API가 최신 데이터부터 내려주는 걸 `.reverse()` 안 해서 시간축이 뒤집혀 있었다. Claude한테 "차트 데이터가 거꾸로 나온다"고 했더니 바로 고쳐왔다. 역시 에러 메시지보다 증상을 말해주는 게 AI한테도 통한다.


---
layout: post
title: "코인자동매매시스템(with. claude) - 2"
date: 2026-02-28 00:00:00 +0900
categories: [개발]
excerpt: "개발자라는 이유 하나로 코인 자동매매 시스템을 만들게 된 이야기"
published: true
---

# 개발자의 꽃 DB 설계하기

<br>

[1편 보러가기](https://mhs442.github.io/2026-02-23/coin-auto-trading-01)

이 프로젝트의 엔터티 도출까지 전편에서 이야기했는데 이제 데이터베이스를 설계할 차례다.

## ERD 툴을 선택해볼까?

ERD를 그리기 위해, 전 회사에서 사용했던 **IntelliJ 의 ERD Editor** 플러그인을 활용해 ERD를 그리려 했는데 치명적인 버그가 발생하고 있었다.

<br>

<img src="/assets/img/coin-auto-trading-02/ERD-editor-error.png">

<br>

컬럼을 작성하고 컬럼의 위치를 수정하려 하면 갑자기 먹통이 되어버리는 모습! 

<br>

이게 IntelliJ 버전 2025.01 버전으로 올라가면서 발생한 버그라고 하는데...
해당 플러그인 개발자가 버그는 인지하고 있다고 하는데 고칠 생각이 없어 보인다. 다른 플러그인을 찾던가 해야지..

<br> 

그래서! 가장 만만한 **ERD Cloud**를 이용해서 그려보자. 그럼 논리적 설계를 먼저 해보자.

(세세한 내용은 제외하고 큰 틀만.. 다 적기에는 너무 양이 방대하다.)

<br>

## 논리 설계부터

이전에 도출한 엔터티는 다음과 같았다.

<br>

```
USERS            → 사용자 정보
PATTERN_QUEUE    → 패턴 큐
PATTERN_STEP     → 패턴 단계
PATTERN          → 패턴
PATTERN_BLOCK    → 패턴 블록
TRADE_HISTORY    → 패턴 단계 히스토리 - 스냅샷 형태
PATTERN_HISTORY  → 패턴 단계 히스토리 - 스냅샷 형태
```

<br>

우선 로그인은 휴대폰 번호를 이용한 로그인을 만들려 한다. 어차피 프로토타입 개발 형태이기 때문에 나중에 휴대폰이 아닌 다른 로그인 방식을 이용하면 된다.
(OAuth2.0 을 사용할 수도..?)

<br>

전 포스팅에는 적지 않았지만 해당 프로그램을 본인만 사용하지 않을수도 있다고 하여.. 로그인 기능을 구현해서 사용자 별 정보를 미리 나눠두는게 맞다고 판단했다.

<br>

추가적으로 필요한 정보들을 그냥 구상만 맞춰두고 나중에 필요없는 데이터를 제거하면 된다고 결론 내리고 작업을 하는 쪽으로 판단했다.

<br>

| USERS      |
|------------|
| 사용자 id     |
| 휴대폰번호      |
| 이메일        |
| 패스워드       |
| 사용자 이름     |
| api_key    |
| api_secret |

<br> 

USERS 엔터티는 위 와 같은 정보를 넣자고 판단! api_key와 api_secret 같은 경우는 ByBit에서 로그인을 통해 사용자 정보(거래 지갑)를 가져와서 사용하는데...

<br>

API로는 로그인 기능을 제공하지 않는다고 한다. 대신, api_key를 사용자가 발급받아서 사용할 수 있다고 하니 회원가입 할때 API키 값을 받아서 넣으면 될 것으로 보인다.

<br>

이처럼 각 엔터티 별로 속성을 정해주고 식별자는 전부 인조 식별자를 쓰기로 했다. 개인 토이프로젝트 수준이니까 관리하기 편하니까..

<br>

## Claude와 함께

자, 여기부터 이제 Claude의 도움을 받기로 했다. IntelliJ 에서 Claude 사용을 위한 플러그인을 제공하는데.. 나는 플러그인으로 쓰기가 싫다.

IntelliJ에 플러그인이 늘어날수록 확실히 느려지는 것을 매우 체감하기에 그냥 cli 방식으로 사용하기로 했다.

<br>

사용법은 매우 간단하다. 아래와 같이 터미널을 열고 claude 명령어를 치면 끝!

<img src="/assets/img/coin-auto-trading-02/claude-intro.png"/>

<br>

명령어를 치면 귀여운 Claude 캐릭터와 함께 현재 프로젝트 경로가 나온다!

<br>

<img src="/assets/img/coin-auto-trading-02/claude-terminal.png"/>

<br>

모델은 그 유명한 Opus 모델을 사용해서 그 강력함을 느껴보기로 했다. (사실 이미 좀 도움을 받은 편..)

그리고 본인의 실행 경로가 나오는데 이 경로가 현재 프로젝트 위치와 동일하면 된다. 그리고 플러그인도 있다고 이야기하는데 어림도 없지. 절대 안 쓸 거다.


아무튼, 사실 엔터티를 도출하고 ERD를 설계하는데에 있어서 이미 Claude의 도움을 받았다. 이녀석 정말 똑똑하다.

내가 설계한 부분에서 모호하거나 모자란 부분을 채워주기에는 부족함이 없었다. 근데 한번씩 허당끼가 있다고 해야할까. 완전히 맡기기에는 무리가 있어 보였다.

<br>

예를 들면, 도출한 엔터티 중에 TRADE_HISTORY 테이블이 존재하는데 이건 사용자별 등록한 자동매매시스템을 통해 거래된 내역을 저장하기 위한 테이블이다.

데이터가 많이 쌓일것이 자명하기에 인덱스 설계를 잘 해야했는데 대충 ERD를 첨부해보면 다음과 같다.

<br>

<img src="/assets/img/coin-auto-trading-02/trade-history-table-erd.png"/>

<br>

위 테이블 정보를 기반으로 아래와 같은 인덱스를 설계했다.

1. 로그인한 사용자의 정보만 보여야 한다 -> user_id 인덱스
2. 특정 코인만 검색가능해야 한다 -> symbol 인덱스
3. 날짜를 기준으로 정렬이나 이후, 이전 데이터를 검색할 수 있어야 한다 -> created_at 인덱스
    - (user_id, created_at)           복합 인덱스 - 사용자별 날짜 조회
    - (user_id, symbol, created_at)   복합 인덱스 - 사용자별 코인 + 날짜 조회
* side, order_result 은 카디널리티가 낮아(2가지 값) 단독 인덱스 제외

<br>

이렇게 설계를 했는데 갑자기 이녀석이 (user_id, created_at) 복합 인덱스는 필요없다고 하는것이 아닌가!
(user_id, symbol, created_at) 복합 인덱스로 사용자 + 거래날짜 조회까지 가능하다고 했는데 말이 안됐다.

<br>

**왜..필요가 없다고 하지?**

<br>

복합 인덱스는 순서가 중요한데 (user_id, symbol, created_at) 인덱스는 user_id 색인 -> symbol 색인 -> created_at 순서로 색인이 되기 때문에 중간에 symbol 없이 조회를 하면 해당 인덱스 활용이 어렵다.

<br>

그래서 역으로 물어봤다. (캡처를 못한 게 한이네..)

<br>

> user_id + created_at 검색은 인덱스 활용이 어렵지 않아?

<br>

그랬더니 죄송하다고 사과를 하는것이 아닌가! 자기가 잘못 생각했다고.. 역시 온전히 맡기기에는 불안하다

하지만 감독만 잘 하면 이 친구를 활용하여 빠르게 개발이 가능할 것 같다.

<br> 

아, 그 전에 나는 Claude에게 사전에 알아야 할 내용들에 대해서 알려주었다. 프로젝트 패키지 최상단에 파일을 작성해서 다음과 같이 적었다.

``` text
BackgroundForClaude.txt

1. git에 commit, push 는 절대 하지 않는다. (직접 내가 할거야)
2. 특정 작업을 요청하면 네가 잘 이해했는지 확인하고 싶어. 그러니, 내가 작업을 요청하면 요약해서 나에게 확인받고 작업을 진행해.
3. 작업을 할 때에는 작업 계획을 먼저 세우고 나에게 확인 받을 것.
4. 파일을 수정 할때에는 수정 전과 후의 상태를 나에게 알려주고 수정 할지 말지 나에게 확인을 필수로 받아.
5. API 응답을 통해 받는 데이터는 Map객체 사용은 지양하고 Mapping할 객체를 직접 만들어서 작업해줘.
6. 요청에 필요한 객체는 ~Request, 응답에 필요한 객체는 ~Response 와 같은 형태로 통일해서 만들어줘
7. TDD 방식으로 개발을 할거야. 특정 비즈니스 로직을 작성해야 한다면 실패하는 테스트 코드를 먼저 만들고 작업해줘.
8. Controller와, Repository 레이어는 통합 테스트, 비즈니스로직인 Service 레이어는 단위 테스트로 작성하도록 해.
9. 테스트코드의 API 요청에 대한 응답은 WireMock을 이용해 모의하고 응답 데이터 모의는 파일로 관리할건데
    test.resources 패키지 하위 __file 이라는 디렉터리를 만들어서 모의할 OpenFeign을 구현한 파일명_모의할객체와 같은 형태로 파일을 생성해서 모의하도록 해.
    예를 들어, MarketClient 파일의 getTickers를 모의하는 파일명은 다음과 같아. MarketClient_getTickers.json
10. 작업시 주요 부분(분기처리, 데이터 파싱 등..)에는 항상 주석을 달아줘.
11. Service 레이어와 Client 레이어에 JavaDoc 형태의 주석을 꼭 달아줘. 파라미터와 응답값들에 대해 기술하도록 해줘.
    Client 레이어에서는 해당 주석을 보고 API 명세 역할을 대신 하도록 할거야.
12. 생성한 객체의 필드값들에 모두 간단한 주석을 달아줘.(인라인 주석 형태로)
13. 때에 따라 새롭게 추가되는 규칙이 있을것. (이건 규칙이 추가되면 알려줄게.)
```

<br>

위와 같은 파일을 만들어 Claude에게 입력시키고 해당 방식을 철저하게 따라서 작업하라고 요청했다.

아마 세션이 유지되고 메모리에 해당 데이터가 유지되는 한 잘 따라서 할 것이다. (지금까지는..)

<br>

## 물리 설계까지

자, 그래서 아주 똑똑한 조수(Claude)와 함께 데이터베이스를 설계했더니 다음과 같은 ddl을 뽑았다.

<br>

``` mariadb
-- =============================================
-- Web Coin Trader - DDL
-- =============================================

-- 1. users (사용자 정보)
-- =============================================
CREATE TABLE users (
                       id            BIGINT        NOT NULL AUTO_INCREMENT  COMMENT '사용자 id',
                       username      VARCHAR(20)   NOT NULL                 COMMENT '사용자 이름',
                       phone_number  VARCHAR(20)   NOT NULL                 COMMENT '휴대폰 번호',
                       email         VARCHAR(50)   NOT NULL                 COMMENT '이메일',
                       password      VARCHAR(200)  NOT NULL                 COMMENT '패스워드',
                       api_key       VARCHAR(500)  NOT NULL                 COMMENT 'Bybit API Key (AES 암호화)',
                       api_secret    VARCHAR(500)  NOT NULL                 COMMENT 'Bybit API Secret (AES 암호화)',
                       PRIMARY KEY (id),
                       CONSTRAINT uq_users_phone UNIQUE (phone_number),
                       CONSTRAINT uq_users_email UNIQUE (email)
);

-- 2. pattern_queue (패턴 큐)
-- =============================================
CREATE TABLE pattern_queue (
                               id                  BIGINT        NOT NULL AUTO_INCREMENT  COMMENT '패턴 큐 id',
                               user_id             BIGINT        NOT NULL                 COMMENT '사용자 id',
                               symbol              VARCHAR(100)  NOT NULL                 COMMENT '코인 종류',
                               use_yn              VARCHAR(2)    NOT NULL DEFAULT 'N'     COMMENT '활성화 여부 (Y/N)',
                               activated_at        DATETIME      NULL                     COMMENT '활성화 시점',
                               created_at          DATETIME      NOT NULL                 COMMENT '생성일',
                               updated_at          DATETIME      NULL                     COMMENT '수정일',
                               trigger_seconds     INT           NOT NULL                 COMMENT '기준 시간(초)',
                               trigger_rate        DECIMAL(5,2)  NOT NULL                 COMMENT '기준 비율(%)',
                               is_full             BOOLEAN       NOT NULL DEFAULT FALSE   COMMENT '패턴 단계 최대 여부',
                               current_step_id     BIGINT        NULL                     COMMENT '현재 진행중 단계 id (FK 없음)',
                               current_pattern_id  BIGINT        NULL                     COMMENT '현재 진행중인 패턴 id (FK 없음)',
                               current_block_order INT           NULL                     COMMENT '현재 진행중인 블록 순서',
                               PRIMARY KEY (id),
                               CONSTRAINT fk_pattern_queue_user FOREIGN KEY (user_id) REFERENCES users (id)
);

CREATE INDEX idx_pattern_queue_user_symbol ON pattern_queue (user_id, symbol);

-- 3. pattern_step (패턴별 단계)
-- =============================================
CREATE TABLE pattern_step (
                              id          BIGINT   NOT NULL AUTO_INCREMENT  COMMENT '패턴별 단계 id',
                              queue_id    BIGINT   NOT NULL                 COMMENT '패턴 큐 id',
                              step_level  INT      NOT NULL                 COMMENT '단계 레벨 (최대 20)',
                              is_full     BOOLEAN  NOT NULL DEFAULT FALSE   COMMENT '패턴 최대 여부',
                              PRIMARY KEY (id),
                              CONSTRAINT fk_pattern_step_queue FOREIGN KEY (queue_id) REFERENCES pattern_queue (id)
);

CREATE INDEX idx_pattern_step_queue_level ON pattern_step (queue_id, step_level);

-- 4. pattern (패턴)
-- =============================================
CREATE TABLE pattern (
                         id                BIGINT        NOT NULL AUTO_INCREMENT  COMMENT '패턴 id',
                         step_id           BIGINT        NOT NULL                 COMMENT '패턴별 단계 id',
                         leverage          INT           NOT NULL                 COMMENT '레버리지',
                         stop_loss_rate    DECIMAL(5,2)  NULL                     COMMENT '손절 기준 비율',
                         take_profit_rate  DECIMAL(5,2)  NULL                     COMMENT '익절 기준 비율',
                         amount            DECIMAL(10,2) NULL                     COMMENT '투자 금액',
                         pattern_order     INT           NOT NULL                 COMMENT '패턴 순서 (1 or 2)',
                         PRIMARY KEY (id),
                         CONSTRAINT fk_pattern_step FOREIGN KEY (step_id) REFERENCES pattern_step (id)
);

CREATE INDEX idx_pattern_step_order ON pattern (step_id, pattern_order);

-- 5. pattern_block (패턴 블록)
-- =============================================
CREATE TABLE pattern_block (
                               id           BIGINT      NOT NULL AUTO_INCREMENT  COMMENT '패턴 블록 id',
                               pattern_id   BIGINT      NOT NULL                 COMMENT '패턴 id',
                               side         VARCHAR(10) NOT NULL                 COMMENT '투자 방향 (LONG / SHORT)',
                               block_order  INT         NOT NULL                 COMMENT '블록 순서 (최대 6)',
                               is_leaf      BOOLEAN     NOT NULL DEFAULT FALSE   COMMENT '마지막 블록 여부',
                               PRIMARY KEY (id),
                               CONSTRAINT fk_pattern_block_pattern FOREIGN KEY (pattern_id) REFERENCES pattern (id)
);

CREATE INDEX idx_pattern_block_order ON pattern_block (pattern_id, block_order);

-- 6. trade_history (거래 히스토리)
--    스냅샷 방식: 원본 삭제와 무관하게 거래 기록 보존, FK 없음
-- =============================================
CREATE TABLE trade_history (
                               id              BIGINT        NOT NULL AUTO_INCREMENT  COMMENT '거래 히스토리 id',
                               queue_step_id   BIGINT        NOT NULL                 COMMENT '패턴별 단계 id (참조용, FK 없음)',
                               user_id         BIGINT        NOT NULL                 COMMENT '사용자 id (스냅샷)',
                               symbol          VARCHAR(100)  NOT NULL                 COMMENT '코인 종류 (스냅샷)',
                               side            VARCHAR(10)   NOT NULL                 COMMENT '체결 방향 (LONG / SHORT)',
                               executed_price  DECIMAL(20,8) NOT NULL                 COMMENT '체결 가격',
                               order_id        VARCHAR(100)  NULL                     COMMENT 'ByBit 주문 id',
                               created_at      DATETIME      NOT NULL                 COMMENT '체결 일시',
                               order_result    VARCHAR(10)   NOT NULL                 COMMENT '주문 결과 (SUCCESS / FAILED)',
                               error_message   TEXT          NULL                     COMMENT '실패 사유',
                               PRIMARY KEY (id)
);

CREATE INDEX idx_trade_user_date ON trade_history (user_id, created_at);
CREATE INDEX idx_trade_user_symbol_date ON trade_history (user_id, symbol, created_at);

-- 7. pattern_history (패턴 단계 히스토리)
--    스냅샷 방식: FK 없음
-- =============================================
CREATE TABLE pattern_history (
                                 id               BIGINT        NOT NULL AUTO_INCREMENT  COMMENT '패턴 단계 히스토리 id',
                                 pattern_step_id  BIGINT        NOT NULL                 COMMENT '패턴 단계 id (참조용, FK 없음)',
                                 user_id          BIGINT        NOT NULL                 COMMENT '사용자 id (스냅샷)',
                                 symbol           VARCHAR(100)  NOT NULL                 COMMENT '코인 종류 (스냅샷)',
                                 action           VARCHAR(10)   NOT NULL                 COMMENT '등록 종류 (create / update / delete)',
                                 created_at       DATETIME      NOT NULL                 COMMENT '등록일',
                                 block_list       TEXT          NOT NULL                 COMMENT '패턴 단계 블록 집합 (JSON)',
                                 PRIMARY KEY (id)
);

CREATE INDEX idx_pattern_history_user_symbol ON pattern_history (user_id, symbol);
CREATE INDEX idx_pattern_history_user_date ON pattern_history (user_id, created_at);

```

<br>

테이블 정보가 좀 적어보이는데 실제로 데이터는 ByBit 서버에서 가져와서 처리할 수 있는 내용들이 많아서 최대한 그 데이터를 활용하기로 했다.

<br>

또한 프로토타입 모델로 개발을 진행할 예정이기 때문에 요구사항이 자주 바뀔 수 있어 최대한 테이블을 적게 두고 개발이 진행될수록 필요한 부분에 대해서는 추가하도록 할 예정이다.

<br>
<br>

## 마치며

DB 설계는 개발의 기초 공사와 같다. 여기서 대충 넘어가면 나중에 반드시 되돌아오게 되어 있다. Claude의 도움을 받으면서도 결국 검증은 사람의 몫이라는 걸 다시 한번 느꼈다.

다음 글에서는 기술 스택 선정과 프로젝트 구조 설계 과정을 다뤄볼 예정이다.

<br>

> **쿠키 🍪**
>
> 이 글을 작성하고 검수하는 과정에서 알게 된 건데, 프로젝트 루트에 `CLAUDE.md` 라는 파일을 만들어두면 Claude Code가 실행될 때 자동으로 해당 파일을 인식한다고 한다. 즉, 내가 했던 것처럼 별도의 txt 파일을 만들어서 "이 파일 읽어" 라고 지시할 필요 없이, `CLAUDE.md`에 규칙을 적어두기만 하면 알아서 읽어간다는 것이다.
>
> 그래서 `BackgroundForClaude.txt` 대신 `CLAUDE.md`로 바꾸려고 했는데 해당 파일은 협업하는 다른 개발자들이 있을 시, git 파일로 관리해서 Claude 설정을 통일해서 사용할때 사용하기도 한다고 한다.
> 
> 나만의 규칙을 사용하고 싶다면 CLAUDE.local.md 라는 파일로 사용해도 자동으로 읽는다고 하니 해당 파일로 변경해서 사용할 것이다.



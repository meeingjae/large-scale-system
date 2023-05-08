# 10장 - 알림 시스템 설계

-- -- 

알림 시스템은 최근 많은 프로그램이 채택한 인기 있는 기능이다

서버는 고객에게 중요할 만한 정보를 비동기적으로 제공한다

## 검토 사항

* 알림 종류는?
    * App 푸시 알림
    * SMS 메세지
    * Email
* 실시간 시스템 이어야 하는가?
    * 실시간
    * 연성 실시간 (soft real-time)
        * 데이터는 최대한 빨리 전달되어야 한다. 하지만 부하 상황에서 약간의 딜레이는 허용한다
* 알람 시스템의 endpoint는?
    * iOS
    * Android
    * 랩탑
    * 데스크탑
* 알림 생성자는?
    * client application program
    * server-side scheduling
* opt-out 지원 여부? (opt-out : 앤드유저가 스스로 알람을 수신하지 않도록 설정하는 것)
* 하루 몇 건의 알람이 발송되어야 하는가?
    * 1000만 건의 Mobile Push
    * 100만 건의 SMS
    * 500만 건의 Email

## 알림 유형벌 지원 방안

### iOS 푸시 알림

* 알림 제공자
    * 알림 요청을 만들어 APNS에 푸시 요청을 보내는 주체
        * 알림 요청에 포함되는 데이터
            * device token (단말 토큰)
                * 알람 고유 식별자
            * payload
                * 알림 내용을 담은 JSON 데이터
                * example
              ```
              {
                "aps": {
                  "alert": {
                    "title": "this is title !",
                    "body": "please read. this is message body",
                    "action-loc-key": "PLAY"
                    },
                  "badge": 5
                }
              } 
              ```
* APNS (Apple Push Notification service)
    * 애플이 제공하는 원격 서비스
    * 푸시 알림을 device에 전달하는 역할
* iOS 단말

### Android 푸시 알림

* iOS 푸시 알림과 유사하다
* APNS 대신 FCM (Firebase Cloud Messaging) 을 사용한다

### SMS 메세지

* 보통 제 3자 서비스를 사용한다
    * Twilio
    * Nexmo
    * 등등

### 이메일 메세지

* 보통은 SMTP를 사용한 자사 이메일 서버를 구축해서 사용한다
* 전송률도 높고, 데이터 분석이 가능한 제 3자 서비스를 이용하기도 한다
    * Sendgrid
    * Mailchimp

### 연락처 정보 수집 절차

* 알림을 보내려면 모바일 단말 토큰, 전화번호, 이메일 주소 등의 정보가 필요하다
* 사용자가 앱을 설치 및 등록할때 정보를 받아 DB에 저장한다
* 사용자 정보를 담는 테이블과 device 정보를 담는 테이블은 1:N 으로 설계한다
    * 하나의 사용자가 Email, Mobile 등 여러 Device를 사용할 수 있다

### 알림 전송 및 수신 절차

1. N개의 API endpoint 서비스
    * 각각의 마이크로서비스
    * 혹은 크론잡(cronjob)
    * 혹은 분산 시스템 컴포넌트
2. 알림 시스템
    * 알림 서버 + 작업 서버로 구현 가능
    * 알림 전송/수신 처리의 핵심
    * 1 ~ N 개의 알림 전송을 위한 API 제공
    * 제3자 서비스에 전달한 payload 생성
3. 제3자 서비스
    * 앤드유저에게 알림을 실제로 발송/전달 하는 역할
    * 확장성에 유의해야 한다
    * 시장에 따라 서비스를 선택해야 할 수도 있다
        * 예를들면 중국에선 FCM 서비스를 사용할 수 없어 Jpsuh, PushY와 같은 서비스로 대체해야 할 수도 있다
4. 앤드유저 device
    * 제3자 서비스에서 전달한 알림을 수신한다

## 알림 서버(알림 전처리 서버) + 작업 서버(완성 된 알림을 제3자 서비스에 전달하는 서버)

### 알림 서버

* 알림 전송 API
    * 사내 스팸 방지 서비스로 알림 내용을 필터링 할 수 있다
    * 클라이언트 인증이 필요하다면 알림 서버에서 한다
* 알림 검증
    * 이메일, 전화번호에 대한 기본적 검증을 수행한다
        * 클라이언트에서 할 수도 있다
* DB or Cache
    * 알림에 포함시킬 데이터를 가져온다
* 알림 전송
    * 알림 데이터를 메세지 큐에 넣는다
    * 해당 메세지 큐의 subscriber는 작업 서버 이다
* Cache
    * User 정보
    * Device 정보
    * 알림 템플릿 정보
* 메세지 큐
    * 작업 서버에 알림을 전달하기 위해 메세지 큘르 사용한다
    * 시스템 컴포넌트 간 의존성을 제거할 수 있다. (시스템 간 약한 결합)
    * 특정 제3자 시스템이 마비되었을 때, 다른 제3자 시스템에 영향을 미치지 않는다
    * Fail count로 retry Controll이 가능하다
* 적용 가능한ㄷ Design Pattern
    * Strategy Pattern (전략 패턴)
    * Observer Pattern (옵저버 패턴)
    * Command Pattern (명령 패턴)
    * Factory Pattern (팩토리 패턴)

### 작업 서버

* 알림 서버가 publish한 알림 이벤트를 subscribe한다
* 알림 이벤트를 제3자 서비스로 전달(중개) 하는 역할을 한다

### 알림 서버 + 작업 서버 의 전송 단계

1. N개의 Microservice 혹은 그 어떤 기능에서 알림 서버로 요청 데이터를 보낸다
2. 알림 서버는 DB 혹은 Cache에서 사용자 정보, Device 정보, 알림 메타 데이터를 가져온다
3. 알림 서버는 전송할 알림의 type에 맞는 이벤트를 생성하여, 각각의 큐에 Publish 한다
4. 작업 서버는 알림 서버가 발행한 이벤트를 subscribe 한다
5. 작업 서버는 알림을 제 3자 서비스로 전달한다
6. 제3자 서비스는 알림을 앤드유저에게 전달한다

## 안정성

### 데이터 손실 방지

* 알림이 소실되면 안되는 상황이 있다
* 위 상황을 해결하려면 알림 데이터를 DB에 보관하고, 재시도 매커니즘을 구현해야 한다
    * 알림 로그를 DB에 유지하는 것도 하나의 구현 방법이다

### 알림 중복 전송 방지

* 같은 알림이 여러 번 반복되는 것을 완전히 막는 것은 불가능하다
    * 분산 시스템 특성 상 가끔은 알림 중복 현상이 발생할 수 있다
* 중복 탐지 메커니즘을 도입하여 중복 전송 빈도를 줄일 수 있다
    * 알림이 도착하면 Event ID를 검사하여, 이미 요청이 있었던 알림인 지 검사한다
    * 중복 이벤트라면 버리고, 아니면 통과시킨다

## 추가 고려사항

### 알림 템플릿

* 알림 메세지의 형식은 대부분 비슷하기에, 이러한 유사성을 고려하여 알림 내용을 템플릿화 한다
* 지정된 형식에 parameter나 tracking link와 같은 데이터를 조정하기 위한 틀이다
* ex
    * $\[Item\] 상품이 출시되었습니다. $\[Link\] 로 접속하세요 !
* 전송 될 알림들을 일관성 있게 유지가 가능하다
* 오류 가능성 및 알림 작성에 드는 시간을 줄일 수 있다

### 알림 설정

* 앤드 유저가 알림의 빈도수를 조절할 수 있도록 옵션을 제공해야 한다
* 유저 정보, 알림 채널, 알림 수신 여부 등을 담고 있는 테이블로 해당 데이터를 관리해야 할 수도 있다
* 위 테이블을 기반으로 사용자가 알림 설정을 켜 두었는지 확인해야 한다

### 전송률 제한

* 사용자가 알림의 빈도가 너무 높다고 느끼는 경우 알림 설정을 아예 꺼버릴 수 있다
* 위와 같은 상황이 발생하지 않게 처리율 제한 장치를 두는 것도 한가지 방법이다

### 알림 전송 재시도 방법

* 제3자 서비스가 알림 전송에 실패하면 해당 알림을 재시도 전용 큐에 넣는다
* 같은 문제가 지속될 경우 서비스 내부 alert를 발행할 수 있다

### 푸시 알림과 보안

* iOS와 Android의 알림 전송 API는 appKey와 appSecret을 사용하여 보안을 유지한다
* 인증 되고, 승인 된 클라이언트만이 알림 전송 서비스를 이용할 수 있다

### 큐 모니터링

* 알림 전송 작업 큐에 대기중인 이벤트가 많을수록 병목의 가능성을 의심해보아야 한다
* 위와 같은 상황일 때, 수평적 규모 확장이 대안이 될 수 있다

### 이벤트 추적

* 알림 확인율, 클릭율, 앱 사용으로 이어지는 비율 과 같은 매트릭은 중요한 데이터이다
* 알림 서비스를 데이터 분석 서비스와 통합해야 한다







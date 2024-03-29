# 12장 - 채팅 시스템 설계

-- -- 

## 채팅 시스템 요구사항

* 응답 지연이 낮은 1:1 채팅 기능
* 최대 100명까지 참여할 수 있는 그룹 채팅 기능
* 사용자의 접속 상태 표시 기능
* 다양한 단말 지원, 하나의 계정으로 여러 단말 동시 접속 지원
* Push 알림

## Client - Server 통신에서의 요구사항

1) Client -> Server 메세지 수신
2) 메세지 수신자 결정 및 전달
3) 수신자가 접속(online) 상태가 아닌 경우, 접속할 때까지 메세지 보관

![client-to-client-채팅흐름도.png](..%2F..%2F..%2F..%2Fimg%2Fchapter12%2Fclient-to-client-%EC%B1%84%ED%8C%85%ED%9D%90%EB%A6%84%EB%8F%84.png)

## 메세지 송신

* 메세지 송신의 주체는 Client이다. (Client to Server) 따라서 Client의 요구가 있을 때 HTTP 프로토콜을 사용할 수 있다
* keep-alive 헤더를 이용하면 connection을 효율적으로 유지할 수 있다
    * keep-alive 헤더를 사용하면 TCP Connection을 맺기 위한 handshake 횟수를 줄일 수 있다

## 메세지 수신

* 메세지 수신 시나리오의 주체는 Server다
* Server가 임의 시점에 Client에게 메세지를 보내는 데에는 HTTP 프로토콜을 쉽게 사용할 수 없다
* Server가 임의 시점에 연결을 만드는 것처럼 동작할 수 있도록 하기 위한 기법이 몇가지 존재한다
    * Polling
    * Long Polling
    * Web Socket

### Polling

* Polling은 Client가 주기적으로 Server에게 새 메세지가 있는지 물어보는 기법이다
* 의미 있는 데이터가 없을 때도 Polling은 주기적으로 일어난다
* Polling을 실행하는 빈도수가 잦을수록 Polling 비용이 늘어난다
* 대부분의 경우 서버 자원이 불필요하게 낭비되는 특징이 있다

### Long Polling

* Long Polling은 Polling과 비슷하지만, Server측에서 데이터를 반환하거나, Timeout이 발생하기 전까지 Connection을 유지하는 기법이다
* Server의 Connection을 Timeout까지 계속 물고있어 비효율적이다
* 메세지를 기다리는 Client와 Connection을 맺은 Server는 메세지를 수신하지 못할 수도 있다
    * ex. 로드 밸런싱을 하는 경우, 수신 메세지가 Connection을 맺지 않은 다른 서버에 수신 될 수 있다
* Server 입장에선 Client에서 연결을 해제 했는지 알 수 있는 방법이 없다

### 웹 소켓

* 웹 소켓 연결의 주체는 Client이다
* 한 번 맺어진 연결은 항구적이며 양방향이다
* 처음에는 HTTP Connection이지만, handshake 절차를 거치면 항구적인 연결이 만들어지고, server <-> client 비동기 양방향 메세지 전송이 가능하다
* 80, 443과 같은 기본 HTTP 통신 포트를 사용하기에 방화벽이 있는 환경에서도 잘 동작한다
* 웹소켓을 사용하면 단순하고 직관적인 설계가 가능하다
* 웹소켓 연결은 항구적으로 유지되기에, 서버 측에서 연결 관리를 효율적으로 해야 한다

![client-to-client-websocat-채팅흐름도.png](..%2F..%2F..%2F..%2Fimg%2Fchapter12%2Fclient-to-client-websocat-%EC%B1%84%ED%8C%85%ED%9D%90%EB%A6%84%EB%8F%84.png)

## 채팅 시스템 개략적 설계안

* Client <-> Server 간의 채팅 흐름에 주 통신은 Web Socket을 사용한다
* 대부분의 기능 (회원가입, 로그인, 사용자 프로필 등) 은 HTTP 프로토콜을 이용하여 구현한다

### 무상태 서비스 (stateless service)

* 무상태 서비스는 일반적인 회원가입, 로그인과 같은 기능을 처리하는 전통적인 요청/응답 서비스이다
* 로드밸런서는 요청에 알맞은 무상태 서비스로 요청을 포워딩 할 수 있어야 한다
* Discovery Service를 이용할 수 있다 (Eureka 등)

![무상태-서비스.png](..%2F..%2F..%2F..%2Fimg%2Fchapter12%2F%EB%AC%B4%EC%83%81%ED%83%9C-%EC%84%9C%EB%B9%84%EC%8A%A4.png)

### 상태 유지 서비스 (stateful service)

* 본 설계안에서 상태 유지가 필요한 서비스는 채팅 서비스이다
    * 각 Client가 Server와 독립적인 Connection을 유지해야 한다
    * Server가 살아 있는 한, 다른 Server로 연결을 변경하지 않는다
    * 앞서 무상태 서비스에서 봤던 Discovery Service는 채팅 서비스와 긴밀히 협력하여, 특정 서비스에 Connection이 몰리지 않도록 Server 선정을 도와야 한다

![상태유지-서비스.png](..%2F..%2F..%2F..%2Fimg%2Fchapter12%2F%EC%83%81%ED%83%9C%EC%9C%A0%EC%A7%80-%EC%84%9C%EB%B9%84%EC%8A%A4.png)

### 제3자 서비스 연동

* 가장 중요한 제 3자 서비스는 Push 알림 서비스이다
* 앱이 실행중이지 않아도 알림을 받아야 해서, 푸시 알림 서비스와의 통합이 중요하다

## 규모 확장성

* 모든 기능을 단일 서버로 구성할 수도 있지만 그렇게 하지 않는 가장 큰 이유는 SPOF(Single Point Of Failure) 때문이다
  ![실시간-메세지-서비스.png](..%2F..%2F..%2F..%2Fimg%2Fchapter12%2F%EC%8B%A4%EC%8B%9C%EA%B0%84-%EB%A9%94%EC%84%B8%EC%A7%80-%EC%84%9C%EB%B9%84%EC%8A%A4.png)
* 채팅 서버는 Client-Client 사이 메세지를 중계하는 역할을 한다
* 접속상태 서버는 사용자의 접속 여부를 관리한다
* API 서버는 로그인, 회원가입, 프로필 변경 등 그 외 나머지 기능을 처리한다
* 알림 서버는 Push 알림을 보낸다
* 키-값 저장소는 채팅 이력을 보관한다
    * 시스템에 접속한 사용자는 이전 채팅 이력을 전부 볼 수 있다

### 저장소

* 데이터 계층을 구성하는 DB를 선택해야 한다
    * 관계형 DB를 사용할 것인가?
    * NoSQL을 사용할 것인가?
    * **데이터의 유형과 읽기/쓰기 연산 패턴이 무엇인지 따져보아야 한다**
* 채팅 시스템의 데이터
    * **일반 데이터**
        * 종류
            * 사용자 프로필
            * 설정
            * 친구 목록
        * 안정성을 보장하는 **RDB**에 보관해야 한다
            * **데이터의 가용성과 규모 확장성을 보증하기 위한 기술**
                * 다중화(replication)
                * 샤딩(Sharding)
    * 채팅 데이터
        * 종류
            * 채팅 히스토리
                * Facebook 같은 경우 하루 600억 건의 채팅 메세지가 발생
                * 위 데이터 중 빈번하게 사용되는 메세지는 최근 메세지
                * 오래된 메세지는 잘 보지 않는다
                * 대체로 최근에 수신한 메세지를 보지만, 이전 히스토리에서 검색, 언급 된 메세지 확인, 특정 메세지로 점프와 같은 무작위 데이터에 대한 접근이 존재한다
            * 채팅 앱의 읽기/쓰기 비율은 보통 1:1 이다
            * 채팅 히스토리는 키-값 저장소에 보관한다
                * 키-값 저장소는 수평적 규모 확장성이 좋다
                * 키-값 저장소는 데이터 접근 지연시간이 낮다
                * RDB는 Long Tail을 잘 처리하지 못한다. RDB는 인덱스가 커지면 random access에 대한 비용이 늘어난다
                * Facebook은 HBase, Discord는 Cassandra를 사용한다

## 데이터 모델

* 1:1 채팅
    * 보통 message_id를 PK로 가진다. (number)
        * 메세지 순서를 쉽게 정할 수 있다
        * created_at 이라는 생성 날짜 컬럼을 별도로 운영하는 것도 좋다
* Group 채팅
    * 보통 message_id와 chatting_group_id를 복합키로 가진다

### message_id

* message_id 요구사항
    * 순서를 보장해야 한다
    * 값이 고유해야 한다
    * 정렬이 가능하며 시간 순서와 일치해야 한다
* 채팅 데이터에 NoSQL을 사용하기로 했다 - NoSQL은 보통 auto-increment를 제공하지 않는다
    * 따라서 snowflake와 같은 대안을 사용하도록 한다
* message_id는 chatting_group 내에서만 고유하면 된다

## 서비스 탐색 기능

* 서비스 탐색 기능의 주된 역할은 Client에게 현재 가용 가능한 채팅 서버를 추천하는 것이다
    * 추천의 기준은 Client의 위치와, server의 용량 등이 있다
* Apache Zookeeper와 같은 오픈 소스 솔루션을 사용할 수 있다
    * 모든 채팅 서버를 Zookeeper에 등록하고, Client가 연결을 시도할 경우 사전에 정한 기준에 따라 최적의 채팅 서버를 Client에게 반환해준다
      ![Zookeeper를-이용한-실시간-채팅-서버-추천-기능.png](..%2F..%2F..%2F..%2Fimg%2Fchapter12%2FZookeeper%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EC%8B%A4%EC%8B%9C%EA%B0%84-%EC%B1%84%ED%8C%85-%EC%84%9C%EB%B2%84-%EC%B6%94%EC%B2%9C-%EA%B8%B0%EB%8A%A5.png)

1. 사용자가 시스템에 로그인 한다
2. Load Balancer가 로그인 요청을 API 서버로 중계한다
3. API 서버가 사용자 인증을 처리하고, Service 탐색 기능이 동작하여 적절한 Web Socket Server를 반환한다
4. 사용자는 채팅 Server 3 과 웹소캣 연결을 맺는다

### 메세지 흐름

### 1:1 채팅 메세지 처리 흐름

![1대1-채팅-메세지-처리-흐름.png](..%2F..%2F..%2F..%2Fimg%2Fchapter12%2F1%EB%8C%801-%EC%B1%84%ED%8C%85-%EB%A9%94%EC%84%B8%EC%A7%80-%EC%B2%98%EB%A6%AC-%ED%9D%90%EB%A6%84.png)

1. 사용자 1이 채팅 서버 1로 메세지 전송
2. 채팅 서버 1은 ID 생성기를 이용해 message_id를 발급
3. 채팅 서버 1은 해당 메세지를 메세지 큐로 전송
4. 메세지 키-값 저장소에 보관
5. 사용자 2가 접속중인 경우라면 해당 채팅 서버로 메세지 중계, 아니라면 Push 알림 서버로 중계
6. 사용자 2가 접속중이라면 채팅 서버가 WebSocket을 이용해 사용자 2에게 메세지 전달

### 여러 단말 사이의 메세지 동기화

![여러-단말-사이의-메세지-동기화.png](..%2F..%2F..%2F..%2Fimg%2Fchapter12%2F%EC%97%AC%EB%9F%AC-%EB%8B%A8%EB%A7%90-%EC%82%AC%EC%9D%B4%EC%9D%98-%EB%A9%94%EC%84%B8%EC%A7%80-%EB%8F%99%EA%B8%B0%ED%99%94.png)

* cur_max_message_id 라는 변수로 현재 단말의 cur_max_message_id 값과 실제 채팅 데이터의 cur_max_message_id 값을 비교한다
* 새 메세지로 간주하는 조건
    * 수신자 ID가 현재 로그인한 사용자와 같다
    * 키-값 저장소에 보관된 메세지로서, 그 ID가 cur_max_message_id보다 큰 데이터이다

### 소규모 그룹 채팅에서의 메세지 흐름

![소규모-채팅-그룹-메세지-흐름.png](..%2F..%2F..%2F..%2Fimg%2Fchapter12%2F%EC%86%8C%EA%B7%9C%EB%AA%A8-%EC%B1%84%ED%8C%85-%EA%B7%B8%EB%A3%B9-%EB%A9%94%EC%84%B8%EC%A7%80-%ED%9D%90%EB%A6%84.png)

* 사용자 A가 B,C 에게 보낸 메세지는 각각의 MQ에 복사된다
* 사용자 B와 C는 자신에게 온 메세지를 확인하기 위해 본인의 메세지 큐만 확인한다
    * 단순한 흐름이다
* 그룹 사이즈가 크지 않으면 MQ에 메세지를 복사해서 넣는 비용이 크지 않아서 소규모 시스템에 적합하다
* 위챗이 위와 같은 접근법을 사용하고 있으며, 그룹당 인원 500명을 제한하고 있다

## 접속 상태 표시

### 사용자 로그인

* 클라이언트와 채팅 서버 간에 web-socket 연결이 맺어지면, last_active_at 이라는 타임스탬프 값을 키-값 저장소에 저장하여 접속상태를 확인한다
* 위 절차가 끝나면 해당 사용자는 접속 중인 것으로 표시된다
  ![사용자-로그인.png](..%2F..%2F..%2F..%2Fimg%2Fchapter12%2F%EC%82%AC%EC%9A%A9%EC%9E%90-%EB%A1%9C%EA%B7%B8%EC%9D%B8.png)

### 로그아웃

* 사용자는 API 서버로 로그아웃 요청을 보내고, API 서버는 접송상태 서버에 접속 상태 변경을 요청한다
  ![사용자-로그아웃.png](..%2F..%2F..%2F..%2Fimg%2Fchapter12%2F%EC%82%AC%EC%9A%A9%EC%9E%90-%EB%A1%9C%EA%B7%B8%EC%95%84%EC%9B%83.png)

### 접속 장애

* WebSocket 과 같은 지속적인 연결을 맺고 있음에도, 일시적인 접속 장애는 늘 발생할 수 있다
    * 예를들면 데이터가 터지지 않는 터널을 지날때, 접속 상태가 offline으로 변해야 하는가?
* 위와 같은 일시적인 접속 장애를 유연하게 대처하기 위해 심장박동(heartbeat) 모델을 사용한다
* 예를들면 client는 10초에 한번씩 서버에 심장박동을 전송하고, 서버는 client로부터 30초간 (약 3번의 임계치) 심장박동이 오지 않을 경우, offline 상태로 바꾸는 것이다

### 상태 정보의 전송

![상태정보-전송-서비스.png](..%2F..%2F..%2F..%2Fimg%2Fchapter12%2F%EC%83%81%ED%83%9C%EC%A0%95%EB%B3%B4-%EC%A0%84%EC%86%A1-%EC%84%9C%EB%B9%84%EC%8A%A4.png)

* 소규모 채팅의 경우 각 사용자 간에 MQ를 두고 본인이 속한 MQ를 구독하는 모델을 사용한다
* 그룹 채팅 참여자의 수가 커질수록 N개 만큼의 MQ의 규모가 커지므로, 효율적이지 못하다
* 따라서 대규모 채팅의 경우 채팅방에 입장하는 순간에 한번 상태 정보를 갱신하거나, 상태 정보를 본인이 원할 때 수동으로 갱신하도록 유도할 수 있다

### 더 나아가 고민해봐야 할 내용들

* 채팅 앱에서 사진, 오디오와 같은 미디어(Media)를 지원하는 방법
    * 이미지, 썸네일, 비디오 제공
    * 압축방식, Cloud 저장소
* 종단 간 암호화
    * 메세지의 발신인과 수신인 이외에는 메세지를 볼 수 없도록 암호화 한다
* 캐시(Cache)
    * Client에 이미 읽은 메세지를 캐시해놓으면 client<->server 사이에 주고 받는 데이터 양을 크게 줄일 수 있다
* 로딩 속도 개선
    * 사용자 데이터를 지역적으로 분산시켜 물리적 거리로 인한 지연 시간을 최소화 한다
* 오류 처리
    * 채팅 서버 오류
        * 수십만의 유저가 동시에 채팅 서버에 접속해 있는 경우, 해당 서버가 죽으면 Zookeeper가 새로운 서버로 채팅 서버를 배정할 수 있도록 한다
    * 메세지 재전송
        * retry , MQ 등을 통해 메세지 전송 안정성을 확보한다

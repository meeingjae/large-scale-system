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

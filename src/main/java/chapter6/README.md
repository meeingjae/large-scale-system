# 6장 - Key - Value Storage 설계

-- -- 

* 키 값 저장소는 비관계형 데이터베이스이다
* 고유 식별자를 키로 가진다
* 키와 값 사이의 관계를 "키-값" 쌍 이라고 지칭한다
* 대표적인 키 값 저장소
    * DynamoDB , Memcached, Redis, Cassandra 등등
* put(key, value) : 키-값 쌍을 저장소에 저장한다
* get(key) : 인자로 주어진 키에 매달린 값을 꺼낸다

### 문제 이해 및 해결  범위 확정

* 키 값 쌍의 크기 (ex. 10kb 이하)
* 큰 데이터를 저장할 수 있어야 한다
* 높은 가용성을 제공해야 한다
* 높은 규모 확장성을 제공해야 한다
* 데이터 일관성 수준은 조정이 가능해야 한다
* 응답 지연시간이 짧아야 한다

### 분산 키 값 저장소

* 분산 해시 테이블이라고도 불린다
* CAP 정리 (Consistency, Availability, Partition Tolerance)
    * 데이터 일관성 (C, Consistency)
        * 분산 시스템에 접속하는 모든 클라이언트는 어떤 노드에 접근하던 항상 같은 값을 제공 받을 수 있어야 한다
    * 데이터 가용성 (A, Availability)
        * 분산 시스템에 접근하는 모든 클라이언트는 일부 노드에 장애가 발생하더라도 항상 정상 응답을 받을 수 있어야 한다
    * 파티션 감내 (P, Partition Tolerance)
        * 두 노드 사이에 장애가 발생하여 네트워크에 파티션이 발생하더라도 시스템은 계속 동작해야 한다
* CP 시스템 (Consistency, Partition Tolerance)
    * 데이터 가용성을 포기하고, 데이터 일관성과 파티션 감내를 챙기는 시스템
    * 최상의 데이터 일관성을 제공하지만, 일부 노드에 접근이 불가능하면 전체 시스템에 대한 접근을 제한한다
* AP 시스템 (Availability, Partition Tolerance)
    * 데이터 일관성을 포기하고, 최상의 가용성과 파티션 감내를 챙기는 시스템
    * 일부 노드가 먹통이 되어도, 다른 노드에서 서비스를 계속 제공한다, 이후 장애 노드가 복구되었을 때 각 노드간 데이터 불일치 현상이 발생할 수 있다

### 시스템 컴포넌트

* 데이터 파티션
* 데이터 다중화 (replication)
* 일관성 (consistency)
* 일관성 불일치 해소 (inconsistency resolution)
* 장애 처리
* 시스템 아키텍쳐 다이어그램
* 쓰기 경로
* 읽기 경로

### 안정 해시를 사용한 데이터 파티션

* 규모 확장 자동화
    * 시스템 부하에 따라 서버가 자동으로 추가 되거나 삭제 되도록 만들 수 있다
* 다양성
    * 각 서버의 용량에 맞게 가상 노드의 수를 조절할 수 있다. 고성능 서버는 더 많은 가상 노드를 갖도록 설정할 수 있다

### 데이터 다중화

* 데이터를 N개의 서버에 비동적으로 다중화 할 필요가 있다 (N은 튜닝 가능할 값)
* 가상 노드를 사용한다면 실제 물리 서버 대수가 N보다 커야한다.

### 데이터 일관성

* 여러 노드에 다중화 된 데이터는 자연히 동기화가 되어야 한다
* 정족수 합의 프로토콜을 사용하면 읽기/쓰기 연산 모두에 일관성을 보장할 수 있다
* 정족수 합의 프로토콜 (Quorum Consensus Protocol)
    * N = 사본 갯수
    * W = Write 연산에 대한 정족수 (Write 연산이 성공했다고 판단하려면 몇 대의 서버로부터 성공 메세지를 받아야 하는가?)
    * R = Read 연산에 대한 정족수 (Read 연산이 성공했다고 판단하려면 몇 대의 서버로부터 성공 메세지를 받아야 하는가?)
    * 중개자가 있고, 중개자에게 응답 해야 한다
        * R=1, W=N : 빠른 읽기 연산에 최적화 된 시스템
        * W=1, R=N : 빠른 쓰기 연산에 최적화 된 시스템
        * W+R > N : 강한 일관성이 보장 됨 (보통 N=3, W=R=2)
        * W+R < N : 강한 일관성이 보장 되지 않음

### 일관성 모델

* 강한 일관성 : 모든 읽기 연산은 가장 최근에 갱신 된 결과를 반환한다 (절대로 낡은 데이터를 보지 못한다)
* 약한 일관성 : 읽기 연산은 가장 최근에 갱신 된 결과를 반환하지 못할 수도 있다
* 최종 일관성 : 약한 일관성의 한 형태로, 갱신 결과가 모든 사본에 반영(동기화) 된다

* 강한 일관성 달성 방법
    * 모든 사본에 현재 쓰기 연산의 결과가 반영될 때까지 해당 데이터에 대한 읽기/쓰기 연산을 금지시킨다
    * 고가용성 시스템에는 적합하지 않다
* 최종 일관성 모델
    * 쓰기 연산이 병렬적으로 발생한다
    * 시스템의 일관성이 깨지는 문제를 클라이언트가 해결해야 한다
    * 비 일관성 해소 기법
        1. 데이터 버저닝 & 벡터 시계

### 장애 감지

* 모든 노드 사이에 멀티캐스팅 채널을 구축하는 것이 가장 손쉬운 노드 간 장애 감지 기법이다 -> 하지만 서버가 많을때는 비효율적이다
* 가십 프로토콜 같은 분산형 장애 감지 솔루션을 채택하는 것이 보다 효과적이다
  1. 각 노드는 멤버십 목록을 유지한다
  2. 각 노드는 주기적으로 자신의 HeartBeat를 증가시킨다
  3. 각 노드는 무작위로 선정 된 노드들에게 주기적으로 자신이 가지고 있는 Heartbeat 목록을 내보낸다
  4. HeartBeat 목록을 받는 노드는 멤버십 목록을 최신으로 갱신한다
  5. 특정 멤버의 HearBeat 카운터 값이 지정한 기간동안 동작하지 않으면 해당 멤버는 장애 상태인 것으로 간주한다

### 일시적 장애 처리

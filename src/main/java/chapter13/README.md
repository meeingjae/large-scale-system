# 13장 - 검색어 자동 완성 시스템

-- -- 

### 요구사항

* 빠른 응답 속도
    * 문자 입력에 대한 자동완성 단어 노출은 100ms 이내여야 한다
* 연관성
    * 입력한 단어와, 자동완성 단어는 연관 된 단어여야 한다
* 정렬
    * 자동완성 단어 선정 계산 결과는 인기도 등의 순위 모델에 의해 정렬 되어야 한다
* 규모 확장성
    * 시스템이 많은 트래픽을 감당할 수 있도록 확장이 가능해야 한다
* 고가용성
    * 시스템의 일부에 장애가 발생하거나, 느려지거나, 예상치못한 네트워크 문제가 발생해도 시스템 사용은 계속 가능해야 한다

-- --

### 개략적 규모 추정

* 일간 능동 사용자 (DAU) 는 1000만 명으로 가정
* 평균적으로 한 사용자는 하루 10건의 검색을 수행
* 평균적으로 20 byte 규모의 데이터를 입력
* 글자가 입력 될 떄마다 백엔드로 요청
* 위 데이터를 조합하면
    * 초당 24,000 건 정도의 요청이 발생(QPS) (1000만 사용자 x 10 질의 / 일 x 20wk / 24시간 / 3600초)
    * 최대 QPS = 평균 x 2 = 48,000 건
* 질의 가운데 20%는 신규 검색어로 가정
    * 매일 0.4GB 정도의 데이터가 시스템에 추가

-- --

## 개략적 설계

### 데이터 수집 서비스 (data gathering service)

* 사용자들의 질의를 실시간으로 수집하는 서비스
* 실시간에 대한 고려가 필요하다
* RDB에 각 질의에 따른 빈도수를 실시간으로 업데이트해 줄 수 있지만, 이는 데이터가 많아질 경우 RDB의 병목으로 이어질 수 있다

### 데이터 질의 서비스 (data query service)

* 주어진 질의에 다섯 개의 인기 검색어를 정렬해 반환하는 서비스

-- --

## 상세 설계

* 트라이(trie) 자료구조
* 트라이 연산
* 데이터 수집 서비스
* 질의 서비스
* 규모 확장이 가능한 저장소

### 트라이 자료구조

* 접두어트리(prefix tree) 라고도 한다
* 트라이는 문자열을 간략하게 저장할 수 있는 자료구조
* 문자열을 꺼내는 데 초점을 맞춘 자료 구조
* 트라이 자료구조 특징
    * 트라이는 트리 형태의 자료구조
    * 트리(tree)의 루트 노드(root node)는 빈 문자열
    * 각 노드는 글자 하나를 저장하며, 26개(다음으로 등자할 수 있는 모든 글자 갯수)의 자식 노드를 가질 수 있다
    * 각 트리 노드는 하나의 단어 또는 접두어 문자열(prefix string)을 나타낸다
* 트라이 자료구조 예시
  ![trie-자료구조-예시.png](..%2F..%2F..%2F..%2Fimg%2Fchapter13%2Ftrie-%EC%9E%90%EB%A3%8C%EA%B5%AC%EC%A1%B0-%EC%98%88%EC%8B%9C.png)
* 트라이 자료구조 with score 예시
  ![trie-자료구조-with-score-예시.png](..%2F..%2F..%2F..%2Fimg%2Fchapter13%2Ftrie-%EC%9E%90%EB%A3%8C%EA%B5%AC%EC%A1%B0-with-score-%EC%98%88%EC%8B%9C.png)
* 트라이 자료구조 용어 정의
    * p : prefix(접두어) 길이
    * n : 트라이 안에 있는 node 개수
    * c : 주어진 노드의 자식 노드 개수
* 시간 복잡도
    * 특정 단어 검색
        * 특정 접두어를 표현하는 노드 탐색
            * O(p)
        * 유효한 문자열을 구성하는 모든 유효 노드 검색
            * O(c)
        * 유효 노드를 정렬하여 인기 있는 검색어 k개 검색
            * O(c log c)
        * 결과 = O(p) + O(c) + O(c log c)
    * 최악의 경우 k개의 결과를 얻으려고 모든 트라이를 전부 검색해야 할 수도 있다
        * 해결 방법
            * 접두어의 최대 길이 제한
                * O(p) -> O(1)
            * 각 노드에 인기 검색어를 캐시
                * 각 노드에 k개의 인기 검색어를 저장해 둔다
                * k개는 정책상 달라질 수 있다
                * 단, 추가적인 저장 공간을 필요로 하는 단점이 존재한다
                * 저장공간 vs 빠른 응답속도에 대한 트레이드오프가 필요할 것
                * ![trie-자료구조-인기검색어-캐싱.png](..%2F..%2F..%2F..%2Fimg%2Fchapter13%2Ftrie-%EC%9E%90%EB%A3%8C%EA%B5%AC%EC%A1%B0-%EC%9D%B8%EA%B8%B0%EA%B2%80%EC%83%89%EC%96%B4-%EC%BA%90%EC%8B%B1.png)
                * 접두어 노드의 시간 복잡도 = O(1)
                * 인기 검색어의 검색 시간 복잡도 = O(1)
                * 인기 검색어 k개를 찾는 전체 알고리즘 시간 복잡도 = O(1)

### 데이터 수집 서비스

* 내용 취합
    * 매일 수천만 건의 질의가 발생할 것이고, 그 때마다 트라이를 갱신한다면 성능이 매우 저하될 것
    * 트라이가 한번 만들어 지면, 인기 검색어가 변경되는 빈도수가 매우 적을 것.
        * 따라서 트라이를 자주 갱신할 필요가 없을 것
* 규모 확장이 쉬운 서비스를 만들려면 데이터가 어디에서 오고, 어떻게 사용되는지 그 성격을 알아야 한다
    * Twitter 같은 경우, 항상 신선한 자동 완성을 제공한다
    * 구글 검색 같은 경우, 항상 신선한 자동 완성을 제공할 필요는 없다
* 트라이 구성 데이터는 보통 데이터 분석 시스템 혹은 데이터 로깅 서비스로부터 발생한다
    * 데이터 분석 서비스 로그
        * 인덱스가 없는 로그성 데이터이다
        * 데이터에 대한 추가 수정 없이 단순 이벤트를 기록만 한다
    * 로그 취합 서버
        * 데이터 분석 서비스로부터 나오는 로그를 필요에 맞게 잘 취합하여 사용해야 한다
        * 실시간성에 따라 로그 취합 빈도수를 정할 수 있다
    * 취합된 데이터
        * 데이터는 다음과 같은 필드를 가진다
            * 키워드
            * 취합 날짜(시간)
            * 빈도 (frequency)
    * 작업 서버
        * 작업 서버는 데이터 취합을 비동기적으로 수행하는 서버이다
        * 트라이 자료구조를 생성하고, DB에 store하는 작업을 담당한다
    * 트라이 캐시
        * 분산 캐시 시스템으로 트라이 데이터를 메모리에 유지해서 연산 속도를 올리는 데 기여한다 (성능 향상)
        * 매주 트라이 DB에 스냅샷을 떠 갱신한다
    * 트라이 데이터베이스
        * 지속성 저장소
            * 문서 저장소 (Document store)
                * 주기적으로 트라이를 직렬화하여 DB에 저장할 수 있다
                * MongoDB와 같은 저장소가 있다
            * 키-값 저장소(key-value store)
                * 트라이에 보관 된 모든 접두어를 해시 테이블 **키**로 변환
                * 각 트라이 노드에 보관 된 모든 데이터를 해시 테이블의 **값**으로 변환
                * 각 트라이 노드는 **<키-값> 쌍**으로 변환된다
    * 질의 서비스
        * ![질의-서비스-흐름도.png](..%2F..%2F..%2F..%2Fimg%2Fchapter13%2F%EC%A7%88%EC%9D%98-%EC%84%9C%EB%B9%84%EC%8A%A4-%ED%9D%90%EB%A6%84%EB%8F%84.png)
            1. 검색 질의가 Load Balancer로 전달
            2. Load Balancer는 해당 질의를 API Server로 포워딩
            3. API Server는 트라이 캐시로부터 데이터를 가져와 응답 데이터 구성
            4. 트라이 캐시가 비어있는 경우, 트라이 DB로부터 값을 불러와 캐싱한다
                5. 캐시 서버의 메모리가 부족하거나, 캐시 서버에 장애가 있으면 캐시 미스가 발생할 수 있다
        * 질의 서비스는 실시간에 가깝도록 성능이 좋아야한다
            * AJAX 요청
                * 페이지 갱신이 필요없는 요청
            * 브라우저 캐싱
                * 대부분의 경우 자동완성 검색어 제안이 자주 바뀌지 않는다
                * 따라서 응답 데이터를 브라우저 캐시에 넣어두면, 후속 질의 시 해당 데이터를 빠르게 사용 가능하다
                * Google이 브라우저 캐싱을 사용중이다
            * 데이터 샘플링
                * 모든 데이터를 캐싱하면 시스템 자원을 많이 소모하게 된다
                * 따라서 데이터를 샘플링해 N개 요청 가운데 1개만을 로깅하도록 한다

### 트라이 연산

* 트라이 생성
    * 트라이 생성은 작업 서버가 담당한다
    * 데이터 분석 서비스의 로그나, DB로부터 취합 된 데이터를 사용한다
* 트라이 갱신
    * 트라이 갱신 방법
        * 매주 한 번 갱신
            * 새로운 트라이를 매 주 생성해서, 기존 트라이를 대체한다
        * 트라이의 각 노드를 개별적으로 갱신
            * 성능이 좋지 않다
                * 특정 노드의 빈도수를 갱신하면, 해당 노드를 캐싱하고 있던 모든 다른 노드(상위 노드) 또한 갱신해줘야 하기 때문이다
            * 트라이가 작을 때는 고려해 볼 만한 방법이다
* 검색어 삭제
    * 폭력적이거나 노골적인 단어는 필터링이 필요할 수 있다
    * 트라이 캐시 앞에 filter 계층을 둘 수 있다
    * ![트라이-필터.png](..%2F..%2F..%2F..%2Fimg%2Fchapter13%2F%ED%8A%B8%EB%9D%BC%EC%9D%B4-%ED%95%84%ED%84%B0.png)
    * 캐시 Filter로 검색 결과를 자유롭게 handling 할 수 있다
    * 부적절 데이터를 물리적으로 삭제하는 것은 다음 갱신 사이클에 비동기적으로 수행이 가능하다

### 저장소 규모 확장

* 영어를 지원하기에 간단하게 첫 글자 기준으로 샤딩(sharding)을 수행할 수 있다
    * 서버의 갯수만큼 알파뱃을 분할한다
        * ex) 서버 2대 = 'a' to 'm' , 'n' to 'z'
        * ex) 서버 3대 = 'a' to 'i' , 'j' to 'r' , 's' to 'z'
    * 위와 같은 경우 최대 사용 가능 서버 갯수는 26개(알파뱃 갯수) 이다
    * 이 이상으로 서버 대수를 늘리려면 계층적 샤딩이 필요하다
        * ex) 'aa' to 'am' , 'an' to 'az' , 'ba' to 'bm', 'bn' to 'bz' ...
    * 하지만 이 방법의 경우 특정 서버로 트래픽이 몰릴 수 있다
        * ex) 'c'로 시작하는 단어가 'x'로 시작하는 단어보다 훨씬 많다
    * 이는 중간에 샤드 관리자(shard map manager)를 두어 해결할 수 있다
        * ![샤드-매니저.png](..%2F..%2F..%2F..%2Fimg%2Fchapter13%2F%EC%83%A4%EB%93%9C-%EB%A7%A4%EB%8B%88%EC%A0%80.png)
        * 샤드 매니저는 어떤 검색어를 어떤 샤드에 배치할 지 관리할 수 있다
            * ex) 'c'로 시작하는 단어 = 1번 샤드, 'x','y','z'로 시작하는 단어 = 2번 샤드 ..

### 추가적인 아이디어
 
* 샤딩을 통하여 작업 대상 데이터의 양을 줄인다
* 순위 모델(ranking model) 정책을 바꾸어, 최근 검색 된 검색어에 보다 높은 가중치를 부여한다
* 데이터가 스트림 형태로 올 수 있다는 점을 명심하자
  * 데이터 스트리밍 시스템
    * 아파치 하둡 맵리듀스
    * 아파치 스파크 스트리밍
    * 아파치 스톰
    * 아파치 카프카
    * ...








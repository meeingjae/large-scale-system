# 8장 - URL 단축키 설계

-- -- 

* 개략적 추정
    * `쓰기 연산` : `매일 1억 개`의 단축 URL 생성
    * `초당 쓰기 연산` : 1억 / 24 / 3600 = `1160`
    * `초당 읽기 연산` : 읽기 연산 비율 : 쓰기 연산 비율 = 10 : 1 ==> `11600`
    * URL 단축 서비스 `10년 운영`을 가정 : 1억 x 365 x 10 = `3650억 개의 레코드`
    * 축약 전 평균 URL 길이 : 100
    * 10년 동안 필욯란 저장 용량 : 3650억 x 100 byte = 36.5 TB

### API 앤드 포인트

1. URL 단축용 앤드 포인트
    * 새 단축 URL을 생성하고자 하는 클라이언트는 이 엔드포인트에 단축할 URL을 인자로 실어서 POST 요청을 보내야 한다. 이 엔드포인트는 다음과 같은 형태를 띈다'
        * POST /api/v1/data/shorten
            * 인자 : longUrlString : String
            * 응답 (Response) : 단축 URL
2. URL 리디렉션용 엔드포인트
    * 단축 URL에 대해서 HTTP 요청이 오면 어ㅜㄴ래 URL로 보내주기 위한 용도의 엔드포인트
        * GET /api/v1/shortUrl
            * 응답 (Response) : HTTP 리디렉션 목적지가 될 원래 URL

### URL 리디렉션

* 단축 URL을 GET 요청 시 301 응답의 Location 에 원래 URL을 넣어 반환한다
* 301 응답과 302 응답의 차이
    * 301 응답 (Permanently Moved)
        * 요청한 URL이 영구적으로 Localtion에 반환된 URL로 이전되었다는 응답이다
        * 따라서 Client는 해당 URL 값을 캐싱하여 다음 요청에 사용할 것이다
        * 첫 번째 요청만 단축 URL 서버로 전송될 것이기에 서버 부하를 최소화할 수 있다
    * 302 응답 (Found)
        * Location에 반환된 URL을 임시적으로 활용하라는 응답이다 즉, Location에 반환된 지정 된 URL에 의해 처리되어야 한다는 응답이다
        * 클라이언트의 요청은 언제나 단축 URL 서버에 먼저 보내진 후에 원래 URL로 리디렉션 되어야 한다
        * 트래픽 분석이 중요할 때는 302 응답을 사용하는 것이 좋다.
            * 클릭 발생률이나 발생 위치를 추적하는 데 좀 더 유리하다
* URL 리디렉션을 구현하는 가장 직관적인 방법은 해시 테이블을 이용하는 것이다
    * 해시 테이블에 {단축 URL, 원래 URL}의 쌍을 저장한다고 가정하면 다음과 같은 프로세스를 거친다
        1. 원래 URL = hashTable.get(단축URL)
        2. 301 혹은 302 응답의 Location 값에 원래 URL을 넣은 후 응답
    * 해시 테이블을 메모리에 적재하는 것은 비용이 크기 떄문에 추천하지 않는다

### URL 단축

* URL을 단축하는 것의 가장 핵심은 역시 URL을 단축하는 해시 함수 `f(x)`를 구현하는 것이다
* URL 단축 함수 `f(x)`는 다음과 같은 요구사항을 만족해야 한다
    * 입력으로 주어지는 긴 URL이 다른 값이면, 해시 값 또한 달라야 한다
    * 계산 된 해시 값은 원래 URL로 복원이 가능해야 한다

### 데이터 모델

* 메모리를 이용하는 해시 테이블의 대안으로 id, originUrl, shortUrl 을 컬럼으로 가지는 RDB Table 구조를 설계할 수 있다

## 해시 함수

* 원래 URl을 단축 URL로 변환하는데 쓰인다
* 단축 URL은 [0-9, a-z, A-Z]의 총 62가지의 문자로 구성될 수 있다
* 단축 URL의 길이를 정하기 위해서는 62^n >= 3650억 인 n의 최소값을 찾아야 한다.
    * n이 7이면 3.5조 개의 URL을 만들 수 있다.

### 해시 후 충돌 해소

* 긴 URL을 줄이려면 원래 URL을 7글자 문자열로 줄이는 해시 함수가 필요하다
* CRC32, MD5, SHA-5와 같이 잘 알려진 해시 함수를 이용할 수 있다
* n = 7 인 단축 URL을 찾아야 하지만, 위 3가지 해시 함수의 결과물은 모두 7자 이상이다
    * 따라서 결과물을 7자로 맞추려면 DB 대신 [블룸 필터](https://ko.wikipedia.org/wiki/%EB%B8%94%EB%A3%B8_%ED%95%84%ED%84%B0)를 사용하면 성능을 높힐
      수 있다
        * 볼륨 필터는 어떤 집합에 특정 원소가 있는지 검사할 수 있도록 하는 확률론에 기초한 공간 효율이 좋은 기술이다

### base-62 변환

* 진법 전환(base conversion)은 URL 단축기를 구현할 때 흔히 사용되는 접근법 중 하나이다
* base-62는 주어진 수를 62진법으로 변환한다
    * 0 == 0
    * 9 == 9
    * 10 == a
    * 11 == b
    * 35 == z
    * 36 == A
    * 61 == Z == 6110
* base-62 기법은 수의 표현 방식이 다른 두 시스템이 같은 수를 공유해야 하는 경우에 유용하다

| 해시 후 충돌 해소 전략                                           | base-62 변환                                                                  |
|---------------------------------------------------------|-----------------------------------------------------------------------------|
| 단축 URL의 길이가 고정 됨                                        | 단축 URL의 길이가 가변적. ID 값이 커지면 같이 길어짐                                           |
| 유일성이 보장되는 ID 생성기가 필요치 않음                                | 유일성 보장 ID 생성기가 필요                                                           |
| 출동이 가능해서 해소 전략이 필요                                      | ID의 유일성이 보장된 후에야 적용이 가능한 전략이라 충돌은 불가능                                       |
| ID로부터 단축 URL을 계산하는 방식이 아니라서 다음에 쓸 수 있는 URL을 알아내는 것이 불가능 | ID가 1씩 증가하는 값이라고 가정하면 다음에 쓸 수 있는 단축 URL이 무엇인지 쉽게 알아낼 수 있어서 보안상 문제가 될 여지가 있음 |

### URL 단축기 상세 설계

1. 입력으로 원래 URL 값을 받는다
2. DB에 해당 URL이 있는지 검사한다
    * DB에 해당 URL이 존재한다면 이미 만든 값이기에 해당 데이터의 단축 URL 값을 불러와 응답한다
3. DB에 해당 URL이 존재하지 않는다면 새로운 URL로, Table의 신규 unique id 값을 발급한다
4. 발급한 id를 62진법으로 변환하여 단축 URL로 만든다
5. id, 원래 URL, 단축 URL로 신규 레코드를 생성 및 저장한 후 Client에 단축 URL 값을 전달한다

### URL 리디렉션 상세 설계

* 리디렉션 시스템 설계
    1. `client` -- GET 단축 URL --> `로드밸런서`
    2. `로드밸런서` --> `WAS`
    3. `WAS` --> `캐시`
    4. `WAS` --> `DB`
    5. `WAS` -- 원래 URL 반환 --> `로드밸런서`
    6. `로드밸런서` -- 원래 URL 반환 --> `client`
* 읽기 연산의 비율이 높은 시스템일수록 캐시를 사용하면 저장 성능을 높힐 수 있다 


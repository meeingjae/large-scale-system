# 5장 - 안정 해시 설계

-- -- 

* 안정 해시의 이점
    * 서버가 추가되거나 삭제 될 때, 재배치 되는 키의 수가 최소화 된다
    * 데이터가 보다 균등하게 분포하게 되므로 수평적 규모 확장성을 달성하기 쉽다
    * 핫스팟 키 문제를 줄인다.
* 안정 해시 사용 사례
    * [아마존 DynamoDB의 Partitioning 관련 컴포넌트](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
    * Apache Casandra 클러스터의 데이터 파티셔닝
    * [디스코드 채팅 어플리케이션](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users)
    * 아카마이 CDN
    * 매그래프 네트워크 부하 분산기

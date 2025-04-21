### 배경

일부 기능(도메인)에서 event기반 아키테처로의 전환을 고려하여 개발중에 있다. 현재 메시지 브로커로 kafka를 활용 준비에 있는 상태로, spring내부 ApplicationEventPublisher 만을 사용하여 event을 사용 중이다. 이 때, 일부 EventListener의 로직은 비동기로 수행된다. event을 호출한 비즈니스 로직과 트랜잭션이 분리되어있고, 원자성과 데이터 정합성을 보장하기 어렵다는 문제가 발생한다. 주로 보상 트랜잭션을 적용하여, 비즈니스 로직이 수행되기 전 상태로 되돌리는 등의 추가적인 로직 구현이 요구되었다. 더해서, 이 보상 트랜잭션의 실패에 대한 처리도 미흡한 상태였기 때문에 다른 방안을 찾게되었다.

이전에 기술블로그를 통해 봤었던 Outbox pattern 이 적절한 대안이 될 것 같았다.

#### 아키텍처
| ------ service ------- | ~~ | before commit phase event listener  |
도메인 로직 + event 발행 -> outbox message 발행, 
                      | after commit phase event listener |
                    -> event sub 로직 수행, outbox message 상태 update, outbox process message 발행

### Point
다른 기술 블로그에서는 대략적인 아키텍처, 사용 사례, 개념 등에 대한 정보는 쉽게 접할 수 있다.
위 패턴을 적용하며 가장 고민이 많았던 부분은 다음이다.
- outbox message 테이블에 수행해야 할 로직을 어떻게 표현하여 담을 것 인가
- outbox message 테이블에 fail 상태로 기록된 event들은 '어떤 주기'로 read하여 event을 다시 수행할 것 인가

outbox message을 read하여, 다시 event을 발행하도록 구현했습니다. 때문에 event정보 자체가 필요했고, outbox message에 insert할 때 event자체를 object mapper 을 이용해서 json string타입으로 직렬화 후 저장합니다. 이 message을 수신하는 쪽에서는 다시 event객체 타입으로 역직렬화를 통해 event객체를 생성해서 로직을 수행하게 됩니다.

그렇다면 server, db에 부하를 주지 않는 선에서의 '주기'를 결정해야한다. 현재 transactional outbox pattern을 적용할 기능에서는, 실패한 event 재처리하는 주기에 민감하지 않는 기능이다. 최종으론 매일 자정마다 (1일 간격) outbox message 에서 `실패한` event들을 재처리하도록 설정하였고, 이와 같이 오래된 message들을 제거하도록 scheduler작업을 등록해놓고 있다


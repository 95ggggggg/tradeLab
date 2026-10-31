# 재전달 — 메시지 중복과 업무 효과를 구분

## 문제와 선택

주문 재요청은 Idempotency-Key, 논리 이벤트의 재전달은 eventId로 구분했다. 주문·Key 기록·Outbox는 주문 DB에 함께 커밋했다. 메모리 콜백 이후 프로세스가 종료되어도 발행 의도가 남도록 Outbox를 사용했고, CDC의 Connector·복제 Slot·로그 보존 운영도 비교했다.

Outbox를 사용해도 Kafka ACK 뒤 DB 표시가 실패하면 다시 발행될 수 있다. 소비 측에서는 영속 eventId 처리 기록과 가상 업무 효과를 소비 DB의 같은 트랜잭션으로 저장했다.

## 코드에서 볼 커밋 경계

[ExecutionService.process](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/execution-service/src/main/java/com/tradelab/execution/application/ExecutionService.java#L23)의 실제 코드:

~~~java
@Override
@Transactional
public boolean process(byte[] key, byte[] payload) {
    var event = decoder.decode(key, payload);
    var now = clock.instant();
    if (!repository.claim(event.eventId(), decoder.fingerprint(event), now)) return false;
    repository.insert(Execution.simulate(event, now));
    return true;
}
~~~

[ExecutionRepository.claim](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/execution-service/src/main/java/com/tradelab/execution/repository/ExecutionRepository.java#L22)은 event_id의 고유 제약과 ON CONFLICT DO NOTHING으로 경합을 처리하고 내용 지문도 비교한다. [Listener](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/event-consumer/src/main/java/com/tradelab/messaging/OrderCreatedListener.java)는 서비스 트랜잭션 반환 뒤 RECORD offset 커밋이 이어지는 경계다.

[Publisher](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/src/main/java/com/tradelab/application/OutboxPublisher.java#L48)는 주문 생성과 별도 트랜잭션에서 행을 잠그고 Kafka ACK를 기다린 뒤 발행 상태를 저장한다. ACK 대기 동안 DB 연결·행 잠금을 유지하는 비용이 있다.

## 실제로 주입했던 실패

| 실패·입력 | 확인한 결과 | 원본 증거 |
| --- | --- | --- |
| 같은 Key·내용의 HTTP 100건 × 3회 | 매회 주문·Key·Outbox 각각 1건 | [멱등성](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/docs/results/idempotency.md) |
| Kafka ACK 뒤 Publisher DB 세션 종료 / 커밋 실패 | Outbox PENDING 보존, 재발행 후 같은 Payload의 Kafka 레코드 2개 | [Outbox](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/docs/results/outbox.md) |
| 동일 eventId의 소비 DB 처리 100건 경합 | 신규 1·중복 99·업무 효과 1건 | [Execution](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/docs/results/execution-service.md) |
| 소비 DB 커밋 후 offset 전 Context 중단 | DB 효과 1건·offset 0, 새 Context 재전달 뒤 효과 1건·offset 1 | [Execution](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/docs/results/execution-service.md) |
| 별도 Order·Execution JVM 종료 후 재시작 | 기존 효과 보존, 재전달 후 추가 효과 없음 | [프로세스 검증](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/docs/results/execution-service.md) |
| 지속 DB 오류·Retry 소진·DLQ 원문 재처리 | DLQ ACK 전 원본 offset 미진행, 원인 제거 후 반복 재처리에도 효과 1건 | [실패·DLQ](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/docs/results/execution-service.md) |

정확한 DB/offset 사이의 Context 중단과 실제 JVM 종료는 서로 다른 실험이다. Publisher의 Kafka 레코드 2개와 Consumer의 효과 1건도 각자의 테스트 근거를 구분한다. 소비 DB 100건 경합을 실제 Kafka 레코드 100개 동시 전달로 설명하지 않는다.

## 재현 코드와 한계

- [Outbox ACK 뒤 실패 테스트](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/src/test/java/com/tradelab/outbox/OutboxKafkaIntegrationTest.java)
- [Consumer의 재전달·롤백·DLQ 테스트](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/execution-service/src/test/java/com/tradelab/execution/ExecutionConsumerIntegrationTest.java)
- [별도 JVM 경계 테스트](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/execution-service/src/test/java/com/tradelab/execution/ServiceBoundaryIntegrationTest.java)

주문 DB·소비 DB·Kafka offset은 별도로 커밋된다. 영속 처리 기록의 보존과 복구 절차를 전제로 업무 효과의 중복 반영을 제어하며, 메시지 자체가 한 번만 전달된다는 보장은 하지 않는다.

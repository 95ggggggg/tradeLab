# 동시 취소 — 응답 수와 커밋 기록을 함께 본 이유

## 계약과 발견

동일한 PENDING 주문에 취소 요청이 동시에 오면 취소 상태로의 변경과 성공 응답은 각각 한 번만 발생하고, 나머지는 HTTP 409로 거절해야 한다.

2026-09-19의 실제 HTTP 100건 경합에서 최종 상태는 CANCELLED였지만 단독 세 회차의 성공 응답은 10·9·10건이었다. 테스트 전용 DB 감사에서 서로 다른 트랜잭션의 CANCELLED→CANCELLED 반복 UPDATE를 확인했다.

| 단독 회차 | 변경 전 성공 / 커밋 UPDATE | 변경 후 성공 / 409 | 변경 후 반복 전이 |
| --- | --- | --- | --- |
| 1 | 10 / 10 | 1 / 99 | 0 |
| 2 | 9 / 9 | 1 / 99 | 0 |
| 3 | 10 / 10 | 1 / 99 | 0 |

근거: [기존 결과와 로그](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/docs/results/concurrency.md). 전체 회귀 중의 Before 3·5·6은 별도 실행이며 위 단독 결과와 합치지 않았다.

## 선택과 구현

READ COMMITTED에서 일반 조회·상태 검사·ID 기준 UPDATE 사이에 같은 상태를 읽는 경합이 있었다. 짧은 단일 행 트랜잭션과 기존 도메인 검증을 유지하려고 행 잠금 이후 반환된 상태로 검사했다. 낙관적 잠금의 version 계약, Redis 잠금의 Lease·늦은 쓰기 보호, 조건부 UPDATE도 비교했다. 성능 우열을 실측해 고른 결과는 아니다.

[OrderService.cancel](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/src/main/java/com/tradelab/application/OrderService.java#L71)의 실제 코드:

~~~java
@Transactional
public Order cancel(UUID orderId) {
    Order current = orders.findByIdForUpdate(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order", orderId));
    Order cancelled = current.cancel(clock.instant());
    orders.updateStatus(cancelled);
    return cancelled;
}
~~~

[OrderRepository.findByIdForUpdate](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/src/main/java/com/tradelab/repository/OrderRepository.java#L42)는 기존 트랜잭션을 요구하고 SELECT 뒤에 FOR UPDATE를 붙인다. 잠금 조회·상태 검사·UPDATE·커밋이 같은 트랜잭션에 속한다. [선택 근거 ADR](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/docs/adr/002-lock-strategy.md).

## 실패와 복구까지 확인한 범위

| 상황 | 기존 실행에서 확인한 상태 |
| --- | --- |
| 선행 연결의 갱신 커밋 / 롤백 | 실제 DB 잠금 대기 후 각각 409 / 취소 성공 |
| 장기 잠금 | 약 2초 후 500 PERSISTENCE_ERROR, 커밋 상태 유지, 잠금 해제 후 새 취소 성공 |
| 취소 UPDATE 뒤 느린 SQL | SQL 제한으로 중단, 취소 롤백·잠금 해제, 후속 취소 성공 |
| 지연 제약 Trigger의 커밋 오류 | 409 DATA_CONFLICT, status·updatedAt 원복, 원인 제거 후 새 취소 성공 |

[동시 HTTP 테스트](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/src/test/java/com/tradelab/concurrency/OrderCancellationConcurrencyIntegrationTest.java)와 [잠금·오류 테스트](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/src/test/java/com/tradelab/concurrency/OrderLockIntegrationTest.java)에서 검증문을 확인할 수 있다.

이 결과는 같은 취소 경로의 정합성에 한정한다. 잠금 대기·Pool 점유 비용은 남으며 다중 인스턴스 처리량이나 실제 체결과 취소의 경합을 검증한 결과는 아니다.

# 성능 — 발행률 개선과 기본값 채택을 나눈 판단

2026-09-25의 기존 실험이다. poll 대기만 1000ms에서 10ms로 바꾸고 배치 크기 10·Pool·SQL·Consumer 설정은 유지했다. 같은 호스트의 다섯 시나리오를 각 두 회차 실행했다.

## 관찰과 결정

| 지표·조건 | 1000ms | 10ms 실험 |
| --- | --- | --- |
| 동시 신규 주문 4개 CASE의 발행률 | 약 9.2~9.5건/s | 약 185.5~187.0건/s |
| 측정 종료 10초 후 미발행 잔여 | 5,030~5,100건 | 0건 |
| 새 주문 없는 WS 조건의 약 29초 Order Pool 대여 | 33회 | 1,846회·1,815회 |
| 동일 계정 신규 주문 1회차 Order DB CPU 최대 표본 | 14.89% | 21.64% |
| SQL별 실행 횟수·시간 | 미측정 | 미측정 |

발행 적체는 해소됐지만 새 주문이 없는 대조 조건에서도 연결 대여와 DB CPU 표본이 늘었다. 사전에 정한 HTTP 오류·P99·DB 부하 악화 시 복원 조건에 따라 1000ms로 되돌리고 10ms 채택을 보류했다.

## 지표의 의미와 한계

Pool 대여는 SQL 실행 횟수·물리 연결 생성 횟수와 다르다. CPU는 CASE별 Docker 표본 4개의 최댓값이며 평균·건당 효율이 아니다. 더 많은 발행·소비 작업을 수행한 조건의 총 CPU 상승을 건당 효율 악화로 단정하지 않았다.

측정 종료 10초 뒤 미발행 0건은 관찰 시점의 결과다. 개별 이벤트가 10초 안에 끝난다는 SLO가 아니다. 허용 지연·자원 상한은 당시 수치로 정하지 않았다.

현재 기본값은 1000ms여서 기존 적체 제약은 남는다. DB 포화나 정합성 실패 때문에 철회한 실험으로 설명하지 않는다.

- [기존 전체 비교·명령·환경·복원 근거](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/docs/results/load-test.md)
- [고정 대기와 Outbox 선택 ADR](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/docs/adr/003-outbox.md)
- [현재 Scheduler](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/src/main/java/com/tradelab/application/OutboxScheduler.java)
- [현재 설정](https://github.com/95ggggggg/invest-prj/blob/5ab986710627cd16a17b3c516959d590ce95cabd/src/main/resources/application.yml)

성능 비교 당시 회귀는 295개, 이후 보존 회귀는 303개다. 두 수를 합산하지 않고 새 실행으로 표시하지 않는다.

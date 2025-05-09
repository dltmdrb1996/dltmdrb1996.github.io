---
title: "Kafka 공부 정리" # 또는 "Kafka 완벽 정복: 핵심 개념부터 실전 운영까지 (종합 가이드)"
date: date: 2025-05-09 11:00:00 +0900 # 실제 글 작성 완료 시간으로 수정
categories: [기술, Kafka, 데이터]
tags: [kafka, msa, event-driven, data-pipeline, stream-processing, distributed-system]
toc: true
comments: true
# image:
#  path: /assets/img/posts/kafka-banner.png
#  alt: Kafka 로고와 데이터 스트림 이미지
# pin: true
---

## Kafka 공부 정리

### 1. Kafka 핵심 개념

*   **1.1 토픽(Topic)**
    *   논리적 메시지 스트림 단위
    *   토픽마다 **파티션**(Partition) 여러 개로 분할 가능s
    *   생성 시 `replication.factor`(RF)와 초기 파티션 수 지정
    *   **추가:** 메시지는 (선택적으로) **키(Key)**를 가질 수 있으며, 이 키는 특정 파티션으로 메시지를 라우팅하는 데 사용될 수 있습니다. (예: 동일 키는 동일 파티션 보장)

*   **1.2 파티션(Partition)**
    *   물리적 메시지 로그 단위, **메시지 저장의 기본 단위**
    *   각 파티션은 0부터 시작하는 순차적 **오프셋(Offset)** 부여
    *   파티션 수만큼 **병렬 처리(throughput)** 가능
    *   여러 파티션 간 전역 순서는 **보장되지 않음** (단, 동일 키를 가진 메시지는 기본 파티셔너 사용 시 동일 파티션으로 전송되어 해당 파티션 내 순서 보장)

*   **1.3 오프셋(Offset)**
    *   **파티션 단위** 순차 번호
    *   파티션마다 0부터 독립 부여 ⇒ 같은 숫자라도 서로 다른 파티션 내 메시지

### 2. 컨슈머 그룹(Consumer Group)

*   `group.id` 로 식별되는 **독립 읽기 단위**
*   같은 그룹 내 컨슈머끼리는 파티션을 나눠 **로드밸런싱** (하나의 파티션은 그룹 내 **단 하나의 컨슈머**에 의해서만 소비됨)
*   다른 그룹이면 **브로드캐스트**: 각 그룹마다 모든 메시지 소비
*   컨슈머가 처리한 위치(오프셋)는 그룹별로 **내부 토픽**(`__consumer_offsets`)에 **컴팩션(compaction) 정책**으로 저장

### 3. 브로커 클러스터 구조

*   **3.1 브로커 노드(Broker)**
    *   클러스터를 이루는 서버 한 대
    *   각 브로커는 여러 파티션의 **리더(Leader)** 혹은 **팔로워(Follower)** 복제본 담당
    *   **추가:** Zookeeper(과거) 또는 KRaft(최신) 프로토콜을 통해 클러스터 메타데이터 관리 및 컨트롤러 선출

*   **3.2 컨트롤러(Controller)**
    *   브로커 중 하나가 선출
    *   토픽 생성·삭제, 파티션 재배치, ISR 관리, 리더 선출 등 클러스터 전반의 관리 작업 담당

*   **3.3 복제(Replication) & ISR (In-Sync Replicas)**
    *   **RF** 개수만큼 복제본 유지 → 가용성·내구성 보장
    *   Leader가 `acks=all` 일 때 ISR(In-Sync Replica) 전체 복제 완료 시 ACK 반환
    *   `min.insync.replicas` 설정 값보다 ISR 수가 적어지면 `acks=all` 설정된 프로듀서 요청 거부 → 데이터 손실 방지 (쓰기 가용성 저하 가능)
    *   `unclean.leader.election.enable=false` (기본) 시, ISR에 없던 (뒤처진) 팔로워가 리더로 승격되는 것 금지 → 데이터 유실 방지 최우선

### 4. 프로듀서 ↔ 브로커: 쓰기 신뢰성

*   **4.1 ACK 설정**
    *   `acks=0` : 응답 안 기다림 → 유실 위험 높음, 속도 가장 빠름
    *   `acks=1` : Leader만 복제 성공 시 ACK → Leader 장애 시 유실 가능 (동기 팔로워가 없는 경우)
    *   `acks=all` (또는 `acks=-1`) : ISR 전체 복제 완료 후 ACK → 가장 높은 내구성 보장

*   **4.2 재시도 · 타임아웃**
    *   `retries`: 메시지 전송 실패 시 재시도 횟수. (멱등성 활성화 시 `Integer.MAX_VALUE` 로 설정하여 순서 보장하며 무한 재시도 권장)
    *   `retry.backoff.ms`: 재시도 간 대기 시간
    *   `request.timeout.ms`: 개별 요청에 대한 브로커 응답 대기 시간 (기본 30s)
    *   `delivery.timeout.ms`: `send()` 호출부터 ACK 수신 또는 최종 실패까지의 총 시간 (재시도 포함, 기본 2min). 이 시간 초과 시 예외 발생.
    *   **추가:** `max.in.flight.requests.per.connection`: 브로커로 한 번에 보낼 수 있는 승인되지 않은 요청 수. 멱등성 비활성화 시 `1`로 설정해야 순서 보장. 멱등성 활성화 시 `5` 이하로 설정 권장.

*   **4.3 배치(Buffering)**
    *   `linger.ms` (기본 0ms): 메시지를 보내기 전 대기하는 시간 (배치 크기 도달 전)
    *   `batch.size` (기본 16KiB): 배치 하나의 최대 크기
    *   `buffer.memory` (기본 32MiB): 프로듀서가 전송 대기 중인 메시지를 쌓아둘 수 있는 전체 메모리 크기 (RecordAccumulator)
    *   `flush()` 호출 시 내부에 남은 모든 배치 전송·ACK 대기

*   **4.4 멱등성(Idempotence)**
    *   `enable.idempotence=true`
    *   **PID(ProducerId) + 시퀀스 번호** 로 중복 판별 (브로커에서)
    *   동일 프로듀서 세션 내 자동 재시도 시 중복 메시지는 브로커에서 drop, 파티션 내 순서 보장 (단, `max.in.flight.requests.per.connection` ≤ 5 조건)
    *   **정정:** 프로듀서 재시작 시 새로운 PID가 할당되거나, 기존 PID의 epoch이 증가하여 새로운 세션이 시작됨. 따라서 애플리케이션 레벨에서 `send()`를 재호출하는 경우(예: `send()` 실패 후 애플리케이션 로직에 의한 재시도)는 멱등성으로 중복이 방지되지 않음. 오직 프로듀서 내부의 자동 재시도에 대해서만 중복을 방지.
    *   **트랜잭셔널 프로듀서** (`transactional.id` 설정) 사용 시, 프로듀서 재시작 후에도 이전 트랜잭션을 이어가거나 중단할 수 있어 "Exactly-Once" 시맨틱스 달성에 핵심.

*   **4.5 트랜잭션(Transactional Producer)**
    *   `transactional.id` 설정 필요, `enable.idempotence=true` 자동 활성화.
    *   `initTransactions()`, `beginTransaction()…send()…commitTransaction()` (또는 `abortTransaction()`)
    *   Produce + (컨슈머의) Offset commit + State store 변경(Streams) 을 **원자적**으로 묶음
    *   실패 시 `abortTransaction()` 으로 롤백 (컨슈머는 `isolation.level=read_committed` 설정 시 커밋된 트랜잭션 메시지만 읽음)

### 5. 브로커 ↔ 컨슈머: 읽기·오프셋 관리

*   **5.1 오프셋 커밋**
    *   `enable.auto.commit=true` (주기적 자동 커밋, `auto.commit.interval.ms`): 간편하지만, 메시지 처리 완료 전 커밋되거나(중복 발생 가능성), 처리 후 커밋 전 장애 시(유실 발생 가능성) 문제 소지.
    *   **수동 커밋**(`enable.auto.commit=false` + `commitSync`/`commitAsync` 또는 스프링 Kafka의 `AckMode` 활용) 권장: 메시지 처리 성공 후 명시적으로 커밋하여 정확성 보장.

*   **5.2 `auto.offset.reset`** (컨슈머 그룹이 처음 시작하거나, 커밋된 오프셋이 더 이상 유효하지 않을 때 적용)
    *   `earliest` : 가장 오래된 메시지(파티션의 시작 오프셋)부터 소비. (리텐션 기간 경과로 삭제된 메시지는 소비 불가)
    *   `latest` : 가장 최신 메시지(파티션의 끝 오프셋)부터 소비 → 과거 메시지 유실(건너뜀).
    *   `none`: 이전 오프셋을 찾을 수 없으면 예외 발생.

*   **5.3 리텐션·컴팩션**
    *   `log.retention.ms` / `log.retention.bytes`: 시간 또는 크기 기반으로 오래된 로그 세그먼트 삭제 → 디스크 공간 관리. 늦은 컨슈머는 메시지 유실 가능.
    *   `log.cleanup.policy=compact`: 키 기반으로 각 키의 마지막 레코드만 보존 (이전 값은 삭제). `__consumer_offsets` 토픽 등에 사용.

*   **5.4 리밸런싱(Rebalancing)**
    *   컨슈머 그룹 내 컨슈머 추가/제거 또는 구독 토픽 변경 시 파티션 재할당 과정.
    *   `session.timeout.ms`: 컨슈머가 브로커에게 살아있음을 알리지 않고 버틸 수 있는 최대 시간. 이 시간 초과 시 브로커는 해당 컨슈머가 죽었다고 판단하고 리밸런싱 유발.
    *   `heartbeat.interval.ms`: `session.timeout.ms` 보다 낮아야 하며, 이 주기로 하트비트 전송.
    *   `max.poll.interval.ms`: `poll()` 호출 간의 최대 시간. 메시지 처리 시간이 이 값을 넘으면 컨슈머가 그룹에서 제외되고 리밸런싱 유발.
    *   **파티션 할당 전략** (`partition.assignment.strategy`): `RangeAssignor`, `RoundRobinAssignor`, `StickyAssignor` (리밸런싱 시 파티션 이동 최소화), `CooperativeStickyAssignor` (Stop-the-world 리밸런싱 방지, 점진적 협력 리밸런싱)

### 6. 오류 처리: 재시도 토픽 & DLT (Dead Letter Topic)

*   **6.1 Spring Kafka `@RetryableTopic` / `NonBlockingRetryableTopic`**
    *   **별도 retry 토픽** 생성 → 원본 파티션 메시지 처리 지연 방지(Head-of-Line Blocking 회피)
    *   단계별 지연(backoff), 재시도 횟수, 파티션 수, 재시도 토픽의 TTL(메시지 만료 시간) 설정 가능.
    *   최종 실패 시 DLT 토픽으로 이동 후 알림·분석.

*   **6.2 순서 보장 이슈**
    *   **파티션 단위만** 순서 보장 기본 원칙.
    *   Retry/DLT 토픽 사용 시, 실패한 메시지가 재시도 토픽으로 갔다가 다시 처리될 때 원래 순서와 달라질 수 있음.
    *   **보강:** 원본 토픽과 재시도 토픽의 파티션 수를 동일하게 하고, 동일한 키로 파티셔닝하면 특정 키에 대한 순서는 최대한 유지하려 하지만, 재시도 과정 자체로 인해 엄격한 순서 보장은 깨질 수 있음. 순서가 매우 중요하다면, 재시도 로직을 컨슈머 내에서 동기적으로 처리하거나 (처리량 저하 감수), 상태 저장소를 활용한 별도 순서 관리 필요.

### 7. Outbox 패턴

*   **7.1 기본 플로우**
    1.  **DB 트랜잭션** 내에서 비즈니스 로직 처리 + Outbox 테이블에 이벤트(메시지) INSERT
    2.  해당 DB 트랜잭션 커밋
    3.  별도의 프로세스(Debezium CDC, 스케줄러, 애플리케이션 리스너 등)가 Outbox 테이블 변경 감지/폴링하여 Kafka로 메시지 `send()` 시도
    4.  Kafka 전송 성공 시 Outbox 테이블에서 해당 이벤트 상태 변경(예: '발송됨') 또는 삭제 (삭제는 데이터 추적 어려움)

*   **7.2 멱등성·중복 방지 (Outbox 발행 시점)**
    *   Kafka 프로듀서의 멱등성(`enable.idempotence=true`)은 프로듀서 내부 자동 재시도에 대한 중복 방지. Outbox 발행기가 메시지를 `send()`하고 응답 받기 전 실패 후 재시작하여 동일 메시지를 다시 `send()`하면, 이는 새로운 `send()` 호출로 간주되어 Kafka 멱등성만으로는 중복 방지 불가.
    *   **해결책:**
        *   **메시지 키(예: Outbox 이벤트 ID) + 컨슈머 레벨 중복 제거:** 컨슈머 측에서 이미 처리한 이벤트 ID인지 확인.
        *   **발행 상태 관리:** Outbox 테이블에 발행 상태 (pending, sent)를 두고, 'sent' 상태면 재발행 방지.
        *   **트랜잭셔널 프로듀서 (Kafka Streams 등 연동 시):** Outbox 테이블 읽기 + Kafka send + Outbox 상태 변경을 하나의 트랜잭션으로 묶는 것은 일반 애플리케이션에서는 복잡. CDC 도구와 연동 시 이점을 가질 수 있음.

*   **7.3 트랜잭셔널 Outbox (Spring Kafka 예시)**
    *   **수정/보강:** 일반적인 DB 트랜잭션과 Kafka 트랜잭션을 하나의 원자적 연산으로 묶는 것은 분산 트랜잭션(XA)이 필요하여 매우 복잡하고 권장되지 않음. `executeInTransaction`은 Kafka 프로듀서의 트랜잭션 범위 내에서 여러 `send()`를 원자적으로 처리하는 것.
    *   **더 현실적인 트랜잭셔널 접근 (At-least-once + 멱등적 컨슈머 또는 Exactly-once Sink):**
        1.  DB TX: 비즈니스 로직 + Outbox 테이블 INSERT (커밋)
        2.  CDC (Debezium 등) 또는 폴러가 Outbox 테이블 읽어서 Kafka로 전송 (At-least-once 보장).
        3.  컨슈머는 멱등하게 설계되거나, Kafka Connect Sink 커넥터가 Exactly-once 지원 시 최종 시스템에 정확히 한 번 반영.
    *   **참고:** `transactional.id`를 사용하는 프로듀서로 Outbox 메시지를 보내고, 메시지 처리 후 Outbox 레코드 상태를 업데이트하는 로직을 구성할 수 있지만, DB 업데이트는 Kafka 트랜잭션 범위 밖임.

### 8. 파티션 증설(Scale-Out)

*   **8.1 파티션 늘리기**
    *   CLI: `kafka-topics.sh --alter --topic <topic_name> --partitions <new_partition_count>` (늘릴 때만 가능, 줄이는 것은 지원 안 함)
    *   Java AdminClient: `adminClient.createPartitions(Map<String, NewPartitions> newPartitionsMap)` (예: `NewPartitions.increaseTo(N)`)
    *   **주의:** 파티션 증설 시 키 기반 파티셔닝 전략을 사용 중이었다면, 기존 키와 새 파티션 간의 매핑이 변경될 수 있음. (즉, 동일 키가 이전과 다른 파티션으로 갈 수 있음). 기존 데이터는 재분배되지 않음.

*   **8.2 파티션 수 vs Consumer 인스턴스**
    *   컨슈머 인스턴스 수 ≤ 파티션 수. 컨슈머 수가 파티션 수보다 많으면 일부 컨슈머는 유휴 상태(idle).
    *   1:1 매핑(컨슈머 수 = 파티션 수)으로 최대 병렬 처리. 여유분(파티션 > 소비자)을 두면 특정 컨슈머 장애 시 다른 컨슈머로 빠르게 작업 이전 가능 (스케일 유연성).

*   **8.3 스케일 업 vs 스케일 아웃**
    *   **스케일 업**: 개별 브로커의 자원(CPU, RAM, Disk I/O, Network) 강화. 한계 존재.
    *   **스케일 아웃**:
        *   **브로커 추가**: 클러스터 전체 처리 용량 및 저장 공간 증가. 이후 파티션 재배치 필요.
        *   **파티션 수 증가**: 특정 토픽의 병렬 처리 능력 향상.

### 9. 브로커 확장 & 복제본 배치

*   RF=N 이면 파티션당 Leader + (N-1) Followers.
*   **브로커 수** ≥ RF 권장 (가용성 측면. 예를 들어 RF=3이면 최소 3대 브로커 필요).
*   **총 복제본 인스턴스 수** = (토픽 A 파티션 수 × RF) + (토픽 B 파티션 수 × RF) + ...
*   새 브로커 추가 후, **파티션 재배치 도구**(`kafka-reassign-partitions.sh`, Cruise Control 등)를 사용하여 기존/신규 파티션의 리더 및 팔로워 복제본들을 새 브로커 포함 전체 클러스터에 균등 분산시켜야 함.
*   **Follower** 는 기본적으로 클라이언트 요청(읽기/쓰기)을 처리하지 않음 (리더를 통해서만). 읽기 요청을 팔로워에서 처리하도록 하는 기능(Follower Fetching)은 별도 설정 및 고려사항 필요.
*   **추가:** **랙 인식(Rack Awareness)** 설정: 브로커들을 다른 물리적 랙에 배치하고 Kafka에 이 정보를 알려주면, Kafka는 복제본들을 서로 다른 랙에 분산시켜 랙 전체 장애 시에도 데이터 유실 방지 및 가용성 향상.

### 10. Kafka Streams vs 일반 Consumer API

| 구분                   | Consumer API (e.g., `KafkaConsumer`)         | Kafka Streams                                                  |
| -------------------- | -------------------------------------------- | -------------------------------------------------------------- |
| 추상화 수준               | 저수준 (메시지 폴링, 오프셋 커밋 등 직접 제어)     | 고수준 (DSL: `map`, `filter`, `join`, `aggregate` 등) + 저수준 Processor API |
| 상태 관리(Stateful)      | 애플리케이션 코드에서 직접 구현 (외부 DB/인메모리 등) | 내장 StateStore (기본 RocksDB) 사용, 변경 사항은 changelog 토픽으로 백업 및 복구 |
| 스트림 처리 오퍼레이터     | 직접 구현                                     | 풍부한 내장 오퍼레이터 제공 (Windowing, Joins, Aggregations 등)  |
| Exactly-Once 시맨틱스   | 프로듀서/컨슈머 트랜잭션 직접 조합 등 복잡한 구현 필요 | 설정(`processing.guarantee=exactly_once` 또는 `exactly_once_v2`)으로 비교적 쉽게 달성 (내부적으로 트랜잭션 활용) |
| 흐름 제어(Back-pressure) | `pause()`/`resume()`, `max.poll.records` 등 수동 제어 | 태스크별 버퍼링(`buffered.records.per.partition`), 내부적으로 컨슈머 API 활용. 애플리케이션 레벨에서 세밀한 제어는 복잡할 수 있음. |
| 병렬화/스레딩 모델        | 컨슈머 인스턴스 단위. 각 인스턴스는 자체 스레드 또는 스레드풀 운영. | 스트림 태스크(StreamTask) 단위. `num.stream.threads`로 스레드 수 지정. 각 스레드가 여러 태스크 처리 가능. |
| 비동기/외부 연동         | 직접 스레드풀, CompletableFuture 등으로 구현      | Processor API 사용 시 가능. DSL에서는 주로 동기적 연산. 비동기 처리는 KIP-472 External Async Service Lookup 등의 패턴 고려. |
| 배포 모델                | 일반 Java 애플리케이션 (독립 실행, 컨테이너 등)   | 경량 라이브러리. 일반 Java 애플리케이션에 임베드 가능. 별도 클러스터 불필요. |



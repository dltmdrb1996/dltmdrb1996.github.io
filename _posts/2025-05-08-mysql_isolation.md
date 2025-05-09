---
title: "MySQL 트랜잭션 격리 수준과 내부 구조 정리" # 또는 "Kafka 완벽 정복: 핵심 개념부터 실전 운영까지 (종합 가이드)"
date: 2025-05-08
categories: [기술, DB, 데이터]
tags: [MySQL, InnoDB, Transaction, IsolationLevel, MVCC, Spring, Database, Concurrency]
toc: true
comments: true
# image:
#  path: /assets/img/posts/kafka-banner.png
#  alt: Kafka 로고와 데이터 스트림 이미지
# pin: true
---

## MySQL 트랜잭션 격리 수준과 내부 구조 정리

데이터베이스 시스템, 특히 동시 사용자가 많은 환경에서 데이터의 정합성과 일관성을 유지하는 것은 매우 중요합니다. 이를 위해 MySQL과 같은 RDBMS는 트랜잭션과 격리 수준이라는 개념을 제공합니다. 이 문서는 MySQL (InnoDB 스토리지 엔진 기준) 트랜잭션의 내부 구조와 격리 수준별 동작 방식, 그리고 Spring 프레임워크에서의 활용법을 심층적으로 다룹니다.

### 1. 트랜잭션과 격리 수준의 필요성

여러 트랜잭션이 동시에 실행될 때, 서로의 작업에 영향을 미쳐 데이터 정합성 문제가 발생할 수 있습니다. 예를 들면 다음과 같은 현상들이 있습니다:

*   **Dirty Read**: 한 트랜잭션이 아직 커밋되지 않은 다른 트랜잭션의 변경 내용을 읽는 현상.
*   **Non-Repeatable Read**: 한 트랜잭션 내에서 같은 쿼리를 두 번 실행했을 때, 그 사이에 다른 트랜잭션이 값을 수정/삭제하고 커밋하여 결과가 다르게 나타나는 현상.
*   **Phantom Read**: 한 트랜잭션 내에서 같은 쿼리를 두 번 실행했을 때, 첫 번째 쿼리에서는 없던 레코드가 두 번째 쿼리에서 나타나는 현상 (다른 트랜잭션이 새 레코드를 삽입하고 커밋).

이러한 문제를 제어하기 위해 SQL 표준은 4가지 트랜잭션 격리 수준을 정의합니다.

### 2. 트랜잭션 격리 수준(Isolation Level)의 기본 개념

SQL 표준에서 정의하는 4가지 격리 수준은 다음과 같으며, 아래로 갈수록 격리 수준이 높아지지만 동시성은 저하될 수 있습니다.

1.  **READ UNCOMMITTED**:
    *   커밋되지 않은 데이터(Dirty Data)를 다른 트랜잭션이 읽는 것을 허용합니다.
    *   Dirty Read, Non-Repeatable Read, Phantom Read 모두 발생 가능합니다.
    *   정합성에 문제가 많아 거의 사용되지 않습니다.

2.  **READ COMMITTED**:
    *   트랜잭션은 커밋된 데이터만 읽습니다. Dirty Read를 방지합니다.
    *   하지만 Non-Repeatable Read와 Phantom Read는 발생 가능합니다.
    *   많은 RDBMS(Oracle, SQL Server 등)의 기본 격리 수준입니다.

3.  **REPEATABLE READ**:
    *   트랜잭션이 시작된 후 다른 트랜잭션에 의해 데이터가 변경되더라도, 해당 트랜잭션 내에서는 일관된 결과를 보장합니다 (Non-Repeatable Read 방지).
    *   MySQL InnoDB의 기본 격리 수준이며, 갭 락(Gap Lock)과 넥스트 키 락(Next-Key Lock)을 통해 Phantom Read도 대부분 방지합니다.

4.  **SERIALIZABLE**:
    *   가장 엄격한 격리 수준으로, 트랜잭션을 순차적으로 실행하는 것과 동일한 결과를 보장합니다.
    *   모든 종류의 동시성 문제를 방지하지만 (Dirty Read, Non-Repeatable Read, Phantom Read 모두 방지), 동시 처리 성능이 가장 낮습니다.
    *   일반적인 SELECT 문에도 공유 락(Shared Lock)이 걸립니다.

### 3. MySQL (InnoDB)의 트랜잭션 구현 핵심 요소

InnoDB 스토리지 엔진은 ACID 특성을 보장하기 위해 다음과 같은 내부 구성 요소와 메커니즘을 활용합니다.

*   **MVCC (Multi-Version Concurrency Control)**:
    *   데이터를 읽을 때 락을 사용하지 않고, 데이터의 여러 버전을 관리하여 동시성을 높이는 기술입니다.
    *   트랜잭션이 시작될 때의 데이터베이스 상태를 기준으로 **스냅샷(Read View)**을 생성합니다.
    *   **Read View**는 해당 트랜잭션이 읽을 수 있는 데이터 버전의 범위를 결정하며, 다음 정보를 포함합니다:
        *   `creator_trx_id`: Read View를 생성한 트랜잭션의 ID.
        *   `min_trx_id` (또는 `low_limit_id`): Read View 생성 시점의 가장 작은 활성 트랜잭션 ID. 이 ID보다 작은 트랜잭션 ID를 가진 변경은 모두 보입니다.
        *   `max_trx_id` (또는 `up_limit_id`): Read View 생성 시점의 가장 큰 (다음에 할당될) 트랜잭션 ID. 이 ID보다 크거나 같은 트랜잭션 ID를 가진 변경은 보이지 않습니다.
        *   `active_trx_ids[]` (또는 `ids`): Read View 생성 시점의 활성(아직 커밋/롤백되지 않은) 트랜잭션 ID 목록. 이 목록에 있는 트랜잭션 ID를 가진 변경은 보이지 않습니다.
    *   각 레코드에는 숨겨진 필드로 `DB_TRX_ID`(데이터를 마지막으로 변경한 트랜잭션 ID)와 `DB_ROLL_PTR`(Undo 로그 포인터)가 있습니다.
    *   `SELECT` 쿼리는 자신의 Read View와 레코드의 `DB_TRX_ID`를 비교하고, 필요시 `DB_ROLL_PTR`을 따라 Undo 로그에서 이전 버전을 찾아 읽습니다.

*   **Undo 로그**:
    *   **목적**:
        1.  **트랜잭션 롤백**: 트랜잭션이 실패했을 때 변경 이전 상태로 복구합니다.
        2.  **MVCC 지원**: 다른 트랜잭션이 특정 시점의 데이터를 읽을 수 있도록 이전 버전의 데이터를 제공합니다.
    *   **동작**: 데이터 변경 시, 변경 전의 값을 Undo 로그에 기록합니다.
    *   **정리**: 더 이상 어떤 트랜잭션의 Read View에도 필요하지 않은 Undo 로그는 Purge 스레드에 의해 주기적으로 삭제됩니다.

*   **Redo 로그**:
    *   **목적**: 시스템 장애 발생 시 데이터 손실을 방지하고 **Durability(지속성)**를 보장합니다. (조회에는 사용되지 않음)
    *   **동작 (WAL, Write-Ahead Logging)**:
        1.  데이터 변경이 발생하면, 변경 사항을 실제 데이터 파일에 쓰기 전에 Redo 로그에 먼저 기록합니다.
        2.  트랜잭션 커밋 시, Redo 로그가 디스크에 안전하게 저장되었음을 보장합니다 (설정에 따라 다름).
        3.  이후 Buffer Pool의 변경된 데이터(Dirty Page)는 비동기적으로 디스크에 반영(flush)됩니다.
    *   **장애 복구**: 시스템 비정상 종료 후 재시작 시, Redo 로그를 사용하여 커밋되었지만 디스크에 완전히 반영되지 않은 변경 사항을 재현합니다.

*   **Buffer Pool**:
    *   InnoDB의 메인 메모리 영역으로, 디스크의 데이터 파일이나 인덱스 정보를 캐싱합니다.
    *   데이터는 16KB 페이지 단위로 관리됩니다.
    *   모든 데이터 읽기 및 쓰기 작업은 Buffer Pool을 통해 이루어집니다.
    *   메모리에서 변경되었지만 아직 디스크에 반영되지 않은 페이지를 **Dirty Page**라고 하며, 이는 백그라운드 스레드에 의해 디스크로 비동기적으로 flush됩니다.
    *   **❓ 트랜잭션이 커밋되고 디스크 반영 전인 경우, SELECT는 어디서 값을 읽는가?**
        *   ✅ SELECT는 기본적으로 Buffer Pool의 데이터를 읽습니다. 만약 트랜잭션이 커밋되었지만 해당 변경 사항이 아직 디스크에 flush되지 않았다면, Buffer Pool의 Dirty Page에 있는 최신 커밋된 데이터를 읽습니다. MVCC 규칙에 따라 현재 트랜잭션이 해당 버전을 볼 수 있어야 하며, 만약 현재 트랜잭션의 Read View가 더 이전 시점의 데이터를 요구한다면 Undo 로그를 참조하여 과거 버전을 가져옵니다.

*   **Doublewrite Buffer**:
    *   부분 페이지 쓰기(Partial Page Write)로 인한 데이터 손상(Torn Page)을 방지하기 위한 메커니즘입니다.
    *   Dirty Page를 실제 데이터 파일에 쓰기 전에, 먼저 연속된 디스크 공간인 Doublewrite Buffer에 기록합니다.
    *   이후 실제 데이터 파일 위치에 해당 페이지를 씁니다.
    *   만약 데이터 파일 쓰기 중 장애가 발생하면, Doublewrite Buffer의 내용을 사용하여 페이지를 복구할 수 있습니다.

### 4. 각 격리 수준에서의 InnoDB 동작 상세

*   **READ UNCOMMITTED**:
    *   락을 거의 사용하지 않으며, 다른 트랜잭션이 커밋하지 않은 변경 내용도 그대로 읽습니다.
    *   InnoDB에서도 이 수준은 MVCC를 거의 활용하지 않고 가장 최신 데이터를 읽으려 시도합니다.

*   **READ COMMITTED**:
    *   **SELECT 쿼리마다 새로운 Read View(스냅샷)를 생성**합니다.
    *   따라서 한 트랜잭션 내에서도 SELECT를 실행할 때마다 다른 트랜잭션이 중간에 커밋한 변경 사항이 반영될 수 있어 Non-Repeatable Read가 발생합니다.
    *   **❓ 왜 READ COMMITTED는 스냅샷을 쓰는데도 결과가 바뀌는가?**
        *   ✅ READ COMMITTED는 "트랜잭션 단위 스냅샷"이 아닌 "SQL 문장 단위 스냅샷"을 사용하기 때문입니다. 각 SELECT 문이 실행될 때마다 새로운 Read View를 생성하므로, 그 사이에 다른 트랜잭션이 데이터를 변경하고 커밋했다면 새로운 Read View는 이 최신 커밋된 데이터를 반영하게 됩니다.

*   **REPEATABLE READ (MySQL InnoDB 기본값)**:
    *   **트랜잭션이 시작될 때 단 한 번 Read View(스냅샷)를 생성**하고, 트랜잭션이 끝날 때까지 모든 SELECT는 이 스냅샷을 기준으로 데이터를 읽습니다.
    *   따라서 트랜잭션 도중 다른 트랜잭션이 데이터를 변경하고 커밋하더라도, 현재 트랜잭션은 자신이 시작될 때의 데이터 버전을 일관되게 읽습니다 (Undo 로그 참조).
    *   MySQL InnoDB는 여기에 추가로 **Next-Key Lock**을 사용하여 Phantom Read 현상도 방지합니다. (일반적인 SELECT에는 락이 걸리지 않고 MVCC로 처리되지만, 특정 조건이나 `FOR UPDATE`, `LOCK IN SHARE MODE` 사용 시 Next-Key Lock이 동작)
    *   **❓ REPEATABLE READ에서도 내 트랜잭션 내의 UPDATE 결과는 SELECT로 반영되나?**
        *   ✅ 네, 반영됩니다. MVCC와 Read View는 *다른* 트랜잭션의 변경 사항으로부터 현재 트랜잭션을 격리하는 것이 주 목적입니다. 현재 트랜잭션 *자신*이 수정한 내용은 항상 최신 상태로 즉시 조회됩니다. Read View는 자신의 `creator_trx_id`를 알고 있어, 자신이 변경한 내용은 필터링하지 않습니다.

*   **SERIALIZABLE**:
    *   가장 강력한 격리 수준으로, 모든 SELECT 문에도 공유 락(Shared Lock)을 설정합니다. `SELECT ... FOR UPDATE`나 `SELECT ... LOCK IN SHARE MODE`를 사용하지 않아도 락이 걸립니다.
    *   읽기 작업도 쓰기 작업과 마찬가지로 다른 트랜잭션의 접근을 제한하므로 동시성은 매우 낮지만 데이터 정합성은 완벽하게 보장됩니다.
    *   Gap Lock, Range Lock 등 사용 가능한 모든 락을 적극적으로 사용하여 동시 접근을 차단합니다.

### 5. 락(Lock)의 종류와 적용

InnoDB는 MVCC를 통해 일반적인 `SELECT` 문에서는 락 없이 일관된 읽기를 제공하지만, 데이터 변경(INSERT, UPDATE, DELETE)이나 특정 읽기(`SELECT ... FOR UPDATE`, `SELECT ... LOCK IN SHARE MODE`) 시에는 락을 사용합니다.

*   **Record Lock**: 인덱스 레코드 자체에 거는 락입니다.
*   **Gap Lock**: 인덱스 레코드 사이의 간격(갭)에 거는 락입니다. 다른 트랜잭션이 이 간격에 새로운 데이터를 삽입하는 것을 방지하여 Phantom Read를 막는 데 기여합니다. 갭 락 자체는 특정 레코드를 잠그지는 않습니다.
*   **Next-Key Lock**: Record Lock과 Gap Lock을 합친 형태입니다. 인덱스 레코드를 잠그고, 그 레코드 이전의 간격까지 잠급니다. REPEATABLE READ 격리 수준에서 Phantom Read를 방지하기 위해 주로 사용됩니다.
    *   **❓ Gap Lock과 Next-Key Lock은 왜 같이 필요할까?**
        *   ✅ Record Lock만으로는 특정 범위 내에 새로운 레코드가 삽입되는 Phantom Read를 막을 수 없습니다. Gap Lock은 이 "간격"에 대한 삽입을 막고, Next-Key Lock은 기존 레코드와 그 앞의 간격을 함께 잠가 특정 범위의 일관성을 더욱 강력하게 보장합니다. REPEATABLE READ에서 Phantom Read를 방지하기 위해 이 조합이 효과적입니다.
*   **일반 `SELECT`와 락**: REPEATABLE READ 및 READ COMMITTED 격리 수준에서 일반적인 `SELECT` 문은 락을 사용하지 않고 MVCC를 통해 데이터를 읽습니다. `SERIALIZABLE` 수준에서는 모든 `SELECT` 문이 공유 락을 획득합니다.

### 6. Undo 로그 vs Redo 로그: 역할 비교

| 구분      | Undo 로그                                    | Redo 로그                                      |
| :-------- | :------------------------------------------- | :--------------------------------------------- |
| **목적**  | 1. 트랜잭션 롤백<br>2. MVCC 과거 버전 조회        | 1. 장애 발생 시 데이터 복구 (Durability 보장)       |
| **내용**  | 변경 전(Before Image)의 데이터 저장            | 변경 후(After Image)의 데이터 또는 변경 내용 저장    |
| **동작**  | 데이터 변경 시 이전 값을 기록                  | 데이터 변경 시 변경 내용을 로그 버퍼에 기록 후 디스크에 기록 (WAL) |
| **조회 시 사용** | MVCC에서 특정 시점의 데이터 조회를 위해 사용 (O) | 조회에는 사용되지 않음 (X)                     |
| **정리**  | Purge 스레드가 더 이상 필요 없는 Undo 레코드 정리 | Checkpoint 발생 시 이전 Redo 로그 정리 가능       |

*   **❓ Redo 로그는 조회 시에도 참고되는가?**
    *   ❌ 아닙니다. Redo 로그는 오직 시스템 장애 시 데이터 복구를 위해, 즉 데이터의 **지속성(Durability)**을 보장하기 위해 사용됩니다. SELECT 쿼리는 Buffer Pool의 데이터나, MVCC를 위해 Undo 로그를 참조하여 과거 버전을 읽습니다.

### 7. Redo 로그의 WAL(Write-Ahead Logging) 구조와 커밋

*   **WAL**: 데이터 변경 사항을 디스크의 데이터 파일에 직접 쓰기 전에, 변경 내용을 Redo 로그에 먼저 기록하는 방식입니다.
*   **커밋 과정**:
    1.  트랜잭션이 데이터를 변경하면, 변경 내용은 Buffer Pool에 반영되고, Redo 로그 버퍼에도 기록됩니다.
    2.  `COMMIT` 명령이 실행되면, Redo 로그 버퍼의 내용이 디스크의 Redo 로그 파일로 flush 됩니다 (이때 `fsync` 호출 여부는 설정에 따름).
    3.  Redo 로그가 디스크에 안전하게 기록되면 트랜잭션은 커밋된 것으로 간주됩니다.
    4.  Buffer Pool의 Dirty Page(변경되었으나 아직 디스크에 반영되지 않은 데이터)는 나중에 백그라운드 스레드에 의해 비동기적으로 디스크 데이터 파일에 flush됩니다.
*   **Checkpoint**: Buffer Pool의 Dirty Page를 디스크로 flush하고, 해당 시점까지의 Redo 로그를 더 이상 복구에 필요하지 않도록 표시하는 작업입니다.
*   **`innodb_flush_log_at_trx_commit` 설정**:
    *   `0`: 매초마다 Redo 로그 버퍼를 디스크로 flush하고 `fsync` 호출. 커밋 시에는 아무 작업도 하지 않음. 성능은 가장 좋지만, OS 다운 시 1초간의 트랜잭션 유실 가능.
    *   `1` (기본값): 매 트랜잭션 커밋 시마다 Redo 로그 버퍼를 디스크로 flush하고 `fsync` 호출. 가장 안전하지만 성능 저하가 가장 큼 (ACID의 D를 완벽히 보장).
    *   `2`: 매 트랜잭션 커밋 시마다 Redo 로그 버퍼를 OS 캐시까지만 flush하고, 1초마다 OS 캐시에서 디스크로 `fsync` 호출. DB 서버 다운에는 안전하지만, OS 다운 시 1초간의 트랜잭션 유실 가능.

### 8. Spring에서의 트랜잭션 격리 수준 설정

Spring 프레임워크는 선언적 트랜잭션 관리를 통해 개발자가 비즈니스 로직에 집중할 수 있도록 지원합니다. `@Transactional` 어노테이션을 사용하여 트랜잭션의 격리 수준, 전파 속성, 타임아웃 등을 설정할 수 있습니다.

*   **`@Transactional(isolation = Isolation.<LEVEL>)`**:
    *   `Isolation.DEFAULT`: 사용하는 DataSource의 기본 격리 수준을 따름 (MySQL InnoDB는 REPEATABLE_READ).
    *   `Isolation.READ_UNCOMMITTED`
    *   `Isolation.READ_COMMITTED`
    *   `Isolation.REPEATABLE_READ`
    *   `Isolation.SERIALIZABLE`
*   **동작 원리**: Spring은 AOP(Aspect-Oriented Programming)를 기반으로 트랜잭션 부가기능을 제공합니다. `@Transactional`이 붙은 메소드가 호출되면, Spring 트랜잭션 매니저는 실제로는 JDBC의 `Connection.setTransactionIsolation()` 메소드를 호출하여 해당 트랜잭션의 격리 수준을 설정합니다.
*   **주의사항 (커넥션 풀 사용 시)**:
    *   HikariCP와 같은 커넥션 풀을 사용할 경우, 커넥션이 재사용됩니다. 만약 특정 트랜잭션에서 격리 수준을 변경했다면, 해당 커넥션이 풀에 반환된 후 다른 트랜잭션에서 재사용될 때 이전 격리 수준이 남아있을 수 있습니다 (커넥션 풀 구현에 따라 다를 수 있지만, 일반적으로 Spring은 트랜잭션 시작 시 설정하고 종료 시 복원).
    *   **그러나 가장 안전한 방법은 필요한 격리 수준을 `@Transactional`에 명시적으로 지정하는 것입니다.** Spring은 트랜잭션 시작 시 지정된 격리 수준을 설정하고, 트랜잭션 종료 시 (커넥션 풀에 반환하기 전) 원래의 기본 격리 수준으로 복원해줍니다. 따라서 "오염" 위험은 적지만, 명시적 설정이 권장됩니다.

*   **기타 `@Transactional` 속성**:
    *   `@Transactional(readOnly = true)`: 해당 트랜잭션이 읽기 전용임을 나타냅니다. DB 드라이버나 DB 자체에서 일부 최적화를 수행할 수 있습니다 (예: 불필요한 락 방지, 복제 지연 허용 등). 또한, JPA 사용 시 flush 모드를 변경하여 Dirty Checking을 덜 수행하게 할 수 있습니다.
    *   `@Transactional(timeout = 5)`: 트랜잭션 실행 시간이 5초를 초과하면 강제로 롤백시킵니다. (초 단위)

### 9. 분산 시스템 환경에서의 락 전략

단일 DB 인스턴스를 넘어선 분산 환경에서는 DB 레벨 락만으로는 동시성 제어에 한계가 있습니다.

*   **분산 락(Distributed Lock)**: 여러 서버나 프로세스에 걸쳐 공유 자원에 대한 접근을 동기화하기 위해 사용됩니다.
    *   구현 방법: Redis(SETNX, Redlock), ZooKeeper, etcd 등을 활용합니다.
    *   **단점**: 분산 락 자체가 병목 지점이 될 수 있으며, 구현이 복잡하고 성능에 영향을 줄 수 있습니다.
    *   **❓ 분산 락을 사용하는데 왜 오토스케일 이점이 줄어드는가?**
        *   ✅ 분산 락은 공유 자원에 대한 접근을 직렬화하는 경향이 있습니다. 특정 자원에 대해 하나의 프로세스/스레드만 접근 가능하게 만들므로, 애플리케이션 서버를 스케일 아웃(오토스케일)하여 처리량을 늘리려 해도, 분산 락으로 보호되는 코드 구간에서는 병렬 처리가 제한되어 전체 시스템의 처리량 증가 효과가 기대만큼 크지 않을 수 있습니다. 즉, 락 경쟁이 심한 경우 락을 획득하기 위한 대기가 발생하여 스케일 아웃의 이점을 상쇄시킵니다.

*   **애플리케이션 레벨 락 전략**:
    *   **낙관적 락 (Optimistic Locking)**: 데이터에 버전(version) 컬럼을 두고, 업데이트 시 버전을 확인하여 충돌을 감지합니다. 충돌 시 재시도하거나 사용자에게 알립니다.
    *   **조합 전략**: 낙관적 락을 기본으로 사용하고, 특정 중요 작업에 대해서는 Redis와 같은 분산 락을 짧게 사용하여 동시 접근을 제어하는 방식이 효과적일 수 있습니다.
    *   **기타**: 스핀락, 세마포어, Actor 모델(Akka, Vert.x 등)을 활용하여 상태 관리 및 동시성 제어를 애플리케이션 로직 내에서 구현할 수도 있습니다.

### 10. 샤딩(Sharding) + 분산 락 구조

대규모 시스템에서는 데이터베이스 샤딩을 통해 부하를 분산시킵니다. 이 환경에서 분산 락을 사용할 때는 다음과 같은 점을 고려해야 합니다.

*   **글로벌 락의 병목**: 전체 시스템에 걸친 단일 분산 락은 여전히 병목이 될 수 있습니다.
*   **락 범위 분산**:
    *   **샤드 키 기반 락**: 특정 샤드 키(예: 사용자 ID)에 대해서만 락을 적용하여 락의 범위를 좁힙니다.
    *   **큐 기반 처리 (Actor 모델 등)**: 특정 자원에 대한 요청을 큐에 넣고 순차적으로 처리하는 Worker 패턴을 적용하여 락의 필요성을 줄일 수 있습니다. 이 경우 큐 자체가 특정 샤드나 노드에 할당될 수 있습니다.

### 11. 고가용성(HA) 및 데이터 무결성 전략

*   **MySQL 복제(Replication)**:
    *   Master-Slave 구조로, Master에서 발생한 변경 사항(binlog)을 Slave로 전달하여 데이터를 동기화합니다.
    *   종류: 비동기(Asynchronous), 반동기(Semi-synchronous), 그룹 복제(Group Replication - Multi-Master 지원).
    *   읽기 부하 분산, 장애 조치(Failover) 등에 활용됩니다.
*   **장애 복구 자동화**:
    *   Orchestrator, MHA (Master High Availability Manager and Tools): Master 장애 시 자동으로 Slave를 새로운 Master로 승격시키는 솔루션.
    *   ProxySQL, HAProxy + Keepalived: DB 연결을 중계하고 장애 감지 시 트래픽을 정상 노드로 라우팅하는 프록시 솔루션.
*   **데이터 무결성 확보**:
    *   **DB 제약조건**: 외래 키(Foreign Key), UNIQUE 제약조건, CHECK 제약조건 (MySQL 8.0.16+).
    *   **트랜잭션**: ACID 원칙을 준수하여 작업의 원자성, 일관성, 격리성, 지속성을 보장.
    *   **애플리케이션 레벨 검증**: DB 제약조건 외에도 애플리케이션 코드에서 비즈니스 규칙에 따른 데이터 유효성 검사를 수행하여 중복 확인 및 데이터 정합성을 강화합니다.

---

이 문서는 MySQL InnoDB의 트랜잭션 처리 방식과 격리 수준의 의미, 그리고 실제 시스템에서 고려해야 할 다양한 요소들을 정리했습니다. 데이터 정합성과 시스템 성능 사이의 균형을 맞추는 것은 어려운 일이지만, 내부 동작 원리를 이해하는 것이 올바른 설계를 위한 첫걸음이 될 것입니다.
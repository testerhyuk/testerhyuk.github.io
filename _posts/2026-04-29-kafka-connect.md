---
title: 카프카 커넥트 개념 정리 1부
date: 2026-04-29 17:55:00 +0900
categories: [Backend, Java]
tags: [Spring, Kafka]
permalink: /posts/kafka-connect-basic-concepts/
---

![kafka-connect](https://blog.nashtechglobal.com/wp-content/uploads/2024/06/Kafak-Connect-custom.png)

<br>

# 카프카 커넥트 1부

카프카 커넥트는 외부 시스템과 카프카 사이에서 데이터를 자동으로 전송하는 프레임워크다

여기서 말하는 외부 시스템은 카프카가 아닌 모든 시스템을 말한다

쉽게 말하면, DB -> Kafka 또는 Kafka -> DB로 데이터를 옮기는 작업을 코드 없이 설정만으로 할 수 있게 해주는 도구다

**카프카 커넥트가 필요한 이유가 뭘까?**

**상황 1.** 만약 DB의 변경사항을 카프카로 보내고 싶을 때

카프카 커넥터 없이 직접 코딩하게 되면

```
@Transactional
public void createOrder(OrderDto dto) {
    orderRepository.save(order);  // DB 저장
    kafkaTemplate.send("order-events", order);  // Kafka 발행
}
```

이런식으로 직접 카프카 메시지를 발행해줘야 하는데 문제가 있다

1. DB 저장은 성공했는데 카프카 메시지 발행이 실패할 가능성 -> 이벤트 유실
2. 카프카 발행은 성공했는데 DB 커밋이 실패할 가능성 -> 이벤트만 발행됨
3. 모든 서비스에 중복 코드 필요
4. 장애 발생 시 재시도 로직을 직접 구현해야함

<br>

**그런데 만약 카프카 커넥트를 사용한다면?**

```
DB → Debezium (자동 감지) → Kafka
```

DB 변경사항을 자동으로 카프카로 전송해주고
장애 시 자동으로 재시도 해주는 기능을 내장하고 있으며,
코드를 작성할 필요 없이 설정만으로 동작할 수 있다는 장점이 있다


<br>

**상황 2.** 카프카 메시지를 다른 DB에 저장하고 싶을 때

```
@KafkaListener(topics = "order-events")
public void consume(String message) {
    Order order = parse(message);
    targetRepository.save(order);  // 매번 수동 저장
}
```

문제점

1. 모든 토픽마다 Consumer 코드 작성이 필요함
2. 배치 처리, 재시도, 에러 핸들링을 직접 구현해야함
3. 성능 튜닝이 어려워짐

**카프카 커넥트를 사용한다면**

```
DB → Debezium (자동 감지) → Kafka
```

카프카 메시지를 자동으로 DB에 저장해주고
배치 처리, 재시도 해주며
코드를 작성할 필요가 없다

<br>

**그럼 카프카 커넥터를 쓰면 Consumer는 구현할 일이 없는가?**

그렇지 않다

단순히 변경 사항을 감지하거나 DB 데이터를 저장만 한다면 카프카 커넥터를 쓰는게 훨씬 효율적이지만,
만약 메시지를 받아서 비즈니스 로직을 처리해야 하는 작업이라면 Consumer를 직접 구현해서 처리해줘야 한다


```
**예시:**

카프카 커넥터로 가능:
- MariaDB 주문 테이블 변경사항 → Kafka → 통합 DB 저장 (단순 복사)

Consumer 직접 구현 필요:
- 결제 완료 이벤트 → 주문 상태를 COMPLETED로 변경하고 재고 차감 (비즈니스 로직)
```

## Task

Task는 실제로 데이터를 옮기는 작업 단위를 말한다

커넥터가 "계획"이라면 Task는 "실행"이다

좀 더 구체적으로 얘기하자면, Connector를 생성하면 카프카 커넥트가 내부적으로 Task를 여러 개 만들어서 실제 작업을 분산 처리한다

```
Debezium Connector 1개 생성
  ↓
Kafka Connect가 자동으로 Task 3개 생성
  ├─ Task 0: MariaDB 샤드 0 담당
  ├─ Task 1: MariaDB 샤드 1 담당  
  └─ Task 2: 예비 (필요 시 동작)
```

여러 개를 만드는 이유는 커넥터 1개가 모든걸 처리하면 속도가 느리기 때문에 task를 여러 개 만들어서 병렬 처리를 하기 위함이다

<br>

실제 동작 예시를 보자

**JDBC Sink Connector**

```
{
  "name": "jdbc-sink",
  "connector.class": "JdbcSinkConnector",
  "tasks.max": 3,  // Task 3개 만들어라
  "topics": "order-events"
}
```

내부 동작

```
order-events 토픽 (파티션 6개)
  ↓
Kafka Connect가 Task 3개 생성
  ├─ Task 0: 파티션 0, 1 읽어서 DB 저장
  ├─ Task 1: 파티션 2, 3 읽어서 DB 저장
  └─ Task 2: 파티션 4, 5 읽어서 DB 저장

→ 3배 빠르게 처리
```

## Task와 Connector의 관계

Connector는 작업의 "관리자"

무슨 작업을 할지 정의하고(DB -> Kafka, Kafka -> DB)
Task를 생성하고 관리할 수 있으며
설정을 검증하는 역할이다

반면, Task는 작업의 "실행자"

실제 데이터를 전송하고, 병렬 처리가 가능하다

## CDC (Change Data Capture)

CDC는 데이터베이스의 변경 사항을 실시간으로 추적하는 기술이다

CDC는 실시간으로 데이터를 감지할 수 있고, DB 부하가 낮다(쿼리 없이 로그만 읽기 때문)

또한 DELETE 감지도 가능하고, 변경 전 / 후 값을 다 알 수가 있다

```
DB에 변경 발생
  ↓
DB가 자동으로 변경 로그(binlog, WAL)에 기록
  ↓
CDC 도구가 로그를 읽어서 감지
  ↓
변경사항을 Kafka로 실시간 전송
```

**CDC는 어떻게 동작할까?**

모든 DB는 변경 로그를 자동으로 기록한다. 원래는 복제나 장애 복구용으로 기록하는건데 CDC는 이걸 활용해서 변경사항을 추적한다

DB별 변경 로그

- MySQL / MariaDB : binlog (Binary Log)
- PostgreSQL : WAL (Write-Ahead Log)
- Oracle : Redo Log
- MongoDB : Oplog

**binlog? WAL?**

- DB의 모든 변경사항을 기록하는 파일이다

```
오전 10시: orders 테이블에 INSERT
오전 10시 5분: users 테이블에 UPDATE
오전 10시 10분: products 테이블에 DELETE
```

이런식으로 모든 변경을 기록한다

<br>

실제 동작 방식

```
1. 주문 생성
   INSERT INTO orders (id, user_id, amount) 
   VALUES (123, 456, 50000);

2. MariaDB가 binlog에 자동 기록
   [timestamp: 12:00:01]
   [type: INSERT]
   [table: orders]
   [data: {id:123, user_id:456, amount:50000}]

3. CDC 도구가 binlog 읽음
   "어? orders 테이블에 INSERT 발생했네?"

4. Kafka로 발행
   {
     "op": "c",  // create
     "before": null,
     "after": {id:123, user_id:456, amount:50000}
   }
```

**CDC의 핵심 개념**

1. Non-Intrusive (비침투적)
    - 애플리케이션 코드 수정이 불필요하고
    - DB에 영향도 없음
    - CDC 도구만 추가하면 끝

2. Real time (실시간)
    - 변경 발생 즉시 감지가 가능함
    - 폴링(주기적 조회)보다 훨씬 빠름
        - 폴링 방식은 주기적으로 SELECT 쿼리를 날리기 때문에 DB 부하고 높고 실시간성이 떨어진다
        - outbox 패턴같은 경우 이벤트를 즉시 전달해야하기 때문에 실시간성이 중요해서 CDC 방식을 쓴다

3. Complete History (완전한 이력)
    - INSERT, UPDATE, DELETE 전부 감지 가능
    - 변경 전 값, 변경 후 값 둘다 제공한다

## Debezium

Debezium이란 CDC를 구현한 오픈 소스 도구를 말한다

CDC는 개념이고, Debezium은 그 개념을 실제로 구현한 도구다

```
CDC (개념)
  ↓ 구현체
Debezium, Maxwell, Canal 등
```

Debezium의 특징

1. 카프카 커넥트 플러그인으로 존재한다
    - 카프카 커넥트 자체는 빈 껍데기다. 플러그인을 꽂아야 동작한다. Debezium은 카프카 커넥트를 동작시키기 위한 플러그인이다
        - Debezium 플러그인이라면 DB -> kafka
        - JDBC Sink 플러그인이라면 Kafka -> DB
        - S3 Sink 플러그인이라면 Kafka -> S3
        - 이런식이다
    - 카프카 커넥트 위에서 동작하고
    - Source Connector로 동작
        - 무슨 말이냐면, Debezium은 오직 DB -> Kafka 방향으로만 가능하다는 의미다
    - 코드 없이 설정만으로 사용 가능하다

2. 다양한 DB 지원
    - MySQL/MariaDB
    - PostgreSQL
    - MongoDB
    - Oracle
    - SQL Server
    - Cassandra

3. 변경 이벤트 상세정보를 제공한다

```
{
  "before": null,  // 변경 전 값 (INSERT라서 null)
  "after": {       // 변경 후 값
    "order_id": "order-123",
    "user_id": "user-456",
    "amount": 50000,
    "status": "PENDING"
  },
  "op": "c",       // operation: c(create), u(update), d(delete)
  "ts_ms": 1710000000000,  // 타임스탬프
  "source": {
    "db": "ecommerce",
    "table": "orders"
  }
}
```

**그럼 이 Debezium은 어떻게 binlog를 읽을 수 있을까?**

우선 DB에서 binlog를 활성화 해야 한다

```
-- MariaDB 설정
SET GLOBAL binlog_format = 'ROW';
SET GLOBAL binlog_row_image = 'FULL';
```

여기서 binlog_format은 binlog가 변경사항을 기록하는 방식을 정한다
3가지 모드가 있는데

1. STATEMENT (기본값)

```
-- binlog에 이렇게 기록
INSERT INTO orders (id, amount) VALUES (123, 50000);
UPDATE users SET login_count = login_count + 1 WHERE id = 456;
```

이런식으로 SQL 문장 자체를 기록한다

그런데 이 방식의 문제는

```
UPDATE orders SET created_at = NOW() WHERE user_id = 123;
```

위와 같은 코드가 있을 때

    - Master에서 실행 -> NOW() = 12:00:00
    - Slave에서 재실행 -> NOW() = 12:00:05 (5초 지연)
    - 즉, 결과가 달라져서 Debezium이 변경된 값을 정확히 알 수 없다

2. ROW

```
-- binlog에 이렇게 기록 (실제 변경된 데이터)
### INSERT INTO orders
### SET
###   @1=123  (id)
###   @2=50000  (amount)
###   @3='2025-04-29 12:00:00'  (created_at)
```

실제 변경된 값을 기록한다

복제시 데이터 일치를 보장하고, Debezium이 변경 전/후 값을 정확히 파악가능하다

단점으로는 파일 크기가 크다는 문제가 있다

3. MIXED

안전한건 ROW로, 간단한건 STATEMENT로 자동으로 섞어서 쓰는 방식이다

<br>

그 다음으로 binlog_row_image는 ROW 모드일 때, 얼마나 자세히 기록할지를 설정하는 옵션이다

여기도 마찬가지로 3가지 옵션이 있는데

1. MINIMAL

```
UPDATE orders SET amount = 60000 WHERE id = 123;

binlog 기록:
before: {id: 123}  // PK만
after: {id: 123, amount: 60000}  // PK + 변경된 컬럼만
```

이 옵션은 필요한 것만 최소로 보여주는 옵션이다

```
# 변경 전
{id: 123}  // PK만
→ 어떤 row인지 식별만 가능하면 됨

# 변경 후
{id: 123, amount: 60000}  // PK + 변경된 컬럼만
→ 뭐가 바뀌었는지만
```

이 옵션에서 문제는 변경 안 된 컬럼 값을 알 수가 없게 되므로 Debezium이 전체 데이터를 카프카로 보낼 수 없다

binlog 크기가 가장 작은 옵션이며, binlog 크기를 최소화하고 싶거나, 단순 복제만 필요할 때(CDC 안 쓸 때) 사용 가능하다

2. NOBLOB

```
before: {id: 123, user_id: 456, amount: 50000}  // TEXT/BLOB 빼고 전부
after: {id: 123, user_id: 456, amount: 60000}
```

TEXT / BLOB 빼고 다 보내는 옵션이다

TEXT는 말 그래도 긴 문자열을 저장할 수 있는 타입이고
BLOB은 Binary Large Object, 바이너리 데이터 저장용 타입이다

```
# 변경 전
{
  id: 123,
  user_id: 456,
  amount: 50000,
  status: 'PENDING'
  // description은 TEXT라서 제외
}

# 변경 후
{
  id: 123,
  user_id: 456,
  amount: 60000,  // 변경됨
  status: 'PENDING'
  // description은 TEXT라서 제외
}
```

이 옵션의 문제는 TEXT / BLOB 컬럼 값을 모른다
큰 데이터는 빠지지만 Debezium이 완전한 데이터를 보낼 수 없다

binlog 크기는 중간정도이며, 복제 성능이 중요하고 큰 컬럼은 별도로 처리할 때 사용하는 옵션이다

3. FULL

```
before: {id: 123, user_id: 456, amount: 50000, status: 'PENDING', ...}  // 모든 컬럼
after: {id: 123, user_id: 456, amount: 60000, status: 'PENDING', ...}
```

이 옵션은 모든 컬럼을 다 포함하는 옵션이다

Debezium에서는 FULL을 보통 쓰는데, Debezium이 변경 전 값과 변경 후 값을 모두 카프카로 보내려면 전체 데이터가 필요하다

binlog는 3 옵션중 가장 크다. 완전한 데이터를 보낼 수 있다는 장점이 있지만 디스크 I/O 증가나 복제 지연이 발생할 수 있다

<br>

그 다음, Debezium의 계정을 생성하고 권한을 부여해야 한다

```
CREATE USER 'debezium'@'%' IDENTIFIED BY 'password';
GRANT SELECT, RELOAD, REPLICATION SLAVE, REPLICATION CLIENT 
ON *.* TO 'debezium'@'%';
```

그런데 왜 계정을 생성하고 권한을 부여해야 할까?

Debezium이 DB에 접속해서 binlog를 읽어야 하기 때문이다

보안상 root 계정을 쓰면 안되기 때문에 계정을 새로 생성해서 최소 권한을 부여한다

1. SELECT는 테이블 데이터를 읽을 권한이다

이는 스냅샷 모드에서 기존 데이터 전체를 읽기 위한 권한이다

```
-- Debezium이 초기 동기화 시 실행
SELECT * FROM orders;
```

2. RELOAD는 FLUSH 명령 실행 권한이다

이는 모든 테이블에 읽기 잠금을 거는 명령이다

```
FLUSH TABLES WITH READ LOCK;  -- Snapshot 찍는 동안 쓰기 차단
SHOW MASTER STATUS;  -- binlog 현재 위치 확인. 현재 binlog 파일과 위치가 기록되며
                     -- 스냅샷이 끝나면 이 위치부터 스트리밍이 시작된다
UNLOCK TABLES;

동작 흐름은 다음과 같다

Debezium Snapshot 시작
↓
FLUSH TABLES WITH READ LOCK  // 쓰기 차단
↓
SHOW MASTER STATUS  // binlog 위치 기록 (예: mysql-bin.000005, position 12345)
↓
SELECT * FROM orders  // 테이블 전체 읽기
↓
UNLOCK TABLES  // 잠금 해제
↓
기록한 위치(12345)부터 binlog 읽기 시작 (Streaming)
```

이 옵션의 효과는

    - 모든 쓰기를 차단하고 읽기만 가능하게 한다
    - 그리고 현재 실행중인 쓰기 작업이 끝날 때까지 대기한다

이 옵션을 쓰는 이유는 스냅샷을 찍는 동안 데이터가 변경되면 안되기 때문이다

3. REPLICATION SLAVE는 binlog 내용을 읽는 권한이다

```
SHOW BINLOG EVENTS;  -- binlog 내용 조회
```

실제 데이터의 변경사항을 읽는다

4. REPLICATION CLIENT는 binlog 메타데이터를 조회하는 권한이다

```
SHOW MASTER STATUS;  -- 현재 binlog 파일, 위치
SHOW BINARY LOGS;  -- binlog 파일 목록
```

binlog 내용은 보지 못하고 위치 / 상태만 확인한다

REPLICATION SLAVE와 CLIENT 둘다 필요한데, 그 이유는 SLAVE에서 실제 변경사항을 읽고, 
CLIENT에서 어디서부터 읽을지 위치를 파악해야 다음 데이터를 읽을 수 있기 때문이다

<br>

준비가 끝났으면 커넥터를 작성한다

```
{
  "name": "orders-connector", // 커넥터 이름(중복 불가)
  "connector.class": "io.debezium.connector.mysql.MySqlConnector", // 어떤 플러그인을 쓸지 결정한다. 여기서는 MySQL용 Debezium
  "database.hostname": "localhost", // DB 서버 주소
  "database.port": 3306, // DB 포트
  "database.user": "debezium", // 방금 위에서 만든 debezium DB 계정
  "database.password": "password", // 비밀번호
  "database.server.id": 1, // Debezium이 Slave처럼 동작할 때 필요한 고유 ID. 같은 DB에 여러 Debezium이 붙으면 각각 다른 ID가 필요함
  "database.server.name": "ecommerce", // 카프카 토픽 이름의 prefix
  "table.include.list": "ecommerce.orders", // 어떤 테이블을 감시할지 명시한다. "DB명.테이블명" 형식
  "database.history.kafka.topic": "schema-changes" // 스키마 변경 이력을 저장할 토픽명
}
```

이밖에도 실무에서 자주 쓰이는 옵션들은 다음과 같다

```
{
  // ===== Snapshot 설정 =====
  "snapshot.mode": "initial",  
  // - initial: 최초 실행 시 전체 스냅샷 + 이후 스트리밍
  // - schema_only: 스키마만 읽고 데이터 안 읽음
  // - never: 스냅샷 안 함, 실시간만
  // - when_needed: 오프셋 없으면 스냅샷

  // ===== 데이터 필터링 =====
  "table.exclude.list": "ecommerce.logs,ecommerce.temp",  
  // 제외할 테이블

  "column.exclude.list": "ecommerce.users.password",  
  // 특정 컬럼 제외 (보안)

  // ===== 성능 튜닝 =====
  "max.batch.size": 2048,  
  // 한 번에 처리할 레코드 수

  "max.queue.size": 8192,  
  // 내부 큐 크기

  "poll.interval.ms": 1000,  
  // binlog 폴링 주기

  // ===== Kafka 토픽 설정 =====
  "topic.prefix": "my-app",  
  // 토픽 이름 접두사 변경 (my-app.orders)

  "topic.creation.default.partitions": 3,  
  // 자동 생성 토픽 파티션 수

  // ===== 변환 설정 =====
  "transforms": "unwrap",  
  "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",  
  // before/after 구조 풀어서 after만 전달

  // ===== 오류 처리 =====
  "errors.tolerance": "all",  
  // 에러 발생 시 계속 진행

  "errors.deadletterqueue.topic.name": "dlq-orders",  
  // 실패한 메시지 보낼 DLQ 토픽

  // ===== 타임아웃 =====
  "connect.timeout.ms": 30000,  
  // DB 연결 타임아웃

  "heartbeat.interval.ms": 5000  
  // Heartbeat 주기 (연결 유지 확인)
}
```

<br>

여기까지 설정해두면 Debezium이 binlog을 읽기 시작한다

```
MariaDB
  ↓ binlog 파일 생성 (mysql-bin.000001)
Debezium이 binlog 읽음
  ↓ 변경사항 감지
Kafka 토픽으로 발행 (ecommerce.orders)
```

<br>

**스냅샷과 스트리밍**

Debezium은 두 가지 모드가 있다

1. 스냅샷 모드 (초기 데이터 복사)

```
Connector 처음 시작 시
  ↓
기존 테이블 전체 데이터를 한 번에 읽음
  ↓
Kafka로 발행 (초기 동기화)
```

예를 들어, order 테이블에 이미 데이터가 1만건이 있으면 전부 카프카로 발행한다

2. 스트리밍 모드 (실시간 변경 추적)

```
Snapshot 완료 후
  ↓
binlog를 실시간으로 읽으면서 변경사항만 감지
  ↓
Kafka로 발행
```

예를 들어, 새로운 주문이 생성되면 즉시 카프카로 발행하고 

## Source Connector vs Sink Connector

커넥터에는 두 종류가 있다

그중, Source Connector는 외부 시스템에서 데이터를 읽어서 Kafka로 보내는 역할이다

Source라는 단어 뜻 그대로, 데이터의 출발점에서 데이터를 가져온다는 뜻이다

```
외부 시스템 (Source) → Source Connector → Kafka
```

**Source Connector는 어떻게 동작할까?**

Debezium

## Converter

Converter는 데이터 형식을 변환해준다

카프카는 바이트 배열만을 저장한다. 하지만 우리는 JSON, String 같은 형식으로 데이터를 주고받는다

```
프로듀서: Order 객체 → JSON 문자열 → byte[]
                         ↓ Kafka에 저장
컨슈머: byte[] → JSON 문자열 → Order 객체
```

이러한 변환 작업을 Converter에서 해준다

이 Converter는 카프카 커넥트에서 3군데에서 사용한다

1. Key Converter - 메시지의 Key를 변환함
2. Value Converter - 메시지의 Value를 변환함
3. Header Converter - 메시지의 헤더를 변환한다

설정 예시를 보자

```
{
  "name": "debezium-connector",
  "key.converter": "org.apache.kafka.connect.json.JsonConverter",
  "value.converter": "org.apache.kafka.connect.json.JsonConverter",
  "value.converter.schemas.enable": false
}
```

Converter의 종류를 보면

1. JsonConverter

```
DB: {"userId": 123, "amount": 5000}
  ↓ JsonConverter
Kafka: {"userId": 123, "amount": 5000} (JSON 문자열 → byte[])
```

-> 사람이 읽기 쉽고, 크기가 크다

2. AvroConverter

```
DB: {"userId": 123, "amount": 5000}
  ↓ AvroConverter (스키마 기반 압축)
Kafka: [바이너리 데이터] (크기 작음)
```

-> 사람이 읽을 수 없음, 크기가 작고 스키마가 필수다

3. StringConverter

```
DB: "simple text"
  ↓ StringConverter
Kafka: "simple text" (문자열 → byte[])
```

-> 단순 문자열 전송용

<br>

**컨버터 선택 기준**

개발 / 디버깅 -> JsonConverter (사람이 보기 편함)

대용량 데이터 -> AvroConverter (압축률이 좋고 빠르기 때문)

단순 문자열 -> StringConverter (오버헤드가 없음)

스키마 관리 필요 -> AvroConverter (스키마 레지스트리 연동)
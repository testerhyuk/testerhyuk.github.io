---
title: 카프카 커넥트 개념 정리 2부
date: 2026-04-29 21:55:00 +0900
categories: [Backend, Java]
tags: [Spring, Kafka]
permalink: /posts/kafka-connect-basic-concepts-2/
---

# 카프카 커넥트 2부

## Debezium Source Connector

1부에서도 Debezium Connector 설정에 대해서 가볍게 알아봤지만

이번에는 하나하나 자세히 뜯어보자

```
{
  "name": "orders-connector",
  "connector.class": "io.debezium.connector.mysql.MySqlConnector",
  "database.hostname": "localhost",
  "database.port": 3306,
  "database.user": "debezium",
  "database.password": "password",
  "database.server.id": 1,
  "database.server.name": "ecommerce",
  "table.include.list": "ecommerce.orders",
  "database.history.kafka.topic": "schema-changes"
}
```

**name**

커넥터의 고유 이름이다

카프카 커넥트 클러스터 내에서 중복 이름이 불가능하다

REST API로 관리할 때 식별자로 사용된다

로그, 모니터링에서 이 이름으로 추적한다

예시로, 커넥터 등록하고나서 상태를 확인하거나, 삭제할 때 사용하는 명령어에서 이 이름을 활용한다

```
# Connector 상태 확인
curl http://localhost:8083/connectors/orders-connector/status

# Connector 삭제
curl -X DELETE http://localhost:8083/connectors/orders-connector
```

이 네이밍의 관례는 다음과 같다

```
{서비스명}-{역할}-connector

예: order-debezium-connector, payment-cdc-connector
```

**connector.class**

어떤 커넥터 플러그인을 사용할지 정한다

Debezium이 지원하는 커넥터

```
MySQL/MariaDB: io.debezium.connector.mysql.MySqlConnector
PostgreSQL: io.debezium.connector.postgresql.PostgresConnector
MongoDB: io.debezium.connector.mongodb.MongoDbConnector
SQL Server: io.debezium.connector.sqlserver.SqlServerConnector
Oracle: io.debezium.connector.oracle.OracleConnector
```

**database.server.id**

MySQL Replication에서 각 slave를 구분하는 고유 ID

Master는 여러 Slave를 구분해야 한다

```
Master
  ├─ Slave 1 (server_id=1) → 실제 복제 DB
  ├─ Slave 2 (server_id=2) → 백업 DB
  └─ Debezium (server_id=3) → CDC 전용
```

각 Slave / Debezium은 고유 ID를 가져야 Master가 "누가 어디까지 읽었는지"를 추적할 수 있다

```
// Debezium 1
"database.server.id": 100

// Debezium 2 (같은 DB, 다른 테이블)
"database.server.id": 101
```

**database.server.name**

카프카 토픽 이름의 prefix를 결정한다

이 부분이 매우 중요하다. 토픽 이름을 결정하기 때문

토픽 이름 규칙

```
{database.server.name}.{database명}.{테이블명}
```

예시

```
"database.server.name": "ecommerce",
"table.include.list": "shop_db.orders"

→ Kafka 토픽: ecommerce.shop_db.orders
```

이렇게 하는 이유는 한 카프카 클러스터에 여러 DB의 CDC를 보낼 수 있기 때문이다

```
ecommerce.shop_db.orders      ← ecommerce DB
ecommerce.shop_db.users

payment.pay_db.transactions   ← payment DB
payment.pay_db.refunds

이렇게 prefix로 구분한다
```

**table.include.list**

어떤 테이블을 감시할지 정한다

형식은 

```
{DB명}.{테이블명}
```

여러 테이블을 명시할 경우

```
"table.include.list": "ecommerce.orders,ecommerce.order_items,ecommerce.payments"
```

와일드카드를 쓸 경우

```
"table.include.list": "ecommerce.*"  // ecommerce DB의 모든 테이블
```

**database.history.kafka.topic**

스키마 변경 이력을 저장할 카프카 토픽명을 정한다

Debezium이 재시작될 때 테이블 구조가 어떻게 바뀌었는지 알아야 하기 때문에 이 설정이 필요하다

시나리오를 봐보자

```
1. Debezium 실행 중
   orders 테이블: (id, user_id, amount)

2. 운영 중 컬럼 추가
   ALTER TABLE orders ADD COLUMN status VARCHAR(20);
   
3. Debezium이 이 DDL을 감지하고 schema-changes 토픽에 기록

4. Debezium 재시작
   schema-changes 토픽 읽어서 "아, status 컬럼 추가됐구나" 파악
```

토픽 내용을 예시로 보면

```
{
  "source": "ecommerce",
  "position": "mysql-bin.000005:12345",
  "ts_ms": 1710000000000,
  "databaseName": "ecommerce",
  "ddl": "ALTER TABLE orders ADD COLUMN status VARCHAR(20)",
  "tableChanges": [...]
}
```

주의할점은 이 토픽을 절대 삭제하면 안된다

Debezium이 스키마 이력을 잃어버리면 재시작이 불가능해진다

이 외에도 다양한 옵션이 있다

```
"table.include.list": "ecommerce.*",  // ecommerce DB 전체

"column.exclude.list": "ecommerce.users.password,ecommerce.users.ssn" // 테이블뿐만 아니라 column도 선택 가능

"max.batch.size": 2048, // 한 번에 Kafka로 보낼 레코드 수
"max.queue.size": 8192 // 내부 버퍼 크기, binlog 읽는 속도 > Kafka 쓰는 속도 → 큐에 쌓임
```

## transforms (SMT - Single Message Transforms)

SMT란 카프카 커넥트에서 메시지를 변환하는 경량 플러그인을 말한다

커넥터가 메시지를 읽거나 쓸 때, 중간에 껴서 메시지를 변환하는 역할이다

```
Source Connector → SMT → Kafka → SMT → Sink Connector
                  (변환)          (변환)
```

보통 Source 커넥터에서 나온 데이터 형식이 Sink가 원하는 형식과 다를 때 쓴다

```
Debezium 출력:
{
  "before": {...},
  "after": {id: 123, amount: 50000},
  "op": "c"
}

JDBC Sink가 원하는 형식:
{id: 123, amount: 50000}
```

**SMT의 종류**

카프카 커넥트에 내장된 SMT들이 있다

1. ExtractNewRecordState (Debezium 전용)

  - Debezium의 before/after 구조를 풀어서 after만 추출하는 설정이다

```
{
  "transforms": "unwrap",
  "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState"
}

# 변환 전
{
  "before": null,
  "after": {
    "id": 123,
    "user_id": 456,
    "amount": 50000
  },
  "op": "c",
  "ts_ms": 1710000000000
}

# 변환 후
{
  "id": 123,
  "user_id": 456,
  "amount": 50000
}
```

여기에 아래와 같은 추가 옵션들도 있다

```
{
  "transforms": "unwrap",
  "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
  
  // DELETE 이벤트의 Tombstone 메시지 삭제 여부
  "transforms.unwrap.drop.tombstones": false,
  
  // 메타데이터 추가 (op, ts_ms 같은)
  "transforms.unwrap.add.fields": "op,ts_ms",
  
  // source 정보 추가
  "transforms.unwrap.add.headers": "db,table"
}

# add.fields 예시

{
  "id": 123,
  "user_id": 456,
  "amount": 50000,
  "__op": "c",           // 추가됨
  "__ts_ms": 1710000000000  // 추가됨
}
```

이 옵션은 Sink가 단순히 INSERT만 하거나, before/after 구분이 불필요하거나, 카프카 메시지 크기를 줄이고 싶을 때 쓰인다

<br>

2. ReplaceField

특정 필드 이름을 변경하거나 제거할 때 쓰는 옵션이다

```
{
  "transforms": "rename",
  "transforms.rename.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
  
  // 필드 이름 변경
  "transforms.rename.renames": "user_id:userId,order_id:orderId",
  
  // 필드 제외
  "transforms.rename.exclude": "internal_field,temp_data"
}

# 변환 전
{
  "order_id": "order-123",
  "user_id": 456,
  "internal_field": "debug"
}

# 변환 후
{
  "orderId": "order-123",
  "userId": 456
}
```

이 옵션은 DB 컬럼명(snake_case)을 Java 스타일(camelCase)로 변환하거나, 민감한 필드를 제거할 때 쓰인다

<br>

3. InsertField

새로운 필드를 추가할 때 쓰는 옵션이다

```
{
  "transforms": "insertTs",
  "transforms.insertTs.type": "org.apache.kafka.connect.transforms.InsertField$Value",
  
  // 정적 값 추가
  "transforms.insertTs.static.field": "source",
  "transforms.insertTs.static.value": "debezium",
  
  // 타임스탬프 추가
  "transforms.insertTs.timestamp.field": "kafka_ts"
}

# 변환 전
{
  "id": 123,
  "amount": 50000
}

# 변환 후
{
  "id": 123,
  "amount": 50000,
  "source": "debezium",
  "kafka_ts": 1710000000000
}
```

메시지 출처를 표시하거나, 처리 시간을 기록하거나, 파티션 정보를 추가할 때 쓰이는 옵션이다

<br>

4. MaskField

특정 필드를 마스킹할 때 쓰는 옵션

```
{
  "transforms": "mask",
  "transforms.mask.type": "org.apache.kafka.connect.transforms.MaskField$Value",
  "transforms.mask.fields": "password,ssn,credit_card",
  "transforms.mask.replacement": "****"
}

# 변환 전
{
  "user_id": 456,
  "password": "secret123",
  "ssn": "123-45-6789"
}

# 변환 후
{
  "user_id": 456,
  "password": "****",
  "ssn": "****"
}
```

민감 정보를 보호할때 사용하는 옵션이다

<br>

5. Filter

특정 메시지를 필터링할 때 쓰는 옵션이다

```
{
  "transforms": "filter",
  "transforms.filter.type": "io.debezium.transforms.Filter",
  "transforms.filter.language": "jsr223.groovy",
  "transforms.filter.condition": "value.op == 'd'"
}

# 효과

DELETE 이벤트는 제외(op == 'd')하고 INSERT와 UPDATE만 통과한다
```

'd'라는 거에 대해서 부연설명을 더 하자면, Debezium이 카프카로 보내는 메시지는 다음과 같다

```
{
  "before": {...},
  "after": {...},
  "op": "c",
  "ts_ms": 1710000000000,
  "source": {
    "db": "ecommerce",
    "table": "orders"
  }
}
```

여기서 op가 바로 operation type인데 쉽게 얘기해서 CRUD라고 생각하면 된다

```
c = INSERT
r = Snapshot(READ)
u = UPDATE
d = DELETE
```

특정 이벤트 타입만 처리하거나, 조건부 필터링이 필요할 때 쓰이는 옵션이다

물론, 위에서는 op에 대해서만 다뤘지만 그 외 타입에 대해서 필터링이 가능하다

<br>

6. Cast

필드 타입을 변환할 때 쓰는 옵션이다

```
{
  "transforms": "cast",
  "transforms.cast.type": "org.apache.kafka.connect.transforms.Cast$Value",
  "transforms.cast.spec": "amount:int64,status:string"
}

# 변환 전
{
  "amount": "50000",  // String
  "status": 1         // Int
}

# 변환 후
{
  "amount": 50000,    // Long
  "status": "1"       // String
}
```

DB가 타입과 카프카 타입이 일치하지 않는 경우나, Sink가 원하는 타입으로 변환할 때 쓰이는 옵션이다

<br>

7. TimestampConverter

타입 스탬프 형식을 변환할 때 쓰이는 옵션이다

```
{
  "transforms": "convertTs",
  "transforms.convertTs.type": "org.apache.kafka.connect.transforms.TimestampConverter$Value",
  "transforms.convertTs.field": "created_at",
  "transforms.convertTs.target.type": "Timestamp",
  "transforms.convertTs.format": "yyyy-MM-dd HH:mm:ss"
}

# 변환 전
{
  "created_at": "2025-04-29 12:00:00"  // String
}

# 변환 후
{
  "created_at": 1710000000000  // Unix timestamp
}
```

String -> Timestamp 또는 Timestamp -> String으로 변환할 때 쓰이는 옵션이다

<br>

8. unwrap

말 그대로 감싸고 있는걸 푸는 옵션이다

Debezium 메시지는 before/after로 감싸져(wrapped) 있다

unwrap은 이 감싸져있는 걸 풀어서 after만 꺼내는 걸 의미한다

```
{
  "before": null,
  "after": {
    "id": 123,
    "user_id": 456,
    "amount": 50000,
    "status": "PENDING"
  },
  "op": "c",
  "ts_ms": 1710000000000,
  "source": {
    "db": "ecommerce",
    "table": "orders"
  }
}

# unwrap 후
{
  "id": 123,
  "user_id": 456,
  "amount": 50000,
  "status": "PENDING"
}

-> after 안의 데이터만 남음
```

이렇게 unwrap을 해주는 이유는 Sink 커넥터가 단순 INSERT만 하기 때문이다

Sink 커넥트는 데이터를 받아서 DB에 데이터를 INSERT 해주는데,

before에 들어있는 데이터나, op 등과 같은 데이터는 필요 없다

<br>

**SMT 체이닝**

여러 SMT를 순서대로 적용할 수 있게 해준다

```
{
  "transforms": "unwrap,addSource,rename",
  
  // 1. Debezium 구조 풀기
  "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
  
  // 2. source 필드 추가
  "transforms.addSource.type": "org.apache.kafka.connect.transforms.InsertField$Value",
  "transforms.addSource.static.field": "source",
  "transforms.addSource.static.value": "gameinfo",
  
  // 3. 필드명 변경
  "transforms.rename.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
  "transforms.rename.renames": "user_id:userId"
}

# 적용 순서

원본 메시지
  ↓ unwrap
{id: 123, user_id: 456}
  ↓ addSource
{id: 123, user_id: 456, source: "gameinfo"}
  ↓ rename
{id: 123, userId: 456, source: "gameinfo"}
```

**내부 클래스 표현법**

옵션들을 보면 중간에 $로 구분된 것들이 보인다

```
org.apache.kafka.connect.transforms.ReplaceField$Value

org.apache.kafka.connect.transforms.InsertField$Value
```

$는 "내부 클래스"를 의미하는 구분자로, "외부클래스$내부클래스"로 사용한다

```
org.apache.kafka.connect.transforms.ReplaceField$Value
```

이걸 예시로 들면, transforms 패키지 안에 ReplaceField 클래스가 있고 이 클래스 내부에 Value라는 또다른 클래스가 있는데 이걸 가져오겠다는 의미다

<br>

실제 코드(간략화)

```
package org.apache.kafka.connect.transforms;

public abstract class ReplaceField<R extends ConnectRecord<R>> implements Transformation<R> {
    
    // Key 변환용 내부 클래스
    public static class Key<R extends ConnectRecord<R>> extends ReplaceField<R> {
        @Override
        public R apply(R record) {
            // record의 key를 가져와서
            Object key = record.key();
            // 필드명 변경, 제거 등 처리
            Object newKey = applyWithSchema(key);
            // 새로운 record 반환
            return record.newRecord(..., newKey, ...);
        }
    }
    
    // Value 변환용 내부 클래스
    public static class Value<R extends ConnectRecord<R>> extends ReplaceField<R> {
        @Override
        public R apply(R record) {
            // record의 value를 가져와서
            Object value = record.value();
            // 필드명 변경, 제거 등 처리
            Object newValue = applyWithSchema(value);
            // 새로운 record 반환
            return record.newRecord(..., newValue);
        }
    }
}
```

<br>

## 에러 처리

카프카 커넥트가 메시지를 처리하다가 실패했을 때 어떻게 할지 정할 수 있는 기능이다

Source Connector 에러

```
DB → Connector → Kafka
     ↑ 여기서 에러

예시:
- DB 연결 끊김
- binlog 포맷 파싱 실패
- Kafka로 전송 실패
- 직렬화(Serialization) 실패
```

Sink Connector 에러

```
Kafka → Connector → DB
        ↑ 여기서 에러

예시:
- DB 연결 끊김
- SQL 문법 오류
- 타입 불일치 (String을 INT에 넣으려고)
- Primary Key 중복
- 역직렬화(Deserialization) 실패
```

SMT 에러

```
메시지 → SMT 변환 → 실패
         ↑ 여기서 에러

예시:
- 필드가 없음 (user_id를 userId로 바꾸려는데 user_id 필드 자체가 없음)
- 타입 변환 실패 ("abc"를 숫자로 변환 시도)
- JSON 파싱 실패
```

만약 에러 처리를 해주지 않는다면

```
에러 발생
  ↓
Connector 즉시 중단 (FAILED 상태)
  ↓
모든 메시지 처리 멈춤
  ↓
수동으로 재시작해야 함
```

이렇게 되면, 만약 100만 건 처리중에 단 하나의 메시지가 깨지게 되면, 그 1건 때문에 나머지 100만 건 처리가 멈추게 된다

<br>

**Error 처리 옵션들**

1. errors.tolerance

```
// 개발/테스트
"errors.tolerance": "none"  // 에러 발생 → 즉시 중단

// 운영
"errors.tolerance": "all"   // 에러 발생 → 로그만 찍고 계속 진행 → 실패한 메시지는 건너뜀 또는 DLQ로
```

<br>

2. errors.retry.timeout

```
"errors.retry.timeout": 300000,      // 5분 (밀리초)
"errors.retry.delay.max.ms": 60000   // 최대 1분 간격
```

에러가 나면 5분동안 1분 간격으로 재시도한다는 의미다

<br>

3. errors.deadletterqueue.topic.name

```
"errors.deadletterqueue.topic.name": "dlq-orders"
```

실패한 메시지를 별도 토픽으로 보내는 옵션

DLQ 메시지 구조는 다음과 같다

```
{
  // 원본 메시지
  "key": {"id": 123},
  "value": {"user_id": 456, "amount": "invalid"},
  
  // 에러 정보 (헤더)
  "__connect.errors.topic": "orders",
  "__connect.errors.partition": 0,
  "__connect.errors.offset": 12345,
  "__connect.errors.connector.name": "jdbc-sink",
  "__connect.errors.task.id": 0,
  "__connect.errors.stage": "VALUE_CONVERTER",
  "__connect.errors.exception.class.name": "org.apache.kafka.connect.errors.DataException",
  "__connect.errors.exception.message": "Failed to convert amount: 'invalid' is not a number",
  "__connect.errors.exception.stacktrace": "..."
}
```

실패 원인 분석이나, 문제 수정 후 재처리, 데이터 유실 방지를 위해 DLQ 토픽으로 보내두기 위한 옵션이다

<br>

4. errors.deadletterqueue.context.headers.enable

```
"errors.deadletterqueue.context.headers.enable": true
```

DLQ 메시지에 에러 컨텍스트 헤더 포함 여부를 결정하는 옵션이다

false가 기본값인데

```
{
  "value": {"user_id": 456, "amount": "invalid"}
  // 에러 정보 없음
}
```

이러면 왜 실패했는지 알 수가 없다

true로 바꾸면

```
{
  "value": {"user_id": 456, "amount": "invalid"},
  // 헤더에 에러 정보 포함
  "__connect.errors.exception.message": "Failed to convert amount",
  "__connect.errors.exception.stacktrace": "..."
}
```

헤더에 에러 정보가 포함되어 원인을 명확하게 알 수 있다

<br>

5. errors.deadletterqueue.topic.replication.factor

```
"errors.deadletterqueue.topic.replication.factor": 3
```

DLQ 토픽의 복제 계수를 정하는 옵션이다

DLQ 토픽이 없으면 카프카 커넥트가 자동으로 replication.factor를 1로 설정하고 한 개의 복제만 만든다

```
Connector 시작
  ↓
에러 발생
  ↓
DLQ 토픽 "dlq-orders" 없음
  ↓
자동 생성 (기본 설정)
```

그런데 자동 생성하게 되면 DLQ 토픽이 한 개만 생기기 때문에, 만약 이 토픽에 문제가 생겨서 다운되면

DLQ로 보내려고 해도 접근이 불가능해서 에러가 발생한다

복제를 쓴다면 최소 3개는 설정해 두는 것이 좋다

<br>

6. errors.log.enable

```
"errors.log.enable": true,
"errors.log.include.messages": true
```

에러를 카프카 커넥트 로그에 기록하는 옵션

## JDBC Sink Connector

JDBC Sink Connector는 카프카 토픽의 메시지를 읽어서 JDBC를 통해 DB에 저장하는 커넥터다

```
Kafka 토픽 (order-events)
  ↓
JDBC Sink Connector
  ↓
JDBC 드라이버 (MySQL, PostgreSQL 등)
  ↓
데이터베이스
```

쉽게 얘기하면, 카프카 -> DB로 데이터를 옮기는 도구다

<br>

**사용하는 옵션들**

1. connector.class

```
"connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector"
```

JDBC Sink Connect 플러그인을 지정해준다

Confluent 오픈소스를 사용한 예시다

<br>

2. topics

어떤 토픽에서 읽을 지 정해주는 옵션이다

```
"topics": "order-events"

# 여러 토픽에서 읽을 경우
"topics": "order-events,payment-events,user-events"

# 토픽별 다른 테이블에서 읽을 경우
"topics": "order-events,payment-events",
"topics.to.tables": "order-events:orders,payment-events:payments"
```

3. connection.url

어떤 DB에 연결할지 정해주는 옵션이다

MySQL의 경우

```
jdbc:mysql://호스트:포트/DB명?옵션
jdbc:mysql://localhost:3306/analytics?useSSL=false
```

PostgreSQL의 경우

```
jdbc:postgresql://호스트:포트/DB명
jdbc:postgresql://localhost:5432/analytics
```

MariaDB의 경우

```
jdbc:mariadb://호스트:포트/DB명
jdbc:mariadb://localhost:3306/analytics
```

<br>

4. insert.mode (핵심)

어떤 방식으로 DB에 저장할지를 결정하는 옵션

<br>

**insert(기본값)**

```
INSERT INTO orders (order_id, user_id, amount)
VALUES ('order-123', 456, 50000);
```

단순히 INSERT만 하는 옵션이고, 중복 시에 에러가 발생한다

```
예시)

메시지 1: {order_id: 'order-123', amount: 50000}
  ↓
INSERT → 성공

메시지 2: {order_id: 'order-123', amount: 60000}  // 같은 ID
  ↓
INSERT → 에러 (Duplicate Key)
  ↓
DLQ로 전송 또는 Connector 중단
```

따라서, 항상 새로운 데이터만 들어오거나, 로그성 데이터일 때 사용하는 옵션이다

<br>

**upsert**

데이터가 없다면 INSERT, 있다면 UPDATE 하는 옵션으로 중복 시에도 에러가 발생하지 않는다

```
INSERT INTO orders (order_id, user_id, amount)
VALUES ('order-123', 456, 50000)
ON DUPLICATE KEY UPDATE
  user_id = 456,
  amount = 50000;
```

```
예시)

메시지 1: {order_id: 'order-123', amount: 50000}
  ↓
INSERT (새로 생성) ✅

메시지 2: {order_id: 'order-123', amount: 60000}  // 같은 ID
  ↓
UPDATE (덮어쓰기) ✅
```

중복 가능성이 있는 데이터나, 최신 상태 유지가 필요할 때 주로 쓰는 옵션이다

실무에서 가장 많이 쓰이는 옵션

<br>

**update**

데이터를 UPDATE 할 때만 쓰이는 옵션으로, 데이터가 없다면 무시한다(에러 발생 X)

```
UPDATE orders
SET user_id = 456, amount = 50000
WHERE order_id = 'order-123';
```

기존 데이터를 갱신만 하거나, 신규 생성을 하지 않을 때 사용하는 옵션

<br>

5. pk.mode, pk.fields

Primary Key를 설정하는 옵션으로 upsert나 update 모드에서 필수로 설정해야 하는 옵션이다

<br>

**pk.mode의 옵션들**

**record_key**

```
"pk.mode": "record_key"
```

카프카 메시지의 key를 PK로 사용하는 옵션

```
{
  "key": {"order_id": "order-123"},  ← 이거를 PK로
  "value": {"user_id": 456, "amount": 50000}
}
```

**record_value**

```
"pk.mode": "record_value",
"pk.fields": "order_id,user_id"
```

카프카 메시지의 value에서 필드를 추출해서 PK로 사용하는 옵션

```
{
  "key": null,
  "value": {
    "order_id": "order-123",  ← 이거를
    "user_id": 456,            ← 이거를 복합 PK로
    "amount": 50000
  }
}
```

**none**

```
"pk.mode": "record_value",
"pk.fields": "order_id,user_id"
```

PK를 사용하지 않음

insert 모드에서만 사용가능한 옵션이다

<br>

**pk.fields의 옵션들**

pk.mode와 달리 pk로 쓸 필드명을 직접 넣어주는 옵션이다

```
"pk.mode": "none"

# 복합 PK의 경우
"pk.fields": "order_id"
```

<br>

6. table.name.format

저장할 테이블의 이름을 넣어주는 옵션이다

```
"table.name.format": "orders" -> 메시지를 받아서 orders 테이블에 저장
```

<br>

7. auto.create, auto.evolve

```
"auto.create": "true",
"auto.evolve": "true"
```

**auto.create**

테이블 자동 생성 여부를 결정하는 옵션이다

true일 경우

```
테이블 없음
  ↓
메시지 스키마 분석
  ↓
CREATE TABLE 자동 실행
  ↓
데이터 INSERT
```

false일 경우

```
테이블 없음
  ↓
에러 발생
```

**auto.evolve**

스키마 변경 시 테이블을 자동으로 수정할지 결정하는 옵션이다

```
// 개발
"auto.evolve": "true"  // 편함

// 운영
"auto.evolve": "false"  // DDL 변경 제어 필요
```

true인 경우

```
기존 메시지: {order_id, user_id, amount}
  ↓
orders 테이블: (order_id, user_id, amount)

새 메시지: {order_id, user_id, amount, status}  ← status 추가
  ↓
ALTER TABLE orders ADD COLUMN status VARCHAR(50);
  ↓
데이터 INSERT
```

false인 경우

```
새 필드 발견
  ↓
에러 발생
```

<br>

8. 성능 튜닝 옵션들

**batch.size**

```
"batch.size": 3000
```

한 번에 INSERT할 레코드 수를 정하는 옵션

```
Kafka에서 메시지 3000개 읽기
  ↓
하나의 트랜잭션으로 묶어서 INSERT
  ↓
COMMIT
```

기본 값은 3000이다

<br>

**max.retries**

```
"max.retries": 10
```

DB 연결 실패 시, 재시도 횟수를 정하는 옵션

```
INSERT 실패 (DB 연결 끊김)
  ↓
재시도 1
  ↓
실패
  ↓
재시도 2
  ↓
...
  ↓
재시도 10
  ↓
모두 실패 → errors.tolerance 정책에 따라 처리
```

기본값은 10이다

<br>

**retry.backoff.ms**

```
"retry.backoff.ms": 3000
```

재시도 간격을 정하는 옵션

```
"retry.backoff.ms": 3000
```

기본값은 3000 (3초)

## 카프카 커넥트의 오프셋

카프카 커넥트도 카프카와 마찬가지로, 재시작하면 오프셋을 통해 중단된 지점부터 읽어온다

저장 위치는

```
Kafka 내부 토픽: connect-offsets
```

```
{
  "file": "mysql-bin.000005",     // binlog 파일
  "pos": 12345,                   // binlog 위치
  "snapshot": true
}

-> mysql-bin.000005 파일의 12345 위치까지 읽었다는 말이다
```
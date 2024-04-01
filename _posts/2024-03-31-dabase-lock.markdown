---
title: "막연하게 알던 DB Lock 알아보기"
date: 2024-03-31
category: [DB, DB]
tag: [database, lock]
---

## Lock?

- 같은 자원에 접근하려는 다중 트랜잭션 환경에서 DB의 일관성과 무결성을 유지하려면 순차적 진행을 위한 직렬화가 필요하고, 이런 직렬화를 위해 모든 DBMS에서 공통으로 사용하는 것이 Lock
- DBMS 마다 Lock 메커니즘이 다르고(ex. oracle 은 Read 작업일 때 Lock X) 격리성 수준을 조정할 수도 있다.(ex. Sql Server에서는 공유 Lock이 작업 수행 완료까지가 아니라 다음 레코드 읽히면 바로 해제됨)

## Lock 종류

### 공유 Lock (Shard Lock, Read Lock, S-Lock)

- 트랜잭션이 데이터를 변경하지 않는 Read 작업을 할 때 잠그는 Lock
- 데이터 조회만 하기 때문에 공유 Lock을 보유한 세션 끼리는 하나의 자원에 동시 접근이 가능함
- 하나의 세션에서 Read 작업 중 일 때, 데이터의 변경이 일어나면 정합성이 지켜지지 않으므로 다른 세션에서 베타 Lock을 걸고 접근할 수 없음

즉, 여러 세션이 동시에 같은 리소스에 대해 읽기 작업은 가능하나 쓰기 작업은 불가

### 베타 Lock (Exclusive Lock, Write Lock, X-Lock)

- 데이터를 변경 하려 할 때 사용되고 트랜잭션이 완료될 때까지 유지됨
- 하나의 세션에서 Write 작업 중 일 때, 다른 세션에서 Read/Write 작업을 하면 결과가 달라질 수 있기 때문에 다른 세션의 공유 Lock/베타 Lock 획득을 막음

즉, 하나의 세션에서 데이터를 변경하려고 할 때 모든 세션은 접근 불가

### 갱신 Lock (Update Lock)

- 데이터를 읽고나서 그 항목을 수정할 가능성이 있는 트랜잭션에서 사용하는 Lock(공유Lock과 베타Lock의 중간 단계 느낌)
- 트랜잭션이 Read 후 실제로 데이터를 변경하면 갱신 Lock은 베타 Lock으로 변경됨
- 갱신 Lock이 설정된 데이터는 다른 트랜잭션에서 공유 Lock을 획득할 수는 있지만 갱신 Lock, 베타 Lock은 불가

```sql
BEGIN TRANSACTION;

SELECT * FROM orders WHERE order_id = 101 FOR UPDATE;

UPDATE orders SET status = 'Processed' WHERE order_id = 101;

COMMIT;
```

`SELECT FOR UPDATE` 구문으로 갱신 Lock과 유사한 기능을 구현 가능

## Blocking

DB에서 데이터의 무결성, 정합성을 보장하기 위해 트랜잭션을 사용하고 각 트랜잭션 마다 Lock이 발동됨.
이런 트랜잭션 상황에서 Blocing은 Lock 경합이 발생해서 특정 트랜잭션이 작업을 실행 못하고 대기하는 상태를 뜻함.

- Blocking 발생 상황 예시
    - 공유 Lock이 설정 된 데이터에 다른 세션이 Write 작업을 하기 위해 베타 Lock을 설정할 할 때
    - 베타 Lock이 설정 된 데이터에 다른 세션이 Read 작업을 하기 위해 공유 Lock을 설정할 때

1. 트랜잭션 A가 시작되서 특정 행에 대해 Lock 획득
2. 거의 동시에 트랜잭션 B가 동일한 행에 접근하려고 하지만, 트랜잭션 A가 이미 Lock을 보유하고 있기 때문에 트랜잭션 B는 대기 상태
3. 이 시점에서 트랜잭션 B는 트랜잭션 A가 해당 Lock을 해제할 때 까지 기다려야하고 이를 Blocking이라고 함
4. 트랜잭션 A가 작업을 완료하면 커밋 후 Lock해제
5. 트랜잭션 B가 이제 Lock을 획득하고 작업 수행

### Blocking 해결 방안

Blocking 상태를 해결하는 방법은 커밋, 롤백 을 하는 것 뿐이므로 Lock 경합 상태가 발생하지 않도록 하는 것이 중요

- 트랜잭션 작업 단위는 최대한 최소로 설정 : 작업 단위를 최소로 설정 할수록 빠르게 트랜잭션이 빨리 종료됨

- 동일한 데이터를 동시에 변경하지 않도록 동시성 관리

- 트랜잭션이 많은 시간 대에는 대용량 데이터 작업 수행 X

## Dead Lock

## Lock 방식

## 2PL 프로토콜
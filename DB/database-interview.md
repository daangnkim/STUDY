# 백엔드 면접 - 데이터베이스 핵심 개념 정리

---

## 목차

1. [트랜잭션 (Transaction)](#1-트랜잭션-transaction)
2. [ACID 속성](#2-acid-속성)
3. [격리 수준 (Isolation Level)](#3-격리-수준-isolation-level)
4. [인덱스 (Index)](#4-인덱스-index)
5. [정규화 (Normalization)](#5-정규화-normalization)
6. [조인 (JOIN)](#6-조인-join)
7. [실행 계획 (Query Execution Plan)](#7-실행-계획-query-execution-plan)
8. [락 (Lock)](#8-락-lock)
9. [데드락 (Deadlock)](#9-데드락-deadlock)
10. [샤딩 (Sharding)](#10-샤딩-sharding)
11. [복제 (Replication)](#11-복제-replication)
12. [CAP 이론](#12-cap-이론)
13. [NoSQL vs RDBMS](#13-nosql-vs-rdbms)
14. [Connection Pool](#14-connection-pool)
15. [N+1 문제](#15-n1-문제)

---

## 1. 트랜잭션 (Transaction)

### 개념

트랜잭션은 **하나의 논리적인 작업 단위**를 구성하는 연산들의 집합이다. "모두 성공하거나, 모두 실패해야 한다"는 원칙을 따른다.

### 실생활 예시: 은행 이체

A가 B에게 10만원을 이체하는 시나리오를 생각해보자.

```sql
-- 이 두 쿼리는 반드시 같이 성공하거나, 같이 실패해야 한다
BEGIN;

UPDATE accounts SET balance = balance - 100000 WHERE user_id = 'A';  -- A 계좌에서 차감
UPDATE accounts SET balance = balance + 100000 WHERE user_id = 'B';  -- B 계좌에 추가

COMMIT;
```

만약 A의 계좌에서 돈을 뺀 후 서버가 다운되면? 트랜잭션이 없다면 A의 돈은 사라지고 B는 돈을 못 받는 상황이 발생한다. 트랜잭션은 이 두 작업을 하나로 묶어 원자성을 보장한다.

### 트랜잭션 상태

```
BEGIN → 활성(Active)
         ↓ 성공
       부분 완료(Partially Committed)
         ↓ 커밋 확정
       완료(Committed)

BEGIN → 활성(Active)
         ↓ 오류 발생
       실패(Failed)
         ↓ 롤백
       철회(Aborted)
```

---

## 2. ACID 속성

ACID는 트랜잭션이 안전하게 처리되기 위한 4가지 핵심 속성이다.

### A - Atomicity (원자성)

트랜잭션의 모든 연산이 **완전히 실행되거나 전혀 실행되지 않아야** 한다.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 50000 WHERE id = 1;
-- 여기서 오류 발생!
UPDATE accounts SET balance = balance + 50000 WHERE id = 2; -- 실행 안 됨

ROLLBACK; -- 첫 번째 UPDATE도 없었던 일이 됨
```

### C - Consistency (일관성)

트랜잭션 실행 전후에 **데이터베이스가 일관된 상태**를 유지해야 한다. 즉, 정의된 규칙(제약 조건, 외래키 등)을 항상 만족해야 한다.

```sql
-- balance는 0 이상이어야 한다는 제약이 있다면
UPDATE accounts SET balance = -1000 WHERE id = 1;
-- → 제약 조건 위반 → 트랜잭션 실패 → 롤백
-- 일관성 덕분에 DB는 항상 유효한 상태를 유지
```

### I - Isolation (격리성)

**동시에 실행되는 트랜잭션들이 서로에게 영향을 주지 않아야** 한다. 각 트랜잭션은 혼자 실행되는 것처럼 동작해야 한다.

```
트랜잭션 A: SELECT balance FROM accounts WHERE id = 1; → 1,000,000
트랜잭션 B: UPDATE accounts SET balance = 500,000 WHERE id = 1; (아직 커밋 안 됨)
트랜잭션 A: SELECT balance FROM accounts WHERE id = 1; → ???
```

격리 수준에 따라 A가 B의 미커밋 변경사항을 볼 수 있는지 여부가 결정된다.

### D - Durability (지속성)

커밋된 트랜잭션의 결과는 **영구적으로 반영**되어야 한다. 서버가 다운되더라도 커밋된 데이터는 사라지지 않는다.

```
WAL(Write-Ahead Logging): 커밋 전에 로그를 먼저 디스크에 기록
→ 서버 크래시 후 재시작 시 로그로 복구 가능
```

---

## 3. 격리 수준 (Isolation Level)

격리성을 완벽히 보장하면 성능이 떨어진다. 그래서 격리 수준을 조절해 **성능 vs 일관성** 사이의 트레이드오프를 선택할 수 있다.

### 발생 가능한 문제들

| 문제 | 설명 |
|------|------|
| Dirty Read | 아직 커밋되지 않은 데이터를 다른 트랜잭션이 읽음 |
| Non-Repeatable Read | 같은 쿼리를 두 번 실행했는데 결과가 다름 (수정/삭제) |
| Phantom Read | 같은 조건으로 조회했는데 없던 행이 생김 (삽입) |

### 4가지 격리 수준

#### READ UNCOMMITTED (가장 낮음)
```sql
-- 트랜잭션 B가 아직 커밋 안 한 데이터를 A가 읽을 수 있음
-- Dirty Read 발생 가능
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

#### READ COMMITTED (대부분의 DB 기본값)
```sql
-- 커밋된 데이터만 읽음, Dirty Read 방지
-- 하지만 같은 트랜잭션 내 두 번 읽으면 결과가 다를 수 있음 (Non-Repeatable Read)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

#### REPEATABLE READ (MySQL InnoDB 기본값)
```sql
-- 같은 트랜잭션 내에서 같은 행을 두 번 읽으면 항상 같은 결과
-- Phantom Read는 여전히 가능 (MySQL InnoDB는 MVCC로 Phantom Read도 방지)
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

#### SERIALIZABLE (가장 높음)
```sql
-- 완전한 직렬화. 동시성 처리 성능이 크게 떨어짐
-- 모든 이상 현상 방지
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

### 격리 수준별 허용 이상 현상

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------|-----------|---------------------|--------------|
| READ UNCOMMITTED | 허용 | 허용 | 허용 |
| READ COMMITTED | 방지 | 허용 | 허용 |
| REPEATABLE READ | 방지 | 방지 | 허용(DB에 따라 다름) |
| SERIALIZABLE | 방지 | 방지 | 방지 |

---

## 4. 인덱스 (Index)

### 개념

인덱스는 데이터를 빠르게 찾기 위한 **자료구조**다. 책의 색인(목차)과 같은 역할을 한다. 인덱스 없이 검색하면 테이블 전체를 스캔(Full Table Scan)해야 한다.

### 내부 구조: B-Tree

대부분의 RDBMS는 **B-Tree(Balanced Tree)** 인덱스를 사용한다.

```
              [50]
            /      \
         [25]      [75]
        /    \    /    \
      [10]  [30][60]  [90]
```

- 루트에서 리프까지의 탐색 시간: O(log N)
- 리프 노드들이 연결 리스트로 이어져 있어 범위 검색도 효율적

### 인덱스 생성 예시

```sql
-- 단일 컬럼 인덱스
CREATE INDEX idx_user_email ON users(email);

-- 복합 인덱스
CREATE INDEX idx_order_user_date ON orders(user_id, created_at);

-- 유니크 인덱스
CREATE UNIQUE INDEX idx_unique_email ON users(email);
```

### 복합 인덱스의 선두 컬럼 규칙

복합 인덱스 `(user_id, created_at)`가 있을 때:

```sql
-- 인덱스 사용 O
WHERE user_id = 1
WHERE user_id = 1 AND created_at > '2024-01-01'

-- 인덱스 사용 X (선두 컬럼인 user_id가 없음)
WHERE created_at > '2024-01-01'
```

### 인덱스가 오히려 독이 될 때

```sql
-- 인덱스가 있어도 풀 스캔이 발생하는 경우들

-- 1. 함수 적용 시
WHERE YEAR(created_at) = 2024  -- 인덱스 무시
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'  -- 인덱스 사용

-- 2. 부정 조건
WHERE status != 'ACTIVE'  -- 대부분 풀 스캔

-- 3. LIKE 앞부분 와일드카드
WHERE name LIKE '%김'  -- 인덱스 무시
WHERE name LIKE '김%'  -- 인덱스 사용

-- 4. NULL 비교
WHERE column IS NULL  -- DB마다 다름, PostgreSQL은 인덱스 지원
```

### 인덱스의 단점

- **INSERT/UPDATE/DELETE 성능 저하**: 데이터 변경 시 인덱스도 함께 갱신해야 함
- **저장 공간 추가 사용**: 인덱스도 디스크를 차지함
- 카디널리티(Cardinality)가 낮은 컬럼(성별 Y/N 등)은 인덱스 효과 미미

---

## 5. 정규화 (Normalization)

### 개념

정규화는 **데이터 중복을 최소화**하고 **이상(Anomaly) 현상을 방지**하기 위해 테이블을 분리하는 과정이다.

### 이상 현상의 종류

정규화되지 않은 테이블:

| 학생ID | 이름 | 강좌명 | 교수명 | 강의실 |
|--------|------|--------|--------|--------|
| 1 | 김철수 | DB | 이교수 | 101호 |
| 1 | 김철수 | 알고리즘 | 박교수 | 202호 |
| 2 | 이영희 | DB | 이교수 | 101호 |

- **삽입 이상**: 학생 없이 강좌만 추가 불가
- **삭제 이상**: 김철수가 DB 수강 취소 시 이교수 정보도 삭제됨
- **갱신 이상**: 이교수의 강의실 변경 시 모든 행을 수정해야 함

### 1NF (제1정규형)

모든 컬럼의 값이 **원자값(Atomic Value)**이어야 한다. 즉, 반복 그룹이나 다중 값이 없어야 한다.

```sql
-- 위반 예시 (취미가 쉼표로 여러 개)
| 이름 | 취미         |
| 김철수 | 독서, 등산, 게임 |

-- 1NF 만족
| 이름   | 취미 |
| 김철수 | 독서 |
| 김철수 | 등산 |
| 김철수 | 게임 |
```

### 2NF (제2정규형)

1NF를 만족하면서, **부분 함수 종속을 제거**한다. 복합 기본키에서 일부 컬럼에만 종속되는 컬럼을 분리한다.

```
기본키: (학생ID, 강좌명)
강의실은 강좌명에만 종속 → 강의실을 강좌 테이블로 분리

수강 테이블: (학생ID, 강좌명)
강좌 테이블: (강좌명, 교수명, 강의실)
```

### 3NF (제3정규형)

2NF를 만족하면서, **이행 함수 종속을 제거**한다. 기본키가 아닌 컬럼이 다른 비기본키 컬럼을 결정해서는 안 된다.

```
강좌 테이블에서: 강좌명 → 교수명 → 강의실
강의실이 교수명에 종속 (이행 종속)

강좌 테이블: (강좌명, 교수명)
교수 테이블: (교수명, 강의실)
```

### 역정규화 (Denormalization)

정규화를 지나치게 하면 JOIN이 많아져 **읽기 성능이 저하**될 수 있다. 이때 의도적으로 중복을 허용하는 역정규화를 고려한다.

```sql
-- 예: 주문마다 상품명을 JOIN 없이 바로 가져오기 위해
-- orders 테이블에 product_name 컬럼을 중복 저장
ALTER TABLE orders ADD COLUMN product_name VARCHAR(255);
```

---

## 6. 조인 (JOIN)

### INNER JOIN

두 테이블에서 **조건이 일치하는 행만** 반환한다.

```sql
SELECT u.name, o.product_name
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
-- 주문이 없는 유저, 유저 없는 주문은 결과에 포함 안 됨
```

### LEFT JOIN

왼쪽 테이블의 **모든 행을 반환**하고, 오른쪽은 일치하는 행만 (없으면 NULL).

```sql
SELECT u.name, o.product_name
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
-- 주문이 없는 유저도 결과에 포함 (o.product_name은 NULL)
```

### JOIN 알고리즘

DB는 JOIN 처리 시 여러 알고리즘 중 하나를 선택한다:

**Nested Loop Join**: 한 테이블을 순회하며 다른 테이블에서 매칭 행을 찾음. 소량 데이터에 유리.
```
for each row in table_A:
    for each row in table_B:
        if join_condition: output
```

**Hash Join**: 작은 테이블을 해시 테이블로 만들고, 큰 테이블로 조회. 대용량 데이터에 유리.

**Merge Join**: 두 테이블을 정렬한 뒤 병합. 이미 정렬된 인덱스가 있을 때 유리.

---

## 7. 실행 계획 (Query Execution Plan)

### 개념

DB 옵티마이저가 SQL을 어떻게 실행할지 결정한 계획이다. 느린 쿼리를 최적화할 때 필수적으로 확인한다.

### EXPLAIN 사용법

```sql
-- PostgreSQL / MySQL
EXPLAIN SELECT * FROM orders WHERE user_id = 1;

-- 실제 실행까지 해보기
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1;
```

### 실행 계획 읽기 (PostgreSQL 예시)

```
Seq Scan on orders  (cost=0.00..450.00 rows=10000 width=50)
  Filter: (user_id = 1)
```

- **Seq Scan**: 순차 탐색 (풀 스캔) → 인덱스가 없거나 옵티마이저가 비효율적이라 판단
- **Index Scan**: 인덱스를 통한 탐색 → 원하는 상태
- **cost**: 예상 비용 (start-up cost .. total cost)
- **rows**: 예상 반환 행 수

### 나쁜 실행 계획의 시그널

```
Seq Scan → 인덱스 추가 또는 쿼리 수정 필요
Hash Join with large rows → 인덱스 기반 Nested Loop 검토
Sort → 정렬 인덱스 검토
```

---

## 8. 락 (Lock)

### 개념

동시에 여러 트랜잭션이 같은 데이터에 접근할 때 **데이터 일관성을 보장**하기 위한 메커니즘.

### 공유 락 (Shared Lock, S Lock) vs 배타 락 (Exclusive Lock, X Lock)

| | 공유 락 (읽기) | 배타 락 (쓰기) |
|---|---|---|
| 공유 락 요청 | 허용 | 대기 |
| 배타 락 요청 | 대기 | 대기 |

```sql
-- 공유 락 (다른 트랜잭션도 읽기 가능, 쓰기는 불가)
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- 배타 락 (다른 트랜잭션 읽기/쓰기 모두 불가)
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
```

### 낙관적 락 vs 비관적 락

**비관적 락 (Pessimistic Lock)**: 충돌이 자주 발생한다고 가정. DB 레벨에서 실제 락을 건다.

```sql
-- 조회 시 락을 걸고, 수정 후 커밋
BEGIN;
SELECT * FROM inventory WHERE product_id = 1 FOR UPDATE;
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 1;
COMMIT;
```

**낙관적 락 (Optimistic Lock)**: 충돌이 드물다고 가정. 버전(version) 컬럼으로 충돌 감지.

```sql
-- version 컬럼으로 충돌 감지
-- 조회 시
SELECT id, quantity, version FROM inventory WHERE product_id = 1;
-- → version = 5

-- 수정 시 (다른 트랜잭션이 먼저 수정했으면 version이 달라져서 0 rows 반환)
UPDATE inventory
SET quantity = quantity - 1, version = version + 1
WHERE product_id = 1 AND version = 5;
-- 영향받은 행이 0이면 → 충돌 감지 → 재시도
```

### MVCC (Multi-Version Concurrency Control)

PostgreSQL, InnoDB 등은 MVCC를 사용해 **읽기와 쓰기가 서로 블로킹하지 않도록** 한다.

- 쓰기 트랜잭션이 새 버전 생성
- 읽기 트랜잭션은 트랜잭션 시작 시점의 스냅샷을 읽음
- 결과: 읽기가 쓰기를 막지 않고, 쓰기도 읽기를 막지 않음

---

## 9. 데드락 (Deadlock)

### 개념

두 개 이상의 트랜잭션이 서로가 가진 락을 기다리며 **영원히 대기 상태**에 빠지는 현상.

### 발생 시나리오

```
트랜잭션 A:
  1. accounts 테이블의 row 1에 락 획득
  2. accounts 테이블의 row 2에 락 요청 → 트랜잭션 B가 가지고 있어서 대기

트랜잭션 B:
  1. accounts 테이블의 row 2에 락 획득
  2. accounts 테이블의 row 1에 락 요청 → 트랜잭션 A가 가지고 있어서 대기

→ A는 B를 기다리고, B는 A를 기다림 → 데드락
```

### 데드락 방지 방법

**1. 락 순서 통일**: 항상 같은 순서로 리소스를 획득

```java
// 이체 시 항상 작은 ID 먼저 락
void transfer(long fromId, long toId, long amount) {
    long firstLock = Math.min(fromId, toId);
    long secondLock = Math.max(fromId, toId);

    lock(firstLock);
    lock(secondLock);
    // ... 처리
}
```

**2. 타임아웃 설정**: 락 대기 시간 제한

```sql
-- MySQL
SET innodb_lock_wait_timeout = 5;  -- 5초 후 타임아웃
```

**3. 데드락 감지**: DB가 자동으로 감지하고 한 트랜잭션을 롤백

```
PostgreSQL: 자동으로 데드락 감지 → 한 트랜잭션에 ERROR 반환
MySQL InnoDB: innodb_deadlock_detect 설정으로 감지
```

---

## 10. 샤딩 (Sharding)

### 개념

데이터를 **여러 데이터베이스 서버에 수평 분할**해서 저장하는 방법. 단일 서버의 한계를 극복하기 위한 수평 확장(Scale-Out) 전략.

### 샤딩 vs 파티셔닝

- **파티셔닝**: 하나의 DB 서버 내에서 테이블을 물리적으로 분할
- **샤딩**: 여러 DB 서버로 데이터를 분산 (각 서버가 Shard)

### 샤딩 방법

**Range-based Sharding**: 값의 범위로 분할

```
Shard 1: user_id 1 ~ 1,000,000
Shard 2: user_id 1,000,001 ~ 2,000,000
Shard 3: user_id 2,000,001 ~ 3,000,000

장점: 범위 쿼리가 효율적
단점: 특정 샤드에 데이터가 몰릴 수 있음 (Hotspot)
```

**Hash-based Sharding**: 해시 함수로 분할

```
shard_number = hash(user_id) % 3

user_id = 1 → hash(1) % 3 = 1 → Shard 2
user_id = 5 → hash(5) % 3 = 2 → Shard 3

장점: 균등한 분배
단점: 범위 쿼리 비효율, 샤드 추가 시 리밸런싱 필요
```

**Directory-based Sharding**: 별도 매핑 테이블로 관리

```
Lookup Table: user_id → Shard ID
유연하지만 Lookup Table 자체가 병목이 될 수 있음
```

### 샤딩의 문제점

```
1. Cross-Shard JOIN 불가 → 애플리케이션 레벨에서 처리해야 함
2. 분산 트랜잭션 어려움
3. 샤드 추가 시 데이터 리밸런싱 필요
4. 운영 복잡도 증가
```

---

## 11. 복제 (Replication)

### 개념

하나의 DB(Primary)의 데이터를 다른 DB(Replica/Secondary)에 **복사해서 동기화**하는 것. 고가용성과 읽기 성능 향상을 위해 사용.

### Primary-Replica 구조

```
[Write 요청] → Primary DB
                     ↓ 복제 (Replication)
               ┌─────────────┐
               Replica 1   Replica 2
               ↑             ↑
         [Read 요청]    [Read 요청]
```

### 동기 복제 vs 비동기 복제

**동기 복제 (Synchronous)**:
```
Primary → 쓰기 → Replica에 전파 → Replica 완료 확인 → 클라이언트에 응답
- 데이터 손실 없음
- 속도 느림 (Replica 응답 대기)
```

**비동기 복제 (Asynchronous)**:
```
Primary → 쓰기 → 클라이언트에 응답 (Replica 전파는 백그라운드)
- 속도 빠름
- Primary 장애 시 일부 데이터 손실 가능 (Replication Lag)
```

### 복제 지연 (Replication Lag) 문제

```
사용자가 데이터를 쓴 직후 읽기를 Replica에서 하면 아직 반영 안 됐을 수 있음

해결책:
1. 중요한 읽기는 Primary에서
2. "Write 후 자신의 데이터 읽기"는 세션 스티키 또는 Primary 라우팅
3. Replica 지연 시간 모니터링
```

---

## 12. CAP 이론

### 개념

분산 시스템에서 다음 세 가지 속성을 **동시에 모두 만족시키는 것은 불가능**하다는 이론.

- **C (Consistency)**: 모든 노드가 같은 시점에 같은 데이터를 봄
- **A (Availability)**: 모든 요청에 응답을 반환 (에러 없이)
- **P (Partition Tolerance)**: 네트워크 파티션(분리)이 발생해도 시스템이 동작

### 네트워크 파티션은 반드시 발생한다

분산 시스템에서 네트워크 장애는 불가피하다. 따라서 **P는 필수 선택**이고, 결국 C와 A 중 하나를 포기해야 한다.

### CP vs AP

**CP 시스템** (일관성 + 파티션 허용, 가용성 희생):

```
네트워크 분리 발생 시 → 일관성을 위해 일부 노드 응답 거부
예: ZooKeeper, HBase, MongoDB (기본 설정)
```

**AP 시스템** (가용성 + 파티션 허용, 일관성 희생):

```
네트워크 분리 발생 시 → 모든 노드가 응답하지만 일관성 보장 안 됨
예: Cassandra, DynamoDB, CouchDB
"Eventually Consistent" (최종적 일관성)
```

### 실무에서의 선택

```
은행 시스템 → CP 선호 (돈과 관련된 데이터는 반드시 일관성)
SNS 좋아요 수 → AP 허용 (잠깐 다를 수 있어도 괜찮음)
```

---

## 13. NoSQL vs RDBMS

### RDBMS

```
장점:
- 강력한 데이터 무결성 (스키마, 제약 조건)
- ACID 트랜잭션 지원
- 복잡한 쿼리 (JOIN, 집계 등) 지원
- 성숙한 생태계, 풍부한 도구

단점:
- 수평 확장(Scale-Out)이 어려움
- 스키마 변경 비용이 높음
- 대용량 비정형 데이터 처리에 부적합
```

### NoSQL 종류와 사용 사례

**Document Store** (MongoDB, CouchDB):
```json
// 유연한 스키마, 중첩 구조 저장 가능
{
  "_id": "user123",
  "name": "김철수",
  "orders": [
    {"product": "노트북", "price": 1500000},
    {"product": "마우스", "price": 50000}
  ]
}
// 사용 사례: CMS, 카탈로그, 사용자 프로필
```

**Key-Value Store** (Redis, DynamoDB):
```
key: "session:abc123"
value: {user_id: 1, expires_at: "2024-12-31"}

사용 사례: 캐싱, 세션 관리, 리더보드
```

**Column Family** (Cassandra, HBase):
```
행 키로 데이터 접근, 컬럼 동적 추가 가능
사용 사례: 시계열 데이터, 로그, IoT 데이터
```

**Graph** (Neo4j):
```
노드와 엣지로 관계 표현
사용 사례: SNS 친구 관계, 추천 시스템, 사기 탐지
```

### 언제 무엇을 선택하나?

```
RDBMS 선택:
- 복잡한 관계, 정규화된 데이터
- 강력한 일관성 필요
- 복잡한 쿼리, 리포팅

NoSQL 선택:
- 빠른 스키마 변화 (스타트업 초기)
- 대용량 단순 읽기/쓰기
- 수평 확장이 중요한 경우
```

---

## 14. Connection Pool

### 개념

DB 연결은 생성 비용이 비싸다 (TCP 연결, 인증, 세션 초기화). **Connection Pool**은 미리 연결을 만들어 두고 재사용하는 패턴이다.

### 커넥션 풀 없을 때의 문제

```
요청마다:
1. TCP 소켓 연결 (3-way handshake)
2. DB 인증
3. 세션 초기화
→ 수십~수백 ms 지연 발생, 대용량 트래픽에서 심각한 문제
```

### 커넥션 풀 동작

```
서버 시작 시: 미리 N개의 커넥션 생성 (min pool size)

요청 처리 시:
1. 풀에서 커넥션 빌려옴 (Acquire)
2. 쿼리 실행
3. 풀에 반납 (Release)

모든 커넥션이 사용 중이면: 대기 (timeout 설정)
```

### HikariCP 설정 예시 (Spring Boot)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10      # 최대 커넥션 수
      minimum-idle: 5            # 최소 유지 커넥션 수
      connection-timeout: 30000  # 커넥션 획득 대기 시간 (30초)
      idle-timeout: 600000       # 유휴 커넥션 제거 시간 (10분)
      max-lifetime: 1800000      # 커넥션 최대 생명주기 (30분)
```

### 적절한 풀 사이즈 계산

```
HikariCP 권장 공식:
pool size = (core_count * 2) + effective_spindle_count

예) CPU 4코어, SSD(spindle=1):
pool size = (4 * 2) + 1 = 9

너무 많은 커넥션 = DB 서버 부하, 컨텍스트 스위칭 오버헤드
너무 적은 커넥션 = 대기 시간 발생
```

---

## 15. N+1 문제

### 개념

1번의 쿼리로 N개의 결과를 가져온 후, 각 결과마다 1번씩 추가 쿼리가 발생하는 문제. 총 1+N번의 쿼리가 실행된다.

### 발생 예시 (JPA/Hibernate)

```java
// Users를 모두 조회 (1번 쿼리)
List<User> users = userRepository.findAll();

// 각 유저의 주문을 조회 (N번 쿼리)
for (User user : users) {
    System.out.println(user.getOrders().size()); // Lazy Loading
}

// 실행된 SQL:
// SELECT * FROM users;                    -- 1번
// SELECT * FROM orders WHERE user_id = 1; -- N번
// SELECT * FROM orders WHERE user_id = 2;
// SELECT * FROM orders WHERE user_id = 3;
// ... N개의 쿼리
```

### 해결 방법 1: Fetch Join

```java
// JPQL
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();

// 실행 SQL (1번으로 해결)
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```

### 해결 방법 2: @BatchSize (Hibernate)

```java
@BatchSize(size = 100)
@OneToMany(mappedBy = "user")
private List<Order> orders;

// IN 절로 묶어서 쿼리
SELECT * FROM orders WHERE user_id IN (1, 2, 3, ..., 100);
```

### 해결 방법 3: DataLoader (GraphQL)

```javascript
// GraphQL에서 N+1 문제 해결
const userLoader = new DataLoader(async (userIds) => {
    const orders = await Order.findAll({ where: { userId: userIds } });
    // userId로 그룹핑해서 반환
    return userIds.map(id => orders.filter(o => o.userId === id));
});

// 여러 user에 대한 orders 요청이 한 번의 쿼리로 묶임
```

### N+1 발견 방법

```properties
# Spring Boot에서 SQL 로그 출력
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.type.descriptor.sql=TRACE
```

```
콘솔에서 같은 패턴의 쿼리가 반복되면 N+1 의심
→ p6spy 라이브러리로 쿼리 수 모니터링 추천
```

---

## 핵심 면접 질문 모음

**Q. 트랜잭션 격리 수준 4가지를 설명하고, 각각 어떤 이상 현상을 허용하나요?**
> READ UNCOMMITTED (Dirty Read 허용) → READ COMMITTED → REPEATABLE READ → SERIALIZABLE (모두 방지) 순으로 격리 수준이 높아지며 성능은 낮아진다.

**Q. 인덱스가 항상 빠른 건 아닌데, 언제 사용을 피해야 하나요?**
> 카디널리티가 낮은 컬럼, 자주 UPDATE되는 컬럼, 테이블이 매우 작을 때, WHERE 절에서 함수로 감싸지는 컬럼에는 인덱스 효과가 없거나 오히려 부담이 된다.

**Q. 낙관적 락과 비관적 락의 차이는?**
> 비관적 락은 충돌이 많다고 가정해 선점, 낙관적 락은 충돌이 적다고 가정해 나중에 감지. 낙관적 락은 DB 락을 걸지 않아 성능이 좋지만, 충돌 시 재시도 로직이 필요하다.

**Q. 샤딩의 단점은?**
> Cross-Shard JOIN 불가, 분산 트랜잭션 어려움, 샤드 키 설계 실수 시 Hotspot 발생, 운영 복잡도 증가.

**Q. MVCC가 무엇인가요?**
> Multi-Version Concurrency Control. 데이터의 여러 버전을 유지해 읽기와 쓰기가 서로를 블로킹하지 않도록 한다. 읽기 트랜잭션은 자신이 시작된 시점의 스냅샷을 본다.

**Q. N+1 문제를 어떻게 해결하나요?**
> Fetch Join으로 한 번의 쿼리에 연관 데이터를 가져오거나, @BatchSize로 IN 절 배치 처리, 또는 쿼리 로그를 통해 문제를 먼저 탐지한다.

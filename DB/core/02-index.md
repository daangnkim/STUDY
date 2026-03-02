# 인덱스 (Index)

---

## 개념

인덱스는 데이터를 빠르게 찾기 위한 **자료구조**다. 책의 색인(목차)과 같은 역할.
인덱스 없이 검색하면 테이블 전체를 순차 스캔(Full Table Scan)해야 한다.

```
인덱스 없을 때:
  SELECT * FROM users WHERE email = 'kim@example.com'
  → 1억 건을 처음부터 끝까지 다 읽음 (O(N))

인덱스 있을 때:
  → B+Tree 타고 내려가서 바로 찾음 (O(log N))
  → 1억 건 기준 log₂(1억) ≈ 27번 비교로 찾음
```

---

## 내부 구조: B+Tree

대부분의 RDBMS(MySQL InnoDB, PostgreSQL)는 **B+Tree** 인덱스를 사용한다.

```
                  [30 | 70]                ← 루트 노드 (키만 저장)
                 /    |    \
          [10|20]  [40|60]  [80|90]        ← 내부 노드 (키만 저장)
          ↓  ↓ ↓   ↓  ↓ ↓   ↓  ↓ ↓
       [리프] [리프] [리프] [리프] [리프]    ← 리프 노드 (키 + Row Pointer 또는 실제 데이터)
          ↔      ↔      ↔      ↔           ← 리프끼리 LinkedList 연결

핵심 특징:
- 루트에서 리프까지 탐색: O(log N)
- 리프 노드가 LinkedList로 연결되어 있어 범위 검색도 효율적
  (예: WHERE age BETWEEN 20 AND 30 → 20 찾은 후 리프를 순서대로 탐색)
```

### Clustered Index vs Non-Clustered Index

```
Clustered Index (MySQL InnoDB의 PK):
  → 리프 노드에 실제 데이터 행이 저장됨
  → 테이블당 1개만 존재
  → PK 순서로 물리적으로 정렬됨

Non-Clustered Index (일반 인덱스):
  → 리프 노드에 인덱스 키 + PK 값(포인터)이 저장됨
  → PK로 실제 데이터 행을 다시 조회 (Bookmark Lookup)
  → 테이블당 여러 개 가능
```

```java
// JPA에서 PK = Clustered Index 자동 생성
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id; // → Clustered Index (데이터 물리적 정렬 기준)

    @Column(unique = true)
    private String email; // → Non-Clustered Index (이메일로 조회 시 PK 찾고 → 데이터 조회)
}
```

---

## JPA에서 인덱스 설정

```java
// 단일 컬럼 인덱스
@Entity
@Table(name = "users",
    indexes = @Index(name = "idx_user_email", columnList = "email"))
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;
    private String name;
    private LocalDateTime createdAt;
}
```

```java
// 복합 인덱스 + 유니크 제약
@Entity
@Table(name = "orders",
    indexes = {
        @Index(name = "idx_order_user_date", columnList = "user_id, created_at"),
        @Index(name = "idx_order_status_created", columnList = "status, created_at DESC")
    },
    uniqueConstraints = {
        @UniqueConstraint(name = "uk_order_number", columnNames = "order_number")
    }
)
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "order_number", nullable = false)
    private String orderNumber;

    @Column(nullable = false)
    private String status;

    @Column(name = "created_at")
    private LocalDateTime createdAt;
}
```

---

## 복합 인덱스 - 선두 컬럼 규칙

복합 인덱스 `(user_id, created_at, status)` 기준:

```sql
-- 인덱스 사용 O
WHERE user_id = 1
WHERE user_id = 1 AND created_at > '2024-01-01'
WHERE user_id = 1 AND created_at > '2024-01-01' AND status = 'COMPLETED'

-- 인덱스 사용 X (선두 컬럼 user_id 없음)
WHERE created_at > '2024-01-01'
WHERE status = 'COMPLETED'
WHERE created_at > '2024-01-01' AND status = 'COMPLETED'
```

```java
// Repository 메서드 예시

// 인덱스 사용 O
List<Order> findByUserId(Long userId);
List<Order> findByUserIdAndCreatedAtAfter(Long userId, LocalDateTime from);
List<Order> findByUserIdAndCreatedAtAfterAndStatus(Long userId, LocalDateTime from, String status);

// 인덱스 사용 X (선두 컬럼 없음)
List<Order> findByCreatedAtAfter(LocalDateTime from);  // Full Scan
List<Order> findByStatus(String status);               // Full Scan
```

### 범위 조건과 등치 조건의 순서

```sql
-- 인덱스: (status, created_at)

-- 권장: 등치 조건을 앞에
WHERE status = 'COMPLETED' AND created_at > '2024-01-01'
-- → status로 먼저 필터링 후 created_at 범위 검색

-- 비권장: 범위 조건이 앞에 오면 그 뒤 컬럼은 인덱스 범위 탐색 불가
WHERE created_at > '2024-01-01' AND status = 'COMPLETED'
-- → created_at 범위 이후 status는 인덱스를 타지 않고 필터링
```

---

## 인덱스가 무효화되는 패턴

```java
// 1. 컬럼에 함수 적용 → 인덱스 무효화
@Query("SELECT o FROM Order o WHERE YEAR(o.createdAt) = :year")  // 인덱스 X
List<Order> findByYear(@Param("year") int year);

// 올바른 방법: 범위 조건으로 변환
@Query("SELECT o FROM Order o WHERE o.createdAt BETWEEN :start AND :end")
List<Order> findByDateRange(
    @Param("start") LocalDateTime start,
    @Param("end") LocalDateTime end);
```

```sql
-- 2. 암묵적 타입 변환
WHERE user_id = 1      -- user_id가 VARCHAR면 인덱스 X (DB가 형변환)
WHERE user_id = '1'    -- 인덱스 O

-- 3. LIKE 앞 와일드카드
WHERE name LIKE '%김'   -- Full Scan (앞에서부터 일치 비교 불가)
WHERE name LIKE '김%'   -- 인덱스 사용 O (앞부분 일치로 B+Tree 탐색 가능)

-- 4. NOT 조건 (대부분 Full Scan)
WHERE status != 'ACTIVE'   -- 인덱스 비효율 (전체의 대부분을 가져와야 하면 Full Scan이 빠름)
WHERE status = 'INACTIVE'  -- 특정 값 조회는 인덱스 효율적

-- 5. OR 조건
WHERE user_id = 1 OR product_id = 5
-- 각 조건별로 인덱스 스캔 후 UNION → 비효율적일 수 있음
-- UNION ALL 쿼리로 분리하거나 별도 처리 고려
```

---

## 커버링 인덱스 (Covering Index)

쿼리에 필요한 **모든 컬럼이 인덱스에 포함**되어 테이블 접근 없이 인덱스만으로 결과 반환.

```java
// 인덱스: (user_id, created_at, status)
// 이 세 컬럼만 SELECT하면 테이블 접근 없음 (커버링 인덱스)

@Query("""
    SELECT new com.example.dto.OrderSummary(o.id, o.createdAt, o.status)
    FROM Order o
    WHERE o.userId = :userId
    ORDER BY o.createdAt DESC
    """)
List<OrderSummary> findOrderSummaryByUserId(@Param("userId") Long userId);

// 반면 아래는 product_name이 인덱스에 없어 테이블 접근 필요
@Query("SELECT o FROM Order o WHERE o.userId = :userId") // SELECT * 와 동일
List<Order> findByUserId(@Param("userId") Long userId);
```

```sql
-- EXPLAIN으로 확인
EXPLAIN SELECT user_id, created_at, status FROM orders WHERE user_id = 1;
-- Extra: Using index  ← 커버링 인덱스 사용 중

EXPLAIN SELECT * FROM orders WHERE user_id = 1;
-- Extra: (없음)  ← 테이블 접근 발생
```

---

## 인덱스 선택 기준

### 인덱스 걸어야 하는 경우

```java
// 1. WHERE 절에 자주 사용되는 컬럼
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);
// → email에 인덱스 필수

// 2. JOIN 조건으로 사용되는 컬럼
@Query("SELECT o FROM Order o JOIN o.user u WHERE u.id = :userId")
// → user_id (FK)에 인덱스 (JPA @JoinColumn은 인덱스 자동 생성 안 함, 직접 추가 필요)

// 3. ORDER BY, GROUP BY에 사용되는 컬럼 (정렬 시 Sort 연산 제거)
// 인덱스: (user_id, created_at DESC)
@Query("SELECT o FROM Order o WHERE o.userId = :userId ORDER BY o.createdAt DESC")
```

### 인덱스 피해야 하는 경우

```java
/*
1. 카디널리티(Cardinality)가 낮은 컬럼
   - 성별(M/F): 전체 데이터의 50%씩 → 인덱스로 걸러도 절반은 읽어야 함
   - status(ACTIVE/INACTIVE/DELETED): 값 종류 적으면 효과 미미
   - boolean 타입 컬럼

2. 자주 UPDATE되는 컬럼
   - 인덱스는 데이터 변경 시 함께 갱신됨
   - UPDATE가 잦으면 인덱스 갱신 비용이 읽기 이득을 상쇄

3. 테이블 데이터가 아주 적을 때
   - 수백 건 이하 → DB가 Full Scan을 선택하는 경우가 많음
   - 옵티마이저가 인덱스보다 Full Scan이 빠르다고 판단

4. 쓰기가 극단적으로 많은 경우
   - 로그성 테이블, 이벤트 테이블 등
   - INSERT 시마다 인덱스 갱신 → 쓰기 성능 저하
*/
```

---

## 인덱스 통계와 옵티마이저

옵티마이저는 **인덱스 통계**를 기반으로 인덱스 사용 여부를 결정한다.

```sql
-- MySQL: 테이블 통계 업데이트
ANALYZE TABLE orders;

-- PostgreSQL: 통계 업데이트
ANALYZE orders;

-- 인덱스가 있는데도 Full Scan이 선택되는 이유:
-- 1. 반환 행이 전체의 10~30% 이상 → Full Scan이 더 빠름
-- 2. 통계가 오래돼서 옵티마이저가 잘못 판단
-- 3. 데이터 분포가 편중됨 (e.g., status = 'ACTIVE'가 95%)
```

```java
// 강제로 인덱스 사용하게 하기 (힌트) - 권장하지 않음, 통계 업데이트가 우선
@Query(value = "SELECT * FROM orders USE INDEX (idx_order_user_date) WHERE user_id = :userId",
    nativeQuery = true)
List<Order> findByUserIdWithHint(@Param("userId") Long userId);
```

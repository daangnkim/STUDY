# 트랜잭션 / ACID / 격리 수준

---

## 1. 트랜잭션 (Transaction)

### 개념

트랜잭션은 **하나의 논리적인 작업 단위**를 구성하는 연산들의 집합이다.
"모두 성공하거나, 모두 실패해야 한다"는 원칙을 따른다.

### 실생활 예시: 은행 이체

A가 B에게 10만원을 이체할 때, 두 UPDATE는 반드시 함께 성공하거나 함께 실패해야 한다.
중간에 서버가 죽으면 A 돈만 빠지고 B는 못 받는 상황이 생긴다.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100000 WHERE user_id = 'A';
UPDATE accounts SET balance = balance + 100000 WHERE user_id = 'B';
COMMIT;
```

### 트랜잭션 상태

```
BEGIN → [활성(Active)]
            ↓ 정상 처리
       [부분 완료(Partially Committed)]
            ↓ 디스크 반영 성공
       [완료(Committed)]

BEGIN → [활성(Active)]
            ↓ 오류 발생
       [실패(Failed)]
            ↓ 롤백
       [철회(Aborted)] → 이전 상태로 복구
```

### Spring @Transactional 기본 사용

Spring은 AOP 프록시를 통해 트랜잭션을 선언적으로 관리한다.
[핵심 로직을 더 깔끔하게: Spring AOP를 활용한 관심사의 분리 | by jjy_joeun | Ne(o)rdinary Tech](https://tech.neordinary.co.kr/%ED%95%B5%EC%8B%AC-%EB%A1%9C%EC%A7%81%EC%9D%84-%EB%8D%94-%EA%B9%94%EB%81%94%ED%95%98%EA%B2%8C-spring-aop%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%9C-%EA%B4%80%EC%8B%AC%EC%82%AC%EC%9D%98-%EB%B6%84%EB%A6%AC-d36ece5d4f22)
`begin/commit/rollback`을 직접 코딩하지 않아도 된다.

```java
@Service
@RequiredArgsConstructor
public class TransferService {

    private final AccountRepository accountRepository;

    @Transactional
    public void transfer(Long fromId, Long toId, long amount) {
        Account from = accountRepository.findById(fromId)
            .orElseThrow(() -> new IllegalArgumentException("계좌 없음"));
        Account to = accountRepository.findById(toId)
            .orElseThrow(() -> new IllegalArgumentException("계좌 없음"));

        from.withdraw(amount); // 잔액 부족 시 예외 발생 → 자동 rollback
        to.deposit(amount);

        // 정상 종료 → 자동 commit
        // RuntimeException 발생 → 자동 rollback (두 변경 모두 취소)
    }
}
```

### @Transactional 동작 원리 (프록시)

```java
// Spring이 내부적으로 만들어주는 프록시 (개념적 표현)
public class TransferServiceProxy extends TransferService {

    @Override
    public void transfer(Long fromId, Long toId, long amount) {
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            super.transfer(fromId, toId, amount); // 실제 비즈니스 로직 호출
            transactionManager.commit(status);
        } catch (RuntimeException e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```

> **주의 - Self-invocation 문제**: 같은 클래스 내부에서 `@Transactional` 메서드를 호출하면
> 프록시를 거치지 않아 트랜잭션이 동작하지 않는다.

```java
@Service
public class OrderService {

    // 이렇게 하면 안 됨! (Self-invocation)
    public void createOrder(OrderRequest req) {
        this.saveOrderWithTransaction(req); // 프록시 안 거침 → @Transactional 무시됨
    }

    @Transactional
    public void saveOrderWithTransaction(OrderRequest req) { ... }
}
```

### Propagation (전파 속성)

트랜잭션이 이미 존재할 때 새 트랜잭션을 어떻게 처리할지 결정한다.

```java
// REQUIRED (기본값): 기존 트랜잭션에 참여, 없으면 새로 생성
@Transactional(propagation = Propagation.REQUIRED)
public void joinExisting() { ... }

// REQUIRES_NEW: 항상 새 트랜잭션 생성, 기존 트랜잭션은 일시 중단
// 사용 예: 감사 로그 - 본 트랜잭션 실패해도 로그는 반드시 남겨야 할 때
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAuditLog(String message) { ... }

// NESTED: 세이브포인트 기반 중첩 트랜잭션 (부분 롤백 가능)
@Transactional(propagation = Propagation.NESTED)
public void nestedWork() { ... }
```

```java
// 실전 예시: 주문 처리 중 알림 발송 실패해도 주문은 성공해야 하는 경우
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final NotificationService notificationService;

    @Transactional
    public void placeOrder(OrderRequest request) {
        Order order = orderRepository.save(Order.from(request)); // 주문 저장

        try {
            notificationService.sendOrderConfirmation(order); // 알림 발송 시도
        } catch (Exception e) {
            // 알림 실패해도 주문은 성공 처리
            log.warn("알림 발송 실패, 주문은 정상 처리됨: {}", e.getMessage());
        }
    }
}

@Service
public class NotificationService {

    // REQUIRES_NEW: 별도 트랜잭션으로 처리
    // → 실패 시 본 트랜잭션(주문)에 영향 없음
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendOrderConfirmation(Order order) {
        // 알림 발송 로직
    }
}
```

### rollbackFor

기본적으로 `@Transactional`은 **unchecked 예외(RuntimeException)에만 롤백**한다.
checked 예외는 롤백하려면 명시해야 한다.

```java
// IOException은 checked 예외 → 기본적으로 롤백 안 됨
@Transactional(rollbackFor = IOException.class) // 명시하면 롤백됨
public void processFile() throws IOException {
    fileRepository.save(file);
    externalFileSystem.write(file); // IOException 발생 시 rollback
}

// 특정 예외에서는 롤백하지 않을 때
@Transactional(noRollbackFor = BusinessException.class)
public void businessLogic() {
    // BusinessException 발생해도 이미 처리된 내용은 커밋
}
```

---

## 2. ACID 속성

트랜잭션의 네 가지 특성으로 ACID가 존재한다.
ACID는 트랜잭션이 안전하게 수행되기 위한 4가지 핵심 속성이다.

### A - Atomicity (원자성)

트랜잭션의 모든 연산이 **완전히 실행되거나 전혀 실행되지 않아야** 한다. (All or Nothing)

```java
@Transactional
public void createOrderWithPayment(OrderRequest request) {
    // 1. 재고 차감
    inventoryService.decrease(request.getProductId(), request.getQuantity());

    // 2. 주문 생성
    Order order = orderRepository.save(Order.from(request));

    // 3. 결제 처리 ← 여기서 예외 발생하면
    paymentService.pay(order.getId(), request.getAmount()); // PaymentException!

    // → 1번 재고 차감, 2번 주문 생성 모두 롤백
    // → 재고는 줄었는데 주문이 없는 상황 방지
}

@Entity
public class Account {
    @Id
    private Long id;
    private long balance;

    public void withdraw(long amount) {
        if (this.balance < amount) {
            // 예외를 던져 트랜잭션 롤백 유발 → 원자성 보장
            throw new InsufficientBalanceException("잔액 부족: " + this.balance);
        }
        this.balance -= amount;
    }
}
```

### C - Consistency (일관성)

트랜잭션 실행 전후에 **DB가 항상 유효한 상태**를 유지해야 한다.
정의된 규칙(제약 조건, 외래키, 도메인 규칙)을 항상 만족해야 한다.

```java
@Entity
@Table(name = "accounts",
    uniqueConstraints = @UniqueConstraint(columnNames = "account_number"))
public class Account {

    @Column(nullable = false)
    private String accountNumber;

    // 애플리케이션 레벨 검증 (Bean Validation)
    @Min(value = 0, message = "잔액은 0 이상이어야 합니다")
    private long balance;

    @Version
    private Long version;
}
```

```sql
-- DB 레벨 제약 조건으로도 일관성 보장
ALTER TABLE accounts ADD CONSTRAINT chk_balance CHECK (balance >= 0);
-- balance가 음수가 되는 트랜잭션은 DB 자체에서 거부
```

### I - Isolation (격리성)

동시에 실행되는 트랜잭션들이 **서로 영향을 주지 않아야** 한다.

```java
// 격리성이 없을 때 발생하는 문제: 재고 1개 남은 상품에 동시 주문
@Transactional
public void orderProduct(Long productId) {
    Product product = productRepository.findById(productId).get();
    // Thread A: stock = 1 읽음
    // Thread B: 동시에 stock = 1 읽음 (문제!)

    if (product.getStock() > 0) {
        product.decreaseStock(); // A: 0으로 변경, B: 이미 0인데 또 감소 → -1!
        orderRepository.save(new Order(productId));
    }
}
// 해결: 격리 수준 조정 또는 락 사용 (다음 섹션 참고)
```

### D - Durability (지속성)

커밋된 트랜잭션 결과는 **영구적으로 반영**되어야 한다.
서버가 다운되어도 커밋된 데이터는 사라지지 않는다.

```
WAL (Write-Ahead Logging) 동작 방식:
1. 트랜잭션 실행
2. COMMIT 요청
3. 변경 내용을 디스크의 WAL 로그에 먼저 기록 (fsync)
4. 클라이언트에 commit 완료 응답
5. 나중에 실제 데이터 파일에 반영

서버 크래시 후 재시작 시:
  → WAL 로그를 읽어 커밋된 트랜잭션 재적용
  → 데이터 손실 없음
```

```java
@Transactional
public Order createOrder(OrderRequest request) {
    Order order = orderRepository.save(Order.from(request));
    // commit 완료 후에는 서버가 죽어도 order는 DB에 영구 저장됨
    return order;
}
```

---

## 3. 격리 수준 (Isolation Level)

격리성을 완벽히 보장하면 동시 처리 성능이 떨어진다.
격리 수준으로 **성능 vs 일관성** 사이의 트레이드오프를 선택한다.

### 발생 가능한 이상 현상 3가지

#### Dirty Read

커밋되지 않은 데이터를 다른 트랜잭션이 읽는 것.

```
TX-A: balance = 1,000,000 → 500,000 으로 수정 (아직 커밋 안 함)
TX-B: balance = 500,000 읽음  ← Dirty Read
TX-A: 롤백 → balance는 다시 1,000,000

TX-B는 존재하지 않는 값(500,000)을 읽은 것!
```

#### Non-Repeatable Read

같은 트랜잭션 안에서 **같은 행을 두 번 읽었는데 값이 다른 것**.

```
TX-A: SELECT balance FROM accounts WHERE id=1 → 1,000,000
    (이 사이에 TX-B가 balance를 500,000으로 수정하고 커밋)
TX-A: SELECT balance FROM accounts WHERE id=1 → 500,000 (다른 값!)

같은 트랜잭션인데 값이 달라짐 → 신뢰할 수 없는 데이터
```

#### Phantom Read

같은 조건으로 조회했는데 **없던 행이 나타나는 것** (삽입에 의해).

```
TX-A: SELECT COUNT(*) FROM orders WHERE user_id=1 → 5건
    (이 사이에 TX-B가 user_id=1 인 주문 1건 추가하고 커밋)
TX-A: SELECT COUNT(*) FROM orders WHERE user_id=1 → 6건 (Phantom!)

같은 조건인데 결과가 다름
```

### 4가지 격리 수준과 Spring 설정

```java
// 1. READ_UNCOMMITTED - Dirty Read 허용. 거의 사용하지 않음.
@Transactional(isolation = Isolation.READ_UNCOMMITTED)
public long getApproximateUserCount() {
    // 정확도가 낮아도 되는 통계 조회에서나 고려
    return userRepository.count();
}
```

```java
// 2. READ_COMMITTED - Oracle, PostgreSQL 기본값
// 커밋된 데이터만 읽음. Non-Repeatable Read 가능.
@Transactional(isolation = Isolation.READ_COMMITTED)
public UserDto getUser(Long userId) {
    return userRepository.findById(userId)
        .map(UserDto::from)
        .orElseThrow();
}
```

```java
// 3. REPEATABLE_READ - MySQL InnoDB 기본값
// 트랜잭션 시작 시 스냅샷 생성 → 같은 행 여러 번 읽어도 동일 결과
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void processUserReport(Long userId) {
    User user = userRepository.findById(userId).get(); // 스냅샷 시점 기준

    // ... 긴 처리 작업 (다른 TX가 이 사용자를 수정하고 커밋해도)

    User same = userRepository.findById(userId).get(); // 여전히 같은 값 반환
    // REPEATABLE_READ이므로 처음 읽은 값과 동일 보장
}
```

```java
// 4. SERIALIZABLE - 완전한 직렬화. 성능 가장 낮음.
// 사실상 한 번에 하나의 트랜잭션만 실행되는 것처럼 동작
@Transactional(isolation = Isolation.SERIALIZABLE)
public void criticalBatchProcess() {
    // 정확성이 극도로 중요한 배치 작업
}
```

### Non-Repeatable Read 코드 시나리오

```java
// READ_COMMITTED에서 발생
@Transactional(isolation = Isolation.READ_COMMITTED)
public void checkAndProcessBalance(Long accountId) {
    Account account = accountRepository.findById(accountId).get();
    long balance = account.getBalance(); // 1,000,000

    // ← 이 사이에 다른 트랜잭션이 balance를 500,000으로 줄이고 커밋

    if (balance > 500000) { // true (1,000,000 > 500,000)
        // 다시 조회하면 500,000이 나옴 (Non-Repeatable Read)
        // 하지만 balance 변수는 이미 1,000,000을 갖고 있음
        // 로직 오류 발생 가능!
        processHighBalance(account);
    }
}
```

### 격리 수준별 이상 현상 허용 여부

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------|-----------|---------------------|--------------|
| READ_UNCOMMITTED | 허용 | 허용 | 허용 |
| READ_COMMITTED | 방지 | **허용** | 허용 |
| REPEATABLE_READ | 방지 | 방지 | **허용** (MySQL InnoDB는 MVCC로 방지) |
| SERIALIZABLE | 방지 | 방지 | 방지 |

### @Transactional(readOnly = true)의 효과

```java
// 읽기 전용 트랜잭션
@Transactional(readOnly = true)
public List<ProductDto> findAll() {
    return productRepository.findAll()
        .stream().map(ProductDto::from).toList();
}

/*
readOnly = true 효과:
1. Hibernate dirty checking 비활성화 → 스냅샷 저장 안 함 (메모리 절약)
2. flush mode = MANUAL → 불필요한 DB flush 방지
3. DB 드라이버/옵티마이저에 읽기 전용 힌트 전달
4. Read/Write 분리 구성 시 → Replica DB로 자동 라우팅
*/
```

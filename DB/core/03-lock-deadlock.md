# 락 (Lock) / 데드락 (Deadlock)

---

## 1. 락 (Lock)

### 개념

동시에 여러 트랜잭션이 같은 데이터에 접근할 때 **데이터 일관성을 보장**하기 위한 메커니즘.

### 공유 락 vs 배타 락

| | 공유 락 요청 | 배타 락 요청 |
|---|---|---|
| **공유 락 보유 중** | 허용 (함께 읽기 가능) | 대기 |
| **배타 락 보유 중** | 대기 | 대기 |

```java
public interface AccountRepository extends JpaRepository<Account, Long> {

    // 공유 락 (S Lock): SELECT ... FOR SHARE
    // 다른 트랜잭션도 읽기는 가능, 쓰기는 이 트랜잭션이 끝날 때까지 불가
    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdForShare(@Param("id") Long id);

    // 배타 락 (X Lock): SELECT ... FOR UPDATE
    // 다른 트랜잭션 읽기/쓰기 모두 이 트랜잭션이 끝날 때까지 불가
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdForUpdate(@Param("id") Long id);
}
```

---

## 2. 비관적 락 (Pessimistic Lock)

충돌이 자주 발생한다고 가정. DB 레벨에서 실제 락을 걸어 동시 접근 자체를 막는다.

### 언제 사용?

- 재고 차감, 좌석 예약, 쿠폰 사용 등 **동시 충돌이 잦은 경우**
- 충돌 시 재시도보다 선점이 더 비용이 낮은 경우

### 구현

```java
@Service
@RequiredArgsConstructor
public class InventoryService {

    private final ProductRepository productRepository;

    @Transactional
    public void decreaseStock(Long productId, int quantity) {
        // SELECT * FROM products WHERE id = ? FOR UPDATE
        // → 이 트랜잭션 끝날 때까지 다른 트랜잭션은 이 행 수정 불가 (대기)
        Product product = productRepository.findByIdWithPessimisticLock(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        if (product.getStock() < quantity) {
            throw new InsufficientStockException("재고 부족: 현재 " + product.getStock());
        }

        product.decreaseStock(quantity);
        // commit 시 락 해제 → 대기 중이던 트랜잭션이 실행됨
    }
}

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithPessimisticLock(@Param("id") Long id);
}
```

### 락 타임아웃 설정

```java
// 락 대기 시간 제한 (무한 대기 방지)
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({
    @QueryHint(
        name = "jakarta.persistence.lock.timeout",
        value = "3000"  // 3000ms = 3초
    )
})
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdWithLockTimeout(@Param("id") Long id);
// 3초 내에 락 획득 못하면 LockTimeoutException 발생
```

```yaml
# 전역 설정
spring:
  jpa:
    properties:
      jakarta:
        persistence:
          lock:
            timeout: 5000  # 기본 5초 대기
```

---

## 3. 낙관적 락 (Optimistic Lock)

충돌이 드물다고 가정. DB 락을 걸지 않고 **@Version 컬럼으로 충돌을 감지**한다.

### 언제 사용?

- 충돌 빈도가 낮은 경우 (읽기가 대부분인 경우)
- 높은 동시성이 필요한 경우 (락으로 인한 병목 제거)
- 사용자 프로필 수정, 게시글 수정 등

### 구현

```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private int stock;

    @Version  // 수정할 때마다 자동으로 1씩 증가
    private Long version;
}
```

```java
@Service
@RequiredArgsConstructor
public class InventoryService {

    private final ProductRepository productRepository;

    @Transactional
    public void decreaseStockOptimistic(Long productId, int quantity) {
        Product product = productRepository.findById(productId).orElseThrow();
        // version = 5 로 읽음

        product.decreaseStock(quantity);

        // JPA가 자동으로 아래 SQL 실행:
        // UPDATE products
        // SET stock = ?, version = 6    ← version +1
        // WHERE id = ? AND version = 5  ← 읽었던 version으로 조건
        //
        // 만약 다른 트랜잭션이 먼저 수정해서 version이 6이 됐으면
        // WHERE id = ? AND version = 5 → 0 rows affected
        // → ObjectOptimisticLockingFailureException 발생
    }
}
```

### 충돌 발생 시 재시도 (@Retryable)

```java
// spring-retry 의존성 필요
// implementation 'org.springframework.retry:spring-retry'
// @EnableRetry 필요

@Service
@RequiredArgsConstructor
public class InventoryService {

    @Retryable(
        retryFor = ObjectOptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(
            delay = 100,       // 첫 재시도: 100ms 후
            multiplier = 2.0   // 이후: 200ms, 400ms ...
        )
    )
    @Transactional
    public void decreaseStockWithRetry(Long productId, int quantity) {
        Product product = productRepository.findById(productId).orElseThrow();
        product.decreaseStock(quantity);
    }

    @Recover // 모든 재시도 실패 시 호출
    public void recoverFromOptimisticLock(
        ObjectOptimisticLockingFailureException e, Long productId, int quantity
    ) {
        log.error("재고 차감 최종 실패 - productId: {}, quantity: {}", productId, quantity);
        throw new StockUpdateFailedException("재고 차감 실패. 잠시 후 다시 시도해주세요.");
    }
}
```

### 낙관적 락 vs 비관적 락 선택 기준

```
비관적 락이 유리한 경우:
  ✓ 동일 데이터 충돌이 잦음 (플래시 세일, 인기 좌석 예약)
  ✓ 충돌 시 재시도 비용이 높음 (복잡한 처리, 외부 API 호출 포함)
  ✓ 실시간 정확성이 중요한 금융 데이터

낙관적 락이 유리한 경우:
  ✓ 충돌이 거의 없음 (사용자 프로필 수정, 게시글 수정)
  ✓ 높은 동시성 필요 (락 대기로 인한 병목 제거)
  ✓ 단순한 업데이트 (재시도 비용이 낮음)
```

---

## 4. MVCC (Multi-Version Concurrency Control)

PostgreSQL, MySQL InnoDB가 사용하는 방식.
**읽기와 쓰기가 서로 블로킹하지 않는다** → 읽기가 쓰기를 기다리지 않음.

### 동작 방식

```
일반 락 방식:
  TX-A (쓰기) ─────────── commit
  TX-B (읽기)    ← 대기 → 읽기 시작  ← TX-A 끝날 때까지 기다려야 함

MVCC 방식:
  TX-A (쓰기): 새 버전 생성 (기존 버전은 Undo Log에 보관)
  TX-B (읽기): TX-A 쓰는 중에도 자신이 시작된 시점의 스냅샷 즉시 읽음
  → 읽기가 쓰기를 블로킹하지 않음
  → 쓰기도 읽기를 블로킹하지 않음
```

```java
// MVCC 덕분에 readOnly 트랜잭션은 락 경쟁 없이 항상 빠르게 처리됨
@Transactional(readOnly = true)
public List<ProductDto> findAllProducts() {
    // 조회 시작 시점의 스냅샷을 읽음
    // 다른 트랜잭션이 동시에 데이터 수정 중이어도 블로킹 없음
    return productRepository.findAll()
        .stream().map(ProductDto::from).toList();
}
```

### MVCC와 Undo Log

```
INSERT 시: 새 버전 생성
UPDATE 시: 새 버전 생성 + 이전 버전을 Undo Log에 보관
DELETE 시: 삭제 마크 표시 + 이전 버전을 Undo Log에 보관

오래된 트랜잭션이 오래된 스냅샷을 읽을수록 Undo Log 공간 증가
→ 장시간 실행되는 읽기 트랜잭션이 Undo Log를 크게 만들 수 있음 (주의!)
```

---

## 5. 데드락 (Deadlock)

### 개념

두 개 이상의 트랜잭션이 서로가 가진 락을 기다리며 **영원히 대기 상태**에 빠지는 현상.

### 발생 시나리오

```
TX-A                              TX-B
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Product(id=1) 락 획득
                                  2. Product(id=2) 락 획득
3. Product(id=2) 락 요청
   → TX-B가 보유 중 → 대기
                                  4. Product(id=1) 락 요청
                                     → TX-A가 보유 중 → 대기

결과: A는 B를 기다리고, B는 A를 기다림 → 영원히 대기 (Deadlock)
DB가 감지 → 한 트랜잭션을 강제 롤백 → Spring: DeadlockLoserDataAccessException
```

### 데드락 유발 코드 예시

```java
@Service
public class DeadlockExampleService {

    // Thread-1이 이 메서드 호출
    @Transactional
    public void transactionA(Long id1, Long id2) {
        productRepository.findByIdWithPessimisticLock(id1); // id=1 락 획득
        Thread.sleep(100); // 일부러 지연 (데드락 유발)
        productRepository.findByIdWithPessimisticLock(id2); // id=2 락 시도 → 대기!
    }

    // Thread-2가 이 메서드 호출
    @Transactional
    public void transactionB(Long id1, Long id2) {
        productRepository.findByIdWithPessimisticLock(id2); // id=2 락 획득
        Thread.sleep(100);
        productRepository.findByIdWithPessimisticLock(id1); // id=1 락 시도 → 대기!
    }
    // → 데드락 발생!
}
```

---

## 6. 데드락 방지 전략

### 전략 1: 항상 같은 순서로 락 획득 (가장 효과적)

```java
@Service
@RequiredArgsConstructor
public class SafeTransferService {

    private final AccountRepository accountRepository;

    @Transactional
    public void transfer(Long fromId, Long toId, long amount) {
        // 항상 ID 오름차순으로 락 획득
        // → A→B 이체나 B→A 이체나 항상 작은 ID 먼저 락 → 데드락 불가
        Long firstId  = Math.min(fromId, toId);
        Long secondId = Math.max(fromId, toId);

        Account first  = accountRepository.findByIdForUpdate(firstId).orElseThrow();
        Account second = accountRepository.findByIdForUpdate(secondId).orElseThrow();

        Account from = first.getId().equals(fromId)  ? first : second;
        Account to   = first.getId().equals(toId)    ? first : second;

        from.withdraw(amount);
        to.deposit(amount);
    }
}
```

### 전략 2: 낙관적 락으로 DB 락 자체를 피함

```java
// 낙관적 락은 DB 락을 걸지 않음 → 데드락 자체가 발생 불가
// 대신 충돌 시 재시도 필요

@Retryable(
    retryFor = ObjectOptimisticLockingFailureException.class,
    maxAttempts = 5,
    backoff = @Backoff(delay = 50, multiplier = 2)
)
@Transactional
public void updateWithOptimisticLock(Long productId, int delta) {
    Product product = productRepository.findById(productId).orElseThrow();
    product.updateStock(delta);
}
```

### 전략 3: 데드락 발생 시 재시도 처리

```java
@Service
@RequiredArgsConstructor
public class ResilientInventoryService {

    private final ProductRepository productRepository;

    // 데드락 발생 시 재시도
    @Retryable(
        retryFor = DeadlockLoserDataAccessException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 1.5)
    )
    @Transactional
    public void decreaseStock(Long productId, int quantity) {
        Product product = productRepository.findByIdWithPessimisticLock(productId)
            .orElseThrow();
        product.decreaseStock(quantity);
    }

    @Recover
    public void recoverFromDeadlock(DeadlockLoserDataAccessException e, Long productId, int quantity) {
        log.error("데드락 재시도 초과 - productId: {}", productId);
        throw new InventoryUpdateFailedException("재고 처리 실패. 잠시 후 다시 시도해주세요.");
    }
}
```

### 전략 4: 트랜잭션 범위 최소화

```java
// Bad: 트랜잭션 안에서 외부 API 호출 등 오래 걸리는 작업 포함
// → 락 보유 시간 길어짐 → 다른 트랜잭션 대기 시간 증가 → 데드락 가능성 증가
@Transactional
public void badPattern(Long productId, OrderRequest request) {
    Product product = productRepository.findByIdWithPessimisticLock(productId).orElseThrow();
    product.decreaseStock(request.getQuantity());

    // 외부 API 호출 (수백 ms ~ 수 초) ← 이 동안 락 유지됨!
    PaymentResult result = paymentGateway.pay(request.getPaymentInfo());
    orderRepository.save(Order.from(request, result));
}

// Good: 트랜잭션 범위를 최소화, 외부 호출은 트랜잭션 밖에서
public void goodPattern(Long productId, OrderRequest request) {
    // 외부 API 호출 먼저 (트랜잭션 없음)
    PaymentResult result = paymentGateway.pay(request.getPaymentInfo());

    // 트랜잭션은 짧게
    decreaseStockAndSaveOrder(productId, request, result);
}

@Transactional
public void decreaseStockAndSaveOrder(Long productId, OrderRequest request, PaymentResult result) {
    Product product = productRepository.findByIdWithPessimisticLock(productId).orElseThrow();
    product.decreaseStock(request.getQuantity());
    orderRepository.save(Order.from(request, result));
    // 빠르게 commit → 락 해제
}
```

### 데드락 감지 로그 확인

```sql
-- MySQL: 마지막 데드락 정보 확인
SHOW ENGINE INNODB STATUS;
-- LATEST DETECTED DEADLOCK 섹션에서 상세 정보 확인

-- 데드락 로그 활성화 (MySQL)
SET GLOBAL innodb_print_all_deadlocks = ON;
-- error log에 모든 데드락 기록됨
```

```java
// Spring: 데드락 예외 처리
@ExceptionHandler(DeadlockLoserDataAccessException.class)
public ResponseEntity<ErrorResponse> handleDeadlock(DeadlockLoserDataAccessException e) {
    log.warn("데드락 발생: {}", e.getMessage());
    return ResponseEntity
        .status(HttpStatus.CONFLICT)
        .body(new ErrorResponse("DEADLOCK", "잠시 후 다시 시도해주세요."));
}
```

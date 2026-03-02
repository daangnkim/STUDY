# JOIN / N+1 문제 / 커넥션 풀

---

## 1. JOIN

### JPA에서 JOIN 방법

```java
// 1. JPQL JOIN (연관관계 없는 테이블 조인)
@Query("SELECT u FROM User u JOIN u.orders o WHERE o.status = 'COMPLETED'")
List<User> findUsersWithCompletedOrders();

// 2. LEFT JOIN
@Query("SELECT u FROM User u LEFT JOIN u.orders o WHERE u.createdAt > :date")
List<User> findActiveUsersAfterDate(@Param("date") LocalDateTime date);

// 3. JOIN FETCH (N+1 해결용 - 연관 엔티티를 한 번에 로드)
@Query("SELECT DISTINCT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();

// 4. @EntityGraph (JOIN FETCH의 어노테이션 방식)
@EntityGraph(attributePaths = {"orders", "orders.product"})
Optional<User> findById(Long id);
```

### QueryDSL로 복잡한 JOIN

```java
@Repository
@RequiredArgsConstructor
public class OrderQueryRepository {

    private final JPAQueryFactory queryFactory;

    // 동적 조건 + 다중 테이블 JOIN
    public List<OrderDto> searchOrders(OrderSearchCondition condition) {
        return queryFactory
            .select(Projections.constructor(OrderDto.class,
                order.id,
                user.name.as("userName"),
                product.name.as("productName"),
                order.createdAt,
                order.status
            ))
            .from(order)
            .innerJoin(user).on(order.userId.eq(user.id))
            .innerJoin(product).on(order.productId.eq(product.id))
            .where(
                userNameContains(condition.getUserName()),
                statusEq(condition.getStatus()),
                createdAtBetween(condition.getFrom(), condition.getTo())
            )
            .orderBy(order.createdAt.desc())
            .offset(condition.getOffset())
            .limit(condition.getSize())
            .fetch();
    }

    // null이면 조건 무시 (동적 쿼리)
    private BooleanExpression userNameContains(String userName) {
        return StringUtils.hasText(userName) ? user.name.contains(userName) : null;
    }

    private BooleanExpression statusEq(String status) {
        return StringUtils.hasText(status) ? order.status.eq(status) : null;
    }

    private BooleanExpression createdAtBetween(LocalDateTime from, LocalDateTime to) {
        if (from == null && to == null) return null;
        if (from == null) return order.createdAt.loe(to);
        if (to == null) return order.createdAt.goe(from);
        return order.createdAt.between(from, to);
    }
}
```

### JOIN 알고리즘 (DB 내부 동작)

```
Nested Loop Join (소량 데이터 + inner 테이블에 인덱스 있을 때):
  for each row in users:        ← outer table (작은 테이블)
      for each row in orders:   ← inner table (인덱스 있어야 효율적)
          if join_condition: output

  시간복잡도: O(M * log N)  ← inner table 인덱스 있을 때
  시간복잡도: O(M * N)      ← 인덱스 없을 때 (최악)

Hash Join (대용량 데이터, 인덱스 없을 때):
  Phase 1 (Build): 작은 테이블로 해시맵 생성
      hashMap[orders.user_id] = order_rows
  Phase 2 (Probe): 큰 테이블 순회하며 해시맵 조회
      for each row in users:
          result = hashMap.get(users.id)

  시간복잡도: O(M + N)
  단점: 해시맵 빌드에 메모리 사용

Merge Join (두 테이블 모두 정렬된 경우):
  이미 정렬된 인덱스가 있을 때 가장 효율적
  시간복잡도: O(M + N)  단, 정렬 비용 O(M log M + N log N)
```

---

## 2. N+1 문제

### 개념

1번의 쿼리로 N개의 결과를 가져온 후, 각 결과마다 1번씩 추가 쿼리가 발생하는 문제.
총 **1 + N번** 쿼리 실행 → 유저 100명이면 101번 쿼리.

### 문제 발생 코드

```java
@Entity
public class User {
    @Id private Long id;
    private String name;

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY) // 기본값: Lazy
    private List<Order> orders;
}

@Service
public class UserService {

    public List<UserDto> getAllUserSummaries() {
        List<User> users = userRepository.findAll();
        // SQL 1번: SELECT * FROM users  (100명 반환)

        return users.stream()
            .map(user -> {
                int orderCount = user.getOrders().size(); // Lazy Loading 트리거!
                // SQL N번: SELECT * FROM orders WHERE user_id = 1
                //          SELECT * FROM orders WHERE user_id = 2
                //          SELECT * FROM orders WHERE user_id = 3
                //          ... 100번
                return new UserDto(user.getName(), orderCount);
            })
            .toList();
        // 합계: 1 + 100 = 101번 쿼리
    }
}
```

### 해결 방법 1: Fetch Join (JPQL)

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // JOIN FETCH로 한 번에 로드
    @Query("SELECT DISTINCT u FROM User u JOIN FETCH u.orders")
    List<User> findAllWithOrders();

    // 실행 SQL (1번):
    // SELECT DISTINCT u.*, o.*
    // FROM users u
    // INNER JOIN orders o ON u.id = o.user_id
}

// 주의: Fetch Join + 페이징은 함께 쓸 수 없음!
// @OneToMany Fetch Join + Pageable → HibernateJpaDialect 경고 + 메모리 페이징
// → 페이징 시에는 @BatchSize 또는 DTO 조회 사용

// 잘못된 예시
@Query("SELECT DISTINCT u FROM User u JOIN FETCH u.orders")
Page<User> findAllWithOrdersPaged(Pageable pageable); // 위험! 전체를 메모리에 올려서 페이징
```

### 해결 방법 2: @EntityGraph

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // 기존 메서드에 Fetch Join 효과 추가 (JOIN FETCH와 동일)
    @EntityGraph(attributePaths = {"orders"})
    List<User> findAll();  // findAll 오버라이드

    // 중첩 연관관계도 가능
    @EntityGraph(attributePaths = {"orders", "orders.product"})
    Optional<User> findWithOrdersAndProductsById(Long id);
}
```

### 해결 방법 3: @BatchSize (페이징 + N+1 함께 해결)

```java
@Entity
public class User {
    @Id private Long id;

    // N+1이 발생해도 IN 절로 묶어서 쿼리 수를 줄임
    @BatchSize(size = 100)
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
}

// 실행 결과 (유저 100명):
// SELECT * FROM users LIMIT 10 OFFSET 0       ← 1번 (페이징)
// SELECT * FROM orders WHERE user_id IN (1,2,...,10)  ← 1번 (배치)
// 합계: 2번 쿼리 (vs 11번)
```

```yaml
# 전역 BatchSize 설정 (application.yml)
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

### 해결 방법 4: DTO 직접 조회 (가장 성능 좋음)

```java
// 엔티티 전체 로드 없이 필요한 컬럼만 직접 DTO로
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("""
        SELECT new com.example.dto.UserOrderSummary(
            u.id,
            u.name,
            COUNT(o.id)
        )
        FROM User u
        LEFT JOIN u.orders o
        GROUP BY u.id, u.name
        ORDER BY u.name
        """)
    List<UserOrderSummary> findUserOrderSummaries();
    // SQL 1번, 필요한 컬럼만, 불필요한 엔티티 로드 없음
}

public record UserOrderSummary(Long userId, String userName, Long orderCount) {}
```

### 해결 방법 5: QueryDSL Projections

```java
@Repository
@RequiredArgsConstructor
public class UserQueryRepository {

    private final JPAQueryFactory queryFactory;

    // 페이징 + N+1 없이 DTO 직접 조회
    public Page<UserOrderSummary> findSummaries(Pageable pageable) {
        List<UserOrderSummary> content = queryFactory
            .select(Projections.constructor(UserOrderSummary.class,
                user.id,
                user.name,
                order.id.count()
            ))
            .from(user)
            .leftJoin(order).on(order.userId.eq(user.id))
            .groupBy(user.id, user.name)
            .orderBy(user.name.asc())
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .fetch();

        Long total = queryFactory
            .select(user.count())
            .from(user)
            .fetchOne();

        return new PageImpl<>(content, pageable, total);
    }
}
```

### N+1 탐지 방법

```yaml
# application.yml - SQL 로그 출력
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE  # 바인딩 파라미터도 출력
```

```java
// 테스트에서 쿼리 수 검증 (Hibernate Statistics)
@SpringBootTest
class UserServiceTest {

    @Autowired
    private EntityManagerFactory emf;

    @Autowired
    private UserService userService;

    @Test
    @DisplayName("N+1 쿼리가 발생하지 않아야 한다")
    void shouldNotHaveNPlusOneProblem() {
        // given
        Statistics stats = emf.unwrap(SessionFactory.class).getStatistics();
        stats.setStatisticsEnabled(true);
        stats.clear();

        // when
        userService.getAllUserSummaries();

        // then
        long queryCount = stats.getPrepareStatementCount();
        assertThat(queryCount)
            .as("쿼리 수가 2번을 초과하면 N+1 문제")
            .isLessThanOrEqualTo(2);
    }
}
```

### 상황별 해결 방법 선택

```
단건 조회 + 연관관계 필요     → Fetch Join 또는 @EntityGraph
목록 조회 + 페이징 없음       → Fetch Join
목록 조회 + 페이징 있음       → @BatchSize 또는 DTO 직접 조회
복잡한 검색 + 페이징          → QueryDSL + DTO Projections
성능이 가장 중요              → DTO 직접 조회 (엔티티 매핑 비용 없음)
```

---

## 3. 커넥션 풀 (Connection Pool)

### 개념

DB 연결 생성은 비용이 크다 (TCP 3-way handshake, DB 인증, 세션 초기화 = 수십~수백ms).
커넥션 풀은 **미리 연결을 만들어 두고 재사용**하는 패턴.

```
풀 없을 때:
  요청 → TCP 연결 → DB 인증 → 쿼리 → 연결 종료  (매 요청마다 수십~수백ms)

풀 있을 때:
  요청 → 풀에서 연결 꺼냄(1ms) → 쿼리 → 풀에 반납
```

### HikariCP 설정 (Spring Boot 기본)

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=Asia/Seoul
    username: user
    password: password
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      # 핵심 설정
      maximum-pool-size: 10        # 최대 커넥션 수 (핵심!)
      minimum-idle: 5              # 최소 유지 커넥션 수

      # 타임아웃 설정
      connection-timeout: 30000    # 풀에서 연결 못 얻으면 에러까지 대기 (30초)
      idle-timeout: 600000         # 유휴 커넥션 제거 시간 (10분)
      max-lifetime: 1800000        # 커넥션 최대 생존 시간 (30분) ← DB timeout보다 짧게!
      keepalive-time: 60000        # 연결 유효성 검사 주기 (60초)

      # 디버깅
      pool-name: MainPool
      leak-detection-threshold: 10000  # 10초 이상 반납 안 되면 경고 로그
```

### 적절한 풀 사이즈 계산

```
HikariCP 권장 공식:
  pool_size = (CPU 코어 수 × 2) + effective_spindle_count

  effective_spindle_count:
    - HDD: 실제 물리 디스크 수
    - SSD: 1

예시) 4코어 서버, SSD:
  pool_size = (4 × 2) + 1 = 9  → 10으로 설정

주의: 서버가 여러 대면 전체 커넥션 수 = 서버 수 × pool_size
  서버 10대 × pool_size 10 = 100개 커넥션
  → DB 서버가 감당 가능한지 확인 (MySQL 기본 max_connections = 151)

너무 많은 커넥션의 문제:
  - DB 서버 메모리 과부하 (커넥션당 메모리 사용)
  - DB 내부 스레드 컨텍스트 스위칭 오버헤드
  - 오히려 성능 저하 발생 가능
```

### 커넥션 풀 모니터링

```java
// Actuator + Micrometer로 HikariCP 메트릭 자동 노출
// build.gradle: implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, hikaricp
```

```
모니터링할 핵심 메트릭:
  hikaricp.connections.active   → 현재 사용 중인 커넥션 수
  hikaricp.connections.idle     → 유휴 커넥션 수
  hikaricp.connections.pending  → 커넥션 기다리는 스레드 수 (이 값이 지속적으로 높으면 pool 부족!)
  hikaricp.connections.timeout  → 타임아웃 발생 횟수 (0이어야 정상)

경고 신호:
  pending > 0 이 지속됨 → maximum-pool-size 증가 필요
  timeout > 0 발생 → 즉시 조사 필요
```

### 커넥션 누수 (Connection Leak)

```java
// 누수 발생 예시: close() 하지 않음
@Service
public class LeakyService {

    @Autowired
    private DataSource dataSource;

    // 절대 이렇게 하면 안 됨!
    public void badMethod() throws SQLException {
        Connection conn = dataSource.getConnection(); // 커넥션 획득
        Statement stmt = conn.createStatement();
        stmt.execute("SELECT 1");
        // conn.close() 누락! → 커넥션 풀로 반납 안 됨 → 누수!
    }

    // 올바른 방법 1: try-with-resources
    public void goodMethod1() throws SQLException {
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute("SELECT 1");
        } // 자동으로 close() 호출 → 풀에 반납
    }

    // 올바른 방법 2: JdbcTemplate (Spring이 관리)
    @Autowired
    private JdbcTemplate jdbcTemplate;

    public void goodMethod2() {
        jdbcTemplate.queryForObject("SELECT 1", Integer.class);
        // Spring이 알아서 커넥션 획득/반납 처리
    }
}
```

```
누수 감지:
  leak-detection-threshold: 10000 설정 시
  → 10초 이상 반납 안 된 커넥션 있으면 WARN 로그 출력
  → 로그에 누수된 커넥션의 스택 트레이스도 함께 출력 → 버그 위치 파악 가능
```

### Read/Write 분리 (Replica 라우팅)

```java
// 읽기는 Replica, 쓰기는 Primary로 자동 라우팅

public class ReplicationRoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        boolean isReadOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        String target = isReadOnly ? "replica" : "primary";
        log.debug("DataSource 라우팅: {}", target);
        return target;
    }
}

@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource dataSource() {
        ReplicationRoutingDataSource routing = new ReplicationRoutingDataSource();

        Map<Object, Object> sources = new HashMap<>();
        sources.put("primary", primaryDataSource());
        sources.put("replica", replicaDataSource());

        routing.setTargetDataSources(sources);
        routing.setDefaultTargetDataSource(primaryDataSource());

        // LazyConnectionDataSourceProxy: 실제 쿼리 시점에 DataSource 결정
        // (트랜잭션 시작 시점이 아니라 첫 쿼리 시점에 readOnly 여부 확인 가능)
        return new LazyConnectionDataSourceProxy(routing);
    }
}

// 사용
@Service
public class ProductService {

    @Transactional(readOnly = true)  // → Replica로 라우팅
    public List<ProductDto> findAll() { ... }

    @Transactional  // → Primary로 라우팅
    public Product create(ProductRequest request) { ... }
}
```

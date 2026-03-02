# 샤딩 / 복제 / CAP 이론 / NoSQL

---

## 1. 샤딩 (Sharding)

### 개념

데이터를 **여러 DB 서버에 수평 분할**해서 저장하는 방법.
단일 서버의 저장 용량 및 처리량 한계를 극복하기 위한 Scale-Out 전략.

```
샤딩 없음:
  모든 데이터 → 단일 DB 서버 (1억 건 → 성능 한계)

샤딩:
  Shard 0: user_id 0,3,6,9... → DB 서버 A
  Shard 1: user_id 1,4,7,10... → DB 서버 B
  Shard 2: user_id 2,5,8,11... → DB 서버 C
```

### 파티셔닝 vs 샤딩

```
파티셔닝: 하나의 DB 서버 내에서 테이블을 물리적으로 분할
  → DB 서버는 하나, 내부적으로 파티션만 나뉨
  → 관리 단순, 쿼리 옵티마이저가 인지함

샤딩: 여러 DB 서버로 데이터를 분산
  → 실제 별도 서버들
  → 관리 복잡, 애플리케이션이 어느 서버에 있는지 알아야 함
```

### 샤딩 방법

```
Hash-based Sharding:
  shard_num = hash(user_id) % shard_count
  → 균등 분배 O
  → 범위 쿼리 비효율 (여러 샤드 조회 필요)
  → 샤드 추가 시 대규모 리밸런싱 필요

Range-based Sharding:
  Shard 0: id 1 ~ 1,000,000
  Shard 1: id 1,000,001 ~ 2,000,000
  → 범위 쿼리 효율적
  → 특정 샤드에 데이터 몰릴 수 있음 (Hotspot)
  → 신규 사용자가 항상 마지막 샤드에 몰림

Directory-based Sharding:
  별도 Lookup 테이블: user_id → Shard ID 매핑
  → 유연성 최고 (샤드 추가 용이)
  → Lookup 테이블 자체가 병목/단일 장애점
```

### Spring에서 Hash-based Sharding 구현

```java
// 샤드 컨텍스트: 현재 요청이 어느 샤드를 사용할지 ThreadLocal로 관리
public class ShardContextHolder {
    private static final ThreadLocal<Integer> shardKey = new ThreadLocal<>();

    public static void setShardKey(int key) { shardKey.set(key); }
    public static Integer getShardKey() { return shardKey.get(); }
    public static void clear() { shardKey.remove(); }
}
```

```java
// AbstractRoutingDataSource: 샤드 키에 따라 DataSource 선택
public class ShardingRoutingDataSource extends AbstractRoutingDataSource {

    private final int shardCount;

    @Override
    protected Object determineCurrentLookupKey() {
        Integer key = ShardContextHolder.getShardKey();
        return (key == null) ? 0 : Math.abs(key % shardCount);
    }
}
```

```java
@Configuration
public class ShardingDataSourceConfig {

    @Bean
    public DataSource dataSource() {
        int shardCount = 3;
        ShardingRoutingDataSource routing = new ShardingRoutingDataSource(shardCount);

        Map<Object, Object> sources = new HashMap<>();
        sources.put(0, createDataSource("jdbc:mysql://shard0:3306/db"));
        sources.put(1, createDataSource("jdbc:mysql://shard1:3306/db"));
        sources.put(2, createDataSource("jdbc:mysql://shard2:3306/db"));

        routing.setTargetDataSources(sources);
        routing.setDefaultTargetDataSource(sources.get(0));
        routing.afterPropertiesSet();

        return routing;
    }
}
```

```java
// 샤드 인식 Service
@Service
@RequiredArgsConstructor
public class UserShardService {

    private final UserRepository userRepository;

    public User findUser(Long userId) {
        try {
            ShardContextHolder.setShardKey(userId.intValue()); // 샤드 결정
            return userRepository.findById(userId).orElseThrow();
        } finally {
            ShardContextHolder.clear(); // 반드시 정리! (ThreadLocal 누수 방지)
        }
    }

    // AOP로 샤드 설정 자동화 (더 깔끔한 방법)
    // @ShardKey(paramName = "userId") 같은 커스텀 어노테이션 만들어 사용
}
```

### 샤딩의 문제점

```java
// 1. Cross-Shard JOIN 불가 → 애플리케이션에서 조합
public List<OrderWithUserDto> findOrdersWithUser(List<Long> userIds) {
    // 각 샤드에서 개별 조회 후 메모리에서 조합
    Map<Long, User> users = userIds.stream()
        .collect(Collectors.toMap(
            id -> id,
            id -> {
                ShardContextHolder.setShardKey(id.intValue());
                try {
                    return userRepository.findById(id).orElseThrow();
                } finally {
                    ShardContextHolder.clear();
                }
            }
        ));

    List<Order> orders = orderRepository.findByUserIdIn(userIds);
    return orders.stream()
        .map(o -> new OrderWithUserDto(o, users.get(o.getUserId())))
        .toList();
}

// 2. 분산 트랜잭션 문제
// 두 샤드에 걸친 트랜잭션은 ACID 보장 어려움
// → Saga 패턴, 2PC 등으로 해결 (복잡도 매우 높음)

// 3. 샤드 추가 시 리밸런싱
// Hash % 3 → Hash % 4 로 변경 시 거의 모든 데이터 이동 필요
// → Consistent Hashing으로 완화 가능
```

---

## 2. 복제 (Replication)

### 개념

Primary DB의 데이터를 Replica DB에 **실시간으로 복사해 동기화**하는 것.
고가용성(HA)과 읽기 성능 향상을 위해 사용.

### Primary-Replica 구조

```
[쓰기 요청] → Primary DB ──── Binary Log ────→ Replica DB (복제)
                                                      ↑
                                               [읽기 요청]

Primary 장점:
  - 쓰기 처리 집중
  - 장애 시 Replica가 Primary로 승격 (Failover)

Replica 장점:
  - 읽기 요청 분산 처리 (읽기 성능 향상)
  - Primary 장애 시 데이터 손실 최소화 (백업)
```

### 동기 복제 vs 비동기 복제

```
동기 복제 (Synchronous):
  Primary → 쓰기 → Replica에 전파 → Replica 완료 확인 → 클라이언트 응답
  장점: 데이터 손실 없음 (RPO = 0)
  단점: Replica 응답 대기 → 쓰기 지연 증가

비동기 복제 (Asynchronous):
  Primary → 쓰기 → 클라이언트 응답 → (백그라운드에서 Replica 전파)
  장점: 쓰기 빠름
  단점: Primary 장애 시 일부 데이터 손실 가능 (Replication Lag만큼)
```

### Replication Lag 문제와 해결

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    // 사용자 정보 업데이트
    @Transactional  // Primary에 씀
    public User updateProfile(Long userId, String newName) {
        User user = userRepository.findById(userId).orElseThrow();
        user.updateName(newName);
        return user;
        // commit → Primary에는 즉시 반영
        // Replica에는 수 ms ~ 수 초 후 반영 (Replication Lag)
    }

    // 문제: 업데이트 직후 Replica에서 읽으면 이전 값이 나올 수 있음
    @Transactional(readOnly = true)  // Replica에서 읽음
    public UserDto getProfile(Long userId) {
        return UserDto.from(userRepository.findById(userId).orElseThrow());
        // 방금 업데이트했는데 이전 값 반환 가능!
    }

    // 해결책 1: 업데이트 후 즉시 조회는 Primary에서 (readOnly = false)
    @Transactional  // Primary에서 읽음
    public UserDto getMyProfileAfterUpdate(Long userId) {
        return UserDto.from(userRepository.findById(userId).orElseThrow());
    }

    // 해결책 2: 같은 트랜잭션 안에서 처리 (업데이트하고 바로 반환)
    @Transactional
    public UserDto updateAndReturn(Long userId, String newName) {
        User user = userRepository.findById(userId).orElseThrow();
        user.updateName(newName);
        return UserDto.from(user); // 메모리에 있는 값 반환 (추가 쿼리 없음)
    }
}
```

### Spring에서 Read/Write 분리 구현

```java
// (04-join-n1-connection-pool.md의 ReplicationRoutingDataSource 참고)

// 핵심 원리:
// @Transactional(readOnly = true) → TransactionSynchronizationManager.isReadOnly() = true
// → ReplicationRoutingDataSource에서 "replica" 키 반환
// → Replica DataSource로 연결

// 사용 패턴
@Service
public class ProductService {

    @Transactional(readOnly = true)  // Replica
    public Page<ProductDto> findAll(Pageable pageable) { ... }

    @Transactional(readOnly = true)  // Replica
    public ProductDto findById(Long id) { ... }

    @Transactional  // Primary
    public Product create(CreateProductRequest req) { ... }

    @Transactional  // Primary
    public Product update(Long id, UpdateProductRequest req) { ... }
}
```

---

## 3. CAP 이론

### 개념

분산 시스템에서 다음 3가지를 **동시에 모두 만족할 수 없다**는 이론 (2000년, Eric Brewer).

```
C (Consistency - 일관성):
  모든 노드가 같은 시점에 같은 데이터를 봄
  쓰기 후 읽기는 항상 최신 값을 반환

A (Availability - 가용성):
  모든 요청에 (오류 없이) 응답을 반환
  일부 노드가 장애여도 나머지가 응답

P (Partition Tolerance - 파티션 허용):
  네트워크 파티션(분리)이 발생해도 시스템이 계속 동작
```

### 왜 P는 포기할 수 없나?

```
분산 시스템 = 여러 서버가 네트워크로 연결
네트워크 장애는 반드시 발생함 (케이블 단선, 스위치 오작동, etc.)
→ P(Partition Tolerance)는 선택이 아닌 필수

결론:
  네트워크 분리 발생 시 C(일관성)을 지키려면 일부 요청을 거부해야 함 (A 포기)
  A(가용성)을 지키려면 오래된 데이터를 반환할 수 있음 (C 포기)
  → C vs A 중 하나를 선택
```

### CP vs AP

```
CP 시스템 (일관성 + 파티션 허용, 가용성 희생):
  네트워크 분리 → 일부 노드가 응답 거부
  항상 최신 데이터 보장
  예: ZooKeeper, HBase, etcd, Redis Sentinel

AP 시스템 (가용성 + 파티션 허용, 일관성 희생):
  네트워크 분리 → 모든 노드가 응답하지만 일관성 보장 안 됨
  Eventually Consistent (최종적 일관성)
  예: Cassandra, DynamoDB, CouchDB, Redis Cluster
```

```java
// CP 선택 예시: 분산 락 (ZooKeeper 기반)
// 네트워크 분리 시 응답 안 할 수 있지만 락의 정확성은 보장
@Service
public class DistributedLockService {

    private final CuratorFramework zookeeperClient;

    public <T> T executeWithLock(String lockPath, Callable<T> action) throws Exception {
        InterProcessMutex lock = new InterProcessMutex(zookeeperClient, lockPath);
        try {
            if (!lock.acquire(5, TimeUnit.SECONDS)) {
                throw new LockAcquisitionException("락 획득 실패");
            }
            return action.call();
        } finally {
            lock.release();
        }
    }
}
```

```java
// AP 선택 예시: Redis 캐시 (최종적 일관성 허용)
// 네트워크 분리 시에도 응답하지만 일부 노드는 오래된 값 반환 가능
@Service
@RequiredArgsConstructor
public class ProductCacheService {

    private final RedisTemplate<String, ProductDto> redisTemplate;

    @Cacheable(value = "products", key = "#id")
    public ProductDto findById(Long id) {
        // 캐시에 없으면 DB 조회
        // 캐시는 Eventually Consistent (DB와 캐시가 잠시 달라도 허용)
        return productRepository.findById(id).map(ProductDto::from).orElseThrow();
    }
}
```

### 실무에서의 선택 기준

```
CP가 필요한 경우:
  - 금융 거래, 결제 (이중 결제, 잔액 불일치 절대 안 됨)
  - 재고 차감 (중복 판매 불가)
  - 예약 시스템 (중복 예약 불가)
  - 분산 락

AP가 허용되는 경우:
  - SNS 좋아요/조회수 (잠깐 다른 값이어도 OK)
  - 상품 검색 인덱스 (약간 오래된 결과 허용)
  - 추천 시스템 (실시간 최신성 불필요)
  - 세션 데이터 (약간의 지연 허용)
```

---

## 4. NoSQL vs RDBMS

### RDBMS 특징

```
장점:
  - 강력한 데이터 무결성 (스키마, 제약조건, 외래키)
  - ACID 트랜잭션
  - 복잡한 쿼리 (JOIN, 집계, 서브쿼리)
  - SQL 표준, 성숙한 생태계

단점:
  - 수평 확장(Scale-Out)이 어려움
  - 스키마 변경 비용 높음 (ALTER TABLE은 대용량에서 위험)
  - 비정형 데이터 처리 부적합
```

### Redis (Key-Value Store)

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}

@Service
@RequiredArgsConstructor
public class RedisUsageService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final StringRedisTemplate stringRedisTemplate;

    // 1. 캐싱
    public void cacheProduct(Long id, ProductDto dto) {
        redisTemplate.opsForValue().set(
            "product:" + id,
            dto,
            Duration.ofMinutes(10)
        );
    }

    // 2. 세션 관리 (@EnableRedisHttpSession 설정 시 자동)
    // HttpSession이 Redis에 저장됨

    // 3. 실시간 랭킹 (Sorted Set)
    public void updateScore(String userId, double score) {
        redisTemplate.opsForZSet().add("leaderboard", userId, score);
    }

    public Set<Object> getTopRanking(int count) {
        return redisTemplate.opsForZSet()
            .reverseRange("leaderboard", 0, count - 1);
    }

    // 4. 분산 락 (Redisson)
    // RLock lock = redissonClient.getLock("lock:product:" + productId);

    // 5. 원자적 카운터
    public long incrementViewCount(Long articleId) {
        return redisTemplate.opsForValue()
            .increment("views:article:" + articleId);
    }
}
```

### Spring Cache Abstraction (@Cacheable)

```java
@Service
@RequiredArgsConstructor
@CacheConfig(cacheNames = "products") // 이 클래스의 기본 캐시 이름
public class ProductCacheService {

    private final ProductRepository productRepository;

    // 캐시에 있으면 반환, 없으면 DB 조회 후 캐시에 저장
    @Cacheable(key = "#id")
    public ProductDto findById(Long id) {
        return productRepository.findById(id)
            .map(ProductDto::from)
            .orElseThrow();
    }

    // 수정 시 캐시 갱신
    @CachePut(key = "#id")
    public ProductDto update(Long id, UpdateProductRequest request) {
        Product product = productRepository.findById(id).orElseThrow();
        product.update(request);
        return ProductDto.from(product);
    }

    // 삭제 시 캐시 제거
    @CacheEvict(key = "#id")
    public void delete(Long id) {
        productRepository.deleteById(id);
    }

    // 전체 캐시 제거
    @CacheEvict(allEntries = true)
    public void clearAll() { }
}
```

### MongoDB (Document Store)

```java
@Document(collection = "product_reviews")
public class ProductReview {
    @Id
    private String id;

    @Indexed  // MongoDB 인덱스
    private Long productId;

    private Long userId;
    private String content;

    @Min(1) @Max(5)
    private int rating;

    // 비정형 속성 (상품 카테고리마다 다를 수 있음)
    private Map<String, Object> metadata;

    @CreatedDate
    private LocalDateTime createdAt;
}

public interface ReviewRepository extends MongoRepository<ProductReview, String> {

    List<ProductReview> findByProductIdOrderByCreatedAtDesc(Long productId);

    // 집계: 평균 평점
    @Aggregation(pipeline = {
        "{ $match: { productId: ?0 } }",
        "{ $group: { _id: '$productId', avgRating: { $avg: '$rating' }, total: { $sum: 1 } } }"
    })
    ReviewStats getStats(Long productId);
}

// MongoTemplate으로 복잡한 쿼리
@Service
@RequiredArgsConstructor
public class ReviewQueryService {

    private final MongoTemplate mongoTemplate;

    public List<ProductReview> findHighRatedReviews(Long productId, int minRating) {
        Query query = new Query(
            Criteria.where("productId").is(productId)
                    .and("rating").gte(minRating)
        );
        query.with(Sort.by(Sort.Direction.DESC, "createdAt"));
        query.limit(20);

        return mongoTemplate.find(query, ProductReview.class);
    }
}
```

### 언제 무엇을 선택하나?

```
RDBMS (MySQL, PostgreSQL):
  → 주문, 결제, 회원 등 강한 일관성 필요
  → 복잡한 관계와 JOIN이 많은 데이터
  → 집계, 리포팅, 복잡한 쿼리

Redis:
  → 캐싱 (DB 부하 감소, 응답 속도 개선)
  → 세션 관리
  → 실시간 랭킹, 카운터
  → 분산 락, Pub/Sub
  → 만료 시간(TTL) 있는 임시 데이터

MongoDB:
  → 스키마가 자주 변하거나 유연한 초기 개발
  → 상품 카탈로그 (카테고리마다 속성이 다름)
  → 이벤트 로그, 사용자 활동 데이터

Cassandra:
  → 시계열 데이터 (IoT 센서, 로그)
  → 쓰기가 극단적으로 많은 경우
  → 지역 분산 데이터
```

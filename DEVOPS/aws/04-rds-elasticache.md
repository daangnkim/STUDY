# RDS / ElastiCache

---

## 1. RDS (Relational Database Service)

AWS가 관리해주는 관계형 DB 서비스. 패치, 백업, 복제, 스냅샷을 AWS가 자동으로 처리한다.

### 지원 DB 엔진

```
MySQL        → 가장 많이 사용
PostgreSQL   → 기능 풍부, 오픈소스
MariaDB      → MySQL 호환
Oracle       → 엔터프라이즈 (라이선스 비쌈)
SQL Server   → Windows 환경
Aurora       → AWS 자체 개발 (MySQL/PostgreSQL 호환, 성능 최대 5배)
```

### RDS Multi-AZ (고가용성)

```
Primary DB (ap-northeast-2a)
  ↓ 동기 복제 (Synchronous)
Standby DB (ap-northeast-2c)   ← 평소에는 트래픽 없음

장애 발생 시:
  → DNS가 Standby로 자동 전환 (Failover)
  → 약 1~2분 소요 (다운타임 발생)
  → 애플리케이션은 DNS로 접근하므로 코드 변경 불필요

주의:
  - Standby는 읽기 불가 (Active-Passive)
  - 읽기 분산은 Read Replica로 별도 처리
```

### RDS Read Replica (읽기 복제본)

```
Primary DB
  ↓ 비동기 복제 (Asynchronous)
┌────────────────────────────────┐
│  Read Replica 1 (같은 리전)    │  ← 읽기 전용
│  Read Replica 2 (같은 리전)    │
│  Read Replica 3 (다른 리전)    │  ← 재해 복구용
└────────────────────────────────┘

Spring Boot 설정:
  @Transactional(readOnly = true) → Replica 엔드포인트
  @Transactional                  → Primary 엔드포인트
```

```yaml
# application.yml - Primary/Replica 분리
spring:
  datasource:
    primary:
      url: jdbc:mysql://primary.xxxxx.ap-northeast-2.rds.amazonaws.com:3306/mydb
    replica:
      url: jdbc:mysql://replica.xxxxx.ap-northeast-2.rds.amazonaws.com:3306/mydb
```

### RDS Aurora

```
Aurora 특징:
  - MySQL/PostgreSQL 호환
  - 스토리지: 3개 AZ에 6개 복사본 자동 저장
  - 최대 15개 Read Replica (RDS는 5개)
  - 자동 스토리지 확장 (10GB ~ 128TB)
  - Serverless 옵션 (트래픽 없을 때 0으로 스케일 다운)

Aurora Cluster Endpoint 구조:
  Writer Endpoint: 쓰기 (Primary에 연결, Failover 시 자동 전환)
  Reader Endpoint: 읽기 (Replica들에 로드밸런싱)
  Custom Endpoint: 특정 Replica 그룹 지정
```

### RDS 백업

```
자동 백업:
  - 매일 지정한 시간에 스냅샷 + 트랜잭션 로그
  - 보존 기간: 1~35일 (기본 7일)
  - 특정 시점으로 복원 가능 (Point-in-Time Recovery)

수동 스냅샷:
  - 원하는 시점에 수동으로 생성
  - 삭제 전까지 영구 보관
  - 다른 리전/계정으로 복사 가능

RDS 삭제 시 final snapshot 권장:
  aws rds delete-db-instance \
    --db-instance-identifier mydb \
    --final-db-snapshot-identifier mydb-final-backup
```

### RDS 파라미터 그룹

```
DB 엔진 설정을 코드로 관리
  max_connections
  innodb_buffer_pool_size
  slow_query_log = 1
  long_query_time = 1  (1초 이상 쿼리 로그)
  character_set_server = utf8mb4

파라미터 그룹 수정 후:
  Dynamic 파라미터 → 즉시 적용
  Static 파라미터  → DB 재시작 필요
```

---

## 2. ElastiCache

AWS가 관리해주는 **인메모리 캐시** 서비스. Redis 또는 Memcached를 지원한다.

### Redis vs Memcached

```
Redis 선택 (대부분의 경우):
  - 다양한 자료구조 (String, List, Set, Sorted Set, Hash)
  - 영속성 (디스크 저장 가능)
  - Pub/Sub 지원
  - 클러스터, 복제 지원
  - Lua 스크립트
  - 세션 저장, 캐시, 분산 락, 랭킹 시스템

Memcached 선택:
  - 단순 Key-Value 캐시
  - 멀티스레드 (단순 캐시에서 더 빠를 수 있음)
  - 수평 확장이 단순
```

### ElastiCache for Redis 구성

```
Standalone (개발용):
  단일 노드, 장애 시 데이터 유실

Redis Cluster (운영 권장):
  Primary + Replica 구성
  Primary 장애 시 Replica가 자동으로 Primary 승격 (Failover ~30초)

Redis Cluster Mode Enabled:
  데이터를 여러 샤드에 분산 (수평 확장)
  최대 500개 노드, 500TB+ 캐시
```

### Spring Boot + ElastiCache 연동

```yaml
# application.yml
spring:
  data:
    redis:
      host: myredis.xxxxx.cache.amazonaws.com   # ElastiCache 엔드포인트
      port: 6379
      # 클러스터 모드 활성화 시
      cluster:
        nodes:
          - myredis.cluster.xxxxx.cache.amazonaws.com:6379
```

```java
@Configuration
@EnableCaching
public class RedisConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .withCacheConfiguration("products",
                config.entryTtl(Duration.ofMinutes(30)))  // products 캐시는 30분
            .build();
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    @Cacheable(value = "products", key = "#id")
    public ProductDto findById(Long id) {
        // 첫 요청: DB 조회 후 Redis에 저장
        // 이후 요청: Redis에서 즉시 반환 (DB 미조회)
        return productRepository.findById(id)
            .map(ProductDto::from)
            .orElseThrow();
    }

    @CacheEvict(value = "products", key = "#id")
    public void updateProduct(Long id, UpdateRequest req) {
        // 수정 시 캐시 삭제 (다음 조회에서 DB에서 새로 로드)
        Product product = productRepository.findById(id).orElseThrow();
        product.update(req);
    }
}
```

### 캐시 전략

```
Cache-Aside (Lazy Loading, 가장 일반적):
  1. 캐시 조회 → HIT → 반환
  2. MISS → DB 조회 → 캐시 저장 → 반환
  단점: 첫 요청은 항상 DB 조회 (Cache Warming으로 완화)

Write-Through:
  쓸 때 DB + 캐시 동시 업데이트
  장점: 캐시 항상 최신
  단점: 쓰기 지연 증가

Write-Behind (Write-Back):
  캐시에만 먼저 쓰고 → 나중에 비동기로 DB에 반영
  장점: 쓰기 성능 최고
  단점: 장애 시 데이터 유실 가능

TTL 전략:
  너무 짧으면 캐시 효율 ↓ (잦은 DB 조회)
  너무 길면 오래된 데이터 반환 (Stale Data)
  수정이 잦은 데이터: 짧은 TTL + CacheEvict
  수정이 드문 데이터: 긴 TTL (상품 정보, 카테고리 등)
```

### ElastiCache 보안

```
VPC 내부에 배치:
  - 인터넷에서 직접 접근 불가
  - 같은 VPC의 EC2/ECS만 접근 가능

보안 그룹:
  - 6379 포트를 EC2 보안 그룹에서만 허용

암호화:
  - 전송 중 암호화 (TLS in-transit)
  - 저장 암호화 (at-rest)
  - AUTH 토큰 설정 (비밀번호)
```

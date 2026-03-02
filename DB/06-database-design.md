# 데이터베이스 설계 방법론

---

## 1. 데이터 모델링 단계

```
요구사항 분석
    ↓
개념적 설계 (ERD - Entity Relationship Diagram)
    ↓
논리적 설계 (정규화, 테이블 구조 정의)
    ↓
물리적 설계 (인덱스, 파티셔닝, 스토리지 최적화)
    ↓
구현 (DDL 작성, 운영)
```

---

## 2. ERD (Entity Relationship Diagram)

### 핵심 구성 요소

```
Entity (엔티티):
  실제 세계의 객체 또는 개념
  예: 회원, 주문, 상품, 카테고리

Attribute (속성):
  엔티티의 특성
  예: 회원(id, 이름, 이메일, 생년월일)

Relationship (관계):
  엔티티 간의 연관성
  예: 회원 "주문" 상품

Cardinality (카디널리티):
  관계의 수량적 표현 - 1:1, 1:N, M:N
```

### 카디널리티 종류

```
1:1 관계:
  회원 ─── 회원_상세정보
  한 회원은 하나의 상세정보를 가짐
  → 주 테이블에 FK를 두거나, 같은 PK 공유

1:N 관계 (가장 일반적):
  회원 ───< 주문
  한 회원은 여러 주문을 가짐. 주문은 하나의 회원에 속함
  → N 쪽(주문)에 FK (member_id) 배치

M:N 관계:
  상품 >───< 태그
  상품은 여러 태그를 가짐. 태그는 여러 상품에 붙음
  → 중간 테이블(product_tag)로 1:N + N:1로 분해 필수
```

### 실무 ERD 예시: 쇼핑몰

```
[Member]
  member_id   PK
  email       UK, NOT NULL
  name        NOT NULL
  phone
  created_at

[Address]  (1:N ← Member)
  address_id  PK
  member_id   FK → Member
  address
  is_default  BOOL
  created_at

[Category]  (셀프 참조 - 계층 구조)
  category_id   PK
  parent_id     FK → Category (NULL이면 최상위)
  name          NOT NULL
  sort_order

[Product]  (1:N ← Category)
  product_id    PK
  category_id   FK → Category
  name          NOT NULL
  price         NOT NULL
  stock         NOT NULL DEFAULT 0
  status        ENUM(ACTIVE, INACTIVE, DELETED)
  created_at

[Order]  (1:N ← Member)
  order_id      PK
  member_id     FK → Member
  total_amount  NOT NULL
  status        ENUM(PENDING, PAID, SHIPPING, DELIVERED, CANCELLED)
  ordered_at    NOT NULL

[OrderItem]  (M:N 분해: Order + Product)
  order_item_id PK
  order_id      FK → Order
  product_id    FK → Product
  quantity      NOT NULL
  unit_price    NOT NULL  ← 주문 당시 가격 스냅샷 (중요!)

[Review]  (1:1 ← OrderItem)
  review_id     PK
  order_item_id FK → OrderItem  UK
  rating        TINYINT (1~5)
  content       TEXT
  created_at
```

---

## 3. 정규화 (Normalization)

데이터 중복을 최소화하고 무결성을 높이기 위해 테이블을 구조화하는 과정.

### 이상(Anomaly) 현상

```
삽입 이상: 데이터 삽입 시 불필요한 데이터도 함께 삽입해야 하는 문제
삭제 이상: 데이터 삭제 시 의도치 않게 다른 데이터도 삭제되는 문제
갱신 이상: 중복 데이터가 있어 일부만 수정되어 데이터 불일치 발생
```

### 제1정규형 (1NF) - 원자성

```
규칙: 모든 속성 값이 원자적(더 이상 분해 불가)이어야 함

위반 예시:
  주문(주문ID, 상품목록)
  → 상품목록: "노트북,마우스,키보드" (반복 그룹)

1NF 변환:
  주문(주문ID, ...)
  주문상품(주문ID, 상품ID, 수량)  ← 원자적으로 분리
```

### 제2정규형 (2NF) - 부분 함수 종속 제거

```
규칙: 1NF + 기본키가 복합키일 때, 비키 속성이 기본키 전체에 종속 (부분 종속 제거)

위반 예시:
  주문상품(주문ID, 상품ID, 수량, 상품명, 상품가격)
  → 상품명, 상품가격은 상품ID에만 종속 (부분 종속)

2NF 변환:
  주문상품(주문ID, 상품ID, 수량)   ← 복합 PK 전체에 종속
  상품(상품ID, 상품명, 상품가격)    ← 상품ID에만 종속 → 분리
```

### 제3정규형 (3NF) - 이행 함수 종속 제거

```
규칙: 2NF + 비키 속성이 기본키에만 종속 (이행 종속 제거)

위반 예시:
  주문(주문ID, 회원ID, 회원이름, 회원이메일)
  → 회원이름, 회원이메일은 회원ID에 종속 (이행 종속)
  → 회원 정보 변경 시 모든 주문 레코드 수정 필요

3NF 변환:
  주문(주문ID, 회원ID)         ← 직접 종속만
  회원(회원ID, 회원이름, 회원이메일)  ← 분리
```

### BCNF (Boyce-Codd NF)

```
3NF보다 엄격한 형태.
모든 결정자가 후보키여야 함.

실무에서는 3NF까지 보통 적용.
BCNF까지 적용하면 성능 저하 가능 (조인 증가).
```

### 역정규화 (De-normalization)

```
정규화로 분리된 테이블을 성능을 위해 다시 합치는 것.

언제 사용:
  - 잦은 조인으로 성능 저하
  - 읽기가 쓰기보다 훨씬 많은 경우
  - 통계/집계 데이터 조회 빈번

예시:
  정규화: 주문 → 주문상품 → 상품 (3번 조인)
  역정규화: 주문상품에 상품명, 단가 컬럼 추가 (주문 당시 스냅샷)

주의:
  역정규화하면 데이터 일관성 관리 책임이 생김
  → 트리거 또는 애플리케이션 로직으로 동기화 필요
```

---

## 4. 키 설계

### PK(Primary Key) 선택

```
자연키 vs 대리키:

자연키 (Natural Key):
  이미 의미를 가진 속성 (주민번호, 이메일, 사업자번호)
  단점: 변경 시 FK 모두 업데이트, 외부에 노출 시 보안 위험

대리키 (Surrogate Key):
  의미 없는 식별자 (AUTO_INCREMENT, UUID)
  권장: 대부분의 경우 대리키 사용

숫자 vs UUID:
  AUTO_INCREMENT (BIGINT):
    - 작은 크기 (8 bytes)
    - B+Tree 인덱스 성능 최적 (순차 삽입)
    - 분산 환경에서 충돌 가능
    - 외부 노출 시 순서 예측 가능 (보안 취약)

  UUID (VARCHAR(36) or BINARY(16)):
    - 전역 유일성 보장 (분산 환경)
    - 16 bytes (BINARY 저장 시)
    - 랜덤 UUID(v4): B+Tree 페이지 분할 증가 → 삽입 성능 저하
    - UUID v7 (시간 정렬): 순차 증가 + 전역 유일 → 권장

  Snowflake ID (Twitter):
    - 64bit 정수 (타임스탬프 + 노드ID + 시퀀스)
    - 분산 환경 + 순차 + 숫자 PK
    - 외부 노출 안전 (예측 어려움)
```

```java
// JPA에서 UUID v7 사용 (Java 19+)
@Entity
public class Order {

    @Id
    @Column(columnDefinition = "BINARY(16)")
    private UUID id;

    @PrePersist
    public void generateId() {
        if (this.id == null) {
            this.id = UUID.randomUUID();  // 또는 UUIDv7 라이브러리
        }
    }
}

// 또는 TSID (Time-Sorted Unique Identifier) 라이브러리 활용
// https://github.com/f4b6a3/tsid-creator
@PrePersist
public void generateId() {
    this.id = TsidCreator.getTsid().toLong();  // 숫자형 TSID
}
```

### FK(Foreign Key) 설계

```
FK 제약 장단점:

장점:
  - 데이터 무결성 DB 레벨에서 보장
  - 잘못된 참조 방지

단점:
  - DML 성능 저하 (삽입/수정/삭제 시 참조 테이블 확인)
  - 대규모 데이터 마이그레이션 어려움
  - 마이크로서비스 환경에서 서비스 간 FK 불가

실무 선택:
  단일 모놀리식 서비스: FK 제약 권장
  마이크로서비스: FK 없이 애플리케이션에서 정합성 관리
  대용량 테이블: FK 없이 배치로 정합성 검증
```

---

## 5. 실무 설계 패턴

### Soft Delete (소프트 삭제)

```sql
-- 물리 삭제 대신 삭제 플래그
ALTER TABLE product ADD COLUMN deleted_at DATETIME NULL;

-- 삭제 처리
UPDATE product SET deleted_at = NOW() WHERE product_id = 1;

-- 조회 시 필터
SELECT * FROM product WHERE deleted_at IS NULL;
```

```java
// JPA - @Where 어노테이션으로 자동 필터
@Entity
@Where(clause = "deleted_at IS NULL")  // 조회 시 자동으로 WHERE 절 추가
@SQLDelete(sql = "UPDATE product SET deleted_at = NOW() WHERE product_id = ?")
public class Product {

    @Id
    private Long id;

    private String name;

    private LocalDateTime deletedAt;
}
```

### Audit 컬럼 (감사 이력)

```java
// Spring Data JPA Auditing
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private Long createdBy;  // 사용자 ID

    @LastModifiedBy
    private Long updatedBy;
}

@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {

    @Bean
    public AuditorAware<Long> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(auth -> ((UserPrincipal) auth.getPrincipal()).getId());
    }
}
```

### 이력 테이블 패턴 (변경 이력 보관)

```sql
-- 상품 가격 변경 이력
CREATE TABLE product_price_history (
    history_id  BIGINT AUTO_INCREMENT PRIMARY KEY,
    product_id  BIGINT NOT NULL,
    price       DECIMAL(15,2) NOT NULL,
    changed_by  BIGINT,
    changed_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_product_id (product_id)
);

-- 상품 가격 변경 트리거
CREATE TRIGGER product_price_audit
AFTER UPDATE ON product
FOR EACH ROW
BEGIN
    IF OLD.price <> NEW.price THEN
        INSERT INTO product_price_history (product_id, price, changed_at)
        VALUES (NEW.product_id, NEW.price, NOW());
    END IF;
END;
```

### 계층형 데이터 설계

```
카테고리처럼 계층(트리) 구조를 저장하는 방법

1. 인접 목록 모델 (Adjacency List) - 가장 단순
   category(id, parent_id, name)
   단점: 전체 트리 조회에 재귀 쿼리 필요

2. 경로 열거 (Path Enumeration)
   category(id, path, name)
   path: "1/3/7"  ← 조상 ID 경로
   장점: LIKE '1/3/%' 로 하위 트리 전체 쉽게 조회
   단점: 이동 시 경로 업데이트

3. 중첩 집합 모델 (Nested Set)
   category(id, lft, rgt, name)
   하위 트리 조회: WHERE lft BETWEEN parent.lft AND parent.rgt
   단점: 삽입/이동 시 lft/rgt 재계산

4. Closure Table (실무 권장)
   category(id, name)
   category_closure(ancestor_id, descendant_id, depth)
   장점: 조회/수정 모두 효율적
   단점: 저장 공간 증가 (노드 수 * 깊이)
```

```java
// MySQL 8.0+ 재귀 CTE로 계층 조회
/*
WITH RECURSIVE category_tree AS (
    SELECT category_id, parent_id, name, 0 AS depth
    FROM category
    WHERE parent_id IS NULL  -- 루트 노드

    UNION ALL

    SELECT c.category_id, c.parent_id, c.name, ct.depth + 1
    FROM category c
    INNER JOIN category_tree ct ON c.parent_id = ct.category_id
)
SELECT * FROM category_tree ORDER BY depth, name;
*/

@Query(value = """
    WITH RECURSIVE category_tree AS (
        SELECT category_id, parent_id, name, 0 AS depth
        FROM category WHERE parent_id IS NULL
        UNION ALL
        SELECT c.category_id, c.parent_id, c.name, ct.depth + 1
        FROM category c
        INNER JOIN category_tree ct ON c.parent_id = ct.category_id
    )
    SELECT * FROM category_tree ORDER BY depth, name
    """, nativeQuery = true)
List<CategoryProjection> findAllWithHierarchy();
```

---

## 6. 데이터 타입 선택

```
정수:
  TINYINT   (1 byte):  -128 ~ 127, 상태 값, 별점(1~5)
  SMALLINT  (2 bytes): -32,768 ~ 32,767, 연도
  INT       (4 bytes): 약 21억, 일반 ID
  BIGINT    (8 bytes): 매우 큰 수, PK (AUTO_INCREMENT 권장)

실수:
  DECIMAL(M, D): 정확한 소수 (금액, 세율)
    예: DECIMAL(15, 2) → 최대 9999999999999.99
  FLOAT/DOUBLE: 근사값 (과학 계산용, 금액에 절대 사용 금지)

문자:
  CHAR(N):    고정 길이, 빠름, 길이 변동 없는 값 (국가코드, 상태코드)
  VARCHAR(N): 가변 길이, 일반 문자열 (이름, 이메일)
  TEXT:       64KB, 인덱스 일부만 가능 (게시글 본문)
  LONGTEXT:   4GB, 대용량 텍스트

날짜/시간:
  DATE:      날짜만 (생년월일)
  DATETIME:  날짜+시간, 타임존 미적용 (로컬 시간 저장)
  TIMESTAMP: 날짜+시간, UTC 저장 + 타임존 자동 변환
             자동 업데이트 가능 (created_at, updated_at)
  → 글로벌 서비스: DATETIME + 애플리케이션에서 UTC 관리

ENUM:
  예: status ENUM('ACTIVE', 'INACTIVE', 'DELETED')
  장점: 가독성, 제약 자동 적용
  단점: 값 추가 시 ALTER TABLE 필요 (대용량에서 위험)
  → 대안: TINYINT + 애플리케이션에서 상수 관리

JSON:
  MySQL 5.7.8+, PostgreSQL 9.2+
  반정형 데이터, 옵션/메타데이터 저장
  → 인덱스 가능하지만 복잡. 자주 조회하는 값은 컬럼으로 분리 권장
```

---

## 7. 스키마 설계 체크리스트

```
PK 설계:
  [ ] 모든 테이블에 PK 존재
  [ ] AUTO_INCREMENT BIGINT 또는 UUID 선택
  [ ] 비즈니스 키(이메일 등)는 PK 사용 지양

컬럼 설계:
  [ ] NOT NULL 제약 적절히 사용
  [ ] 금액은 DECIMAL 사용 (FLOAT/DOUBLE 금지)
  [ ] ENUM 대신 TINYINT + 코드 상수 검토
  [ ] 날짜는 DATETIME(UTC) 또는 TIMESTAMP 선택
  [ ] VARCHAR 길이 적절히 (너무 크게 잡지 말 것)

관계 설계:
  [ ] M:N 관계는 중간 테이블로 분해
  [ ] FK 컬럼에 인덱스 추가
  [ ] 주문 상품 가격 등 스냅샷 필요한 값 별도 저장

공통 컬럼:
  [ ] created_at, updated_at 모든 테이블에 추가
  [ ] Soft Delete 필요 시 deleted_at 추가
  [ ] 비즈니스 키에 UNIQUE 제약 추가 (이메일, 주문번호 등)

성능 고려:
  [ ] 자주 조회하는 컬럼에 인덱스 추가
  [ ] FK 컬럼 인덱스 추가 확인
  [ ] 필요 이상의 정규화 방지 (역정규화 검토)
  [ ] 파티셔닝 필요 여부 검토 (대용량 이력 테이블)
```

---

## 8. JPA Entity 설계 vs DB 설계

```java
// DB 설계를 JPA로 표현할 때 주의사항

@Entity
@Table(name = "orders",
    indexes = {
        @Index(name = "idx_member_id", columnList = "member_id"),
        @Index(name = "idx_status_ordered_at", columnList = "status, ordered_at")
    }
)
public class Order extends BaseEntity {  // BaseEntity: createdAt, updatedAt

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // FK 컬럼 직접 저장 (API 응답에 memberId만 필요한 경우)
    @Column(nullable = false)
    private Long memberId;

    // 또는 연관 관계로 (JOIN이 필요한 경우)
    @ManyToOne(fetch = FetchType.LAZY)  // LAZY 필수 (N+1 방지)
    @JoinColumn(name = "member_id", insertable = false, updatable = false)
    private Member member;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> orderItems = new ArrayList<>();  // 빈 컬렉션으로 초기화

    @Enumerated(EnumType.STRING)  // ORDINAL 사용 금지 (순서 변경 위험)
    @Column(nullable = false, length = 20)
    private OrderStatus status;

    @Column(nullable = false)
    private BigDecimal totalAmount;  // DECIMAL → BigDecimal

    @Column(nullable = false)
    private LocalDateTime orderedAt;

    // 비즈니스 메서드
    public void cancel() {
        if (this.status != OrderStatus.PENDING) {
            throw new IllegalStateException("취소 불가능한 주문 상태입니다.");
        }
        this.status = OrderStatus.CANCELLED;
    }
}
```

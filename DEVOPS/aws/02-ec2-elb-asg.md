# EC2 / ELB / Auto Scaling

---

## 1. EC2 (Elastic Compute Cloud)

AWS의 가상 서버(VM). 용량을 필요에 따라 늘리거나 줄일 수 있다.

### 인스턴스 타입

```
이름 구조: [패밀리][세대][크기]
  예) t3.medium, m5.xlarge, c6g.large

패밀리:
  t  → 버스트 가능 범용 (개발/테스트, 낮은 비용)
  m  → 범용 (균형잡힌 CPU/메모리)
  c  → 컴퓨팅 최적화 (CPU 집약적 작업)
  r  → 메모리 최적화 (인메모리 DB, 캐시)
  i  → 스토리지 최적화 (NoSQL DB, 데이터 웨어하우스)
  g  → GPU (머신러닝, 그래픽)
  a  → ARM 기반 (Graviton, 가성비 높음)

크기: nano < micro < small < medium < large < xlarge < 2xlarge...

백엔드 서버 일반적 선택:
  개발 환경: t3.small / t3.medium
  운영 환경: m5.large / m5.xlarge (트래픽에 따라)
```

### 구매 옵션

```
On-Demand:
  - 사용한 시간만큼 청구 (초 단위)
  - 예측 불가한 워크로드, 단기 사용
  - 가장 비쌈

Reserved Instances (예약):
  - 1년 또는 3년 약정
  - On-Demand 대비 최대 72% 할인
  - 장기 운영 서버에 적합

Spot Instances:
  - AWS 여유 컴퓨팅 사용, 최대 90% 할인
  - 언제든 AWS가 회수 가능 (2분 전 경고)
  - 배치 처리, 데이터 분석 등 중단 허용되는 작업

Savings Plans:
  - 시간당 특정 사용량 약정 (유연한 Reserved)
  - EC2, Lambda, Fargate에 적용
```

### 주요 구성 요소

```
AMI (Amazon Machine Image):
  - EC2 인스턴스의 템플릿 (OS + 소프트웨어 설정)
  - Amazon Linux 2, Ubuntu, Windows 등
  - 커스텀 AMI 생성 가능 (배포 자동화에 활용)

Security Group:
  - 인스턴스 레벨 방화벽 (Stateful)
  - 인바운드/아웃바운드 규칙 (허용 규칙만 설정)
  - 기본: 모든 인바운드 차단, 모든 아웃바운드 허용

Key Pair:
  - EC2 SSH 접속용 키
  - Private Key는 절대 분실하면 안 됨 (복구 불가)
  - AWS Systems Manager Session Manager 사용 시 불필요

EBS (Elastic Block Store):
  - EC2에 붙는 영구 블록 스토리지 (하드디스크)
  - EC2 종료해도 데이터 유지
  - gp3 (SSD, 범용), io2 (고성능 SSD), st1 (HDD, 대용량)
```

### User Data (부팅 시 자동 스크립트)

```bash
#!/bin/bash
# EC2 최초 시작 시 한 번 실행
yum update -y
yum install -y java-21-amazon-corretto

# 앱 배포
aws s3 cp s3://my-deploy-bucket/app.jar /opt/app/app.jar
systemctl start myapp
```

---

## 2. ELB (Elastic Load Balancer)

들어오는 트래픽을 여러 EC2 인스턴스에 **자동으로 분산**하는 서비스.

### ELB 종류

```
ALB (Application Load Balancer):
  - L7 (HTTP/HTTPS) 레벨 로드밸런서
  - URL 경로, 헤더, 쿼리 파라미터 기반 라우팅
  - 가장 많이 사용 (REST API, 웹 애플리케이션)

NLB (Network Load Balancer):
  - L4 (TCP/UDP) 레벨
  - 초고성능, 초저지연 (수백만 req/sec)
  - 고정 IP 제공
  - 게임 서버, 금융 거래

CLB (Classic Load Balancer):
  - 구형, 사용 지양
```

### ALB 라우팅 규칙

```
경로 기반 라우팅 (Path-based):
  /api/*    → Target Group A (API 서버)
  /admin/*  → Target Group B (어드민 서버)
  /*        → Target Group C (프론트엔드)

호스트 기반 라우팅 (Host-based):
  api.example.com   → Target Group A
  admin.example.com → Target Group B

헤더 기반:
  X-Version: v2 → Target Group (v2 서버)
```

### Health Check

```
ALB가 주기적으로 각 인스턴스에 헬스체크 요청을 보내
비정상 인스턴스로는 트래픽을 보내지 않음

설정:
  프로토콜: HTTP
  경로: /actuator/health  (Spring Boot)
  정상 임계값: 2 (연속 2번 성공 시 healthy)
  비정상 임계값: 3 (연속 3번 실패 시 unhealthy)
  간격: 30초
```

```java
// Spring Boot Actuator 헬스체크 엔드포인트
// build.gradle: implementation 'org.springframework.boot:spring-boot-starter-actuator'

// application.yml
management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      show-details: when-authorized
```

### Sticky Session (고정 세션)

```
기본 동작: 매 요청이 다른 인스턴스로 분산
문제: 세션 정보가 특정 인스턴스 메모리에 있으면 다른 인스턴스에서 로그인 풀림

해결책 1: Sticky Session (쿠키 기반)
  → 같은 사용자는 항상 같은 인스턴스로 라우팅
  → 인스턴스 장애 시 세션 유실 가능

해결책 2: Redis 세션 스토어 (권장)
  → 세션을 Redis에 중앙 저장
  → 어느 인스턴스로 가도 같은 세션 공유
  → @EnableRedisHttpSession
```

---

## 3. Auto Scaling Group (ASG)

트래픽에 따라 EC2 인스턴스 수를 **자동으로 증감**시키는 서비스.

### 스케일링 정책

```
동적 스케일링:
  Target Tracking (가장 간단, 권장):
    "CPU 평균 50% 유지" → ASG가 자동으로 인스턴스 수 조절
    "ALB RequestCountPerTarget 1000" → 인스턴스당 1000 req/s 유지

  Step Scaling:
    CPU 60~70% → +1대 추가
    CPU 70~80% → +2대 추가
    CPU 80%+   → +3대 추가

  Simple Scaling:
    CPU 80% 초과 → +1대 (쿨다운 후 다시 평가)

예약 스케일링:
  "매일 오전 9시 최소 10대, 오후 6시 최소 2대"

예측 스케일링:
  ML로 미래 트래픽 예측해서 선제적 스케일아웃
```

### ASG 설정 값

```
Min Size: 최소 인스턴스 수 (0도 가능, 보통 2 이상)
Max Size: 최대 인스턴스 수 (비용 상한선)
Desired Capacity: 현재 목표 인스턴스 수

예시:
  Min: 2  ← 장애 시에도 최소 2대 보장 (Multi-AZ에 1대씩)
  Desired: 4
  Max: 10 ← 비용 폭탄 방지
```

### Scale-In 시 인스턴스 보호

```
Lifecycle Hook:
  Scale-In(종료) 직전에 훅 실행
  → 처리 중인 요청 완료 대기 (Connection Draining)
  → 로그 S3 업로드, 캐시 플러시 등

Termination Policy:
  - OldestInstance: 가장 오래된 인스턴스 먼저 종료
  - NewestInstance: 가장 새 인스턴스 먼저 종료
  - Default: AZ 균형 → 가장 오래된 AMI 인스턴스 순
```

### 전형적인 백엔드 아키텍처

```
인터넷
  ↓
Route 53 (DNS)
  ↓
ALB (Multi-AZ)
  ↓ (라운드로빈)
┌────────────────────────────────┐
│  Auto Scaling Group            │
│  ┌─────────┐  ┌─────────┐     │
│  │ EC2 (2a)│  │ EC2 (2c)│ ... │  ← 트래픽에 따라 자동 증감
│  └─────────┘  └─────────┘     │
└────────────────────────────────┘
  ↓
RDS Multi-AZ (Primary / Standby)
Redis ElastiCache
```

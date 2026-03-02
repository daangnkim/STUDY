# VPC / 보안 그룹 / NAT

---

## 1. VPC (Virtual Private Cloud)

AWS 내에 격리된 **가상 사설 네트워크**. 모든 AWS 리소스는 VPC 안에 배치된다.

### VPC 기본 구조

```
VPC (10.0.0.0/16)  ← 최대 65,536개 IP
├── Public Subnet (10.0.1.0/24)   ← 인터넷 직접 접근 가능
│     EC2 (웹서버, Bastion Host)
│     ALB
├── Private Subnet (10.0.2.0/24)  ← 인터넷 직접 접근 불가
│     EC2 (앱 서버)
│     RDS
│     ElastiCache
└── Isolated Subnet (10.0.3.0/24) ← 인터넷 및 NAT도 불가
      RDS (가장 민감한 DB)
```

### CIDR 블록

```
VPC CIDR: 10.0.0.0/16
  → 10.0.0.0 ~ 10.0.255.255 (65,536개 IP)

Subnet CIDR (VPC를 잘게 쪼갬):
  Public:  10.0.1.0/24  (256개 IP, 실제 사용 가능 251개 - AWS 5개 예약)
  Private: 10.0.2.0/24
  Private: 10.0.3.0/24

AWS 예약 IP (각 서브넷에서 5개):
  .0   → 네트워크 주소
  .1   → VPC 라우터
  .2   → DNS
  .3   → 미래 사용 예약
  .255 → 브로드캐스트
```

---

## 2. 인터넷 연결 컴포넌트

### Internet Gateway (IGW)

```
VPC ↔ 인터넷 연결 게이트웨이

Public Subnet에서 인터넷 접근하려면:
  1. IGW를 VPC에 연결
  2. Public Subnet의 라우팅 테이블에 0.0.0.0/0 → IGW 추가
  3. EC2에 퍼블릭 IP 또는 Elastic IP 할당

Route Table (Public Subnet):
  10.0.0.0/16  → local (VPC 내부 통신)
  0.0.0.0/0    → igw-xxxxxxxx (인터넷 트래픽)
```

### NAT Gateway

```
Private Subnet의 EC2가 인터넷 아웃바운드 접근이 필요할 때 사용
(예: 패키지 업데이트, 외부 API 호출)

구조:
  Private EC2 → NAT Gateway (Public Subnet에 배치) → IGW → 인터넷

NAT Gateway 특징:
  - AWS 관리형 (고가용성, 자동 스케일)
  - Public Subnet에 배치, Elastic IP 할당 필요
  - 인바운드 불가 (외부에서 Private EC2로 직접 접근 불가)
  - AZ당 하나씩 배치 권장 (AZ 장애 대비)

Route Table (Private Subnet):
  10.0.0.0/16  → local
  0.0.0.0/0    → nat-xxxxxxxx (NAT Gateway)
```

```
비용 고려:
  NAT Gateway: 시간당 + 데이터 처리량 과금 (꽤 비쌈)
  개발 환경: NAT Instance (EC2 기반, 저렴하지만 관리 필요)
  또는 VPC Endpoint 사용 (S3, DynamoDB 등 AWS 서비스)
```

---

## 3. 보안 그룹 (Security Group)

**인스턴스 레벨 방화벽** (Stateful - 응답 트래픽 자동 허용).

### 보안 그룹 특징

```
Stateful:
  인바운드 허용 → 응답 아웃바운드 자동 허용
  (NACL과 달리 응답 규칙 별도 설정 불필요)

허용 규칙만 존재:
  화이트리스트 방식 (명시적 허용만, 나머지는 자동 차단)
  Deny 규칙 없음

여러 인스턴스에 적용 가능:
  하나의 보안 그룹을 여러 EC2, RDS에 연결 가능
```

### 보안 그룹 실무 설계

```
ALB 보안 그룹 (sg-alb):
  인바운드:
    HTTPS 443  0.0.0.0/0 (전체 인터넷)
    HTTP  80   0.0.0.0/0 (HTTPS로 리다이렉트 목적)
  아웃바운드:
    8080  sg-app (EC2 앱 서버 보안 그룹에만)

앱 서버 보안 그룹 (sg-app):
  인바운드:
    8080  sg-alb (ALB에서만 접근 허용 - 보안 그룹 참조)
    22    sg-bastion (배스천 호스트에서만 SSH)
  아웃바운드:
    3306  sg-db  (RDS 접근)
    6379  sg-cache (ElastiCache 접근)
    443   0.0.0.0/0 (외부 API 호출)

DB 보안 그룹 (sg-db):
  인바운드:
    3306  sg-app (앱 서버에서만)
  아웃바운드:
    없음 (모든 아웃바운드 차단)

→ 핵심: IP 주소 대신 보안 그룹 ID를 소스로 사용
  (IP가 바뀌어도 자동 반영, 더 안전하고 유연)
```

---

## 4. NACL (Network ACL)

**서브넷 레벨 방화벽** (Stateless - 응답 트래픽도 명시 필요).

```
보안 그룹 vs NACL 비교:
┌──────────────┬──────────────────┬──────────────────┐
│              │   보안 그룹       │    NACL          │
├──────────────┼──────────────────┼──────────────────┤
│ 적용 레벨    │ 인스턴스          │ 서브넷           │
│ 상태         │ Stateful         │ Stateless        │
│ 규칙 유형    │ 허용만           │ 허용 + 차단      │
│ 규칙 평가    │ 모든 규칙 평가   │ 번호순 (낮을수록 우선) │
│ 기본값       │ 모든 인바운드 차단 │ 모든 트래픽 허용 │
└──────────────┴──────────────────┴──────────────────┘

NACL 예시 (Inbound):
  Rule 100: Allow TCP 443 0.0.0.0/0
  Rule 200: Allow TCP 80  0.0.0.0/0
  Rule 300: Deny  TCP ALL 192.168.1.0/24  ← 특정 IP 차단
  Rule *  : Deny  ALL ALL 0.0.0.0/0       ← 나머지 차단

NACL은 보통 기본값(허용) 유지 + 특정 IP 차단 용도로 활용
세밀한 제어는 보안 그룹으로
```

---

## 5. VPC Endpoint

```
Private Subnet의 서비스 → S3, DynamoDB 접근 시
  방법 1: NAT Gateway 경유 (인터넷 우회 + 비용 발생)
  방법 2: VPC Endpoint (인터넷 거치지 않고 AWS 내부망으로)

Gateway Endpoint (S3, DynamoDB):
  - 라우팅 테이블에 경로 추가
  - 무료

Interface Endpoint (다른 AWS 서비스):
  - ENI(탄력적 네트워크 인터페이스) 생성
  - 시간당 비용 발생

장점:
  - 인터넷 미경유 → 보안 향상
  - NAT Gateway 트래픽 감소 → 비용 절감
  - EC2 → S3 대규모 전송 시 특히 유리
```

---

## 6. Bastion Host (Jump Server)

```
Private EC2에 SSH 접근하는 방법:

구조:
  개발자 PC → (SSH 22) → Bastion Host (Public Subnet) → (SSH 22) → Private EC2

Bastion Host 설정:
  - 작은 인스턴스 (t3.nano)
  - 보안 그룹: 특정 IP(사무실, 개발자 IP)에서만 22 허용
  - Private EC2의 보안 그룹: Bastion 보안 그룹에서만 22 허용

SSH 접속:
  # ~/.ssh/config
  Host bastion
    HostName 3.x.x.x  # Bastion 퍼블릭 IP
    User ec2-user
    IdentityFile ~/.ssh/my-key.pem

  Host private-ec2
    HostName 10.0.2.10  # Private IP
    User ec2-user
    IdentityFile ~/.ssh/my-key.pem
    ProxyJump bastion   # Bastion 경유

  # 접속
  ssh private-ec2

더 나은 방법 - AWS Systems Manager Session Manager:
  - SSH 키 없이 브라우저 또는 CLI로 Private EC2 접속
  - 포트 22 불필요 (보안 그룹에 SSH 규칙 제거 가능)
  - 모든 세션 로그 CloudWatch/S3에 기록
```

---

## 7. VPC Peering & Transit Gateway

```
VPC Peering:
  두 VPC를 1:1로 연결 (같은 계정 또는 다른 계정/리전 가능)
  단점: 전이적 라우팅 불가 (A-B, B-C 피어링 → A-C 통신 불가)

Transit Gateway:
  여러 VPC를 허브-앤-스포크 방식으로 연결
  온프레미스(VPN/Direct Connect)도 연결 가능
  전이적 라우팅 지원

  VPC A ─┐
  VPC B ─┤ Transit Gateway ─── 온프레미스
  VPC C ─┘
```

---

## 8. 실제 아키텍처 예시

```
                  인터넷
                    ↓
              Internet Gateway
                    ↓
        ┌──── Public Subnet ────┐
        │  ALB (Multi-AZ)       │
        │  NAT Gateway          │
        │  Bastion Host         │
        └───────────────────────┘
                    ↓
        ┌──── Private Subnet ───┐
        │  EC2 App (Multi-AZ)   │
        │  ECS Task             │
        └───────────────────────┘
                    ↓
        ┌──── Isolated Subnet ──┐
        │  RDS Primary          │
        │  RDS Standby          │
        │  ElastiCache          │
        └───────────────────────┘

보안 그룹 연결:
  ALB → sg-alb: 443 from internet
  EC2 → sg-app: 8080 from sg-alb only
  RDS → sg-db:  3306 from sg-app only
```

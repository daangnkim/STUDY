# AWS 글로벌 인프라 / IAM

---

## 1. AWS 글로벌 인프라

### Region (리전)

물리적으로 분리된 지리적 위치. 각 리전은 독립된 전력, 네트워크, 냉각 시스템을 가진다.

```
현재 주요 리전:
  ap-northeast-2   → 서울
  ap-northeast-1   → 도쿄
  us-east-1        → 버지니아 북부 (AWS 기본 리전, 가장 많은 서비스)
  us-west-2        → 오레곤
  eu-west-1        → 아일랜드

리전 선택 기준:
  1. 사용자와의 물리적 거리 (레이턴시)
  2. 법적 규제 (한국 금융/공공 데이터 → 국내 리전)
  3. 서비스 제공 여부 (신규 서비스는 us-east-1 먼저 출시)
  4. 비용 (리전마다 가격 다름, us-east-1이 보통 저렴)
```

### AZ (Availability Zone, 가용 영역)

리전 안에 있는 독립된 데이터 센터 클러스터. 서울 리전(ap-northeast-2)은 4개 AZ를 가진다.

```
ap-northeast-2a
ap-northeast-2b
ap-northeast-2c
ap-northeast-2d

AZ 간 특징:
  - 물리적으로 수십 km 이상 분리 (화재, 홍수 등 독립)
  - 전용 고속 광케이블로 연결 (저지연)
  - 하나의 AZ 장애 시 다른 AZ가 계속 서비스

고가용성(HA) 설계:
  → 최소 2개 이상 AZ에 리소스 배치 (Multi-AZ)
```

### Edge Location (엣지 로케이션)

CloudFront(CDN)의 캐시 서버. 리전보다 훨씬 많은 수백 개 위치.
사용자와 가장 가까운 엣지에서 콘텐츠 전달 → 레이턴시 최소화.

---

## 2. IAM (Identity and Access Management)

AWS 리소스에 대한 **인증(Authentication)과 인가(Authorization)**을 관리하는 서비스.
전 세계 단일 서비스 (리전 구분 없음, Global).

### 핵심 구성 요소

```
User (사용자):
  → 사람 또는 애플리케이션을 나타내는 개체
  → Access Key (프로그래밍 방식) 또는 Console 로그인

Group (그룹):
  → User들의 모음
  → 그룹에 정책 부여 → 멤버 User 모두 적용
  → User는 여러 그룹에 속할 수 있음

Role (역할):
  → AWS 서비스/리소스가 다른 AWS 서비스에 접근하기 위한 임시 자격증명
  → EC2가 S3에 접근, Lambda가 DynamoDB에 접근 등
  → 사람이 아닌 서비스에게 부여

Policy (정책):
  → JSON으로 정의된 권한 문서
  → User/Group/Role에 연결
```

### IAM Policy 구조

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadOnly",
      "Effect": "Allow",                    // Allow 또는 Deny
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {                        // 조건 (선택)
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"  // 특정 IP에서만
        }
      }
    }
  ]
}
```

### EC2에 IAM Role 부여 (Spring Boot 예시)

```java
// EC2에 IAM Role이 부여되어 있으면 별도 Key 없이 AWS SDK 사용 가능
// build.gradle
// implementation 'software.amazon.awssdk:s3'
// implementation 'software.amazon.awssdk:sts'

@Configuration
public class AwsConfig {

    @Bean
    public S3Client s3Client() {
        return S3Client.builder()
            .region(Region.AP_NORTHEAST_2)
            // EC2에 IAM Role 있으면 자동으로 인스턴스 메타데이터에서 credential 읽음
            // 로컬 개발 시에는 ~/.aws/credentials 파일 또는 환경 변수 사용
            .credentialsProvider(DefaultCredentialsProvider.create())
            .build();
    }
}
```

```yaml
# EC2가 아닌 로컬 개발 시 application.yml
cloud:
  aws:
    credentials:
      access-key: ${AWS_ACCESS_KEY_ID}        # 환경 변수로 주입
      secret-key: ${AWS_SECRET_ACCESS_KEY}
    region:
      static: ap-northeast-2
```

### IAM 보안 모범 사례

```
1. Root 계정 MFA 필수 활성화
   - Root 계정으로 일상 작업 금지
   - 관리자 IAM User 생성해서 사용

2. 최소 권한 원칙 (Least Privilege)
   - 필요한 권한만 부여
   - S3 읽기만 필요하면 s3:GetObject만 허용
   - AdministratorAccess는 꼭 필요한 경우만

3. Access Key 관리
   - Access Key는 절대 코드에 하드코딩하지 말 것
   - EC2에서는 IAM Role 사용 (Key 불필요)
   - 주기적으로 Key 교체 (Rotation)
   - 사용하지 않는 Key 삭제

4. IAM Access Analyzer
   - 외부에서 접근 가능한 리소스 자동 탐지
   - 불필요한 공개 권한 감지
```

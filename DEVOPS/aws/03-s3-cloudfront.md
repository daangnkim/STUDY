# S3 / CloudFront

---

## 1. S3 (Simple Storage Service)

**무제한 용량의 객체 스토리지** 서비스. 파일을 Key-Value 형태로 저장한다.
99.999999999% (11 nine) 내구성을 제공한다.

### 핵심 개념

```
Bucket (버킷):
  - 파일을 담는 컨테이너 (최상위 디렉토리 개념)
  - 버킷 이름은 전 세계 고유해야 함
  - 리전에 속함

Object (객체):
  - 실제 저장하는 파일 (Key + Value + Metadata)
  - Key: 파일 경로 (예: images/profile/user1.jpg)
  - Value: 파일 데이터
  - 최대 5TB

S3는 파일시스템이 아님:
  - 폴더처럼 보이지만 실제로는 key의 prefix일 뿐
  - "images/user1.jpg"는 슬래시 포함한 하나의 Key
```

### 스토리지 클래스

```
S3 Standard:
  - 자주 접근하는 데이터
  - 가용성 99.99%
  - 프로필 이미지, 정적 파일

S3 Standard-IA (Infrequent Access):
  - 자주 접근하지 않지만 빠른 접근 필요
  - Standard보다 저렴, 조회 시 추가 비용
  - 백업, 재해 복구 파일

S3 Glacier / Glacier Deep Archive:
  - 아카이빙용 (수 분 ~ 수 시간 후 복원)
  - 매우 저렴
  - 규정 준수 데이터 장기 보관

S3 Intelligent-Tiering:
  - 접근 패턴 모니터링 후 자동으로 티어 이동
  - 예측 불가한 접근 패턴에 적합
```

### Spring Boot에서 S3 사용

```java
// build.gradle
// implementation 'software.amazon.awssdk:s3:2.x.x'
// 또는 spring-cloud-starter-aws-messaging

@Service
@RequiredArgsConstructor
public class S3Service {

    private final S3Client s3Client;

    @Value("${aws.s3.bucket}")
    private String bucketName;

    // 파일 업로드
    public String upload(MultipartFile file, String key) throws IOException {
        PutObjectRequest request = PutObjectRequest.builder()
            .bucket(bucketName)
            .key(key)
            .contentType(file.getContentType())
            .contentLength(file.getSize())
            .build();

        s3Client.putObject(request,
            RequestBody.fromInputStream(file.getInputStream(), file.getSize()));

        return "https://%s.s3.ap-northeast-2.amazonaws.com/%s".formatted(bucketName, key);
    }

    // 파일 다운로드
    public byte[] download(String key) {
        GetObjectRequest request = GetObjectRequest.builder()
            .bucket(bucketName)
            .key(key)
            .build();

        return s3Client.getObjectAsBytes(request).asByteArray();
    }

    // 파일 삭제
    public void delete(String key) {
        s3Client.deleteObject(b -> b.bucket(bucketName).key(key));
    }

    // Presigned URL 생성 (임시 접근 URL, 프론트에서 직접 업로드)
    public String generatePresignedUrl(String key, Duration expiration) {
        try (S3Presigner presigner = S3Presigner.builder()
                .region(Region.AP_NORTHEAST_2)
                .build()) {

            PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                .signatureDuration(expiration)
                .putObjectRequest(b -> b.bucket(bucketName).key(key))
                .build();

            return presigner.presignPutObject(presignRequest).url().toString();
        }
    }
}
```

### Presigned URL 활용 패턴

```
일반 업로드 (서버 경유):
  사용자 → [파일] → Backend → [파일] → S3
  단점: 서버가 대용량 파일 중계 → 서버 부하, 느림

Presigned URL 직접 업로드:
  사용자 → "업로드 URL 주세요" → Backend
  Backend → Presigned URL 생성 → 사용자
  사용자 → [파일] → S3 (직접 업로드)
  장점: 서버 부하 없음, 빠름

Presigned Download URL:
  비공개 S3 파일을 임시로 공개하는 URL
  만료 시간 설정 가능 (1시간 등)
  → 로그인한 사용자에게만 다운로드 링크 제공
```

### 버킷 정책

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForWebsite",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-website-bucket/*"
    }
  ]
}
```

### S3 정적 웹사이트 호스팅

```
React/Vue 빌드 결과물을 S3에 배포:
  1. 버킷 생성 (퍼블릭 액세스 차단 해제)
  2. 정적 웹사이트 호스팅 활성화
  3. index.html, error.html 설정
  4. 버킷 정책으로 GetObject 허용
  5. (권장) CloudFront 앞에 붙여서 HTTPS + CDN 적용
```

---

## 2. CloudFront

AWS의 **CDN(Content Delivery Network)** 서비스.
전 세계 Edge Location에서 콘텐츠를 캐싱해 사용자에게 빠르게 전달한다.

### 동작 원리

```
1. 사용자가 img.example.com/logo.png 요청
2. 가장 가까운 Edge Location으로 라우팅
3. Edge에 캐시 있으면 → 즉시 반환 (Cache HIT)
4. 캐시 없으면 → Origin(S3, ALB)에서 가져와 캐시 후 반환 (Cache MISS)

Origin 종류:
  - S3 버킷 (정적 파일)
  - ALB (동적 콘텐츠)
  - API Gateway
  - 커스텀 HTTP 서버
```

### 주요 기능

```
HTTPS 강제:
  HTTP → HTTPS 자동 리다이렉트
  ACM(AWS Certificate Manager)으로 무료 SSL 인증서 발급

지리적 제한 (Geo Restriction):
  특정 국가에서만 접근 허용 or 차단

캐시 정책:
  - TTL 설정 (기본 24시간)
  - Cache-Control 헤더 준수
  - 쿼리 스트링/헤더/쿠키 기반 캐시 키 설정

Lambda@Edge / CloudFront Functions:
  Edge에서 요청/응답 처리 (A/B 테스트, 인증, URL 재작성)

WAF 통합:
  CloudFront에 WAF 연결 → DDoS, SQL Injection 방어
```

### S3 + CloudFront 구성

```
                  CloudFront Distribution
                  ┌─────────────────────────────────────┐
사용자 요청        │  /images/*    → Origin A: S3 버킷    │
─────────────→  │  /api/*       → Origin B: ALB       │
(가까운 Edge)    │  /*           → Origin C: S3 (SPA)  │
                  └─────────────────────────────────────┘

장점:
  - HTTPS 자동 적용
  - 글로벌 캐싱으로 원본 서버 부하 감소
  - DDoS 기본 방어 (AWS Shield Standard)
  - 사용자 위치에 따른 최적 경로
```

### Cache Invalidation (캐시 무효화)

```bash
# 배포 후 CloudFront 캐시 삭제
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*"

# 특정 파일만 무효화
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/index.html" "/static/main.js"

# CI/CD 파이프라인에서 배포 후 자동 실행
```

```java
// Spring Boot에서 CloudFront 캐시 무효화
@Service
@RequiredArgsConstructor
public class CloudFrontService {

    private final CloudFrontClient cloudFrontClient;

    @Value("${aws.cloudfront.distribution-id}")
    private String distributionId;

    public void invalidateCache(List<String> paths) {
        InvalidationBatch batch = InvalidationBatch.builder()
            .paths(Paths.builder()
                .items(paths)
                .quantity(paths.size())
                .build())
            .callerReference(String.valueOf(System.currentTimeMillis()))
            .build();

        cloudFrontClient.createInvalidation(b -> b
            .distributionId(distributionId)
            .invalidationBatch(batch));
    }
}
```

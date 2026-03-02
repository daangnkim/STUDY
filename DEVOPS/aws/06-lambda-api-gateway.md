# Lambda / API Gateway

---

## 1. Lambda

서버 없이 코드를 실행하는 **FaaS(Function as a Service)** 서비스.
인프라 관리 없이 코드만 업로드하면 AWS가 실행 환경을 자동으로 관리한다.

### Lambda 동작 방식

```
트리거 → Lambda 함수 실행 → 결과 반환

트리거 종류:
  - API Gateway (HTTP 요청)
  - S3 이벤트 (파일 업로드/삭제)
  - SQS/SNS (메시지 수신)
  - DynamoDB Streams
  - EventBridge (스케줄, 이벤트)
  - CloudFront (Lambda@Edge)
```

### Lambda 핵심 개념

```
Cold Start vs Warm Start:
  Cold Start: 새 컨테이너 초기화 → 함수 실행 (처음 호출, 일정 시간 미호출 후)
    - 지연 시간: Java 1~5초, Node.js 200ms, Python 200ms
  Warm Start: 기존 컨테이너 재사용 → 즉시 실행
    - Provisioned Concurrency로 Cold Start 제거 가능 (비용 발생)

실행 환경:
  - 메모리: 128MB ~ 10,240MB (설정 값이 CPU도 비례해서 결정)
  - 타임아웃: 최대 15분
  - 임시 스토리지: /tmp 최대 10GB
  - 동시 실행: 기본 1,000개/리전 (상향 요청 가능)

과금:
  - 요청 수 × 실행 시간(ms) × 메모리(GB)
  - 월 100만 요청 무료, 400,000 GB-초 무료
```

### Lambda Handler (Java / Spring)

```java
// build.gradle
// implementation 'com.amazonaws:aws-lambda-java-core:1.2.3'
// implementation 'com.amazonaws:aws-lambda-java-events:3.11.0'

// 1. 단순 함수형 핸들러
public class HelloHandler implements RequestHandler<Map<String, String>, String> {

    @Override
    public String handleRequest(Map<String, String> event, Context context) {
        LambdaLogger logger = context.getLogger();
        logger.log("Event: " + event.toString());

        String name = event.getOrDefault("name", "World");
        return "Hello, " + name + "!";
    }
}
```

```java
// 2. API Gateway 이벤트 처리
public class ApiHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public APIGatewayProxyResponseEvent handleRequest(
            APIGatewayProxyRequestEvent event, Context context) {

        try {
            String body = event.getBody();
            Map<String, String> request = objectMapper.readValue(body, Map.class);

            // 비즈니스 로직
            String result = processRequest(request);

            return APIGatewayProxyResponseEvent.builder()
                .withStatusCode(200)
                .withHeader("Content-Type", "application/json")
                .withBody(objectMapper.writeValueAsString(Map.of("result", result)))
                .build();

        } catch (Exception e) {
            return APIGatewayProxyResponseEvent.builder()
                .withStatusCode(500)
                .withBody("{\"error\": \"Internal Server Error\"}")
                .build();
        }
    }
}
```

### Lambda + Spring Boot (AWS Lambda Web Adapter)

```java
// Spring Boot 앱을 Lambda에서 실행하는 방법
// aws-lambda-web-adapter 사용 (컨테이너 이미지 기반)

// 일반 Spring Boot 컨트롤러 그대로 사용
@RestController
@RequestMapping("/api")
public class ProductController {

    @GetMapping("/products/{id}")
    public ResponseEntity<ProductDto> getProduct(@PathVariable Long id) {
        // 기존 코드 그대로
        return ResponseEntity.ok(productService.findById(id));
    }
}

// Dockerfile (Lambda 컨테이너 이미지)
// FROM public.ecr.aws/lambda/java:21
// COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.8.1 /lambda-adapter /opt/extensions/lambda-adapter
// COPY target/app.jar ${LAMBDA_TASK_ROOT}/app.jar
// CMD ["app.jar"]
```

### Lambda Cold Start 최적화 (Java)

```java
// 1. GraalVM Native Image - 네이티브 컴파일로 Cold Start 제거에 가까운 성능
//    Spring Boot 3.x: spring-native 지원
//    빌드 시간 오래 걸리지만 실행은 즉시

// 2. 핸들러 외부에서 초기화 (Warm Start 활용)
public class OptimizedHandler implements RequestHandler<SQSEvent, Void> {

    // Lambda 컨테이너 초기화 시 한 번만 실행 (Cold Start 시)
    // 이후 호출에서는 재사용됨
    private static final DynamoDbClient dynamoDbClient = DynamoDbClient.builder()
        .region(Region.AP_NORTHEAST_2)
        .build();

    private static final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public Void handleRequest(SQSEvent event, Context context) {
        // 이미 초기화된 클라이언트 재사용
        event.getRecords().forEach(record -> {
            processMessage(record.getBody());
        });
        return null;
    }
}
```

### Lambda Layers

```
공통 라이브러리를 Layer로 분리해서 여러 함수에서 공유

Layer 구조:
  /opt/java/lib/ → Java 라이브러리 (.jar)
  /opt/python/   → Python 라이브러리

활용:
  - 공통 유틸리티 코드
  - AWS SDK (함수 코드 크기 감소)
  - 데이터 처리 라이브러리
  - 환경 설정 파일

Lambda 크기 제한:
  배포 패키지: 50MB (압축), 250MB (압축 해제)
  Layer 포함 총 크기: 250MB
  컨테이너 이미지: 10GB (제한 없음에 가까움 → 대규모 앱에 적합)
```

---

## 2. API Gateway

HTTP/WebSocket API 엔드포인트를 생성하고 Lambda, ALB, HTTP 백엔드에 연결하는 서비스.

### API Gateway 종류

```
REST API (v1):
  - 가장 많은 기능 (캐싱, WAF, 사용 플랜, API 키 등)
  - 복잡한 설정

HTTP API (v2):
  - REST API보다 70% 저렴
  - 더 빠름 (낮은 지연)
  - 기능은 적지만 대부분의 케이스에 충분
  - Lambda, HTTP 백엔드 연결

WebSocket API:
  - 실시간 양방향 통신 (채팅, 알림)
  - 연결 유지 관리
```

### API Gateway + Lambda 아키텍처

```
클라이언트
  ↓ HTTPS 요청
API Gateway
  ↓ 이벤트 전달
Lambda 함수
  ↓ 쿼리/쓰기
DynamoDB / RDS

특징:
  - 완전 서버리스 (서버 관리 불필요)
  - 요청 수에 따라 자동 확장
  - 초당 수만 요청 처리 가능
  - 비용: 호출 수 기반 (서버 항상 실행 X)
```

### API Gateway 주요 기능

```
인증/인가:
  - Lambda Authorizer: 커스텀 인증 로직 (JWT 검증 등)
  - Cognito User Pool: AWS Cognito와 통합
  - IAM 인증: AWS Signature V4

요청/응답 변환:
  - Mapping Template (VTL): 요청/응답 형식 변환
  - HTTP API: 변환 기능 제한적

캐싱 (REST API만):
  - TTL 설정으로 Lambda 호출 감소
  - 캐시 키: 경로, 쿼리 파라미터, 헤더

스로틀링:
  - 계정 레벨: 10,000 req/s, 5,000 버스트
  - API/스테이지 레벨 개별 설정 가능

CORS:
  API Gateway에서 CORS 헤더 자동 추가 설정 가능
```

### Lambda Authorizer 구현

```java
// JWT 검증 Lambda Authorizer
public class JwtAuthorizer implements RequestHandler<APIGatewayCustomAuthorizerEvent, APIGatewayCustomAuthorizerResponse> {

    private static final String SECRET = System.getenv("JWT_SECRET");

    @Override
    public APIGatewayCustomAuthorizerResponse handleRequest(
            APIGatewayCustomAuthorizerEvent event, Context context) {

        String token = event.getAuthorizationToken();

        try {
            // Bearer 토큰 추출
            String jwt = token.replace("Bearer ", "");

            // JWT 검증
            Claims claims = Jwts.parserBuilder()
                .setSigningKey(Keys.hmacShaKeyFor(SECRET.getBytes()))
                .build()
                .parseClaimsJws(jwt)
                .getBody();

            String userId = claims.getSubject();
            String methodArn = event.getMethodArn();

            // 허용 정책 반환
            return buildPolicy(userId, "Allow", methodArn);

        } catch (JwtException e) {
            // 인증 실패 → 401
            throw new RuntimeException("Unauthorized");
        }
    }

    private APIGatewayCustomAuthorizerResponse buildPolicy(
            String principalId, String effect, String resource) {

        Statement statement = Statement.builder()
            .withEffect(effect)
            .withAction("execute-api:Invoke")
            .withResource(resource)
            .build();

        PolicyDocument policyDocument = PolicyDocument.builder()
            .withVersion("2012-10-17")
            .withStatement(Collections.singletonList(statement))
            .build();

        return APIGatewayCustomAuthorizerResponse.builder()
            .withPrincipalId(principalId)
            .withPolicyDocument(policyDocument)
            .build();
    }
}
```

---

## 3. 서버리스 패턴

### 이벤트 드리븐 이미지 처리

```
S3 버킷에 이미지 업로드
  ↓ S3 이벤트
Lambda (이미지 리사이징)
  ↓ 결과 저장
S3 버킷 (썸네일)

코드:
public class ImageResizeHandler implements RequestHandler<S3Event, Void> {

    @Override
    public Void handleRequest(S3Event event, Context context) {
        event.getRecords().forEach(record -> {
            String bucket = record.getS3().getBucket().getName();
            String key = record.getS3().getObject().getKey();

            // S3에서 원본 이미지 다운로드
            byte[] imageData = downloadFromS3(bucket, key);

            // 리사이징 (Thumbnailator 등 라이브러리 사용)
            byte[] thumbnail = resize(imageData, 150, 150);

            // 썸네일 저장
            uploadToS3(bucket, "thumbnails/" + key, thumbnail);
        });
        return null;
    }
}
```

### 서버리스 vs 서버 기반 선택 기준

```
Lambda 적합:
  - 간헐적 워크로드 (이벤트 기반, 배치)
  - 파일 처리 (이미지 변환, 문서 처리)
  - 짧은 실행 시간 (15분 이하)
  - 트래픽 예측 불가한 서비스
  - 마이크로서비스 단순 API

EC2/ECS 적합:
  - 항상 실행 중인 서비스 (상시 트래픽)
  - 긴 실행 시간 필요
  - 상태 유지 (WebSocket, 게임 서버)
  - 콜드 스타트 허용 불가 (금융, 실시간)
  - 복잡한 Spring Boot 앱
```

---

## 4. SAM (Serverless Application Model)

AWS 서버리스 앱을 코드로 정의하는 IaC 프레임워크.

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: java21
    MemorySize: 512
    Timeout: 30
    Environment:
      Variables:
        DB_URL: !Ref DatabaseUrl

Resources:
  # API Lambda 함수
  ProductFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: com.example.ProductHandler::handleRequest
      CodeUri: target/app.jar
      Events:
        GetProduct:
          Type: HttpApi
          Properties:
            Path: /products/{id}
            Method: get
        CreateProduct:
          Type: HttpApi
          Properties:
            Path: /products
            Method: post

  # 비동기 처리 Lambda
  OrderProcessor:
    Type: AWS::Serverless::Function
    Properties:
      Handler: com.example.OrderHandler::handleRequest
      CodeUri: target/app.jar
      Events:
        SQSEvent:
          Type: SQS
          Properties:
            Queue: !GetAtt OrderQueue.Arn
            BatchSize: 10

  # SQS 큐
  OrderQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 60
```

```bash
# 로컬 테스트
sam local invoke ProductFunction --event event.json
sam local start-api  # 로컬 API Gateway 에뮬레이션

# 배포
sam build
sam deploy --guided
```

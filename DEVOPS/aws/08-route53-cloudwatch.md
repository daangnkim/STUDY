# Route 53 / CloudWatch

---

## 1. Route 53

AWS의 **DNS(Domain Name System)** 서비스. 도메인 등록, DNS 라우팅, 헬스체크를 제공한다.

### DNS 레코드 종류

```
A 레코드:     도메인 → IPv4 주소
              example.com → 1.2.3.4

AAAA 레코드:  도메인 → IPv6 주소

CNAME:        도메인 → 다른 도메인 (루트 도메인에는 불가)
              www.example.com → example.com

Alias:        AWS 리소스를 직접 가리킴 (CNAME의 AWS 버전, 루트 도메인 가능)
              example.com → ALB 엔드포인트
              example.com → CloudFront 배포

NS 레코드:    도메인의 네임서버 (Route 53 호스팅 존 생성 시 자동 생성)
SOA 레코드:   도메인의 권한 정보

TTL (Time To Live):
  DNS 응답을 캐시하는 시간 (초)
  짧으면: 변경 빠르게 반영, DNS 조회 증가
  길면:   DNS 조회 감소, 변경 반영 느림
  배포 전: TTL 낮게 → 배포 후: TTL 원복
```

### Route 53 라우팅 정책

```
Simple:
  단순히 IP 반환. 여러 IP 설정 시 랜덤 반환

Weighted (가중치 기반):
  A 레코드: 70% → 서버 1
  A 레코드: 30% → 서버 2
  → A/B 테스트, 점진적 트래픽 이동에 활용

Latency-based:
  사용자 위치에서 가장 레이턴시 낮은 리전으로 라우팅
  → 글로벌 서비스에서 사용자 경험 향상

Failover:
  Primary: ap-northeast-2 ALB (메인 리전)
  Secondary: us-east-1 ALB (재해 복구 리전)
  Health Check 실패 시 자동으로 Secondary로 전환

Geolocation:
  사용자 지리적 위치에 따라 다른 서버로
  한국 → 서울 리전
  미국 → 버지니아 리전

Geoproximity:
  지리적 위치 + 편향(bias) 값 조합
  Traffic Flow로 설정

Multi-value Answer:
  여러 A 레코드 반환 + 각 레코드에 Health Check
  비정상 엔드포인트는 응답에서 제외
```

### Route 53 Health Check

```
HTTP/HTTPS/TCP 엔드포인트 모니터링

설정:
  프로토콜: HTTPS
  경로: /actuator/health
  간격: 30초 (Standard), 10초 (Fast)
  실패 임계값: 3회 연속 실패 시 비정상

Health Check 연동:
  - Failover 라우팅: Primary 비정상 → Secondary로 자동 전환
  - CloudWatch Alarm과 연결 → SNS 알림

Private Subnet Health Check:
  Route 53는 인터넷에서 체크하므로 Private EC2 직접 체크 불가
  → CloudWatch Metric → Health Check 연동으로 대안
```

---

## 2. CloudWatch

AWS 리소스와 애플리케이션의 **모니터링** 서비스.

### CloudWatch 핵심 구성요소

```
Metrics (지표):
  - AWS 서비스에서 자동 수집 (EC2 CPU, RDS 연결 수 등)
  - 사용자 정의 지표 직접 발행 가능
  - 네임스페이스 / 지표명 / 디멘션(태그)으로 구분
  - 15개월 보존

Logs (로그):
  - Log Group → Log Stream 구조
  - EC2, Lambda, ECS 등에서 로그 수집
  - Metric Filter: 로그 내 패턴 → 지표로 변환

Alarms (알람):
  - 지표가 임계값 초과 시 액션 수행
  - 액션: SNS 알림, EC2 Auto Scaling, Systems Manager

Dashboards:
  - 지표 시각화 (그래프, 숫자 위젯)
  - 여러 리전/계정 통합 뷰

Insights:
  - Logs Insights: SQL 유사 언어로 로그 쿼리/분석
  - Container Insights: ECS/EKS 컨테이너 모니터링
  - Application Insights: 애플리케이션 레벨 모니터링
```

### EC2 모니터링

```
기본 모니터링 (무료):
  - 5분 간격
  - CPU, 네트워크 I/O, 디스크 I/O, 상태 체크

상세 모니터링 (유료):
  - 1분 간격
  - Auto Scaling 반응 속도 향상에 도움

주의: 메모리 사용률은 기본 포함 안 됨
  → CloudWatch Agent 설치 필요 (메모리, 디스크 사용률 등)
```

### Spring Boot → CloudWatch Logs

```yaml
# application.yml - CloudWatch Logs로 로그 전송
logging:
  level:
    root: INFO
    com.myapp: DEBUG

# CloudWatch Logs Appender (logback-spring.xml)
```

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="CLOUDWATCH" class="ca.pjer.logback.AwsLogsAppender">
        <layout>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </layout>
        <logGroupName>/my-app/production</logGroupName>
        <logStreamUuidPrefix>app-</logStreamUuidPrefix>
        <logRegion>ap-northeast-2</logRegion>
        <maxBatchLogEvents>50</maxBatchLogEvents>
        <maxFlushTimeMillis>3000</maxFlushTimeMillis>
        <maxBlockTimeMillis>0</maxBlockTimeMillis>
    </appender>

    <appender name="ASYNC_CLOUDWATCH" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="CLOUDWATCH"/>
    </appender>

    <root level="INFO">
        <appender-ref ref="ASYNC_CLOUDWATCH"/>
    </root>
</configuration>
```

### 커스텀 메트릭 발행

```java
@Service
@RequiredArgsConstructor
public class MetricsService {

    private final CloudWatchClient cloudWatchClient;

    // 주문 생성 수 지표 발행
    public void recordOrderCreated(String orderType) {
        PutMetricDataRequest request = PutMetricDataRequest.builder()
            .namespace("MyApp/Orders")
            .metricData(MetricDatum.builder()
                .metricName("OrderCreated")
                .value(1.0)
                .unit(StandardUnit.COUNT)
                .timestamp(Instant.now())
                .dimensions(
                    Dimension.builder()
                        .name("OrderType")
                        .value(orderType)
                        .build()
                )
                .build())
            .build();

        cloudWatchClient.putMetricData(request);
    }

    // 처리 시간 지표
    public void recordProcessingTime(String operation, long milliseconds) {
        cloudWatchClient.putMetricData(b -> b
            .namespace("MyApp/Performance")
            .metricData(MetricDatum.builder()
                .metricName("ProcessingTime")
                .value((double) milliseconds)
                .unit(StandardUnit.MILLISECONDS)
                .dimensions(Dimension.builder()
                    .name("Operation")
                    .value(operation)
                    .build())
                .build()));
    }
}
```

```java
// Spring Micrometer + CloudWatch 통합 (권장)
// build.gradle: implementation 'io.micrometer:micrometer-registry-cloudwatch2'

@Configuration
public class MetricsConfig {

    @Bean
    public CloudWatchMeterRegistry cloudWatchMeterRegistry(
            CloudWatchConfig config, Clock clock) {
        return new CloudWatchMeterRegistry(config, clock,
            CloudWatchAsyncClient.builder()
                .region(Region.AP_NORTHEAST_2)
                .build());
    }
}

// 서비스에서 사용
@Service
@RequiredArgsConstructor
public class OrderService {

    private final MeterRegistry meterRegistry;

    public Order createOrder(OrderRequest request) {
        // 카운터
        meterRegistry.counter("orders.created",
            "type", request.getType().name()).increment();

        // 타이머 (처리 시간 자동 측정)
        return meterRegistry.timer("orders.processing.time").record(() -> {
            return processOrder(request);
        });
    }
}
```

### CloudWatch Alarms

```
주요 알람 설정:

EC2:
  - CPUUtilization > 80% → SNS 알림 + Auto Scaling
  - StatusCheckFailed = 1 → 인스턴스 재시작

RDS:
  - DatabaseConnections > 80 → 연결 풀 부족 경고
  - FreeStorageSpace < 10GB → 용량 부족 경고
  - ReplicaLag > 30초 → 복제 지연 경고

ALB:
  - TargetResponseTime > 2초 → 응답 지연 경고
  - HTTPCode_Target_5XX_Count > 10/분 → 서버 오류 급증

Lambda:
  - Errors > 0 → 에러 발생
  - Throttles > 0 → 동시 실행 한계 초과
  - Duration > 타임아웃 80% → 타임아웃 위험

커스텀:
  - orders.created < 10/시간 → 주문 급감 (이상 감지)
  - error.rate > 1% → 에러율 임계값 초과
```

### CloudWatch Logs Insights 쿼리

```sql
-- 최근 1시간 ERROR 로그 분석
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100

-- API 응답 시간 분포
fields @timestamp, duration
| filter operation = "createOrder"
| stats avg(duration), max(duration), pct(duration, 95), pct(duration, 99)
  by bin(5m)

-- 특정 사용자 요청 추적
fields @timestamp, @message
| filter @message like /userId=12345/
| sort @timestamp asc

-- 에러율 계산
fields @timestamp
| stats
    count(*) as total,
    count_distinct(requestId) as errors by bin(1m)
| filter @message like /ERROR/
```

### X-Ray (분산 추적)

```
마이크로서비스에서 요청 흐름을 추적하는 서비스

Spring Boot + X-Ray:
  - AWS X-Ray SDK 추가
  - 서비스 간 요청 추적 (Trace ID 전파)
  - 각 세그먼트(DB 쿼리, 외부 API 호출) 시간 측정
  - 병목 지점 시각화

// build.gradle
// implementation 'com.amazonaws:aws-xray-recorder-sdk-spring:2.14.0'

@Configuration
public class XRayConfig {

    @Bean
    public AWSXRayServletFilter xrayFilter() {
        return new AWSXRayServletFilter("my-app");
    }
}
```

---

## 3. CloudWatch + 알람 실전 예시

```
알람 → SNS → Slack 연동:

1. SNS 토픽 생성 (alarm-notifications)
2. Lambda 함수 생성 (SNS → Slack Webhook 전달)
3. CloudWatch Alarm → SNS 토픽 연결
4. Lambda → SNS 구독

Lambda 코드 (Slack 알림):
public class SlackNotifier implements RequestHandler<SNSEvent, Void> {

    @Override
    public Void handleRequest(SNSEvent event, Context context) {
        event.getRecords().forEach(record -> {
            String message = record.getSNS().getMessage();
            sendToSlack(message);
        });
        return null;
    }

    private void sendToSlack(String message) {
        String webhookUrl = System.getenv("SLACK_WEBHOOK_URL");
        String payload = "{\"text\": \"🚨 AWS Alert: " + message + "\"}";

        // HTTP POST to Slack Webhook
        HttpClient.newHttpClient().sendAsync(
            HttpRequest.newBuilder()
                .uri(URI.create(webhookUrl))
                .POST(HttpRequest.BodyPublishers.ofString(payload))
                .header("Content-Type", "application/json")
                .build(),
            HttpResponse.BodyHandlers.ofString()
        );
    }
}
```

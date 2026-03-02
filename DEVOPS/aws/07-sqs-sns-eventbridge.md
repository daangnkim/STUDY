# SQS / SNS / EventBridge

---

## 1. SQS (Simple Queue Service)

**메시지 큐** 서비스. 컴포넌트 간 비동기 통신으로 느슨한 결합(Loose Coupling)을 구현한다.

### SQS 동작 원리

```
Producer(생산자) → [SQS Queue] → Consumer(소비자)

특징:
  - Pull 방식: Consumer가 주기적으로 큐에서 메시지 꺼냄 (Polling)
  - 메시지 보존: 최대 14일 (기본 4일)
  - 메시지 크기: 최대 256KB
  - 처리량: 표준 큐 무제한, FIFO 큐 초당 300~3,000 TPS
```

### SQS 큐 종류

```
Standard Queue (표준 큐):
  - 무제한 처리량
  - At-Least-Once Delivery: 메시지가 최소 1번 전달 (중복 가능)
  - Best-Effort Ordering: 순서 보장 안 됨
  - 사용: 이메일 전송, 이미지 처리 등 중복 허용되는 작업

FIFO Queue:
  - 정확히 한 번 처리 (Exactly-Once Processing)
  - 순서 보장 (First-In-First-Out)
  - 초당 300 메시지 (배치로 3,000까지)
  - 사용: 금융 거래, 재고 차감 등 순서/중복 민감한 작업
```

### Visibility Timeout

```
메시지를 꺼낸 Consumer가 처리 중일 때 다른 Consumer가 같은 메시지 처리 방지

동작:
  1. Consumer A가 메시지 수신
  2. 메시지가 Visibility Timeout 동안 비가시 상태 (기본 30초)
  3. A가 처리 완료 → DeleteMessage 호출 → 메시지 삭제
  4. A가 실패/타임아웃 → Visibility Timeout 만료 → 메시지 다시 가시

설정:
  처리 시간보다 충분히 크게 설정
  처리 시간: 5분 → Visibility Timeout: 10분

Dead Letter Queue (DLQ):
  일정 횟수 이상 처리 실패한 메시지를 별도 DLQ로 이동
  → 실패 메시지 원인 분석, 알람 설정 가능
  maxReceiveCount: 3 → 3번 실패 시 DLQ로
```

### Spring Boot + SQS

```java
// build.gradle
// implementation 'software.amazon.awssdk:sqs'
// 또는 spring-cloud-aws-messaging

@Service
@RequiredArgsConstructor
@Slf4j
public class SqsMessageProducer {

    private final SqsClient sqsClient;

    @Value("${aws.sqs.order-queue-url}")
    private String queueUrl;

    public void sendOrderMessage(OrderCreatedEvent event) {
        try {
            String messageBody = objectMapper.writeValueAsString(event);

            SendMessageRequest request = SendMessageRequest.builder()
                .queueUrl(queueUrl)
                .messageBody(messageBody)
                // FIFO 큐인 경우:
                // .messageGroupId("order-" + event.getUserId())
                // .messageDeduplicationId(event.getOrderId())
                .build();

            SendMessageResponse response = sqsClient.sendMessage(request);
            log.info("Message sent. MessageId: {}", response.messageId());

        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize message", e);
        }
    }
}
```

```java
// Spring Cloud AWS를 사용한 SQS 컨슈머
@Component
@Slf4j
public class OrderEventConsumer {

    @SqsListener("${aws.sqs.order-queue-url}")
    public void processOrder(@Payload String message,
                              @Headers Map<String, String> headers) {
        try {
            OrderCreatedEvent event = objectMapper.readValue(message, OrderCreatedEvent.class);
            log.info("Processing order: {}", event.getOrderId());

            // 비즈니스 로직
            orderService.processOrder(event);

            // 정상 처리 시 자동으로 DeleteMessage 호출

        } catch (Exception e) {
            log.error("Failed to process order", e);
            // 예외 throw 시 Visibility Timeout 후 재처리
            // maxReceiveCount 초과 시 DLQ로
            throw new RuntimeException(e);
        }
    }
}
```

```java
// Lambda에서 SQS 배치 처리
public class OrderLambdaHandler implements RequestHandler<SQSEvent, SQSBatchResponse> {

    @Override
    public SQSBatchResponse handleRequest(SQSEvent event, Context context) {
        List<SQSBatchResponse.BatchItemFailure> failures = new ArrayList<>();

        for (SQSEvent.SQSMessage message : event.getRecords()) {
            try {
                OrderCreatedEvent order = objectMapper.readValue(
                    message.getBody(), OrderCreatedEvent.class);
                processOrder(order);

            } catch (Exception e) {
                log.error("Failed to process: {}", message.getMessageId(), e);
                // 실패한 메시지만 다시 큐에 (성공한 것은 자동 삭제)
                failures.add(SQSBatchResponse.BatchItemFailure.builder()
                    .withItemIdentifier(message.getMessageId())
                    .build());
            }
        }

        return SQSBatchResponse.builder()
            .withBatchItemFailures(failures)
            .build();
    }
}
```

---

## 2. SNS (Simple Notification Service)

**Pub/Sub** 메시징 서비스. 하나의 메시지를 여러 구독자에게 동시에 전달한다.

### SNS 동작 원리

```
Publisher → [SNS Topic] → 여러 Subscriber에 Fan-Out

Subscriber 종류:
  - SQS 큐 (가장 일반적)
  - Lambda 함수
  - HTTP/HTTPS 엔드포인트
  - 이메일
  - SMS
  - Mobile Push (iOS, Android)

예시: 주문 완료 이벤트
  SNS Topic: order-completed
    ├── SQS: 재고 감소 큐
    ├── SQS: 배송 처리 큐
    ├── Lambda: 영수증 이메일 발송
    └── Lambda: 통계 데이터 업데이트
```

### SNS 메시지 필터링

```json
// 구독 시 필터 정책 설정
// 예: OrderType이 "EXPRESS"인 메시지만 처리
{
  "orderType": ["EXPRESS"],
  "amount": [{ "numeric": [">=", 100000] }]
}

// SNS 메시지 속성 포함해서 발행
{
  "Message": "{...}",
  "MessageAttributes": {
    "orderType": {
      "DataType": "String",
      "StringValue": "EXPRESS"
    },
    "amount": {
      "DataType": "Number",
      "StringValue": "150000"
    }
  }
}
```

### Spring Boot + SNS

```java
@Service
@RequiredArgsConstructor
public class EventPublisher {

    private final SnsClient snsClient;

    @Value("${aws.sns.order-topic-arn}")
    private String topicArn;

    public void publishOrderCompleted(Order order) {
        OrderCompletedEvent event = OrderCompletedEvent.builder()
            .orderId(order.getId())
            .userId(order.getUserId())
            .amount(order.getTotalAmount())
            .build();

        try {
            PublishRequest request = PublishRequest.builder()
                .topicArn(topicArn)
                .message(objectMapper.writeValueAsString(event))
                .subject("Order Completed")
                // 메시지 필터링용 속성
                .messageAttributes(Map.of(
                    "orderType", MessageAttributeValue.builder()
                        .dataType("String")
                        .stringValue(order.getType().name())
                        .build()
                ))
                .build();

            snsClient.publish(request);
            log.info("Published order completed event: {}", order.getId());

        } catch (JsonProcessingException e) {
            throw new RuntimeException("Serialization failed", e);
        }
    }
}
```

### SNS + SQS Fan-Out 패턴

```
주문 완료 이벤트 Fan-Out:

  OrderService
      ↓ publish
  SNS Topic (order-completed)
      ├── SQS A → Lambda/EC2: 재고 감소
      ├── SQS B → Lambda/EC2: 배송 요청
      └── SQS C → Lambda/EC2: 적립금 지급

장점:
  - OrderService는 SNS에만 발행 (구독자 모름)
  - 새로운 기능 추가 = SNS에 구독 추가만 (OrderService 수정 불필요)
  - 각 SQS가 독립적으로 재처리 (하나 실패해도 나머지 영향 없음)
  - SQS 버퍼링으로 처리 속도 차이 흡수
```

---

## 3. EventBridge

**이벤트 버스** 서비스. AWS 서비스 이벤트, 애플리케이션 이벤트, 외부 SaaS 이벤트를 라우팅한다.

### EventBridge vs SNS

```
SNS:
  - 단순 메시지 발행/구독
  - 필터링: 메시지 속성 기반

EventBridge:
  - 풍부한 이벤트 라우팅 (이벤트 내용 기반 필터링)
  - AWS 서비스 이벤트 자동 연동 (EC2 상태 변경, S3 이벤트 등)
  - SaaS 연동 (Zendesk, Datadog 등)
  - 스케줄 규칙 (cron, rate)
  - 이벤트 아카이브 및 재처리
  - Schema Registry (이벤트 스키마 관리)
```

### EventBridge 규칙

```json
// EC2 인스턴스 상태 변경 시 Lambda 호출
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["stopped", "terminated"]
  }
}

// 커스텀 앱 이벤트 라우팅
{
  "source": ["my-app.orders"],
  "detail-type": ["OrderCompleted"],
  "detail": {
    "amount": [{ "numeric": [">=", 1000000] }]  // 100만원 이상 주문만
  }
}
```

### Spring Boot + EventBridge

```java
@Service
@RequiredArgsConstructor
public class EventBridgePublisher {

    private final EventBridgeClient eventBridgeClient;

    public void publishOrderEvent(Order order, String eventType) {
        OrderEvent event = OrderEvent.builder()
            .orderId(order.getId())
            .userId(order.getUserId())
            .amount(order.getTotalAmount())
            .status(order.getStatus())
            .build();

        try {
            PutEventsRequestEntry entry = PutEventsRequestEntry.builder()
                .eventBusName("my-app-events")
                .source("my-app.orders")
                .detailType(eventType)  // "OrderCompleted", "OrderCancelled" 등
                .detail(objectMapper.writeValueAsString(event))
                .build();

            PutEventsRequest request = PutEventsRequest.builder()
                .entries(entry)
                .build();

            PutEventsResponse response = eventBridgeClient.putEvents(request);

            if (response.failedEntryCount() > 0) {
                log.error("Failed to publish events: {}", response.entries());
            }

        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }
}
```

### EventBridge 스케줄 (Cron)

```
정기 작업을 Lambda/Step Functions에 연결

예시:
  매일 오전 2시 데이터 집계: cron(0 17 * * ? *)  → UTC 기준
  5분마다 헬스체크:           rate(5 minutes)
  평일 오전 9시 보고서:       cron(0 0 ? * MON-FRI *)

// Terraform/CDK로 정의:
// resource "aws_cloudwatch_event_rule" "daily_batch" {
//   schedule_expression = "cron(0 17 * * ? *)"
// }
```

---

## 4. 메시지 패턴 비교

```
SQS 사용:
  - 단일 Consumer가 처리해야 하는 작업
  - 처리 속도 차이 완충 (버퍼링)
  - 재처리(DLQ)가 중요한 경우
  - 예: 주문 처리, 이메일 발송 큐

SNS 사용:
  - 하나의 이벤트를 여러 곳에 알려야 할 때 (Fan-Out)
  - 실시간 알림 (이메일, SMS, Push)
  - 예: 이벤트 브로드캐스트

EventBridge 사용:
  - AWS 서비스 이벤트 반응
  - 복잡한 이벤트 라우팅 (내용 기반 필터링)
  - 정기 스케줄 작업
  - 마이크로서비스 이벤트 버스
  - 예: 이벤트 드리븐 아키텍처 허브

조합 패턴 (SNS + SQS):
  SNS → 여러 SQS 큐 (Fan-Out + 큐잉)
  각 SQS가 Consumer를 가져 비동기 처리
  가장 많이 사용하는 패턴
```

---

## 5. 실무 패턴: 주문 처리 시스템

```
사용자 주문
  ↓
Order API (Spring Boot)
  ↓ SNS publish (order.completed)
  ├──→ SQS (inventory-queue)
  │      ↓ Consumer
  │    재고 서비스 (재고 차감)
  │
  ├──→ SQS (delivery-queue)
  │      ↓ Consumer (또는 Lambda)
  │    배송 서비스 (배송 요청)
  │
  └──→ SQS (notification-queue)
         ↓ Lambda
       알림 서비스 (이메일/SMS)

EventBridge 스케줄:
  매시간 → Lambda (미완료 주문 확인 및 알림)
  매일 오전 2시 → Lambda (전날 매출 집계)
```

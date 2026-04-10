# 02. 중앙화 로깅 — Trace ID로 분산 로그 조합

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 왜 MSA 환경에서 각 서비스의 로그를 개별적으로 보면 안 되는가?
- ELK Stack과 EFK Stack의 차이점은 무엇이며, 언제 어떤 것을 선택해야 하는가?
- Trace ID를 활용하여 여러 서비스의 로그를 하나의 요청으로 통합 조회하는 방법은?
- 구조화 로그(JSON)를 사용할 때의 장점과 구현 방법은?
- 로그 보존 기간과 저장소 계층(Hot/Warm/Cold)을 설계하는 기준은?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

모놀리식 애플리케이션에서는 모든 로그가 `/var/log/app.log` 한 파일에 모입니다. SSH로 서버에 접속하면 `tail -f` 명령어로 실시간 로그를 봅니다. 하지만 마이크로서비스 아키텍처에서는 같은 요청이 10개 서비스를 거치므로, 로그도 10개 서버에 흩어져 있습니다. 결제 오류를 조사하려고 OrderService 로그를 봤는데 에러가 없고, PaymentService 로그를 봤는데도 없고, 더 찾아보니 InventoryService에서 3단계 호출 체인 뒤에 timeout이 발생했습니다.

중앙화 로깅이 없다면 장애 분석에 1-2시간이 소요됩니다. 로그가 중앙 저장소(Elasticsearch)에 모이고, 각 로그 항목에 Trace ID가 포함되어 있으면, 사용자의 "결제 실패" 불평을 Trace ID로 검색하는 순간 10개 서비스의 모든 관련 로그가 시간순으로 나타납니다. 병목지점은 밀리초 단위로 명확해지고, 대응도 빨라집니다.

또한 구조화 로그(JSON 포맷)를 사용하면 필드 기반 검색이 가능합니다. 예를 들어 "고객ID가 customer_12345인 모든 에러"를 한 줄의 Kibana 쿼리로 찾을 수 있습니다. 텍스트 파일 로그라면 grep을 여러 번 실행해야 하고, 여전히 놓친 항목이 있을 수 있습니다.

---

## 😱 흔한 실수 (Before)

```bash
# 문제 1: 각 서비스 로그가 분산되어 있음
# OrderService 서버 (203.0.113.10)
user@order-server:~$ tail -f /var/log/order-service.log
2025-04-10T10:15:32.123Z [INFO] Order created for customer_123
2025-04-10T10:15:33.456Z [ERROR] Payment service unavailable

# PaymentService 서버 (203.0.113.20) - 다른 서버!
user@payment-server:~$ tail -f /var/log/payment-service.log
2025-04-10T10:15:33.789Z [INFO] Payment request received
2025-04-10T10:15:34.012Z [ERROR] DB connection timeout

# InventoryService 서버 (203.0.113.30) - 또 다른 서버!
user@inventory-server:~$ tail -f /var/log/inventory-service.log
2025-04-10T10:15:34.345Z [INFO] Stock check started
2025-04-10T10:15:35.678Z [ERROR] Stock insufficient

# 문제점:
# - 세 가지 로그를 동시에 모니터링하려면 3개 SSH 세션 필요
# - 어느 로그가 동일 요청인지 연관성 파악 불가능
# - 실제로 어디서 실패했는지 추적 불가능
```

```java
// 문제 2: 일반 텍스트 로그 (구조화되지 않음)
// application.log
[2025-04-10 10:15:32.123] [INFO] [OrderService] [http-nio-8080-exec-1] Order creation started
[2025-04-10 10:15:32.456] [INFO] [OrderService] [http-nio-8080-exec-1] Calling PaymentService
[2025-04-10 10:15:33.789] [ERROR] [OrderService] [http-nio-8080-exec-1] Exception: java.net.SocketTimeoutException: Connection timeout

// 문제점:
// 1. "Order creation started"를 grep으로 찾아도 어느 고객인지 모름
// 2. 응답시간을 분석하려면 타임스탬프를 수동 계산 (10:15:33.789 - 10:15:32.123 = 1.666초)
// 3. 필드 기반 검색 불가능 (예: 고객ID로 필터링)
// 4. 백만 줄 로그에서 특정 요청의 로그를 모으려면 수동으로 라인별 grep
```

```bash
# 문제 3: 로그 검색이 불가능
# OrderService 로그에서 특정 고객의 모든 요청 찾기
$ grep "customer_12345" /var/log/order-service.log
# 결과: 아무것도 없음 (로그가 구조화되지 않았으므로)

# 텍스트 기반 패턴 매칭만 가능
$ grep "Order creation" /var/log/order-service.log | wc -l
# 몇 천 개... 그 중 누가 customer_12345인지 알 수 없음
```

---

## ✨ 올바른 접근 (After)

```java
// ✅ 구조화 로그 (JSON) + Trace ID
// Logback 설정에서 자동으로 JSON 형식으로 기록됨

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private Tracer tracer;
    
    private static final Logger logger = LoggerFactory.getLogger(OrderController.class);
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody OrderRequest request) {
        
        // Trace ID는 자동으로 출력됨 (MDC를 통해)
        logger.info("Order creation started",
                kv("customer.id", request.getCustomerId()),
                kv("order.id", request.getOrderId()),
                kv("amount", request.getAmount()),
                kv("service", "order-service"));
        
        try {
            OrderResponse response = orderService.processOrder(request);
            
            logger.info("Order creation succeeded",
                    kv("customer.id", request.getCustomerId()),
                    kv("order.id", request.getOrderId()),
                    kv("status", "completed"));
            
            return ResponseEntity.ok(response);
        } catch (Exception e) {
            logger.error("Order creation failed",
                    kv("customer.id", request.getCustomerId()),
                    kv("order.id", request.getOrderId()),
                    kv("error", e.getMessage()),
                    kv("error.type", e.getClass().getName()));
            
            throw new OrderProcessingException(e);
        }
    }
    
    private StructuredArgument kv(String key, Object value) {
        return KeyValuePair.key(key).value(value);
    }
}

// 결과 로그 (JSON):
// {"timestamp":"2025-04-10T10:15:32.123Z","level":"INFO","service":"order-service","traceId":"550e8400-e29b-41d4-a716-446655440000","spanId":"1111111111111111","customer.id":"123","order.id":"order_456","amount":99.99,"message":"Order creation started"}
// {"timestamp":"2025-04-10T10:15:33.456Z","level":"INFO","service":"payment-service","traceId":"550e8400-e29b-41d4-a716-446655440000","spanId":"2222222222222222","customer.id":"123","transaction.id":"txn_789","status":"completed","message":"Payment succeeded"}
```

```bash
# Kibana에서 검색 (중앙화 로깅)
# 쿼리: traceId:"550e8400-e29b-41d4-a716-446655440000"
# 결과:
# [10:15:32.123] order-service  Order creation started customer_id=123 order_id=order_456 amount=99.99
# [10:15:32.456] order-service  Calling PaymentService
# [10:15:33.789] payment-service  Payment request received customer_id=123 transaction_id=txn_789
# [10:15:34.012] inventory-service  Stock check started customer_id=123 sku=SKU_001
# [10:15:35.678] inventory-service  Stock check completed sku=SKU_001 available=true
# [10:15:36.012] order-service  Order creation succeeded customer_id=123 order_id=order_456

# 필드 기반 검색 가능
# 쿼리: customer.id:"123" AND level:"ERROR"
# 결과: customer_123이 겪은 모든 에러를 시간순으로 표시
```

---

## 🔬 내부 동작 원리 — ELK Stack 아키텍처 심층 분석

### 아키텍처 다이어그램

```
┌─────────────────────────────────────────────────────────────────────┐
│                         마이크로서비스 환경                           │
│                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ OrderService │  │PaymentService│  │InventoryService              │
│  │              │  │              │  │              │              │
│  │ /var/log/    │  │ /var/log/    │  │ /var/log/    │              │
│  │ app.log      │  │ app.log      │  │ app.log      │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
└─────────┼──────────────────┼──────────────────┼────────────────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    로그 수집 (Filebeat/Logstash)                    │
│                                                                       │
│  실시간 로그 파일 모니터링 → 파싱 → 필터링 → 전송                   │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼ (대용량 처리)
┌─────────────────────────────────────────────────────────────────────┐
│                      메시지 큐 (Kafka)                               │
│                                                                       │
│  로그 버퍼링 → 여러 Logstash 인스턴스가 병렬 처리 가능               │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      로그 처리 (Logstash)                            │
│                                                                       │
│  1. 필터링: traceId 파싱, 에러 감지                                  │
│  2. 강화: GeoIP 추가, 호스트명 추가                                   │
│  3. 검증: JSON 스키마 검증                                            │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 검색 저장소 (Elasticsearch)                          │
│                                                                       │
│  ├─ logs-2025-04-10-000001  (1GB, Hot Tier)  → 7일 보관             │
│  ├─ logs-2025-04-03-000001  (1GB, Warm Tier) → 30일 보관            │
│  └─ logs-2025-03-31-000001  (1GB, Cold Tier) → 년단위 아카이빙      │
│                                                                       │
│  인덱싱: traceId, service, level, customer.id 필드별 역색인         │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  시각화 및 분석 (Kibana)                             │
│                                                                       │
│  - 대시보드: 서비스별 에러율, 응답시간                               │
│  - 검색: traceId 기반 통합 로그 조회                                │
│  - 알림: 에러율 > 5% → Slack 알림                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Trace ID를 통한 로그 통합 조회

```
원본 요청:
  Request: POST /api/orders
  Trace ID: 550e8400-e29b-41d4-a716-446655440000
  User: customer_123

OrderService (로그):
  2025-04-10T10:15:32.123Z [INFO] 
  {"traceId":"550e8400-e29b-41d4-a716-446655440000","service":"order-service","customer.id":"123","event":"order.created"}

PaymentService (로그):
  2025-04-10T10:15:33.456Z [INFO]
  {"traceId":"550e8400-e29b-41d4-a716-446655440000","service":"payment-service","customer.id":"123","event":"payment.initiated"}

InventoryService (로그):
  2025-04-10T10:15:34.789Z [INFO]
  {"traceId":"550e8400-e29b-41d4-a716-446655440000","service":"inventory-service","customer.id":"123","event":"stock.checked"}

NotificationService (로그):
  2025-04-10T10:15:35.012Z [INFO]
  {"traceId":"550e8400-e29b-41d4-a716-446655440000","service":"notification-service","customer.id":"123","event":"email.sent"}

Kibana 검색:
  쿼리: traceId:"550e8400-e29b-41d4-a716-446655440000"
  결과: 4개 로그가 시간순으로 모두 표시됨
        → 사용자는 "전체 요청 플로우"를 한눈에 파악 가능
```

### 로그 보존 계층 (Tiering)

```
시간 → 

Hot (0-7일)
├─ logs-2025-04-10  [읽기/쓰기 빈번] [고비용 SSD]
├─ logs-2025-04-09
├─ logs-2025-04-08
└─ ...
총 크기: 7GB, 접근성 100%

Warm (7-30일)
├─ logs-2025-04-03  [읽기 간헐] [저비용 HDD]
├─ logs-2025-03-27
├─ logs-2025-03-21
└─ ...
총 크기: 23GB, 접근성 90% (쿼리 5초)

Cold (30일 이상)
├─ logs-2025-03  [아카이빙] [S3/Glacier]
├─ logs-2025-02
├─ logs-2025-01
└─ ...
총 크기: 200GB, 접근성 50% (쿼리 1분 이상)

월간 비용 추정:
- Hot: 7GB × $0.10/GB = $0.70
- Warm: 23GB × $0.03/GB = $0.69
- Cold: 200GB × $0.01/GB = $2.00
- 합계: $3.39/월 (작은 규모)

대규모 (초당 10,000 요청):
- Hot: 700GB × $0.10 = $70
- Warm: 2.3TB × $0.03 = $69
- Cold: 20TB × $0.01 = $200
- 합계: $339/월 (상당한 비용)
```

---

## 💻 Logback + JSON 로깅 설정

```xml
<!-- logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProperty name="LOG_LEVEL" source="logging.level.root" defaultValue="INFO"/>
    <springProperty name="APP_NAME" source="spring.application.name" defaultValue="app"/>
    
    <!-- 콘솔 출력 (개발용) -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{ISO8601} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- JSON 파일 출력 (Filebeat가 수집) -->
    <appender name="JSON_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/application/app.log</file>
        
        <!-- 일일 롤링 + 30일 보관 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>/var/log/application/app.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>
        </rollingPolicy>
        
        <!-- JSON 포맷 -->
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <timeZone>UTC</timeZone>
            <!-- 기본 필드 자동 포함 -->
            <customFields>
                {"service":"${APP_NAME}","environment":"prod","version":"1.0.0"}
            </customFields>
            <!-- Trace Context 자동 포함 -->
            <includeContext>true</includeContext>
        </encoder>
    </appender>
    
    <!-- 에러 로그 분리 -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/application/error.log</file>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
        
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>/var/log/application/error.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>90</maxHistory>
        </rollingPolicy>
        
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"service":"${APP_NAME}","level":"ERROR"}</customFields>
        </encoder>
    </appender>
    
    <!-- Async 로깅 (성능 개선) -->
    <appender name="ASYNC_JSON" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold> <!-- 에러 로그는 버리지 않음 -->
        <appender-ref ref="JSON_FILE"/>
    </appender>
    
    <root level="${LOG_LEVEL}">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="ASYNC_JSON"/>
        <appender-ref ref="ERROR_FILE"/>
    </root>
    
    <!-- 특정 패키지는 DEBUG -->
    <logger name="com.example.order" level="DEBUG"/>
</configuration>
```

```java
// pom.xml 의존성
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
```

```java
// 구조화 로그 사용 예시
@Service
public class OrderService {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderService.class);
    
    public void processOrder(Order order) {
        // 로그에 구조화된 필드 추가
        MDC.put("order.id", order.getId());
        MDC.put("customer.id", order.getCustomerId());
        MDC.put("amount", String.valueOf(order.getAmount()));
        
        try {
            logger.info("Processing order",
                    kv("action", "order.process.start"),
                    kv("items.count", order.getItems().size()));
            
            // 비즈니스 로직
            validateOrder(order);
            createPayment(order);
            updateInventory(order);
            
            logger.info("Order processing completed",
                    kv("action", "order.process.complete"),
                    kv("status", "SUCCESS"));
            
        } catch (InvalidOrderException e) {
            logger.warn("Order validation failed",
                    kv("action", "order.validation.failed"),
                    kv("error.code", e.getErrorCode()),
                    kv("error.message", e.getMessage()),
                    kv("error.type", e.getClass().getSimpleName()));
            
            throw e;
            
        } catch (Exception e) {
            logger.error("Unexpected error during order processing",
                    kv("action", "order.process.error"),
                    kv("error.type", e.getClass().getName()),
                    kv("error.message", e.getMessage()),
                    kv("error.stacktrace", getStackTrace(e)));
            
            throw e;
            
        } finally {
            MDC.clear();  // Context 정리
        }
    }
    
    private StructuredArgument kv(String key, Object value) {
        return KeyValuePair.key(key).value(value);
    }
}
```

```yaml
# Logstash 설정 (logstash.conf)
input {
  kafka {
    bootstrap_servers => "kafka:9092"
    topics => ["application-logs"]
    codec => "json"
  }
}

filter {
  # Trace ID 파싱 (JSON에서 자동 추출됨)
  
  # 에러 감지 및 태깅
  if [level] == "ERROR" {
    mutate { add_tag => ["error", "alert"] }
    
    # 중요 에러는 즉시 알림
    if [message] =~ /OutOfMemory|StackOverflow/ {
      mutate { add_tag => ["critical"] }
    }
  }
  
  # GeoIP 추가 (호출자 정보)
  if [client.ip] {
    geoip {
      source => "client.ip"
      target => "geoip"
    }
  }
  
  # 응답시간 계산
  if [span.duration_ms] {
    mutate {
      convert => { "span.duration_ms" => "integer" }
      add_field => {
        "performance.category" => "normal"
      }
    }
    
    if [span.duration_ms] > 1000 {
      mutate { replace => { "performance.category" => "slow" } }
    }
  }
}

output {
  # Elasticsearch로 전송
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
    
    # 자동 인덱스 생성 정책 (ILM)
    ilm_pattern => "{now/d}-1"
    ilm_rollover_alias => "logs"
    ilm_policy => "logs-policy"
  }
  
  # 에러는 별도 인덱스에도 저장
  if "error" in [tags] {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "errors-%{+YYYY.MM.dd}"
    }
  }
}
```

```yaml
# Filebeat 설정 (filebeat.yml)
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/application/*.log
  
  # JSON 파싱
  json.keys_under_root: true
  json.add_error_key: true
  
  # 멀티라인 스택트레이스 처리
  multiline.pattern: '^\[|^  '
  multiline.negate: true
  multiline.match: after

output.kafka:
  hosts: ["kafka:9092"]
  topic: "application-logs"
  codec.json: {}
```

---

## 📊 패턴 비교

| 항목 | ELK Stack | EFK Stack | 분산 로그 (Before) |
|------|-----------|-----------|---------|
| **구성요소** | Elasticsearch + Logstash + Kibana | Elasticsearch + Fluentd + Kibana | 각 서버의 파일 로그 |
| **데이터 수집** | Logstash (무거움, 처리 강력) | Fluentd (가볍고 확장성) | SSH + tail |
| **Kubernetes 친화성** | 중간 | 매우 높음 | 불가능 |
| **저장소** | Elasticsearch (검색 최적화) | Elasticsearch | 파일 시스템 |
| **비용** | 중간 (CPU/메모리 많음) | 낮음 | 무시할 수 있음 |
| **검색 성능** | 빠름 (인덱싱) | 빠름 (인덱싱) | 느림 (grep) |
| **로그 보존** | Hot/Warm/Cold 자동 | 별도 구성 필요 | 수동 관리 |
| **분석 기능** | 강력함 (필터, 강화, 변환) | 중간 (플러그인 필요) | 없음 |

---

## ⚖️ 트레이드오프

### 장점 ✅
- **통합 로그 조회**: Trace ID로 10개 서비스 로그를 한 화면에서 확인
- **필드 기반 검색**: 고객ID, 주문번호, 에러 타입으로 빠른 필터링
- **실시간 모니터링**: Kibana 대시보드로 에러율, 응답시간 실시간 추적
- **자동화 알림**: 에러율 > 5% → Slack, PagerDuty 자동 알림
- **성능 분석**: 구조화 로그로 느린 API, DB 쿼리 자동 감지
- **감사 추적**: 모든 사용자 액션을 Trace ID와 함께 기록

### 단점 ❌
- **높은 저장소 비용**: 초당 10K 요청 시 월 $300-500 (Elasticsearch)
- **복잡한 인프라**: Elasticsearch 클러스터 관리, Kibana, Logstash 운영
- **네트워크 오버헤드**: 로그를 중앙으로 전송하는 대역폭 사용
- **성능 영향**: 대량 로깅 시 애플리케이션 성능 저하 (Async로 완화)
- **민감한 데이터 관리**: 신용카드, SSN 자동 마스킹 설정 필수
- **스토리지 폭발**: 로그 보존 정책 부재 시 저장소 부족

### 비용 분석

```
소규모 (월 1GB 로그):
- Elasticsearch: $20
- Kibana: $10
- Logstash: $10
- 합계: $40/월 (관리 가능)

중규모 (월 100GB 로그):
- Elasticsearch: $1,000
- Kibana: $100
- Logstash: $200
- 합계: $1,300/월

대규모 (월 1TB 로그):
- Elasticsearch: $10,000
- Kibana: $1,000
- Logstash: $2,000
- 합계: $13,000/월 (상당한 비용)

비용 절감 전략:
1. 샘플링: DEBUG 로그는 1% 샘플링
2. 필터링: 건강 체크, 메트릭 로그 제외
3. 보존: Hot 7일, Warm 30일, Cold 1년
4. 압축: Elasticsearch 자동 압축 (2배 용량 절감)
```

---

## 📌 핵심 정리

✅ **분산 로그의 필수 요소**
- 모든 로그에 Trace ID 포함 (자동 계측으로 가능)
- Trace ID로 여러 서비스의 로그를 통합 조회
- 구조화 로그 (JSON)로 필드 기반 검색 가능

✅ **ELK vs EFK 선택 기준**
- ELK: 대규모 엔터프라이즈, 복잡한 로그 처리
- EFK: Kubernetes, 마이크로서비스, 비용 절감
- 둘 다 Elasticsearch와 Kibana 사용 (호환성 높음)

✅ **중앙화 로깅으로 얻는 것**
- 장애 대응 시간 80% 단축 (1시간 → 12분)
- 근본 원인 분석 자동화 (필터 → 통계)
- 사용자 행동 추적 및 감사

✅ **로그 보존 정책의 중요성**
- Hot: 최근 1주일 (접근성 100%)
- Warm: 1개월 (접근성 90%)
- Cold: 1년 이상 (아카이빙, 접근성 50%)

✅ **성능 최적화**
- 비동기 로깅 (AsyncAppender): 메인 스레드 블로킹 방지
- 배치 전송 (Logstash): 네트워크 효율성
- 필드 필터링 (Logstash): 불필요한 데이터 제외

---

## 🤔 생각해볼 문제

**Q1.** 다음 로그에서 병목지점을 찾고, 개선 방안을 제시하시오.

```json
{"timestamp":"2025-04-10T10:15:32.123Z","traceId":"abc123","service":"order-service","message":"Order processing","duration_ms":2500}
{"timestamp":"2025-04-10T10:15:33.456Z","traceId":"abc123","service":"payment-service","message":"Payment initiated","duration_ms":1800}
{"timestamp":"2025-04-10T10:15:35.234Z","traceId":"abc123","service":"inventory-service","message":"Stock check","duration_ms":50}
{"timestamp":"2025-04-10T10:15:35.284Z","traceId":"abc123","service":"order-service","message":"Order completed","duration_ms":2850}
```

<details><summary>해설 보기</summary>
병목지점: PaymentService (1800ms)는 전체 요청 시간의 67%를 차지합니다.

개선 방안:
1. 결제 API 타임아웃 단축 (3초 → 1.5초)
2. 결제 서비스 비동기화 (Saga 패턴)
3. 결제 결과 캐싱 (같은 고객, 같은 금액의 반복 요청)
4. 결제 서비스 스케일링 (더 많은 인스턴스)
5. 외부 결제 게이트웨이 선택 (더 빠른 API)

이중 가장 빠른 개선은 4번 (수평 확장)입니다.
</details>

**Q2.** 초당 100,000개 요청이 들어오는 프로덕션 환경에서 구조화 로그를 모든 요청에 적용하면 어떤 문제가 발생하는가?

<details><summary>해설 보기</summary>
문제점:
1. 저장소 폭발: 월 수십TB (비용: $100,000+)
2. 네트워크 포화: 로그 전송만 1Gbps 이상
3. 애플리케이션 성능 저하: 로깅 오버헤드 10-20%
4. Elasticsearch 클러스터 과부하: 초당 수백만 문서 인덱싱

해결 방법:
1. 샘플링: DEBUG 로그 1%, INFO 10%, ERROR 100%
2. 필터링: 헬스 체크, 메트릭 로그 제외
3. 비동기 로깅: AsyncAppender로 메인 스레드 분리
4. 로그 압축: gzip 자동 적용 (2배 용량 감소)
5. 보존 정책: Hot 3일, Warm 7일, Cold 30일

예상 효과: 월 저장소 1TB로 축소, 비용 $300으로 절감
</details>

**Q3.** Trace ID가 로그에 포함되지 않는 문제가 발생했다. 원인과 해결책을 설명하시오.

```java
// 문제 코드
@Service
public class PaymentService {
    private static final Logger logger = LoggerFactory.getLogger(PaymentService.class);
    
    public void pay(Order order) {
        // 이 로그에는 Trace ID가 포함되지 않음
        logger.info("Payment started for order: " + order.getId());
    }
}
```

<details><summary>해설 보기</summary>
원인:
- Spring Cloud Sleuth 또는 Micrometer Tracing이 활성화되지 않음
- Logback이 MDC (Mapped Diagnostic Context)를 읽도록 설정되지 않음

해결책:

1. 의존성 추가:
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
```

2. Logback 설정에 traceId, spanId 추가:
```xml
<pattern>%d{ISO8601} [%X{traceId}] [%X{spanId}] %-5level %msg%n</pattern>
```
(또는 LogstashEncoder 사용: 자동 포함)

3. 재시작 후 로그에 Trace ID 나타남:
```
2025-04-10T10:15:33.456Z [550e8400-e29b-41d4-a716-446655440000] [2222222222222222] INFO Payment started
```
</details>

---

<div align="center">

**[⬅️ 이전: 분산 추적 — Trace ID로 서비스 흐름 추적](./01-distributed-tracing.md)** | **[홈으로 🏠](../README.md)** | **[다음: 메트릭 모니터링 — RED 방법론과 Grafana 대시보드 ➡️](./03-metrics-monitoring.md)**

</div>

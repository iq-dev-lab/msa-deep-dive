# 01. 분산 추적 — Trace ID로 서비스 흐름 추적

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Trace ID와 Span ID는 무엇이고, 어떤 차이가 있는가?
- 분산 추적이 MSA 환경에서 성능 병목지점을 찾는 데 어떻게 도움이 되는가?
- OpenTelemetry 표준을 따르는 추적 시스템의 기본 구성은 무엇인가?
- 샘플링 전략의 종류와 각각 언제 사용해야 하는가?
- Spring Boot에서 Micrometer Tracing으로 자동 계측을 어떻게 설정하는가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

마이크로서비스 아키텍처에서는 하나의 사용자 요청이 10개, 20개의 서비스를 거칳니다. API Gateway에서 시작된 요청이 OrderService → PaymentService → InventoryService → NotificationService를 순회합니다. 모놀리식 애플리케이션에서는 스택 트레이스 한 줄로 전체 호출 흐름을 볼 수 있었지만, MSA에서는 이러한 정보가 여러 프로세스, 여러 장비에 흩어져 있습니다.

분산 추적(Distributed Tracing)이 없다면 최종 사용자가 "결제가 느리다"고 불평할 때 어느 서비스가 문제인지 파악하는 데 수십 분이 소요됩니다. OrderService는 빠르고, PaymentService도 빠르지만, 숨겨진 3단계 호출 체인 때문에 InventoryService의 데이터베이스 쿼리가 3초 걸렸을지도 모릅니다. 분산 추적을 통해 우리는 각 서비스의 응답시간을 나노초 단위로 측정하고, 정확한 병목지점을 밀리초 단위로 찾아낼 수 있습니다.

OpenTelemetry는 모니터링 계층을 표준화하는 벤더 중립적인 프레임워크입니다. Zipkin, Jaeger, Datadog, New Relic 등 다양한 백엔드를 동일한 계측 코드로 지원합니다. 따라서 추적 백엔드를 바꾸더라도 애플리케이션 코드는 손대지 않아도 됩니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ 분산 추적이 없는 상황
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody OrderRequest request) {
        // 요청이 여기서 들어옴 - 하지만 어디서 왔는지 식별 정보 없음
        long startTime = System.currentTimeMillis();
        
        OrderResponse response = orderService.processOrder(request);
        
        long duration = System.currentTimeMillis() - startTime;
        System.out.println("Total time: " + duration + "ms"); // 최상위 레이어만 시간 측정
        
        return ResponseEntity.ok(response);
    }
}

// PaymentService 내부 - 이 서비스가 느린지 알 수 없음
@Service
public class PaymentService {
    @Autowired
    private PaymentGateway gateway;
    
    public PaymentResult pay(String orderId, BigDecimal amount) {
        // 어느 요청인지 식별할 방법이 없음
        // OrderService 호출이 500개 있을 때, 이 호출은 어느 주문 처리 중인가?
        return gateway.chargeCard(amount);
    }
}

// 로그 분석
// OrderService: "[2025-04-10 10:15:32] Order processed in 2.5s"
// PaymentService: "[2025-04-10 10:15:33] Payment charged" (어느 주문의 결제인지 불명확)
// InventoryService: "[2025-04-10 10:15:34] Stock updated" (동일 시각대이지만 연관성 불명확)
```

문제점:
- 서비스 A에서 "느렸다"는 로그가 있어도 왜 느린지 모름
- 서비스 B, C, D 중 누가 책임인지 파악 불가
- 네트워크 지연인지 DB 쿼리인지 알 수 없음
- 엔드-투-엔드 요청 흐름을 추적할 방법이 없음

---

## ✨ 올바른 접근 (After)

```java
// ✅ OpenTelemetry를 통한 분산 추적
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    // Tracer는 자동으로 주입됨 (Spring Boot Auto-Configuration)
    @Autowired
    private Tracer tracer;
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody OrderRequest request) {
        // 새로운 Span 생성 (이 엔드포인트의 작업)
        // Trace ID는 자동으로 전파됨
        Span span = tracer.spanBuilder("createOrder")
                .setAttribute("order.id", request.getOrderId())
                .setAttribute("customer.id", request.getCustomerId())
                .setAttribute("amount", request.getAmount())
                .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            OrderResponse response = orderService.processOrder(request);
            span.addEvent("order.processed", 
                Attributes.of(AttributeKey.stringKey("status"), "success"));
            return ResponseEntity.ok(response);
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, "Order processing failed");
            throw e;
        } finally {
            span.end();
        }
    }
}

// PaymentService는 동일한 Trace ID를 전파받음
@Service
public class PaymentService {
    
    @Autowired
    private PaymentGateway gateway;
    
    @Autowired
    private Tracer tracer;
    
    public PaymentResult pay(String orderId, BigDecimal amount) {
        // 새로운 Span이지만, 동일한 Trace ID를 가짐
        Span span = tracer.spanBuilder("payment.process")
                .setAttribute("order.id", orderId)
                .setAttribute("amount", amount.doubleValue())
                .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // 이 호출의 성능도 자동으로 추적됨
            PaymentResult result = gateway.chargeCard(amount);
            span.addEvent("payment.success", 
                Attributes.of(AttributeKey.stringKey("transaction.id"), 
                    result.getTransactionId()));
            return result;
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, "Payment failed");
            throw e;
        } finally {
            span.end();
        }
    }
}

// Trace 및 Span 정보 로그에 포함
@Configuration
public class LoggingConfiguration {
    
    @Bean
    public LoggingSpanExporter loggingExporter() {
        // Trace ID와 Span ID가 자동으로 로그에 포함됨
        return LoggingSpanExporter.create();
    }
}

// 결과: Zipkin 또는 Jaeger 백엔드에서 waterfall 차트로 시각화
// Trace ID: 550e8400-e29b-41d4-a716-446655440000
// ├─ Span: createOrder (0ms - 2500ms) [OrderService]
// │  ├─ Span: order.validate (0ms - 50ms)
// │  ├─ Span: payment.process (50ms - 1500ms) [PaymentService] ← 1450ms 소요
// │  │  ├─ Span: payment.gateway.call (100ms - 1400ms) ← 병목!
// │  ├─ Span: inventory.update (1500ms - 2200ms) [InventoryService]
// │  └─ Span: notification.send (2200ms - 2500ms) [NotificationService]
```

개선점:
- 동일 Trace ID로 모든 서비스 요청이 연결됨
- 각 작업 단위(Span)의 정확한 시간 측정
- 병목지점을 밀리초 단위로 지정 가능
- Span에 메타데이터(주문ID, 고객ID) 저장 → 나중에 필터링 가능

---

## 🔬 내부 동작 원리 — OpenTelemetry 아키텍처 심층 분석

### Trace, Span, Context 개념

```
┌─────────────────────────────────────────────────────────────────────┐
│ Trace ID: 550e8400-e29b-41d4-a716-446655440000 (요청 전체)          │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Span ID: 1111 (OrderService 진입) [0ms - 2500ms]             │  │
│  │ - service_name: "order-service"                               │  │
│  │ - span_name: "createOrder"                                    │  │
│  │ - attributes: {order.id: "123", amount: 99.99}               │  │
│  │                                                                │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │ Span ID: 2222 (PaymentService 호출) [50ms - 1500ms]     │ │  │
│  │  │ - Parent Span ID: 1111 (계층 구조 표현)               │ │  │
│  │  │ - service_name: "payment-service"                       │ │  │
│  │  │ - span_name: "payment.process"                          │ │  │
│  │  │ - attributes: {order.id: "123", amount: 99.99}         │ │  │
│  │  │                                                          │ │  │
│  │  │  ┌───────────────────────────────────────────────────┐ │ │  │
│  │  │  │ Span ID: 3333 (외부 API 호출) [100ms - 1400ms]   │ │ │  │
│  │  │  │ - Parent Span ID: 2222                            │ │ │  │
│  │  │  │ - span_name: "payment.gateway.call"              │ │ │  │
│  │  │  │ - status: ERROR (이 구간에서 느림!)              │ │ │  │
│  │  │  └───────────────────────────────────────────────────┘ │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  │                                                                │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │ Span ID: 4444 (InventoryService 호출) [1500ms - 2200ms] │ │  │
│  │  │ - Parent Span ID: 1111                                   │ │  │
│  │  │ - service_name: "inventory-service"                      │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

**핵심 용어:**
- **Trace ID**: 요청 처음부터 끝까지 동일한 고유 ID (UUID 또는 128-bit)
- **Span ID**: 개별 서비스의 작업 단위 (각 Span마다 고유)
- **Parent Span ID**: 호출 계층 표현 (OrderService의 Span이 PaymentService의 Span을 호출)
- **Context**: 현재 활성 Trace와 Span 정보 (ThreadLocal 또는 구조화된 동시성으로 전파)

### Context 전파 메커니즘

HTTP 헤더를 통한 전파:
```
요청 1 (API Gateway)
  Header: traceparent: 00-550e8400e29b41d4a716446655440000-1111111111111111-01

  → OrderService
    자동으로 Trace ID 파싱: 550e8400-e29b-41d4-a716-446655440000
    Span ID 생성: 2222222222222222
    Header: traceparent: 00-550e8400e29b41d4a716446655440000-2222222222222222-01
    
    → PaymentService
      동일 Trace ID 파싱
      새로운 Span ID 생성: 3333333333333333
      Header: traceparent: 00-550e8400e29b41d4a716446655440000-3333333333333333-01
```

**W3C Trace Context 표준:**
```
traceparent: 00-TraceID-SpanID-Flags
             ││  │        │     │
             ││  │        │     └─ 추적 활성화 플래그 (01)
             ││  │        └─ 부모 Span ID (16자리 hex)
             ││  └─ Trace ID (32자리 hex)
             └─ 버전 (현재 00)
```

### 샘플링 전략

```java
// 1. 전체 샘플링 (ALWAYS_ON)
// 모든 요청을 추적 - 개발/테스트 환경
@Bean
public Sampler alwaysOnSampler() {
    return Sampler.alwaysOn();
    // 프로덕션에서는 매우 비싼 옵션 (로그 저장량 증가)
}

// 2. 확률적 샘플링 (PROBABILITY)
// 1% 샘플링 - 요청 100개 중 1개만 추적
@Bean
public Sampler probabilitySampler() {
    return Sampler.traceIdRatioBased(0.01);
    // 프로덕션 권장: 0.01 ~ 0.1 (1% ~ 10%)
}

// 3. Head-based 샘플링 (결정: 처음 서비스에서)
// OrderService에서 샘플 여부 결정 → 모든 다운스트림 서비스도 동일 결정
@Bean
public Sampler headBasedSampler() {
    return Sampler.parentBased(Sampler.traceIdRatioBased(0.05));
    // 부모 Span이 없으면 (= 최초 진입점), 5% 확률로 샘플
}

// 4. Tail-based 샘플링 (결정: 마지막 서비스에서)
// Collector가 전체 Trace 수집 후, 백엔드에서 샘플링 판단
// 예: 에러 발생한 Trace는 100%, 정상은 1%
// 이 방식은 전체 Trace를 수집하므로 네트워크/저장소 비용 증가
```

**샘플링 비교:**

| 전략 | 오버헤드 | 정확도 | 사용 시기 |
|------|--------|--------|---------|
| Always On | 높음 | 완벽함 | 개발/디버깅 |
| Probability | 낮음 | 통계적 | 프로덕션 (1-10%) |
| Head-based | 낮음 | 통계적 | 권장 (일관성) |
| Tail-based | 매우 높음 | 높음 | 에러 위주 분석 |

---

## 💻 Spring Boot + Micrometer Tracing 설정

```yaml
# application-prod.yml
spring:
  application:
    name: order-service
  
  # OpenTelemetry 활성화
  otel:
    exporter:
      otlp:
        endpoint: http://localhost:4317  # Collector 주소
    traces:
      exporter: otlp
      sampler:
        jaeger:
          type: parentbased_traceidratio
          param: 0.05  # 5% 샘플링
  
  # Micrometer Tracing (Spring Boot 3.0+)
  tracing:
    propagation:
      type: jaeger  # W3C Trace Context 사용
    enabled: true

management:
  tracing:
    sampling:
      probability: 0.05  # 5% 샘플링
  otlp:
    metrics:
      export:
        endpoint: http://localhost:4317
    tracing:
      endpoint: http://localhost:4317
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus,env
```

```java
// pom.xml 의존성
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
    <version>1.32.0-SNAPSHOT</version>
</dependency>
```

```java
// TracingConfiguration.java - 고급 설정
@Configuration
public class TracingConfiguration {
    
    // 커스텀 SpanExporter (Trace 데이터를 어디로 보낼지 정의)
    @Bean
    public OtlpGrpcSpanExporter otlpGrpcSpanExporter() {
        return OtlpGrpcSpanExporter.builder()
                .setEndpoint("http://localhost:4317")
                .setTimeout(Duration.ofSeconds(5))
                .build();
    }
    
    // Sampler 설정 (프로덕션에서 오버헤드 제어)
    @Bean
    public Sampler sampler(Environment env) {
        String profile = env.getProperty("spring.profiles.active");
        
        if ("prod".equals(profile)) {
            // 프로덕션: 에러 또는 느린 요청만 샘플링
            return Sampler.parentBased(
                Sampler.traceIdRatioBased(0.01)  // 기본 1%
            );
        }
        return Sampler.alwaysOn();  // 개발 환경: 모두 추적
    }
    
    // 특정 요청 필터링 (민감한 엔드포인트 제외)
    @Bean
    public SpanFilteringSpanExporter spanFilteringExporter() {
        return new SpanFilteringSpanExporter();
    }
}

// 커스텀 SpanFilter - 민감한 정보 필터링
public class SpanFilteringSpanExporter implements SpanExporter {
    
    @Override
    public CompletableResultCode export(Collection<SpanData> spans) {
        // 신용카드 처리 관련 Span은 전송하지 않음 (보안)
        List<SpanData> filteredSpans = spans.stream()
                .filter(span -> !span.getName().contains("payment.gateway"))
                .collect(Collectors.toList());
        
        // 필터된 데이터만 Collector로 전송
        return CompletableResultCode.ofSuccess();
    }
}
```

```java
// 비동기 호출에서의 Context 전파
@Service
public class AsyncOrderService {
    
    @Autowired
    private Tracer tracer;
    
    @Autowired
    private ThreadContextElement contextElement;
    
    public void processOrderAsync(Order order) {
        // 현재 Context 캡처
        SpanContext currentContext = tracer.currentSpan().getSpanContext();
        
        CompletableFuture.runAsync(() -> {
            // 비동기 스레드에서 Context 복원
            try (Scope scope = tracer.withSpan(
                    tracer.spanBuilder("async.process")
                            // Parent Context 지정
                            .setParent(Context.current())
                            .startSpan())) {
                
                processOrderInternal(order);
            }
        });
    }
}
```

---

## 📊 패턴 비교

| 항목 | 수동 추적 (Before) | OpenTelemetry (After) |
|------|--------|-----------|
| **구현 방식** | 수동으로 Thread ID, 타임스탬프 기록 | 자동 계측 (Auto-instrumentation) |
| **코드 침투성** | 높음 (모든 함수에 로깅 코드 필요) | 낮음 (데코레이터/인터셉터) |
| **의존성** | 로깅 라이브러리 (SLF4J, Log4j) | OpenTelemetry (벤더 중립) |
| **백엔드 전환** | 코드 수정 필요 | 설정만 변경 (Zipkin → Jaeger) |
| **성능 오버헤드** | 5-15% (동기 로깅) | 1-3% (비동기 배치) |
| **메타데이터** | 제한적 (텍스트 로그) | 구조화됨 (attributes, events) |
| **샘플링** | 불가능 (모든 요청 기록) | 효율적 (확률적 샘플링) |
| **외부 서비스 자동 감지** | 불가능 | 가능 (HTTP, gRPC, DB) |

---

## ⚖️ 트레이드오프

### 장점 ✅
- **정확한 병목지점 파악**: 나노초 단위로 각 서비스 성능 측정
- **근본 원인 분석 (RCA) 가능**: Waterfall 차트로 시각화된 호출 경로
- **자동 계측**: 라이브러리 수준에서 HTTP, DB, 메시지 브로커 자동 추적
- **프로덕션 안전**: 샘플링으로 오버헤드 1-3%로 제어
- **벤더 중립성**: 추적 백엔드 변경 시 코드 수정 불필요
- **구조화된 쿼리**: Trace ID로 관련 로그, 메트릭 자동 상관관계 분석

### 단점 ❌
- **추가 인프라**: Collector, Zipkin/Jaeger 설치/운영 필요
- **저장소 비용**: 대규모 트래픽 시 Trace 저장 비용 상당 (Elasticsearch, S3)
- **복잡한 분석**: Waterfall 차트 해석에 경험 필요
- **Context 전파 복잡성**: 비동기, 메시지 큐 환경에서 Context 수동 관리
- **민감한 데이터 필터링**: 신용카드번호, 개인정보 포함된 Span 자동 필터링 필요
- **네트워크 오버헤드**: Collector로 Trace 전송 시 네트워크 대역폭 사용

### 비용-편익 분석
```
개발 환경: Sampler.alwaysOn() 사용
- 오버헤드: 10-20%
- 저장소: 무시해도 됨
- 편익: 버그 조기 발견

스테이징: 10% 샘플링
- 오버헤드: 1-2%
- 저장소: 월 100GB
- 편익: 운영 전 병목지점 검증

프로덕션 (높은 트래픽): 1% 샘플링
- 오버헤드: 0.1-0.3%
- 저장소: 월 50GB (비용: ~$500)
- 편익: 장애 대응 시 근본 원인 분석 가능
```

---

## 📌 핵심 정리

✅ **Trace ID와 Span의 이해**
- 하나의 요청 = 1개 Trace ID + 여러 Span ID
- Span ID는 부모-자식 관계로 호출 계층 표현
- Context 전파로 마이크로서비스 간 식별 정보 자동 연결

✅ **OpenTelemetry 표준의 이점**
- 벤더 중립성: Zipkin, Jaeger, Datadog 모두 지원
- 자동 계측: Spring AOP, HTTP ClientInterceptor로 라이브러리 수준 지원
- W3C Trace Context 표준: 엔터프라이즈 환경에서 호환성 보장

✅ **샘플링으로 운영 비용 제어**
- 개발: 100% (모든 요청)
- 테스트: 10-50%
- 프로덕션: 1-5% (장애 추적 + 성능 가용성 균형)

✅ **병목지점을 정확하게 찾기**
- Waterfall 차트에서 가장 긴 Span 식별
- Span의 attributes (order.id, customer.id)로 필터링
- DB 슬로우 쿼리, API 타임아웃, 네트워크 지연 분리 진단

✅ **Context 전파의 핵심**
- HTTP 헤더 (traceparent)로 자동 전파
- 비동기/메시지 큐: 수동으로 Context 캡처 및 복원
- ThreadLocal (동기) vs 구조화된 동시성 (Virtual Thread) 대응

---

## 🤔 생각해볼 문제

**Q1.** 다음 시나리오에서 병목지점은 어디인가? Waterfall 차트를 해석하시오.

```
Trace ID: abc123
├─ OrderService.createOrder [0ms - 1000ms]
│  ├─ OrderService.validate [0ms - 10ms]
│  ├─ PaymentService.pay [10ms - 800ms]  ← 790ms 소요
│  │  ├─ DB.insert [10ms - 50ms]
│  │  └─ PaymentGateway.charge [50ms - 800ms]  ← 750ms 소요!
│  └─ NotificationService.send [800ms - 1000ms]
```

<details><summary>해설 보기</summary>
병목지점은 PaymentGateway.charge (50ms - 800ms, 750ms)입니다. 이는 외부 결제 API 응답이 느린 상황입니다. 개선 방법:
1. 결제 API 타임아웃 단축 (불가능시 Circuit Breaker 적용)
2. 결제 프로세스 비동기화 (Saga 패턴)
3. 결제 서비스 재시도 로직 추가

OrderService.validate와 NotificationService.send는 실제로 병목이 아닙니다.
</details>

**Q2.** 프로덕션 환경에서 1% 샘플링을 사용할 때, 초당 10,000개 요청이 들어온다면 하루 수집되는 Trace 데이터량은 약 얼마인가? (Span당 평균 2KB 가정)

<details><summary>해설 보기</summary>
- 1% 샘플링 → 초당 100개 요청 추적
- 요청당 평균 5개 Span (OrderService → PaymentService → ... → NotificationService)
- 하루: 86,400초 × 100 요청/초 × 5 Span × 2KB = 약 86GB

하루 86GB는 매우 많으므로, 실제로는:
1. 샘플링을 0.5%로 낮추거나
2. Tail-based 샘플링 (에러만 100%, 정상은 0.1%)
3. Trace 보존 기간 7일로 제한 → 월 600GB → $300 정도

따라서 프로덕션에서는 1% 미만의 샘플링이 권장됩니다.
</details>

**Q3.** 비동기 작업에서 Context가 전파되지 않는 문제가 발생했다. 다음 코드의 문제점과 해결방법을 설명하시오.

```java
@Service
public class OrderService {
    @Async
    public void processOrderAsync(Order order) {
        // 비동기 스레드에서 실행됨
        // 이 메서드의 Trace ID는?
    }
}
```

<details><summary>해설 보기</summary>
문제: 비동기 메서드는 다른 스레드에서 실행되므로, ThreadLocal 기반의 Context가 전파되지 않습니다. 결과적으로 원본 요청의 Trace ID와 단절됩니다.

해결 방법:
1. Spring Cloud Sleuth 사용: `@EnableSpringCloudSleuth` 자동 Context 전파
2. 수동 처리:
```java
@Async
public void processOrderAsync(Order order) {
    Span span = tracer.spanBuilder("async.process")
            .setParent(Context.current())  // 부모 Context 지정
            .startSpan();
    
    try (Scope scope = span.makeCurrent()) {
        // 비동기 작업
    } finally {
        span.end();
    }
}
```

3. Java 21 Virtual Thread 사용: Context가 자동 전파됨
</details>

---

<div align="center">

**[⬅️ 이전: Chapter 6 — BFF 패턴](../api-gateway/05-bff-pattern.md)** | **[홈으로 🏠](../README.md)** | **[다음: 중앙화 로깅 — Trace ID로 분산 로그 조합 ➡️](./02-centralized-logging.md)**

</div>

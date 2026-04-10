# 01. 동기 통신 vs 비동기 통신 — 선택 기준

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 동기 호출이 전체 시스템의 가용성을 낮추는 수학적 원리는 무엇인가?
- 결제 서비스는 왜 동기 통신을 사용해야 하고, 알림 서비스는 비동기를 사용해야 하는가?
- Eventual Consistency를 도입했을 때의 실제 비용과 수용 기준은 무엇인가?
- 동기와 비동기를 섞어 사용할 때 각 단계에서 결합도와 지연시간이 어떻게 변하는가?
- 비동기 메시지 손실 시나리오와 그에 따른 설계 패턴의 차이는?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

MSA에서 동기 vs 비동기 통신의 선택은 단순한 기술 결정이 아니라 **시스템 전체의 가용성, 응답 시간, 결합도를 결정하는 근본적인 아키텍처 결정**입니다.

동기 호출이 많을수록 서비스 간 강한 결합도가 생기며, 한 서비스의 지연이나 장애가 전체 시스템에 연쇄적으로 영향을 미칩니다. 예를 들어 결제를 위해 5개 서비스(인증, 주문, 재고, 결제, 알림)를 순차적으로 호출하면, 각 서비스의 가용성이 99.9%일 때 전체 가용성은 99.9% ^ 5 ≈ 99.5%로 하락합니다. 이것이 MSA에서 "비동기 우선" 원칙이 강조되는 이유입니다.

그러나 모든 통신을 비동기로 할 수는 없습니다. 결제 승인, 신용카드 검증, 실시간 재고 확인처럼 **즉시 응답이 필수적인 비즈니스 로직**은 동기 호출로만 구현 가능합니다. 비동기로 처리하면 최종 일관성(Eventual Consistency)에 따라 수 초 또는 수 분 뒤에 승인 여부가 결정되는데, 이는 사용자 경험을 해칩니다.

따라서 MSA의 핵심은 **각 시나리오에 맞는 통신 방식을 정확히 선택하고, 불가피한 동기 호출의 영향을 최소화하는 설계**입니다.

---

## 😱 흔한 실수 (Before — 모든 서비스 호출을 동기로 처리)

```java
// ❌ 안티패턴: 주문 생성 시 모든 확인을 동기로 처리
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest request) {
    // 1. 사용자 검증 (User Service)
    User user = userServiceClient.getUser(request.getUserId()); // 100ms
    if (user == null) throw new UserNotFoundException();
    
    // 2. 재고 확인 (Inventory Service)
    boolean hasStock = inventoryServiceClient.hasStock(request.getProductId()); // 150ms
    if (!hasStock) throw new OutOfStockException();
    
    // 3. 가격 조회 (Pricing Service)
    Price price = pricingServiceClient.getPrice(request.getProductId()); // 120ms
    
    // 4. 배송지 검증 (Shipping Service)
    shippingServiceClient.validateAddress(request.getAddress()); // 180ms
    
    // 5. 결제 처리 (Payment Service)
    PaymentResult payment = paymentServiceClient.charge(
        user.getCreditCard(), price.getAmount()); // 1000ms (네트워크 지연)
    if (!payment.isSuccess()) throw new PaymentFailedException();
    
    // 6. 알림 발송 (Notification Service)
    notificationServiceClient.sendEmail(user.getEmail(), 
        "주문이 완료되었습니다."); // 200ms
    
    // 총 소요 시간: 100 + 150 + 120 + 180 + 1000 + 200 = 1750ms
    // 문제점: 
    // - 한 서비스 장애 → 전체 주문 시스템 다운
    // - 결제 완료까지 1.75초 (사용자는 클릭 후 답답함)
    // - Inventory 장애로 Payment, Notification도 대기
    
    Order order = orderRepository.save(new Order(user, price, request.getAddress()));
    return ResponseEntity.ok(new OrderResponse(order.getId()));
}
```

**문제점:**
1. **가용성 악화**: 5개 서비스 중 1개 장애 → 전체 주문 시스템 다운 (가용성: 0.99^5 = 0.995 = 99.5%)
2. **지연 누적**: 모든 호출이 순차적 → 1.75초 이상 소요
3. **결합도 증가**: 새 서비스 추가 시 createOrder 메서드 수정 필요
4. **리소스 낭비**: Notification Service 느려도 결제 확인 지연

---

## ✨ 올바른 접근 (After — 동기/비동기 하이브리드)

```java
// ✅ 개선된 패턴: 필수 항목은 동기, 나머지는 비동기
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest request) {
    // === CRITICAL PATH (동기): 반드시 완료되어야 진행 ===
    
    // 1. 사용자 검증 (User Service) - 동기 필요
    User user = userServiceClient.getUser(request.getUserId()); // 100ms
    if (user == null) throw new UserNotFoundException();
    
    // 2. 재고 차감 (Inventory Service) - 동기 필요 (동시성 문제)
    InventoryResult inventory = inventoryServiceClient.reserveStock(
        request.getProductId(), request.getQuantity()); // 150ms
    if (!inventory.isSuccess()) throw new OutOfStockException();
    
    // 3. 가격 확정 (Pricing Service) - 동기 필요 (결제 정확성)
    Price price = pricingServiceClient.getPrice(request.getProductId()); // 120ms
    
    // 4. 결제 처리 (Payment Service) - 동기 필수 (금융거래)
    PaymentResult payment = paymentServiceClient.charge(
        user.getCreditCard(), price.getAmount()); // 1000ms
    if (!payment.isSuccess()) {
        // 결제 실패 시 예약 취소
        inventoryServiceClient.cancelReservation(inventory.getReservationId());
        throw new PaymentFailedException();
    }
    
    // 동기 호출 완료 시점: ~1370ms
    Order order = orderRepository.save(new Order(
        user, price, request.getAddress(), inventory.getReservationId()));
    
    // === NON-CRITICAL PATH (비동기): 나중에 처리해도 됨 ===
    
    // 5. 배송지 검증 → 이벤트로 발행
    eventBus.publish(new OrderCreatedEvent(order.getId(), request.getAddress()));
    // → ShippingService가 구독하여 비동기 처리
    
    // 6. 알림 발송 → 메시지 큐로 발행
    notificationQueue.send(new EmailNotificationMessage(
        user.getEmail(), 
        "주문 #" + order.getId() + "이 완료되었습니다."));
    // → NotificationService가 구독하여 비동기 처리
    
    // 사용자에게 반환되는 시간: ~1370ms (충분히 빠름)
    // 장점:
    // - 결제 완료 후 즉시 주문 ID 반환
    // - 배송지 검증·알림 지연 → 사용자 경험 영향 없음
    // - 배송지/알림 장애 → 주문 시스템에 영향 없음
    
    return ResponseEntity.ok(new OrderResponse(order.getId()));
}

// === 비동기 핸들러 (별도 프로세스) ===
@EventListener
public void onOrderCreated(OrderCreatedEvent event) {
    // 배송지 검증 (실패해도 주문은 완료)
    try {
        shippingServiceClient.validateAddress(event.getAddress());
    } catch (Exception e) {
        logger.warn("배송지 검증 실패: {}", event.getOrderId());
        // Compensation: 사용자에게 알림, 수동 개입 프로세스 시작
    }
}

@KafkaListener(topics = "order-created")
public void onOrderNotification(Order order) {
    // 알림 발송 (재시도 가능)
    for (int attempt = 0; attempt < 3; attempt++) {
        try {
            emailService.send(order.getUser().getEmail(), 
                "주문 완료: #" + order.getId());
            return;
        } catch (Exception e) {
            if (attempt < 2) Thread.sleep(1000 * (attempt + 1));
        }
    }
    // 최종 실패 시 DLQ로 이동
}
```

**개선 효과:**
1. **지연 감소**: 1.75초 → 1.37초 (21% 개선)
2. **가용성 향상**: 배송·알림 장애 → 주문에 영향 없음
3. **확장성**: 새로운 구독자(예: 분석) 추가 시 OrderCreatedEvent 구독만 하면 됨

---

## 🔬 내부 동작 원리 — 동기/비동기 선택의 수학

### 1. 동기 호출의 가용성 계산

```
전체 가용성 = A₁ × A₂ × A₃ × ... × Aₙ

예: 5개 서비스 동기 호출, 각 99.9% 가용성
    전체 = 0.999 ^ 5 = 0.995004999 ≈ 99.5%
    
실제 다운타임 예상:
- 월 99.9% = 43분 다운타임
- 월 99.5% = 3.6시간 다운타임 ← 3배 증가!
```

### 2. 응답 시간 누적 (Critical Path Method)

```
동기: 선형 가산
Sum(호출1, 호출2, ...) = 100ms + 150ms + 120ms + 1000ms = 1370ms

비동기: 병렬 처리 가능
Max(Critical Path) = 1370ms (동기 부분)
Non-Critical Path = 병렬 처리 (사용자 경험에 미치지 않음)
```

### 3. 결합도 분석

**동기 호출의 결합도:**
```
Order Service → User Service
Order Service → Inventory Service  (시간적 결합도: 호출 시점에 응답 필요)
Order Service → Payment Service   (논리적 결합도: 코드에 하드코딩)
Order Service → Notification Service
```

**비동기 이벤트의 결합도 감소:**
```
Order Service → Event Bus (이벤트 발행)
               ↓
           [OrderCreatedEvent]
               ↓
         ┌─────┬─────┐
         ↓     ↓     ↓
      Service1 Service2 Service3
      (독립적으로 구독, 실패해도 Order Service 영향 없음)
```

### 4. Eventual Consistency 지연 분석

```
동기 결제:     사용자 "주문 완료" → 즉시 UI 업데이트
             시간: T

비동기 결제:   사용자 "주문 처리 중" → 이벤트 큐에 발행 → 결제 서비스 처리
             시간: T + (네트워크 + 처리) ≈ T + 5~10초
             
트레이드오프:
- 동기: 빠르지만 장애 시 전체 시스템 영향
- 비동기: 느리지만 격리, 재시도 가능, 확장 용이
```

---

## 💻 코드 예제 — 동기/비동기 하이브리드 구현

### 동기 호출 (RestTemplate + Timeout)
```java
@Configuration
public class RestClientConfig {
    @Bean
    public RestTemplate restTemplate() {
        HttpComponentsClientHttpRequestFactory factory = 
            new HttpComponentsClientHttpRequestFactory();
        factory.setConnectTimeout(5000);      // 5초 타임아웃
        factory.setReadTimeout(10000);        // 10초 타임아웃
        return new RestTemplate(factory);
    }
}

@Service
public class OrderService {
    @Autowired private RestTemplate restTemplate;
    
    public Order createOrder(OrderRequest request) {
        try {
            // Timeout으로 보호: 10초 이상 대기 금지
            ResponseEntity<User> userResp = restTemplate.getForEntity(
                "http://user-service/users/" + request.getUserId(),
                User.class);
            User user = userResp.getBody();
            // ...
        } catch (ResourceAccessException e) {
            throw new ServiceTimeoutException("User Service 타임아웃");
        }
    }
}
```

### 비동기 이벤트 발행
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("order-async-");
        executor.initialize();
        return executor;
    }
}

@Service
public class OrderService {
    @Autowired private ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(...));
        
        // 트랜잭션 커밋 후 이벤트 발행 (중요!)
        eventPublisher.publishEvent(new OrderCreatedEvent(this, order));
        return order;
    }
}

@Component
public class OrderEventListener {
    @Async // 별도 스레드에서 처리
    @EventListener
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        Order order = event.getOrder();
        logger.info("Order created: {}", order.getId());
        // 배송, 알림 등 처리
    }
}
```

### Kafka를 이용한 이벤트 기반 통신
```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      acks: all                    # 모든 레플리카 확인
      retries: 3
      properties:
        linger.ms: 10              # 배치 대기 시간
        compression.type: snappy
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      max-poll-records: 100
```

```java
@Component
public class OrderEventProducer {
    @Autowired private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderCreated(Order order) {
        OrderEvent event = new OrderEvent(
            order.getId(),
            order.getUserId(),
            order.getTotalPrice(),
            System.currentTimeMillis());
        
        kafkaTemplate.send("order-events", 
            order.getId().toString(), 
            event);
    }
}

@Component
public class OrderEventConsumer {
    @KafkaListener(topics = "order-events", groupId = "notification-service")
    public void handleOrderCreated(OrderEvent event) {
        emailService.sendConfirmation(event.getUserId(), event.getOrderId());
    }
}
```

---

## 📊 패턴 비교 — 동기 vs 비동기

| 특성 | 동기 (Request-Response) | 비동기 (Event-Driven) |
|------|----------------------|-------------------|
| **응답 시간** | 즉시 (밀리초) | 지연됨 (초~분) |
| **가용성** | 낮음 (순차 곱셈) | 높음 (독립적) |
| **결합도** | 높음 (시간적·논리적) | 낮음 (느슨한 결합) |
| **트랜잭션** | ACID 보장 가능 | Eventual Consistency |
| **개발 복잡도** | 낮음 | 높음 (상태 관리) |
| **재시도 용이성** | 어려움 | 쉬움 (메시지 재처리) |
| **스케일링** | 병목(동기 호출자) | 높음 (Consumer 추가) |
| **사용 사례** | 결제, 조회, 검증 | 알림, 분석, 부가 기능 |
| **순서 보장** | 호출 순서로 보장 | 토픽/파티션 레벨 보장 |

---

## ⚖️ 트레이드오프

```
동기 호출의 장점과 비용:
✅ 장점
  - 응답 즉시성 (사용자 경험 좋음)
  - 트랜잭션 처리 간단 (ACID)
  - 에러 처리 명확 (동기식 try-catch)
  
❌ 비용
  - 한 서비스 장애 → 전체 시스템 다운 (Cascading Failure)
  - 지연 누적 (응답 시간 증가)
  - 확장성 제한 (수평 확장 어려움)
  - 버전 관리 복잡 (API 진화)

────────────────────────────────────────

비동기 이벤트의 장점과 비용:
✅ 장점
  - 높은 가용성 (서비스 독립성)
  - 낮은 결합도 (새 구독자 추가 쉬움)
  - 좋은 확장성 (Consumer 추가로 처리량 증가)
  - 지연 분산 (큐에 버퍼링)
  
❌ 비용
  - Eventual Consistency (지연된 일관성)
  - 복잡한 상태 관리 (Saga 패턴, 보정 트랜잭션)
  - 메시지 손실 가능성 (재시도 필요)
  - 디버깅 어려움 (비동기 추적)
  - 운영 복잡도 (메시지 큐 관리)
```

---

## 📌 핵심 정리

```
✅ 동기 통신 선택 기준
  1. 즉시 응답이 비즈니스 요구사항인 경우
     → 결제 승인, 주문 확인, 실시간 조회
  2. 트랜잭션 일관성이 필수인 경우
     → 금전 거래, 재고 차감
  3. 에러 처리가 동기식으로 필요한 경우
     → 사용자에게 즉시 피드백 (실패/성공)

✅ 비동기 통신 선택 기준
  1. 지연이 수용 가능한 경우
     → 알림, 이메일, 분석, 로깅
  2. 높은 가용성이 필요한 경우
     → 많은 서비스 연계 시스템
  3. 이벤트 기반 확장이 필요한 경우
     → 새로운 서비스 추가 시 기존 코드 수정 최소화

✅ 하이브리드 설계 원칙
  1. Critical Path (동기): 주요 비즈니스 로직
  2. Non-Critical Path (비동기): 부가 기능
  3. Timeout 설정: 동기 호출은 반드시 타임아웃 정의
  4. 재시도 정책: 비동기는 3회 재시도 + DLQ
  5. 모니터링: 각 호출의 지연, 실패율 추적
```

---

## 🤔 생각해볼 문제

**Q1.** 전자상거래 시스템에서 "상품 추천" 기능을 추가하려고 합니다. 사용자가 주문을 완료한 후 추천 상품 목록을 받는데, 동기/비동기 중 어느 방식을 선택해야 하고 그 이유는?

<details>
<summary>해설 보기</summary>

**비동기 이벤트 기반 추천이 최적**

1. **비즈니스 요구사항**: 추천은 "나중에 봐도 괜찮음" (지연 수용 가능)
2. **가용성**: 추천 엔진 장애 → 주문 완료에 영향 없음
3. **확장성**: 추천 알고리즘 개선 시 Order Service 코드 수정 없음
4. **구현**:
   ```java
   // Order Service가 발행
   eventBus.publish(new OrderCompletedEvent(order.getId(), order.getUserId()));
   
   // Recommendation Service가 구독 (별도 배포)
   @KafkaListener(topics = "order-completed")
   public void generateRecommendations(OrderCompletedEvent event) {
       List<Product> recommendations = mlModel.recommend(event.getUserId());
       notificationService.sendPushNotification(
           event.getUserId(), 
           "추천 상품: " + recommendations);
   }
   ```

반면 동기 호출이면: 추천 생성 지연 → 사용자 주문 완료 지연 → UX 악화

</details>

**Q2.** 동기 호출의 가용성을 0.99^n에서 0.999^n으로 개선하려면? (n=5 서비스 기준)

<details>
<summary>해설 보기</summary>

**선택지:**
1. **각 서비스 가용성 개선** (0.99 → 0.999)
   - 비용: 높음 (인프라, 복제, 모니터링)
   - 효과: 0.999^5 = 99.95% (약간만 개선)
   - 현실: 매우 어려움

2. **Circuit Breaker + Fallback** (동기 호출 수 감소)
   ```java
   @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
   public PaymentResult charge(Payment payment) {
       return paymentClient.process(payment);
   }
   
   public PaymentResult paymentFallback(Payment payment, Exception e) {
       // 결제 실패 → 대기 큐에 저장, 나중에 재시도
       paymentQueue.offer(payment);
       return new PaymentResult(Status.PENDING);
   }
   ```
   - 장애 시: 동기 호출 실패 → 비동기로 전환
   - 가용성: 부분 개선

3. **비동기로 마이그레이션** (근본 해결) ⭐ **권장**
   - 5개 서비스 중 2-3개를 비동기로 변경
   - 새로운 가용성: 동기 2-3개 + 비동기 2-3개
   - 효과: 99.95% 이상 달성 가능

**결론**: 가용성 개선은 동기 호출 수를 줄이는 것이 가장 효과적

</details>

**Q3.** 한 Kafka 토픽이 OrderCreatedEvent를 발행하는데, 3개 서비스(Notification, Shipping, Analytics)가 구독합니다. Notification Service만 장애가 나면?

<details>
<summary>해설 보기</summary>

**Kafka의 Consumer Group 메커니즘:**

```
Topic: order-events (Partition 0, 1, 2)

Consumer Group: notification-service
  - offset: 1000 (처리한 메시지까지)
  - Status: 중단 (장애)

Consumer Group: shipping-service
  - offset: 1050 (계속 처리 중)
  - Status: 정상

Consumer Group: analytics-service
  - offset: 1030 (계속 처리 중)
  - Status: 정상
```

**각 서비스에 미치는 영향:**
- ❌ Notification Service: 메시지 소비 안 됨, offset 진전 안 됨
- ✅ Shipping Service: 영향 없음 (독립적 offset 유지)
- ✅ Analytics Service: 영향 없음 (독립적 offset 유지)

**복구 시나리오:**
1. Notification Service 재시작
2. Kafka에서 마지막 처리한 offset(1000)부터 메시지 다시 소비
3. 손실된 1000~1050 범위의 메시지 재처리 (자동)

**동기 호출이었다면:**
```
Order Service → Notification Service (실패 발생)
              ↓
         전체 주문 시스템 중단 ❌
```

이것이 비동기 + Kafka가 높은 가용성을 제공하는 이유!

</details>

---

<div align="center">

**[⬅️ 이전: 서비스 자율성 원칙 — 독립 배포·확장·기술·데이터](../service-decomposition/06-service-autonomy-principles.md)** | **[홈으로 🏠](../README.md)** | **[다음: REST API 설계 — 서비스 간 계약과 버저닝 ➡️](./02-rest-api-design.md)**

</div>

# 05. 운영 중 발생하는 장애 패턴 — 순환 의존성과 카스케이드 장애

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 순환 의존성(Circular Dependency)이란 무엇이고, MSA에서 발생하는 이유는?
- 카스케이드 장애(Cascade Failure)의 메커니즘을 이해하고, 그것을 방지하는 방법은?
- Circuit Breaker, Timeout, Bulkhead 패턴이 각각 어떤 문제를 해결하는가?
- 분산 데드락 상황에서 감지 및 복구 절차는?
- Saga 패턴에서 실패한 트랜잭션을 어떻게 추적하고 복구할 것인가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

마이크로서비스 아키텍처는 "독립적으로 배포 가능한 서비스들의 느슨한 연결"을 목표로 합니다. 하지만 실제로는 서비스들이 강하게 연결되어 있습니다. OrderService가 PaymentService를 호출하고, PaymentService가 RiskService를 호출하고, RiskService가 다시 OrderService를 호출한다면? 또는 PaymentService가 느려져서 OrderService의 스레드 풀이 다 떨어지고, 결국 OrderService 자체도 느려지면? 이러한 상황들이 MSA 환경에서 자주 발생합니다.

모놀리식 애플리케이션에서는 같은 프로세스 내에서 동기 호출만 일어나므로, 문제가 발생해도 한 부분에 국한됩니다. 하지만 MSA에서는 네트워크를 통한 호출이므로, 한 서비스의 장애가 전체 시스템으로 파급됩니다. 이를 "카스케이드 장애"라고 부릅니다. 결제 시스템 장애로 시작한 문제가 주문 시스템, 재고 시스템, 알림 시스템까지 전파되어 결국 모든 서비스가 다운될 수 있습니다.

따라서 MSA 운영에서는 "전체 시스템이 한 서비스의 장애로 함께 다운되지 않도록" 설계해야 합니다. 장애를 격리하고, 일부 기능을 제한하더라도 핵심 기능은 살리는 "우아한 성능 저하(Graceful Degradation)"를 구현해야 합니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ 순환 의존성 문제
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private PaymentService paymentService;
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody OrderRequest request) {
        
        // OrderService → PaymentService 호출
        PaymentResult result = paymentService.pay(request.getCustomerId(), request.getAmount());
        
        if (result.isSuccess()) {
            return ResponseEntity.ok(new OrderResponse("OK"));
        }
        return ResponseEntity.status(500).build();
    }
}

// PaymentService가 OrderService를 다시 호출
@Service
public class PaymentService {
    
    @Autowired
    private OrderController orderController;
    
    public PaymentResult pay(String customerId, BigDecimal amount) {
        // 결제 이전에 주문 정보 검증
        try {
            // PaymentService → OrderService → PaymentService
            // 이렇게 되면 무한 재귀!
            OrderResponse order = orderController.getOrder(customerId);
            
            // ...처리...
            return new PaymentResult(true);
        } catch (Exception e) {
            return new PaymentResult(false);
        }
    }
}

// 문제점:
// 1. OrderService와 PaymentService가 서로 호출
// 2. 만약 circular resolve가 발생하면 StackOverflowError
// 3. 순환 구조 자체가 아키텍처 문제 신호
```

```java
// ❌ 카스케이드 장애 (아무 보호도 없음)
@Service
public class OrderService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    public OrderResponse processOrder(Order order) {
        // PaymentService 호출 (timeout 없음!)
        PaymentResult result = restTemplate.postForObject(
                "http://payment-service/pay",
                order,
                PaymentResult.class);
        
        if (result.isSuccess()) {
            // InventoryService 호출 (timeout 없음!)
            inventoryService.updateStock(order.getItems());
            
            // NotificationService 호출 (timeout 없음!)
            notificationService.sendEmail(order.getCustomerId());
        }
        
        return new OrderResponse(result);
    }
}

// 시나리오: PaymentService 장애 발생
// 시간 10:00 - PaymentService 에러율 50%
// 시간 10:01 - OrderService 스레드 풀 75% 사용 (PaymentService 호출 대기)
// 시간 10:02 - OrderService 스레드 풀 100% 사용 (모든 스레드 blocked)
// 시간 10:03 - OrderService도 장애! (신규 요청 처리 불가)
// 시간 10:04 - API Gateway가 OrderService 타임아웃으로 감지
// 시간 10:05 - 모든 다운스트림 서비스도 OrderService 호출 중단
// → 카스케이드 장애 발생!

// 문제점:
// - PaymentService 5초 timeout 없음 → 무한 대기
// - OrderService 스레드 풀이 고갈 → 모든 요청 못받음
// - 하나의 느린 서비스가 전체 시스템 다운시킴
```

```java
// ❌ 분산 트랜잭션 실패 (Saga 복구 불가)
@Service
public class OrderSagaService {
    
    public void processOrderSaga(Order order) {
        try {
            // Step 1: 주문 생성
            orderRepository.save(order);
            
            // Step 2: 결제 수행
            paymentService.pay(order.getId(), order.getAmount());  // 실패!
            // Exception 발생!
            
            // Step 3: 재고 업데이트 (실행 안 됨)
            inventoryService.updateStock(order.getItems());
            
        } catch (PaymentException e) {
            // 보상 트랜잭션 없음!
            // 문제:
            // - 주문은 생성됨 (orderRepository.save)
            // - 결제는 실패함 (paymentService.pay)
            // - 재고는 변경되지 않음
            // → 데이터 일관성 깨짐! (주문만 남음)
            
            // 복구 방법:
            // 1. 주문 삭제? (고객이 이미 확인했을 수도)
            // 2. 수동 개입? (관리자가 매번 확인?)
            // 3. 보상 로직? (없음)
            
            throw e;
        }
    }
}

// 결과:
// - 주문 DB: 주문 데이터 존재 (status=PENDING)
// - 결제 DB: 결제 기록 없음 (실패)
// - 고객 요청: "왜 주문은 생성됐는데 결제는 안 됐어?"
// - 운영팀: 수동으로 주문 삭제 또는 환불 처리 필요
```

---

## ✨ 올바른 접근 (After)

```java
// ✅ 순환 의존성 해결: 이벤트 기반 아키텍처로 전환
// Before: OrderService → PaymentService → OrderService (순환)
// After: OrderService → Event(PaymentProcessed) → InventoryService

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private EventPublisher eventPublisher;
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody OrderRequest request) {
        
        // 1. 주문 생성
        Order order = orderRepository.save(
                new Order(request.getCustomerId(), request.getAmount()));
        
        // 2. "주문 생성됨" 이벤트 발행 (Pub/Sub)
        // PaymentService는 이 이벤트를 구독하여 처리
        eventPublisher.publish(
                new OrderCreatedEvent(order.getId(), order.getAmount()));
        
        // 3. 즉시 응답 (비동기 처리)
        return ResponseEntity.ok(new OrderResponse(order.getId(), "PENDING"));
    }
}

// PaymentService (이벤트 구독)
@Service
public class PaymentService {
    
    @Autowired
    private EventPublisher eventPublisher;
    
    // "주문 생성됨" 이벤트를 받으면 결제 처리
    @EventListener
    public void handleOrderCreatedEvent(OrderCreatedEvent event) {
        try {
            PaymentResult result = chargeCard(event.getAmount());
            
            // 결제 성공 이벤트 발행
            eventPublisher.publish(
                    new PaymentCompletedEvent(event.getOrderId(), true));
        } catch (Exception e) {
            // 결제 실패 이벤트 발행
            eventPublisher.publish(
                    new PaymentCompletedEvent(event.getOrderId(), false));
        }
    }
}

// InventoryService (이벤트 구독)
@Service
public class InventoryService {
    
    @EventListener
    public void handlePaymentCompletedEvent(PaymentCompletedEvent event) {
        if (event.isSuccess()) {
            updateStock(event.getOrderId());
        }
    }
}

// 장점:
// 1. 순환 의존성 제거 (A → Event → B 형태)
// 2. 느슨한 결합 (서비스들이 직접 의존 안 함)
// 3. 비동기 처리로 응답성 향상
```

```java
// ✅ 카스케이드 장애 방지: Circuit Breaker + Timeout + Bulkhead
@Configuration
public class ResilientRpcConfiguration {
    
    @Bean
    public RestClient paymentServiceClient(RestClientBuilder builder) {
        return builder
                .baseUrl("http://payment-service")
                // Timeout 설정
                .requestFactory(factory -> {
                    HttpComponentsClientHttpRequestFactory f =
                            (HttpComponentsClientHttpRequestFactory) factory;
                    f.setConnectTimeout(Duration.ofSeconds(2));
                    f.setReadTimeout(Duration.ofSeconds(5));
                })
                .build();
    }
}

@Service
@CircuitBreaker(
        name = "paymentService",
        fallbackMethod = "paymentServiceFallback")
@Retry(name = "paymentService")
@Bulkhead(name = "paymentService")
@Timeout(duration = "5s")
public class OrderService {
    
    @Autowired
    private RestClient paymentServiceClient;
    
    public PaymentResult processPayment(Order order) {
        // Circuit Breaker 적용
        // 상태:
        // 1. CLOSED: 정상 (호출 통과)
        // 2. OPEN: 장애 (호출 차단, 즉시 실패)
        // 3. HALF_OPEN: 회복 중 (테스트 호출 1개 허용)
        
        try {
            return paymentServiceClient.post()
                    .uri("/pay")
                    .body(order)
                    .retrieve()
                    .toEntity(PaymentResult.class)
                    .getBody();
        } catch (Exception e) {
            // Fallback: 대체 로직
            return paymentServiceFallback(order, e);
        }
    }
    
    // Circuit Breaker가 열림 (PaymentService 장애)
    public PaymentResult paymentServiceFallback(Order order, Exception e) {
        // 방법 1: 캐시된 결제 정보 사용
        PaymentResult cached = paymentCache.get(order.getCustomerId());
        if (cached != null) {
            return cached;
        }
        
        // 방법 2: 결제 유보 (나중에 처리)
        // 결제 없이 주문 생성만 하고, 배치 작업이 나중에 결제 처리
        pendingPaymentQueue.add(order);
        return new PaymentResult(PaymentStatus.PENDING);
    }
}

// Resilience4j 설정 (application.yml)
resilience4j:
  circuitbreaker:
    configs:
      default:
        registerHealthIndicator: true
        slidingWindowSize: 100  # 최근 100개 요청으로 판단
        failureRateThreshold: 50  # 50% 실패 → OPEN
        slowCallRateThreshold: 50  # 50% 느린 요청(>2초) → OPEN
        waitDurationInOpenState: 30s  # 30초 후 HALF_OPEN으로 전환
        permittedNumberOfCallsInHalfOpenState: 3  # HALF_OPEN에서 3개 호출 허용
        automaticTransitionFromOpenToHalfOpenEnabled: true
    instances:
      paymentService:
        baseConfig: default
        slowCallDurationThreshold: 2s  # 2초 이상 = slow call
  
  retry:
    configs:
      default:
        maxAttempts: 3
        waitDuration: 1000  # 1초 대기 후 재시도
        retryExceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
    instances:
      paymentService:
        baseConfig: default
  
  bulkhead:
    configs:
      default:
        maxConcurrentCalls: 20  # 동시 호출 최대 20개
        maxWaitDuration: 2s  # 2초 대기 후 실패
    instances:
      paymentService:
        baseConfig: default

// Bulkhead 효과 (스레드 풀 격리):
// OrderService 스레드 풀: 50개
// ├─ PaymentService 호출용: 20개 (별도 Bulkhead)
// └─ 기타 작업: 30개 (다른 요청 처리 가능)
//
// PaymentService가 느려지면:
// ├─ PaymentService 호출 20개 스레드 모두 blocked
// └─ 기타 작업 30개 스레드는 계속 작동 (부분 서비스 가능)
```

```java
// ✅ Saga 패턴 with 보상 트랜잭션
@Service
public class OrderSagaService {
    
    @Autowired
    private SagaOrchestrator sagaOrchestrator;
    
    public void processOrderSaga(Order order) {
        // Saga: 여러 서비스의 분산 트랜잭션
        // 각 스텝이 실패하면 그 이전의 모든 스텝을 취소
        
        sagaOrchestrator.executeSaga(
                order,
                
                // 스텝 1: 주문 생성
                step()
                        .action(step -> {
                            orderRepository.save(order);
                            return SagaStepResult.success(order);
                        })
                        // 보상 트랜잭션: 주문이 이후 스텝에서 실패하면 취소
                        .compensation(step -> {
                            orderRepository.delete(order);
                            return SagaStepResult.success();
                        })
                        .build(),
                
                // 스텝 2: 결제 수행
                step()
                        .action(step -> {
                            PaymentResult result = paymentService.pay(
                                    order.getId(),
                                    order.getAmount());
                            
                            if (!result.isSuccess()) {
                                // 실패 시 모든 이전 스텝의 보상 트랜잭션 실행
                                return SagaStepResult.failure(
                                        "Payment failed: " + result.getReason());
                            }
                            return SagaStepResult.success(result);
                        })
                        // 보상 트랜잭션: 결제 취소
                        .compensation(step -> {
                            paymentService.refund(order.getId(), order.getAmount());
                            return SagaStepResult.success();
                        })
                        .build(),
                
                // 스텝 3: 재고 업데이트
                step()
                        .action(step -> {
                            inventoryService.updateStock(order.getItems());
                            return SagaStepResult.success();
                        })
                        // 보상 트랜잭션: 재고 복구
                        .compensation(step -> {
                            inventoryService.restoreStock(order.getItems());
                            return SagaStepResult.success();
                        })
                        .build(),
                
                // 스텝 4: 알림 발송
                step()
                        .action(step -> {
                            notificationService.sendEmail(
                                    order.getCustomerId(),
                                    "Order created: " + order.getId());
                            return SagaStepResult.success();
                        })
                        .build()  // 알림 실패는 무시 (보상 필요 없음)
        );
    }
}

// Saga 실행 흐름:
// 성공 경로:
// Step 1 ✓ → Step 2 ✓ → Step 3 ✓ → Step 4 ✓ → 완료!
//
// 실패 경로 (Step 3에서 실패):
// Step 1 ✓ → Step 2 ✓ → Step 3 ✗
//   ↓ 역순 보상:
// Step 2 보상 (refund) ✓
// Step 1 보상 (delete order) ✓
// → 최종 상태: 주문 없음, 결제 없음, 재고 복구됨
```

---

## 🔬 내부 동작 원리 — 분산 장애 탐지 및 복구

### Circuit Breaker 상태 전이 다이어그램

```
┌─────────────────────────────────────────────────────────────────────┐
│               Circuit Breaker 상태 머신                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│                         CLOSED (정상)                                │
│                       ┌─────────────┐                                │
│                       │             │                                │
│     ┌─────────────────┤  요청 통과  ├────────┐                       │
│     │                 │             │        │                       │
│     │                 └─────────────┘        │                       │
│     │                      ▲                 │                       │
│     │                      │                 │                       │
│     │                      │          에러율 > 50%                   │
│     │                      │          또는                           │
│     │                      │          느린 호출(>2초) > 50%          │
│     │                      │                 │                       │
│     │                      │                 ▼                       │
│     │                  ┌─────────────┐                               │
│     │                  │             │                               │
│     │                  │   OPEN      │ ◄─────────────┐               │
│     │                  │ (장애 확인)  │              │               │
│     │                  │             │          즉시 실패            │
│     │                  └─────────────┘          Exception 반환       │
│     │                       │                    (통과 불가)        │
│     │                       │                                        │
│     │                 30초 대기 후                                   │
│     │               (waitDurationInOpenState)                        │
│     │                       │                                        │
│     │                       ▼                                        │
│     │                  ┌─────────────┐                               │
│     │                  │             │                               │
│     │     ┌─────────────┤ HALF_OPEN   ├────────┐                    │
│     │     │             │ (회복 시도) │        │                    │
│     │     │             │             │        │                    │
│     │     │             └─────────────┘        │                    │
│     │     │                  ▲                 │                    │
│     │     │                  │                 │                    │
│     │  통과 3개 허용          │          1개 실패하면               │
│     │  (permittedNumberOf...│          다시 OPEN으로              │
│     │   CallsInHalfOpenState)│                 │                    │
│     │                        │                 ▼                    │
│     │                        │            OPEN으로 돌아감           │
│     │                        │                                       │
│     └────────────────────────┘                                       │
│                                                                       │
│  예시:                                                               │
│  CLOSED → (에러율 50%+) → OPEN → (30초 대기) → HALF_OPEN           │
│                              ↑ (실패) ↓                             │
│                              └─(다시 OPEN)                          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 카스케이드 장애 메커니즘

```
시간대별 장애 전파:

10:00:00 - PaymentService 장애 시작
  ┌─────────────────────────────────┐
  │ PaymentService: 500 req/s        │
  │ - 응답시간: 100ms → 5초 증가    │
  │ - 에러율: 0% → 50% 증가         │
  └─────────────────────────────────┘

10:00:10 - OrderService 영향 받기 시작
  ┌─────────────────────────────────┐
  │ OrderService:                    │
  │ - Thread Pool (50개):            │
  │   ├─ 사용 가능: 50개             │
  │   ├─ PaymentService 호출 대기: 0개
  │                                  │
  │ 각 요청이 PaymentService 호출:    │
  │ (대기 시간: 5초)                 │
  │                                  │
  │ Thread 사용 계산:                │
  │ = 들어오는 요청 / 응답시간       │
  │ = 1000 req/s / 5s                │
  │ = 5,000개 필요!                  │
  │                                  │
  │ 하지만 Thread는 50개만 존재      │
  │ → 모두 blocked!                  │
  └─────────────────────────────────┘

10:00:20 - OrderService 완전 장애
  ┌─────────────────────────────────┐
  │ OrderService:                    │
  │ - Thread Pool: 50개 모두 사용    │
  │ - 신규 요청: 모두 타임아웃       │
  │ - 응답시간: 5초 → 30초 증가      │
  │                                  │
  │ ApiGateway가 OrderService 감지:  │
  │ → 자동으로 트래픽 차단           │
  └─────────────────────────────────┘

10:00:30 - 다운스트림 서비스 영향
  ┌─────────────────────────────────┐
  │ FrontendService (API Gateway):   │
  │ - OrderService 호출 실패         │
  │ - InventoryService 호출도 실패   │
  │   (OrderService 없이 재고 조회?)  │
  │                                  │
  │ 모든 e-commerce 기능 다운!       │
  └─────────────────────────────────┘

해결책 적용 시간대:

10:00:10 - Circuit Breaker 적용
  PaymentService 에러율 50% → OPEN
  → 더 이상 PaymentService 호출 안 함 (즉시 실패)
  → OrderService 스레드 해제!

10:00:15 - OrderService 회복
  - 스레드 이미 사용 가능 (25개)
  - 신규 요청 처리 가능
  - 응답시간 정상화

효과: 카스케이드 차단! ✓
```

### Saga 패턴의 보상 트랜잭션

```
분산 트랜잭션 성공 경로:

Order {
  id: "ORD-001",
  customerId: "CUST-123",
  amount: 99.99,
  items: [SKU-A, SKU-B]
}

Step 1: 주문 생성 ✓
  OrderDB:
  ├─ ORD-001: {status: CREATED, amount: 99.99}

Step 2: 결제 수행 ✓
  PaymentDB:
  ├─ TXN-001: {orderId: ORD-001, amount: 99.99, status: SUCCESS}
  
  OrderDB (업데이트):
  ├─ ORD-001: {status: PAYMENT_CONFIRMED}

Step 3: 재고 업데이트 ✓
  InventoryDB:
  ├─ SKU-A: quantity = 100 - 1 = 99
  ├─ SKU-B: quantity = 50 - 2 = 48
  
  OrderDB (업데이트):
  ├─ ORD-001: {status: INVENTORY_RESERVED}

Step 4: 알림 발송 ✓
  NotificationDB:
  ├─ NOTIF-001: {orderId: ORD-001, type: ORDER_CREATED}
  
  OrderDB (최종):
  ├─ ORD-001: {status: COMPLETED}

최종 상태: 모두 성공! ✓

─────────────────────────────────────────────

분산 트랜잭션 실패 경로 (Step 3에서 재고 부족):

Step 1: 주문 생성 ✓
  OrderDB:
  ├─ ORD-002: {status: CREATED}

Step 2: 결제 수행 ✓
  PaymentDB:
  ├─ TXN-002: {orderId: ORD-002, amount: 199.98}

Step 3: 재고 업데이트 ✗ (SKU-C 재고 부족)
  InventoryDB:
  ├─ SKU-C: quantity = 2 (필요: 5개)
  → Exception: InsufficientStockException

즉시 보상 트랜잭션 시작 (역순):

보상 2: 결제 취소 ✓
  PaymentDB (업데이트):
  ├─ TXN-002: {status: REFUNDED}
  
  OrderDB (업데이트):
  ├─ ORD-002: {status: PAYMENT_REFUNDED}

보상 1: 주문 취소 ✓
  OrderDB (최종):
  ├─ ORD-002: {status: CANCELLED}

최종 상태:
├─ OrderDB: 주문 취소됨
├─ PaymentDB: 환불됨
├─ InventoryDB: 변경 없음 (Step 3 전에 실패)
→ 데이터 일관성 유지! ✓
```

---

## 💻 실제 구현 코드

```java
// 순환 의존성 감지 (DFS 기반)
@Component
public class CircularDependencyDetector {
    
    public void detectCircularDependencies(
            Map<String, Set<String>> dependencyGraph) {
        
        // 각 서비스 노드에서 DFS 시작
        for (String service : dependencyGraph.keySet()) {
            Set<String> visited = new HashSet<>();
            Set<String> recursionStack = new HashSet<>();
            
            if (hasCycle(service, visited, recursionStack, dependencyGraph)) {
                logger.error("Circular dependency detected starting from: " + service);
                
                // 순환 경로 추출
                List<String> cycle = extractCyclePath(
                        service, recursionStack, dependencyGraph);
                
                logger.error("Cycle path: " + String.join(" → ", cycle));
                
                // 알림 발송 (운영팀)
                alerting.sendAlert(AlertLevel.CRITICAL,
                        "Circular Dependency: " + String.join(" → ", cycle));
            }
        }
    }
    
    private boolean hasCycle(String current, Set<String> visited,
                             Set<String> recursionStack,
                             Map<String, Set<String>> graph) {
        
        visited.add(current);
        recursionStack.add(current);
        
        // 현재 서비스가 호출하는 모든 서비스 검사
        for (String neighbor : graph.getOrDefault(current, new HashSet<>())) {
            if (!visited.contains(neighbor)) {
                if (hasCycle(neighbor, visited, recursionStack, graph)) {
                    return true;
                }
            } else if (recursionStack.contains(neighbor)) {
                // 순환 발견!
                return true;
            }
        }
        
        recursionStack.remove(current);
        return false;
    }
}

// Circuit Breaker 상세 모니터링
@Component
public class CircuitBreakerMonitor {
    
    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;
    
    @Scheduled(fixedRate = 5000)  // 5초마다
    public void monitorCircuitBreakers() {
        for (CircuitBreaker cb : circuitBreakerRegistry.getAllCircuitBreakers()) {
            CircuitBreaker.State state = cb.getState();
            
            // 상태 메트릭 기록
            meterRegistry.gauge("circuitbreaker.state",
                    Tags.of("name", cb.getName(), "state", state.toString()),
                    stateToNumber(state));
            
            // 상태 변화 감지
            if (state == CircuitBreaker.State.OPEN) {
                logger.warn("Circuit Breaker OPEN: {}", cb.getName());
                alerting.sendAlert(AlertLevel.WARNING,
                        "Circuit Breaker opened: " + cb.getName());
                
                // 메트릭 확인
                CircuitBreakerMetrics metrics = cb.getMetrics();
                logger.warn("Failure rate: {}%, Call count: {}",
                        metrics.getFailureRate() * 100,
                        metrics.getNumberOfCalls());
            }
        }
    }
    
    private int stateToNumber(CircuitBreaker.State state) {
        return switch (state) {
            case CLOSED -> 0;      // 정상
            case OPEN -> 1;        // 장애
            case HALF_OPEN -> 2;   // 회복 시도
            default -> -1;
        };
    }
}

// 분산 데드락 감지
@Component
public class DistributedDeadlockDetector {
    
    @Autowired
    private SagaExecutionRepository sagaRepo;
    
    @Scheduled(fixedRate = 30000)  // 30초마다
    public void detectAndRecoverFromDeadlock() {
        // 실행 중인 모든 Saga 조회
        List<SagaExecution> activeSagas = sagaRepo.findAllActive();
        
        for (SagaExecution saga : activeSagas) {
            // 실행 시간 확인
            Duration duration = Duration.between(
                    saga.getStartTime(), Instant.now());
            
            // 임계값 (10분) 초과 → 데드락 의심
            if (duration.toMinutes() > 10) {
                logger.error("Potential deadlock detected: sagaId={}, duration={}",
                        saga.getId(), duration);
                
                // 데드락 상태 분석
                analyzeDeadlockState(saga);
                
                // 자동 복구 시도
                attemptRecovery(saga);
            }
        }
    }
    
    private void analyzeDeadlockState(SagaExecution saga) {
        // 각 스텝의 상태 확인
        for (SagaStep step : saga.getSteps()) {
            logger.info("Step: {}, Status: {}, StartTime: {}",
                    step.getName(), step.getStatus(), step.getStartTime());
            
            // 특정 스텝에서 오래 머물러있으면 해당 스텝의 호출 서비스 확인
            if (step.getStatus() == SagaStepStatus.IN_PROGRESS) {
                // 외부 서비스 상태 확인
                ServiceHealth health = healthChecker.check(step.getTargetService());
                if (!health.isHealthy()) {
                    logger.error("Service {} is unhealthy, blocking saga step",
                            step.getTargetService());
                }
            }
        }
    }
    
    private void attemptRecovery(SagaExecution saga) {
        // 방법 1: 타임아웃된 스텝 재시도
        SagaStep failedStep = saga.getSteps().stream()
                .filter(s -> s.getStatus() == SagaStepStatus.IN_PROGRESS)
                .filter(s -> Duration.between(s.getStartTime(), Instant.now())
                        .toMinutes() > 5)
                .findFirst()
                .orElse(null);
        
        if (failedStep != null) {
            logger.info("Retrying saga step: {}", failedStep.getName());
            failedStep.setStatus(SagaStepStatus.RETRY);
            sagaRepo.save(saga);
        }
        
        // 방법 2: 전체 보상 (Saga 롤백)
        if (shouldCompensate(saga)) {
            logger.info("Initiating saga compensation for: {}", saga.getId());
            compensateSaga(saga);
        }
    }
    
    private void compensateSaga(SagaExecution saga) {
        // 모든 완료된 스텝을 역순으로 보상
        List<SagaStep> completedSteps = saga.getSteps().stream()
                .filter(s -> s.getStatus() == SagaStepStatus.COMPLETED)
                .sorted(Comparator.comparingInt(SagaStep::getSequence).reversed())
                .toList();
        
        for (SagaStep step : completedSteps) {
            try {
                logger.info("Executing compensation for step: {}", step.getName());
                step.executeCompensation();
            } catch (Exception e) {
                logger.error("Compensation failed for step: {}", step.getName(), e);
                // 보상 실패 → 관리자 알림
                alerting.sendAlert(AlertLevel.CRITICAL,
                        "Saga compensation failed: " + saga.getId());
            }
        }
        
        saga.setStatus(SagaExecutionStatus.COMPENSATED);
        sagaRepo.save(saga);
    }
}
```

---

## 📊 패턴 비교

| 항목 | Circuit Breaker | Timeout | Bulkhead | Fallback |
|------|---------|---------|----------|----------|
| **목적** | 연쇄 장애 차단 | 무한 대기 방지 | 리소스 격리 | 우아한 저하 |
| **동작** | 에러율 초과 시 호출 차단 | N초 후 실패 | 스레드 풀 분리 | 대체 로직 실행 |
| **부작용** | 호출자가 즉시 실패 | 응답 지연 | 리소스 2배 사용 | 부정확한 결과 가능 |
| **복합도** | 낮음 | 낮음 | 중간 | 중간 |
| **권장도** | 필수 | 필수 | 선택 | 권장 |

---

## ⚖️ 트레이드오프

### 장점 ✅
- **장애 격리**: 한 서비스의 장애가 전체로 전파되지 않음
- **자동 복구**: Circuit Breaker 자동으로 HALF_OPEN 시도
- **데이터 일관성**: Saga의 보상 트랜잭션으로 분산 트랜잭션 관리
- **모니터링**: 순환 의존성, 데드락 자동 감지
- **우아한 저하**: 일부 기능만 제한하고 핵심 기능 유지

### 단점 ❌
- **복잡한 로직**: 보상 로직, Fallback 구현 복잡
- **데이터 불일치 가능**: Saga는 최종 일관성만 보장
- **모니터링 오버헤드**: Circuit Breaker, Bulkhead 메트릭 수집
- **리소스 낭비**: Bulkhead로 인한 메모리 2배 사용
- **디버깅 어려움**: 분산 트랜잭션 실패 원인 파악 어려움

---

## 📌 핵심 정리

✅ **순환 의존성 해결**
- 이벤트 기반 아키텍처로 직접 호출 제거
- 서비스 A → Event → 서비스 B (느슨한 결합)

✅ **카스케이드 장애 방지**
- Circuit Breaker: 장애 서비스 호출 차단
- Timeout: 무한 대기 방지 (기본 5초)
- Bulkhead: 스레드 풀 분리 (20개 격리)
- Fallback: 캐시 또는 대체 로직

✅ **분산 트랜잭션 관리**
- Saga 패턴: 각 스텝마다 보상 로직 정의
- 실패 시 이전 스텝들을 역순으로 보상
- 최종 일관성(Eventual Consistency) 보장

✅ **자동 감지 및 복구**
- DFS로 순환 의존성 감지
- 10분 이상 실행되는 Saga 데드락으로 판단
- 자동 재시도 또는 수동 개입 알림

✅ **운영 체크리스트**
- ✓ Circuit Breaker 설정 (모든 외부 호출)
- ✓ Timeout 설정 (기본 5초, API별 조정)
- ✓ Bulkhead 설정 (격리 필요한 서비스)
- ✓ Saga 보상 로직 구현
- ✓ 순환 의존성 정기 검사 (CI/CD 파이프라인)
- ✓ 데드락 감지 알림 (30초 간격)

---

## 🤔 생각해볼 문제

**Q1.** OrderService가 PaymentService와 InventoryService를 동시에 호출한다. PaymentService가 장애나면, InventoryService도 함께 다운되는 이유는?

<details><summary>해설 보기</summary>
Circuit Breaker 없는 경우:

```
OrderService 스레드 풀: 50개

각 요청마다:
1. PaymentService 호출 (5초 timeout) ← 느려짐
2. InventoryService 호출 (1초 timeout)

초당 100개 요청:
- PaymentService: 100 req/s × 5s = 500개 스레드 필요 ← 50개만 있음!
- 50개 스레드 모두 blocked
- InventoryService도 호출 불가 (스레드 없음)
```

해결책: Bulkhead 적용
```
OrderService 스레드 풀: 50개
├─ PaymentService 풀: 20개 (격리)
├─ InventoryService 풀: 20개 (격리)
└─ 기타 작업: 10개

PaymentService 장애:
- PaymentService 20개 스레드 blocked
- InventoryService 20개 스레드는 계속 작동
- 부분 서비스 가능!
```
</details>

**Q2.** Saga 패턴에서 보상 트랜잭션도 실패하면 어떻게 해야 할까?

<details><summary>해설 보기</summary>
보상 트랜잭션 실패는 데이터 불일치의 심각한 상황입니다.

예시:
```
Step 1: 주문 생성 ✓
Step 2: 결제 완료 ✓
Step 3: 재고 업데이트 ✗ (실패)

보상 시작:
보상 2: 환불 ✗ (결제 게이트웨이 다운!)
```

해결책:

1. **자동 재시도**
```java
@Retryable(maxAttempts = 5, backoff = @Backoff(delay = 1000))
public void compensatePayment(String transactionId) {
    paymentService.refund(transactionId);
}
```

2. **보상 스케줄링**
```java
// 실패한 보상을 배치 작업으로 예약
compensationScheduler.schedule(() -> {
    compensatePayment(transactionId);
}, "2025-04-10 03:00:00");  // 새벽 3시에 재시도
```

3. **관리자 알림 + 수동 개입**
```java
// 자동 복구 실패 → 운영팀 알림
if (compensationFailed) {
    alerting.sendAlert(AlertLevel.CRITICAL,
            "Manual intervention required: Saga " + sagaId + 
            " failed at step " + failedStep);
    
    // 관리자 대시보드에 표시
    manualInterventionQueue.add(new ManualTask(
            "Refund transaction " + transactionId,
            "Payment compensation failed automatically",
            sagaId));
}
```

결론: **보상 실패는 반드시 감지하고 알림**. 자동 복구 불가능하면 수동 개입 필요.
</details>

**Q3.** Canary 배포 중 실시간으로 에러율을 10%까지 올려도 괜찮을까? 즉시 롤백해야 할까?

<details><summary>해설 보기</summary>
**상황 판단:**

에러율 10% = 100개 요청 중 10개 실패

초당 1,000개 요청 기준:
- 초당 실패: 100개
- 분당 실패: 6,000개
- 시간당 실패: 360,000개

이는 엄청난 규모입니다!

**결정 기준:**

1. **에러 타입 확인**
```
- 연결 오류? → 인프라 문제 (즉시 롤백)
- 400 Bad Request? → 호환성 문제 (즉시 롤백)
- 5초 이상 timeout? → 성능 문제 (이미 조정 완료면 다음 단계)
- 0.1% 수준의 에러? → 허용 범위 (계속)
```

2. **SLO 확인**
```
목표: 에러율 < 0.5%
현재: 에러율 10%
→ SLO 위반 심각 → 즉시 롤백!
```

3. **사용자 영향도**
```
에러 유형:
- 주문 생성 실패? (핵심 기능) → 즉시 롤백
- 추천 상품 로드 실패? (부가 기능) → 계속 모니터링
```

**권장:**
```
에러율 5% 이상 → 즉시 롤백
이유: 
- 대부분의 SLO는 0.1% ~ 1% 수준
- 10%는 개발 버전 수준의 품질
- 롤백 후 원인 분석 후 재시도
```
</details>

---

<div align="center">

**[⬅️ 이전: 배포 전략 — Blue/Green, Canary, Feature Flag](./04-deployment-strategies.md)** | **[홈으로 🏠](../README.md)**

</div>

# 02. Choreography Saga — 이벤트 기반 자율 실행

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Choreography Saga에서 각 서비스는 어떻게 다음 단계를 실행하게 될까?
- 이벤트 체인 중간에 하나의 서비스가 실패했을 때 보상 트랜잭션은 어떻게 역방향으로 전파되는가?
- Choreography가 Orchestration보다 느슨한 결합을 제공하는 대신 무엇을 포기하는가?
- 메시지 브로커(Kafka, RabbitMQ)를 사용하는 Choreography 구현에서 멱등성을 어떻게 보장할 것인가?
- Saga 실패 시 보상 이벤트 체인이 정상적으로 완료되었는지 어떻게 검증할 것인가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

Choreography Saga는 MSA의 핵심 가치인 "느슨한 결합(Loose Coupling)"을 구현하는 가장 직관적인 패턴입니다. 각 서비스가 이벤트를 발행하고 다른 서비스가 관심 있는 이벤트를 구독함으로써, 서비스 간의 강제적인 의존성 없이 비즈니스 흐름을 이룹니다.

전통적인 요청-응답 방식(Request/Response)에서는 서비스 A가 서비스 B의 API를 호출할 때 B의 가용성에 직접 영향을 받습니다. 하지만 이벤트 기반 방식에서는 A가 이벤트를 발행하고 즉시 반환하므로, B의 처리 지연이나 실패가 A에 직접 영향을 주지 않습니다. 이는 높은 가용성과 확장성을 실현하는 핵심입니다.

그러나 이 느슨한 결합이라는 장점은 **흐름 추적의 어려움**이라는 대가가 따릅니다. 주문에서 배송까지의 전체 흐름이 이벤트 체인으로 분산되어 있으므로, 중간에 어디서 실패했는지 파악하기 어려워집니다. 이를 해결하기 위한 전술(Saga ID 추적, 모니터링, 분산 추적)이 필수적입니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ 잘못된 접근: 이벤트 발행은 하지만 멱등성 보장 없음
// 같은 이벤트가 여러 번 처리될 수 있음

@Service
public class OrderService {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Transactional
    public void createOrder(OrderRequest request) {
        // Step 1: 주문 생성 (DB에 커밋)
        Order order = new Order(
            request.getCustomerId(),
            request.getProductId(),
            request.getQuantity(),
            request.getAmount()
        );
        orderRepository.save(order);
        
        // Step 2: 이벤트 발행 (Kafka로 전송)
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getAmount()
        );
        kafkaTemplate.send("order-events", String.valueOf(order.getId()), event);
        // ❌ 문제 1: DB 커밋 후 이벤트 발행 사이에 장애 가능
        //           Order는 생성됐는데 이벤트는 발행 안 됨
        // ❌ 문제 2: 이벤트가 Kafka에 중복으로 들어갈 수 있음
        //           PaymentService가 같은 이벤트를 여러 번 처리
    }
}

@Service
public class PaymentService {
    
    @KafkaListener(topics = "order-events", groupId = "payment-group")
    public void handleOrderCreatedEvent(OrderCreatedEvent event) {
        // ❌ 문제: 이 메서드가 여러 번 실행될 수 있음
        // Kafka는 최소 한 번(At-Least-Once) 전달을 보장하지만, 
        // 정확히 한 번(Exactly-Once) 전달을 보장하지 않음
        
        // 예: 같은 OrderCreatedEvent가 두 번 도착
        Payment payment1 = new Payment(event.getOrderId(), event.getAmount());
        paymentRepository.save(payment1);  // 첫 번째 저장
        
        // ... 처리 중 장애 발생 ...
        
        // Kafka 재전송 → 같은 이벤트 다시 도착
        Payment payment2 = new Payment(event.getOrderId(), event.getAmount());
        paymentRepository.save(payment2);  // 두 번째 저장
        
        // 결과: 같은 고객이 같은 금액으로 두 번 결제됨!
        
        PaymentCompletedEvent resultEvent = new PaymentCompletedEvent(
            event.getOrderId(),
            payment1.getId(),  // 첫 번째 결제 ID
            "SUCCESS"
        );
        kafkaTemplate.send("payment-events", resultEvent);
    }
}

/*
실제 장애 흐름:
T1: OrderService가 Order 생성 (DB 커밋)
T2: OrderService가 OrderCreatedEvent 발행 (Kafka 전송 중)
T3: 네트워크 오류로 Kafka 전송 실패 → retry 로직 작동
T4: PaymentService가 이벤트 처리 시작 → Payment 생성
T5: PaymentService가 PaymentCompletedEvent 발행
T6: OrderService의 retry 로직이 다시 OrderCreatedEvent 발행
T7: PaymentService가 같은 이벤트를 다시 처리 → Payment 중복 생성!

결과: 고객의 카드에서 같은 금액이 여러 번 결제됨 (심각한 장애)
*/
```

```yaml
# ❌ 잘못된 설정: Kafka 설정에서 멱등성 보장 없음

spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: payment-group
      # ❌ auto-offset-reset: earliest  # 메시지 재처리 위험
      # ❌ enable-auto-commit: true     # 자동 커밋 (처리 실패해도 오프셋 증가)
    producer:
      # ❌ acks: 1  # 리더 복제본까지만 기다림 (일부 손실 가능)

# 결과: Saga 실패 시 복구 불가능한 상태 발생
```

---

## ✨ 올바른 접근 (After)

```java
// ✅ 올바른 접근: 멱등성 보장 + Outbox 패턴으로 안정적 이벤트 발행

@Service
public class OrderService {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private OrderEventOutboxRepository outboxRepository;
    
    /**
     * 주문 생성과 이벤트 발행을 원자적으로 처리
     * Outbox 패턴: 이벤트를 DB에 먼저 저장한 후
     *            별도의 Polling Service가 Kafka로 발행
     */
    @Transactional
    public Order createOrder(OrderRequest request) {
        // Step 1: 주문 생성
        Order order = new Order(
            request.getCustomerId(),
            request.getProductId(),
            request.getQuantity(),
            request.getAmount()
        );
        order = orderRepository.save(order);
        
        // Step 2: 이벤트를 DB에 저장 (Outbox 테이블)
        // 이 트랜잭션 내에서 Order와 Event가 같이 커밋됨
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getAmount()
        );
        OrderEventOutbox outbox = new OrderEventOutbox(
            "OrderCreatedEvent",
            event,
            order.getId(),
            false  // published = false
        );
        outboxRepository.save(outbox);
        
        // Step 3: 별도의 트랜잭션에서 Kafka로 발행하므로
        //         여기서는 DB에만 저장하고 반환
        
        return order;
    }
}

// Outbox 테이블 구조
@Entity
@Table(name = "order_event_outbox")
public class OrderEventOutbox {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String eventType;
    
    @Column(columnDefinition = "TEXT")
    private String eventPayload;  // JSON 직렬화된 이벤트
    
    private Long aggregateId;  // 주문 ID
    
    private Boolean published;
    
    private LocalDateTime createdAt;
    
    private LocalDateTime publishedAt;
}

// Outbox Polling Service: 주기적으로 미발행 이벤트를 Kafka로 발행
@Service
public class OrderEventPublisher {
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Autowired
    private OrderEventOutboxRepository outboxRepository;
    
    /**
     * 1분마다 실행: 미발행 이벤트를 Kafka로 발행
     * 실패하면 다음 실행 때 재시도됨 (재발행되더라도 멱등성 보장)
     */
    @Scheduled(fixedRate = 60000)
    public void publishOrderEvents() {
        List<OrderEventOutbox> unpublished = 
            outboxRepository.findByPublishedFalse();
        
        for (OrderEventOutbox outbox : unpublished) {
            try {
                // Kafka로 발행 (publishedAt 타임스탬프 포함으로 멱등성 보장)
                kafkaTemplate.send(
                    "order-events",
                    String.valueOf(outbox.getAggregateId()),
                    outbox.getEventPayload()
                );
                
                // DB에서 published = true로 업데이트
                outbox.setPublished(true);
                outbox.setPublishedAt(LocalDateTime.now());
                outboxRepository.save(outbox);
                
            } catch (Exception e) {
                // 실패해도 다음 주기에 재시도 (중복 발행이 멱등성으로 처리됨)
                log.warn("Failed to publish event: {}", outbox.getId(), e);
            }
        }
    }
}

/**
 * ✅ PaymentService: 멱등성 보장으로 중복 처리 방지
 */
@Service
public class PaymentService {
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Autowired
    private IdempotencyKeyRepository idempotencyKeyRepository;
    
    @KafkaListener(topics = "order-events", groupId = "payment-group")
    public void handleOrderCreatedEvent(OrderCreatedEvent event) {
        /**
         * 멱등성 보장: Idempotency Key 패턴
         * 같은 이벤트가 여러 번 들어와도 처음 한 번만 처리됨
         */
        String idempotencyKey = generateIdempotencyKey(
            "OrderCreatedEvent",
            event.getOrderId(),
            event.getEventTimestamp()
        );
        
        // Step 1: 이미 처리된 이벤트인지 확인
        if (idempotencyKeyRepository.existsByKey(idempotencyKey)) {
            log.info("Event already processed: {}", idempotencyKey);
            return;  // 이미 처리됐으므로 스킵
        }
        
        // Step 2: 새로운 이벤트 처리
        try {
            Payment payment = new Payment(
                event.getOrderId(),
                event.getAmount(),
                "PROCESSING"
            );
            paymentRepository.save(payment);
            
            // 결제 API 호출 (외부 PG사)
            PGResponse pgResponse = externalPaymentGateway.charge(
                event.getAmount(),
                event.getOrderId()
            );
            
            payment.setPgTransactionId(pgResponse.getTransactionId());
            payment.setStatus("COMPLETED");
            paymentRepository.save(payment);
            
            // Step 3: Idempotency Key 기록 (이 트랜잭션과 함께 커밋)
            IdempotencyKeyRecord keyRecord = new IdempotencyKeyRecord(
                idempotencyKey,
                "OrderCreatedEvent",
                event.getOrderId(),
                "SUCCESS"
            );
            idempotencyKeyRepository.save(keyRecord);
            
            // Step 4: 다음 단계 이벤트 발행
            PaymentCompletedEvent resultEvent = new PaymentCompletedEvent(
                event.getOrderId(),
                payment.getId(),
                "COMPLETED"
            );
            publishPaymentEvent(resultEvent);
            
        } catch (PaymentException e) {
            // 결제 실패 → 보상 이벤트 발행
            PaymentFailedEvent failureEvent = new PaymentFailedEvent(
                event.getOrderId(),
                e.getMessage()
            );
            publishPaymentEvent(failureEvent);
            
            throw new SagaCompensationException("Payment processing failed", e);
        }
    }
    
    private String generateIdempotencyKey(String eventType, Long orderId, LocalDateTime timestamp) {
        return String.format("%s:%d:%d", eventType, orderId, timestamp.getTime());
    }
    
    private void publishPaymentEvent(Object event) {
        // Outbox 패턴으로 발행
        // ...
    }
}

// Idempotency Key 저장소
@Entity
@Table(name = "idempotency_keys")
public class IdempotencyKeyRecord {
    @Id
    private String key;
    
    private String eventType;
    private Long aggregateId;
    private String result;
    private LocalDateTime createdAt;
    
    // TTL: 24시간 후 삭제 (storage 절약)
}
```

```yaml
# ✅ 올바른 설정: 멱등성 보장 + 신뢰할 수 있는 메시지 처리

spring:
  kafka:
    bootstrap-servers: kafka:9092
    consumer:
      group-id: payment-group
      # 수동 커밋으로 처리 후 커밋 보장
      enable-auto-commit: false
      # 초기 오프셋: 가장 최신부터 시작 (이미 처리한 것 재처리 방지)
      auto-offset-reset: latest
      # 한 번에 1개 메시지만 처리 (안정성)
      max-poll-records: 1
    producer:
      # 모든 복제본이 확인할 때까지 대기 (최고 신뢰성)
      acks: all
      # 멱등성 활성화: Kafka가 자동으로 중복 제거
      enable-idempotence: true
      # 재시도 설정
      retries: 3
      retry-backoff-ms: 100
    
  jpa:
    hibernate:
      ddl-auto: validate

# Outbox 주기적 발행 설정
app:
  outbox:
    polling-interval-ms: 60000  # 1분마다
    batch-size: 100             # 한 번에 100개씩
```

---

## 🔬 내부 동작 원리 — Choreography 이벤트 흐름 심층 분석

### 정상 흐름: 주문 → 결제 → 재고 → 배송

```
┌────────────────────────────────────────────────────────────────────┐
│           Choreography Saga: 정상 완료 시나리오                   │
└────────────────────────────────────────────────────────────────────┘

시각   OrderService        PaymentService      InventoryService  ShippingService
────────────────────────────────────────────────────────────────────────────────

T1     [주문 생성]
       ┌─ Order INSERT
       └─ OrderCreatedEvent → Outbox
       
T2     OrderCreatedEvent 발행 (Kafka)
       ───────────────────────────────→
       
T3                         [이벤트 수신]
                           ┌─ Payment INSERT
                           └─ PaymentCompletedEvent → Outbox
                           
T4     (결제 완료 이벤트 기다리는 중)
       
       PaymentCompletedEvent 발행 (Kafka)
       ────────────────────────────────────────────→
       
T5                                         [이벤트 수신]
                                           ┌─ Stock UPDATE
                                           └─ InventoryReservedEvent → Outbox
                                           
T6     (재고 예약 이벤트 기다리는 중)
       
       InventoryReservedEvent 발행 (Kafka)
       ───────────────────────────────────────────────────→
       
T7                                                          [이벤트 수신]
                                                            ┌─ Shipment INSERT
                                                            └─ ShippingCreatedEvent → Outbox

T8     [Saga 완료]
       Order.status = "COMPLETED"
       
────────────────────────────────────────────────────────────────────────────────

✅ 특징:
- 각 서비스가 독립적으로 실행됨 (비동기)
- 이벤트 체인으로 느슨한 결합 유지
- 전체 처리 시간 = T1 + T3 + T5 + T7 (모두 병렬 처리 아님, 순차적)
- 각 서비스는 자신의 DB 트랜잭션만 관리
```

### 장애 시나리오: 재고 부족 시 보상 흐름

```
┌────────────────────────────────────────────────────────────────────┐
│          Choreography Saga: 보상 (Compensation) 시나리오          │
└────────────────────────────────────────────────────────────────────┘

시각   OrderService        PaymentService      InventoryService  ShippingService
────────────────────────────────────────────────────────────────────────────────

T1-T4  [정상 흐름: 주문 생성 → 결제 완료]
       Order.status = "PENDING"
       Payment.status = "COMPLETED"

T5     InventoryReservedEvent 수신 중...
       
                                           ❌ 재고 부족!
                                           ┌─ InventoryOutOfStockEvent → Outbox
                                           
T6     InventoryOutOfStockEvent 발행 (역방향 보상 시작)
       
       [보상 시작]
       ← InventoryOutOfStockEvent 수신
       ┌─ Order.status = "CANCELLED"
       └─ OrderCancelledEvent → Outbox

T7     PaymentService로 환불 요청
       OrderCancelledEvent 발행 (Kafka)
       ────────────────────────────────→
       
                                           ← OrderCancelledEvent 수신
                                           ┌─ Payment.status = "REFUNDED"
                                           └─ PaymentRefundedEvent → Outbox

T8     PaymentRefundedEvent 발행 (Kafka)
       ← PaymentRefundedEvent 수신
       ┌─ Order.status = "REFUND_COMPLETED"

T9     [보상 완료]
       전체 Saga 롤백 → 마치 트랜잭션이 시작되지 않은 것처럼
       
       최종 상태:
       - Order: CANCELLED (또는 REFUND_COMPLETED)
       - Payment: REFUNDED
       - Inventory: 원래 상태 (변경 안 됨)
       - Shipping: 생성 안 됨

────────────────────────────────────────────────────────────────────────────────

❌ 문제점:
T1-T4 사이의 중간 상태가 외부에 노출됨
- T2에서 T4 사이에 고객이 주문을 조회하면 "PENDING" 상태
- T5에서 보상 시작되면 상태가 "CANCELLED"로 변경됨
- 고객 UI에서 "잠깐 주문됨 → 갑자기 취소됨" 표시

✅ 해결책:
- Saga ID로 추적 가능하게 설계
- 중간 상태를 Order.sagaStatus로 명시 (예: "AWAITING_INVENTORY")
- 모니터링 대시보드에서 "처리 중" 상태 표시
```

### 이벤트 체인 흐름도

```
                    ┌──────────────────────────────────────────┐
                    │        Choreography Saga Flow            │
                    └──────────────────────────────────────────┘

                              [OrderService]
                                   │
                                   ▼
                          ┌────────────────┐
                          │  Order 생성    │
                          │ (로컬 TX)      │
                          └────────┬───────┘
                                   │
                                   ▼ OrderCreatedEvent
                          ┌──────────────────────┐
                          │   Message Broker     │
                          │   (Kafka Topic:      │
                          │    order-events)     │
                          └──────────┬───────────┘
                                     │
                  ┌──────────────────┼──────────────────┐
                  │                  │                  │
                  ▼                  ▼                  ▼
           [PaymentService]   [InventoryService]  [Other Services]
                  │                  │                  │
                  ▼                  ▼                  ▼
          (구독하지 않음)   (구독하지 않음)    (구독 가능)
          (리스닝 안 함)    (리스닝 안 함)
          
                  ▼
         ┌─────────────────┐
         │ Payment 처리    │
         │ (로컬 TX)       │
         └────────┬────────┘
                  │
                  ▼ PaymentCompletedEvent
         ┌──────────────────────┐
         │   Message Broker     │
         │   (Kafka Topic:      │
         │    payment-events)   │
         └──────────┬───────────┘
                    │
         ┌──────────┴──────────┐
         │                     │
         ▼                     ▼
    [InventoryService]   [ShippingService]
         │                     │
         ▼                     ▼
    Stock 감소           (리스닝 안 함)
    (로컬 TX)
         │
         ▼ InventoryReservedEvent
    ┌──────────────────────┐
    │   Message Broker     │
    │   (Kafka Topic:      │
    │    inventory-events) │
    └──────────┬───────────┘
               │
               ▼
        [ShippingService]
               │
               ▼
        Shipment 생성
        (로컬 TX)

════════════════════════════════════════════════════════════════

보상 흐름 (InventoryOutOfStock 발생):

        InventoryOutOfStockEvent
         ┌──────────────────────────┐
         │   Message Broker         │
         │   (보상용 Topic)         │
         └──────────┬───────────────┘
                    │
         ┌──────────┴──────────┐
         │                     │
         ▼                     ▼
    [OrderService]      [PaymentService]
         │                     │
         ▼                     ▼
    Order 취소          Payment 환불
    (보상 TX)           (보상 TX)
```

---

## 💻 Choreography 구현: 실무 코드 예제

### 이벤트 정의

```java
// 이벤트는 불변(Immutable)하게 정의
@Value
@AllArgsConstructor
public class OrderCreatedEvent {
    private String eventId;
    private String sagaId;
    private Long orderId;
    private Long customerId;
    private BigDecimal amount;
    private LocalDateTime createdAt;
    
    // 이벤트 생성 팩토리
    public static OrderCreatedEvent of(Order order, String sagaId) {
        return new OrderCreatedEvent(
            UUID.randomUUID().toString(),
            sagaId,
            order.getId(),
            order.getCustomerId(),
            order.getAmount(),
            LocalDateTime.now()
        );
    }
}

@Value
@AllArgsConstructor
public class PaymentCompletedEvent {
    private String eventId;
    private String sagaId;
    private Long paymentId;
    private Long orderId;
    private String status;  // COMPLETED, REFUNDED
    private LocalDateTime completedAt;
}

@Value
@AllArgsConstructor
public class InventoryReservedEvent {
    private String eventId;
    private String sagaId;
    private Long reservationId;
    private Long productId;
    private Integer quantity;
    private LocalDateTime reservedAt;
}

// 보상 이벤트들
@Value
@AllArgsConstructor
public class PaymentRefundedEvent {
    private String eventId;
    private String sagaId;
    private Long paymentId;
    private String refundReason;
    private LocalDateTime refundedAt;
}

@Value
@AllArgsConstructor
public class OrderCancelledEvent {
    private String eventId;
    private String sagaId;
    private Long orderId;
    private String cancellationReason;
    private LocalDateTime cancelledAt;
}
```

### 이벤트 리스너 구현 (InventoryService)

```java
@Service
@Slf4j
public class InventoryEventListener {
    
    @Autowired
    private InventoryRepository inventoryRepository;
    
    @Autowired
    private ReservationRepository reservationRepository;
    
    @Autowired
    private SagaLogRepository sagaLogRepository;
    
    /**
     * PaymentCompletedEvent 처리
     * 재고 예약 시도 → 실패하면 보상 이벤트 발행
     */
    @KafkaListener(
        topics = "payment-events",
        groupId = "inventory-group",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handlePaymentCompletedEvent(PaymentCompletedEvent event) {
        String sagaId = event.getSagaId();
        
        try {
            log.info("[Saga:{}] PaymentCompleted event received", sagaId);
            
            // Step 1: Saga 로그 기록
            recordSagaStep(sagaId, "PAYMENT_COMPLETED_RECEIVED", "SUCCESS");
            
            // Step 2: 재고 조회 및 예약
            processInventoryReservation(event, sagaId);
            
        } catch (InsufficientInventoryException e) {
            // 재고 부족 → 보상 트랜잭션 발행
            handleInventoryFailure(event, sagaId, e);
        } catch (Exception e) {
            log.error("[Saga:{}] Unexpected error", sagaId, e);
            recordSagaStep(sagaId, "INVENTORY_ERROR", "FAILED");
            throw new SagaException("Inventory processing failed", e);
        }
    }
    
    @Transactional
    private void processInventoryReservation(
        PaymentCompletedEvent event,
        String sagaId
    ) throws InsufficientInventoryException {
        
        // DB에서 상품별 재고 조회
        Inventory inventory = inventoryRepository
            .findByProductId(event.getOrderId())
            .orElseThrow(() -> new EntityNotFoundException("Product not found"));
        
        // 충분한 재고 확인 (비즈니스 로직)
        if (inventory.getAvailableQuantity() < 1) {
            throw new InsufficientInventoryException(
                "Insufficient inventory for product: " + event.getOrderId()
            );
        }
        
        // 재고 예약
        Reservation reservation = new Reservation(
            event.getOrderId(),
            event.getOrderId(),
            1,
            sagaId
        );
        reservationRepository.save(reservation);
        
        // 재고 감소
        inventory.decreaseAvailable(1);
        inventoryRepository.save(inventory);
        
        // Saga 로그 기록
        recordSagaStep(sagaId, "INVENTORY_RESERVED", "SUCCESS");
        
        // ✅ 이벤트는 Outbox 패턴으로 발행 (별도 스레드)
        // 이 메서드의 트랜잭션과 함께 커밋됨
        publishInventoryReservedEvent(event, reservation, sagaId);
    }
    
    @Transactional
    private void handleInventoryFailure(
        PaymentCompletedEvent event,
        String sagaId,
        Exception cause
    ) {
        // 보상 이벤트 발행: 결제 환불 요청
        log.warn("[Saga:{}] Inventory insufficient, initiating compensation", sagaId);
        
        // Saga 로그: 실패 기록
        recordSagaStep(sagaId, "INVENTORY_RESERVED_FAILED", "FAILED");
        recordSagaStep(sagaId, "COMPENSATION_STARTED", "COMPENSATION");
        
        // 보상 이벤트를 Outbox에 저장
        InventoryCompensationEvent compensationEvent = 
            new InventoryCompensationEvent(
                UUID.randomUUID().toString(),
                sagaId,
                event.getOrderId(),
                event.getOrderId(),
                "Insufficient inventory: " + cause.getMessage(),
                LocalDateTime.now()
            );
        
        publishCompensationEvent(compensationEvent);
    }
    
    private void publishInventoryReservedEvent(
        PaymentCompletedEvent event,
        Reservation reservation,
        String sagaId
    ) {
        InventoryReservedEvent outboundEvent = new InventoryReservedEvent(
            UUID.randomUUID().toString(),
            sagaId,
            reservation.getId(),
            event.getOrderId(),
            1,
            LocalDateTime.now()
        );
        
        // Outbox 테이블에 저장 (이 트랜잭션과 함께 커밋)
        InventoryEventOutbox outbox = new InventoryEventOutbox(
            "InventoryReservedEvent",
            outboundEvent,
            event.getOrderId(),
            false
        );
        // outboxRepository.save(outbox);
    }
    
    private void recordSagaStep(String sagaId, String step, String status) {
        SagaLog log = new SagaLog(sagaId, step);
        log.setStatus(status);
        log.setService("InventoryService");
        log.setTimestamp(LocalDateTime.now());
        sagaLogRepository.save(log);
    }
}
```

---

## 📊 패턴 비교

| 항목 | 요청-응답 (Request/Response) | 이벤트 기반 (Choreography) |
|------|------|------|
| **결합도** | 강결합 (A가 B의 API 호출) | 느슨한 결합 (이벤트로만 통신) |
| **처리 방식** | 동기 | 비동기 |
| **흐름 제어** | 발신자가 제어 | 구독자가 결정 |
| **순서 보장** | 강함 (요청→응답) | 약함 (이벤트 순서 의존) |
| **장애 영향** | 즉시 전파 (B 장애 → A 영향) | 지연 처리 (B 재시작 시 처리) |
| **추적 난이도** | 쉬움 (단순 호출 체인) | 어려움 (분산 이벤트) |
| **확장성** | 낮음 (강결합) | 높음 (느슨한 결합) |
| **구현 복잡도** | 낮음 | 높음 (멱등성, Outbox 등) |

---

## ⚖️ 트레이드오프

### Choreography 선택 시
- ✅ **느슨한 결합**: 새 서비스 추가가 기존 코드에 영향 없음
- ✅ **높은 가용성**: 일부 서비스 지연이 다른 부분에 영향 없음
- ✅ **확장성**: 이벤트 구독자를 동적으로 추가 가능
- ✅ **복원력**: 메시지 브로커가 메시지 보관하므로 재처리 가능
- ❌ **복잡한 흐름 추적**: Saga 전체 흐름을 파악하기 어려움
- ❌ **순환 참조 위험**: 이벤트 체인이 순환할 수 있음 (무한 루프)
- ❌ **멱등성 관리**: 모든 이벤트 핸들러가 멱등성을 보장해야 함
- ❌ **디버깅 어려움**: 분산 환경에서 장애 원인 파악 복잡
- ❌ **사가 완료 검증**: 보상이 정상 완료되었는지 확인 필요

---

## 📌 핵심 정리

✅ **Choreography Saga**는 각 서비스가 이벤트에 반응하여 자율적으로 실행하는 패턴입니다. 느슨한 결합과 높은 가용성을 제공합니다.

✅ **이벤트 체인**은 OrderCreatedEvent → PaymentCompletedEvent → InventoryReservedEvent → ShippingCreatedEvent 순서로 자동으로 진행됩니다.

✅ **보상 흐름**은 역방향으로 실행됩니다. InventoryOutOfStock → PaymentRefunded → OrderCancelled 순서로 보상 트랜잭션이 실행됩니다.

✅ **멱등성 보장**은 필수입니다. Idempotency Key 패턴으로 같은 이벤트의 중복 처리를 방지합니다.

✅ **Outbox 패턴**으로 DB 저장과 이벤트 발행을 원자적으로 처리합니다. 이를 통해 "DB는 저장되었는데 이벤트는 발행 안 된" 상황을 방지합니다.

✅ **트레이드오프**: 느슨한 결합의 대가로 흐름 추적이 어려워지며, Saga ID 기반 모니터링이 필수입니다.

---

## 🤔 생각해볼 문제

**Q1.** Choreography에서 이벤트가 "정확히 한 번(Exactly-Once)" 처리되지 않는다면, 어떻게 멱등성을 보장할 수 있을까요? 멱등성이 없으면 어떤 문제가 발생하나요?

<details><summary>해설 보기</summary>
**멱등성 없을 시 문제**:
- Kafka가 PaymentCompletedEvent를 두 번 전송 (네트워크 오류로 재전송)
- InventoryService가 같은 이벤트를 두 번 처리
- 재고가 두 번 감소됨 → 고객이 주문하지 않은 수량까지 재고 손실

**해결책: Idempotency Key 패턴**
```java
// Event 처리 전에 이미 처리됐는지 확인
String key = "PaymentCompleted:" + event.getPaymentId() + ":" + event.getSagaId();
if (idempotencyRepository.exists(key)) {
    return;  // 이미 처리된 이벤트, 스킵
}

// 처리 수행
processPayment(event);

// 처리 완료 기록
idempotencyRepository.save(key);
```

이를 통해 같은 이벤트가 여러 번 도착해도 처음 한 번만 처리됩니다.
</details>

**Q2.** 보상 이벤트가 다시 장애를 만나면 어떻게 될까요? 예: PaymentRefundedEvent를 발행했는데, InventoryService가 오프라인이라 메시지를 수신하지 못했다면?

<details><summary>해설 보기</summary>
**시나리오**:
1. InventoryOutOfStock 발생 → 보상 시작
2. PaymentService에게 "환불하세요" 이벤트 발행
3. InventoryService가 오프라인 (메시지 브로커에 메시지만 남음)
4. InventoryService가 복구되면 → 메시지 수신 → 환불 처리

**Choreography의 장점**:
- 메시지 브로커(Kafka)가 메시지를 계속 보관
- InventoryService 복구 후 자동으로 메시지 처리
- 추가 코드 없이 자동 재시도

**하지만 문제**:
- 얼마나 오래 보류될지 불명확 (타임아웃 설정 필요)
- "보상이 정상 완료됐는가?"를 확인할 방법이 없음
- Saga 전체가 "PENDING" 상태로 계속 유지

**해결책**:
- Dead Saga Detector: 일정 시간 미완료된 Saga 감지
- Saga 타임아웃: 예: "1시간 안에 완료 안 되면 수동 개입"
- 모니터링: Saga 상태 대시보드로 "현재 진행 중인 Saga" 조회
</details>

**Q3.** 만약 보상 이벤트가 원래 이벤트보다 늦게 도착하면 어떻게 될까요? 시간 역순으로 이벤트가 처리되는 경우를 생각해보세요.

<details><summary>해설 보기</summary>
**시나리오**:
- T1: OrderCreatedEvent 발행
- T2: PaymentCompletedEvent 발행
- T3: InventoryOutOfStockEvent 발행 (보상 시작)
- T4: PaymentRefundedEvent 발행
- 하지만 메시지 브로커의 지연으로:
  - 실제 도착 순서: T1 → T4 → T2 → T3

**결과**: Saga 순서가 뒤바뀜 → 상태 불일치

**해결책**:
1. **Saga ID로 순서 보장**: 같은 Saga ID는 순서대로 처리
   ```yaml
   # Kafka에서 같은 key는 같은 파티션으로
   key: ${sagaId}  # 같은 Saga는 같은 파티션
   ```

2. **타임스탬프 검증**: 이벤트 수신 시 원본 시간 확인
   ```java
   if (event.getCreatedAt().isAfter(lastProcessedEventTime)) {
       processEvent(event);
   } else {
       // 이전 이벤트 → 무시 (이미 처리됨)
       return;
   }
   ```

3. **Saga 상태 머신**: 상태별로 받을 수 있는 이벤트 명시
   ```
   PENDING → PaymentCompleted (O)
   PENDING → PaymentRefunded (X) // 아직 결제 안 됨
   ```

이를 통해 이벤트 순서 보장 없이도 논리적 일관성을 유지합니다.
</details>

---

<div align="center">

**[⬅️ 이전: 분산 트랜잭션의 문제 — 2PC와 Saga](./01-distributed-transaction-problem.md)** | **[홈으로 🏠](../README.md)** | **[다음: Orchestration Saga — 중앙 오케스트레이터 설계 ➡️](./03-orchestration-saga.md)**

</div>

# 03. Orchestration Saga — 중앙 오케스트레이터 설계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Orchestrator가 각 서비스를 어떻게 제어하고, 상태를 어떻게 관리하는가?
- 상태 머신(State Machine)으로 Saga 진행 상황을 어떻게 추적할 것인가?
- Orchestrator 자신이 장애를 만나면 어떻게 복구할 것인가?
- Command 패턴과 Event 패턴의 차이는 무엇이고, Orchestration에서는 어느 것을 사용하는가?
- 복잡한 분기 로직(조건부 실행, 병렬 단계)을 어떻게 표현할 것인가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

Choreography 패턴에서는 각 서비스가 이벤트에 반응하여 자율적으로 실행되므로, 전체 흐름을 한눈에 파악하기 어렵습니다. 특히 Saga 진행이 막혔을 때 "지금 어느 단계인가?", "왜 진행이 안 되는가?"를 파악하는 데 막대한 시간이 걸립니다.

Orchestration 패턴은 이러한 문제를 해결하기 위해 중앙의 **Saga Orchestrator**가 명시적으로 각 서비스에 "무엇을 하라"는 명령(Command)을 보내고, 응답을 받아 다음 단계를 결정합니다. 이를 통해:

1. **명확한 흐름 제어**: Orchestrator가 전체 로직을 한 곳에서 관리
2. **장애 추적 용이**: "주문 생성 완료 → 결제 대기 중 → 결제 성공" 각 단계를 명확히 기록
3. **복잡한 조건 처리**: 조건부 단계, 병렬 처리, 루프 등을 Orchestrator에서 처리
4. **빠른 디버깅**: 중앙 로그에서 전체 흐름을 추적

다만 이 명확함의 대가로 **강한 결합도**가 생깁니다. Orchestrator가 모든 서비스를 알고 있어야 하고, 새 서비스 추가 시 Orchestrator 코드 수정이 필수입니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ 잘못된 접근: Orchestrator가 없이 수동으로 순서 제어
// 강결합, 중복된 로직, 추적 불가능

@Service
public class OrderProcessingService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private PaymentServiceClient paymentClient;  // 강결합
    
    @Autowired
    private InventoryServiceClient inventoryClient;  // 강결합
    
    @Autowired
    private ShippingServiceClient shippingClient;  // 강결합
    
    public void processOrder(OrderRequest request) {
        // Step 1: 주문 생성
        Order order = new Order(request);
        orderRepository.save(order);
        
        // Step 2: 결제 호출 (동기)
        // ❌ 문제: PaymentService가 느리면 전체 블로킹
        PaymentResult payment = paymentClient.pay(
            order.getId(),
            request.getAmount()
        );
        
        if (!payment.isSuccess()) {
            // ❌ 문제: 어떻게 취소할 것인가?
            // rollbackOrder(order.getId());
            throw new PaymentException("Payment failed");
        }
        
        // Step 3: 재고 예약
        try {
            InventoryResult inventory = inventoryClient.reserve(
                request.getProductId(),
                request.getQuantity()
            );
            
            // Step 4: 배송 생성
            ShippingResult shipping = shippingClient.createShipment(
                order.getId()
            );
            
        } catch (InventoryException e) {
            // ❌ 문제: 결제는 됐는데 재고는 없음
            // 환불해야 하는데...?
            paymentClient.refund(payment.getId());  // 추가 호출 필요
            orderRepository.delete(order);
            throw e;
        }
    }
}

/*
❌ 문제점:
1. 서비스 간 강결합: OrderProcessing이 3개 클라이언트를 모두 알고 있음
2. 흐름 제어 코드 분산: 각 장소에서 조건문과 취소 로직이 섞임
3. 상태 추적 불가: "지금 어디까지 진행했는가?" 알 수 없음
4. 보상 로직 누락: 복잡한 실패 경우를 모두 처리하지 못함
5. 재사용 불가: 다른 곳에서 같은 Saga를 사용하려면 코드 복사
6. 테스트 어려움: 모든 클라이언트를 Mock해야 함
*/
```

```yaml
# ❌ 잘못된 구조: 상태 관리가 없음
# Saga가 어디까지 진행했는지 알 수 없음

spring:
  jpa:
    hibernate:
      ddl-auto: create-drop

# 문제: Order 테이블에는 status가 있지만,
#      "지금 어떤 단계인가?" (ORDER_CREATED? PAYMENT_PROCESSING? INVENTORY_RESERVED?)
#      이런 세부 상태를 추적할 방법이 없음
```

---

## ✨ 올바른 접근 (After)

```java
// ✅ 올바른 접근: Orchestrator가 명시적으로 Saga를 관리
// 각 단계를 상태 머신으로 추적, 실패 시 자동 복구

@Service
@Slf4j
public class OrderSagaOrchestrator {
    
    @Autowired
    private OrderSagaRepository sagaRepository;
    
    @Autowired
    private PaymentServiceClient paymentClient;
    
    @Autowired
    private InventoryServiceClient inventoryClient;
    
    @Autowired
    private ShippingServiceClient shippingClient;
    
    /**
     * Saga 시작: 주문 요청을 받아 전체 흐름을 제어
     */
    public String startOrderSaga(OrderRequest request) {
        String sagaId = UUID.randomUUID().toString();
        
        // Saga 생성 및 초기 상태로 시작
        OrderSaga saga = new OrderSaga(sagaId);
        saga.setState(SagaState.PENDING);
        saga.setData(request);
        sagaRepository.save(saga);
        
        // 비동기로 Saga 실행 (별도 스레드 또는 메시지 큐)
        executeSaga(sagaId);
        
        return sagaId;
    }
    
    /**
     * Saga 실행: 상태 머신에 따라 각 단계를 실행
     */
    public void executeSaga(String sagaId) {
        OrderSaga saga = sagaRepository.findById(sagaId)
            .orElseThrow(() -> new EntityNotFoundException("Saga not found"));
        
        try {
            switch (saga.getState()) {
                case PENDING:
                    executeOrderCreation(saga);
                    break;
                    
                case ORDER_CREATED:
                    executePayment(saga);
                    break;
                    
                case PAYMENT_COMPLETED:
                    executeInventoryReservation(saga);
                    break;
                    
                case INVENTORY_RESERVED:
                    executeShipmentCreation(saga);
                    break;
                    
                case SHIPMENT_CREATED:
                    saga.setState(SagaState.SAGA_COMPLETED);
                    sagaRepository.save(saga);
                    log.info("[Saga:{}] Order saga completed successfully", sagaId);
                    break;
                    
                case COMPENSATION_STARTED:
                    executeCompensation(saga);
                    break;
                    
                default:
                    log.warn("[Saga:{}] Unknown state: {}", sagaId, saga.getState());
            }
            
        } catch (Exception e) {
            log.error("[Saga:{}] Error during saga execution", sagaId, e);
            saga.setState(SagaState.COMPENSATION_STARTED);
            saga.setErrorMessage(e.getMessage());
            sagaRepository.save(saga);
            executeSaga(sagaId);  // 보상 시작
        }
    }
    
    /**
     * Step 1: 주문 생성 (로컬 트랜잭션)
     */
    @Transactional
    private void executeOrderCreation(OrderSaga saga) {
        OrderRequest request = saga.getDataAs(OrderRequest.class);
        
        Order order = new Order(
            request.getCustomerId(),
            request.getProductId(),
            request.getQuantity(),
            request.getAmount(),
            saga.getId()  // Saga ID 추적
        );
        orderRepository.save(order);
        
        saga.setState(SagaState.ORDER_CREATED);
        saga.setOrderId(order.getId());
        sagaRepository.save(saga);
        
        log.info("[Saga:{}] Order created: {}", saga.getId(), order.getId());
        
        // 다음 단계 실행
        executeSaga(saga.getId());
    }
    
    /**
     * Step 2: 결제 처리 (외부 서비스 호출)
     */
    private void executePayment(OrderSaga saga) {
        try {
            PaymentRequest request = PaymentRequest.builder()
                .orderId(saga.getOrderId())
                .amount(saga.getDataAs(OrderRequest.class).getAmount())
                .sagaId(saga.getId())
                .build();
            
            // Orchestrator가 PaymentService에 명령 전송
            PaymentResponse response = paymentClient.processPayment(request);
            
            saga.setState(SagaState.PAYMENT_COMPLETED);
            saga.setPaymentId(response.getPaymentId());
            sagaRepository.save(saga);
            
            log.info("[Saga:{}] Payment completed: {}", saga.getId(), response.getPaymentId());
            
            // 다음 단계로 진행
            executeSaga(saga.getId());
            
        } catch (PaymentException e) {
            // 결제 실패 → 보상 시작
            log.warn("[Saga:{}] Payment failed: {}", saga.getId(), e.getMessage());
            saga.setState(SagaState.COMPENSATION_STARTED);
            saga.setCompensationReason("PAYMENT_FAILED");
            sagaRepository.save(saga);
            executeSaga(saga.getId());
        }
    }
    
    /**
     * Step 3: 재고 예약 (외부 서비스 호출)
     */
    private void executeInventoryReservation(OrderSaga saga) {
        try {
            InventoryRequest request = InventoryRequest.builder()
                .orderId(saga.getOrderId())
                .productId(saga.getDataAs(OrderRequest.class).getProductId())
                .quantity(saga.getDataAs(OrderRequest.class).getQuantity())
                .sagaId(saga.getId())
                .build();
            
            InventoryResponse response = inventoryClient.reserveInventory(request);
            
            saga.setState(SagaState.INVENTORY_RESERVED);
            saga.setReservationId(response.getReservationId());
            sagaRepository.save(saga);
            
            log.info("[Saga:{}] Inventory reserved: {}", saga.getId(), response.getReservationId());
            
            executeSaga(saga.getId());
            
        } catch (InventoryException e) {
            // 재고 부족 → 보상 시작
            log.warn("[Saga:{}] Inventory reservation failed: {}", saga.getId(), e.getMessage());
            saga.setState(SagaState.COMPENSATION_STARTED);
            saga.setCompensationReason("INVENTORY_UNAVAILABLE");
            sagaRepository.save(saga);
            executeSaga(saga.getId());
        }
    }
    
    /**
     * Step 4: 배송 생성 (외부 서비스 호출)
     */
    private void executeShipmentCreation(OrderSaga saga) {
        try {
            ShippingRequest request = ShippingRequest.builder()
                .orderId(saga.getOrderId())
                .sagaId(saga.getId())
                .build();
            
            ShippingResponse response = shippingClient.createShipment(request);
            
            saga.setState(SagaState.SHIPMENT_CREATED);
            saga.setShipmentId(response.getShipmentId());
            sagaRepository.save(saga);
            
            log.info("[Saga:{}] Shipment created: {}", saga.getId(), response.getShipmentId());
            
            executeSaga(saga.getId());
            
        } catch (ShippingException e) {
            // 배송 생성 실패 → 보상 시작
            log.warn("[Saga:{}] Shipment creation failed: {}", saga.getId(), e.getMessage());
            saga.setState(SagaState.COMPENSATION_STARTED);
            saga.setCompensationReason("SHIPMENT_CREATION_FAILED");
            sagaRepository.save(saga);
            executeSaga(saga.getId());
        }
    }
    
    /**
     * 보상 Saga 실행: 진행된 단계를 역순으로 취소
     */
    private void executeCompensation(OrderSaga saga) {
        SagaState currentState = saga.getState();
        
        try {
            switch (currentState) {
                case COMPENSATION_STARTED:
                    // 어느 단계까지 진행했는지에 따라 다르게 보상
                    if (saga.getShipmentId() != null) {
                        cancelShipment(saga);
                    }
                    // fallthrough: 다음 보상 단계로
                    
                case SHIPMENT_CANCELLED:
                    if (saga.getReservationId() != null) {
                        releaseInventory(saga);
                    }
                    
                case INVENTORY_RELEASED:
                    if (saga.getPaymentId() != null) {
                        refundPayment(saga);
                    }
                    
                case PAYMENT_REFUNDED:
                    if (saga.getOrderId() != null) {
                        cancelOrder(saga);
                    }
                    
                case ORDER_CANCELLED:
                    saga.setState(SagaState.SAGA_FAILED);
                    sagaRepository.save(saga);
                    log.info("[Saga:{}] Compensation completed - Saga failed", saga.getId());
                    break;
            }
            
        } catch (Exception e) {
            log.error("[Saga:{}] Error during compensation", saga.getId(), e);
            saga.setErrorMessage(e.getMessage());
            saga.setState(SagaState.COMPENSATION_FAILED);
            sagaRepository.save(saga);
            // 수동 개입 필요 (관리자 알림)
        }
    }
    
    @Transactional
    private void cancelOrder(OrderSaga saga) {
        Order order = orderRepository.findById(saga.getOrderId())
            .orElseThrow();
        order.setStatus("CANCELLED");
        orderRepository.save(order);
        
        saga.setState(SagaState.ORDER_CANCELLED);
        sagaRepository.save(saga);
        
        log.info("[Saga:{}] Order cancelled", saga.getId());
        executeSaga(saga.getId());
    }
    
    private void refundPayment(OrderSaga saga) {
        try {
            RefundRequest request = RefundRequest.builder()
                .paymentId(saga.getPaymentId())
                .sagaId(saga.getId())
                .build();
            
            paymentClient.refundPayment(request);
            
            saga.setState(SagaState.PAYMENT_REFUNDED);
            sagaRepository.save(saga);
            
            log.info("[Saga:{}] Payment refunded", saga.getId());
            executeSaga(saga.getId());
            
        } catch (Exception e) {
            log.error("[Saga:{}] Refund failed", saga.getId(), e);
            throw new CompensationException("Unable to refund payment", e);
        }
    }
    
    private void releaseInventory(OrderSaga saga) {
        try {
            ReleaseRequest request = ReleaseRequest.builder()
                .reservationId(saga.getReservationId())
                .sagaId(saga.getId())
                .build();
            
            inventoryClient.releaseReservation(request);
            
            saga.setState(SagaState.INVENTORY_RELEASED);
            sagaRepository.save(saga);
            
            log.info("[Saga:{}] Inventory released", saga.getId());
            executeSaga(saga.getId());
            
        } catch (Exception e) {
            log.error("[Saga:{}] Release failed", saga.getId(), e);
            throw new CompensationException("Unable to release inventory", e);
        }
    }
    
    private void cancelShipment(OrderSaga saga) {
        try {
            CancelShipmentRequest request = CancelShipmentRequest.builder()
                .shipmentId(saga.getShipmentId())
                .sagaId(saga.getId())
                .build();
            
            shippingClient.cancelShipment(request);
            
            saga.setState(SagaState.SHIPMENT_CANCELLED);
            sagaRepository.save(saga);
            
            log.info("[Saga:{}] Shipment cancelled", saga.getId());
            executeSaga(saga.getId());
            
        } catch (Exception e) {
            log.error("[Saga:{}] Shipment cancellation failed", saga.getId(), e);
            throw new CompensationException("Unable to cancel shipment", e);
        }
    }
}

/**
 * Saga 상태 머신 정의
 */
public enum SagaState {
    PENDING,                    // Saga 시작
    ORDER_CREATED,              // Step 1 완료
    PAYMENT_COMPLETED,          // Step 2 완료
    INVENTORY_RESERVED,         // Step 3 완료
    SHIPMENT_CREATED,           // Step 4 완료
    SAGA_COMPLETED,             // Saga 성공
    
    COMPENSATION_STARTED,       // 보상 시작
    SHIPMENT_CANCELLED,         // 보상 Step 1
    INVENTORY_RELEASED,         // 보상 Step 2
    PAYMENT_REFUNDED,           // 보상 Step 3
    ORDER_CANCELLED,            // 보상 Step 4
    SAGA_FAILED,                // Saga 최종 실패
    COMPENSATION_FAILED         // 보상 자체 실패 (수동 개입 필요)
}

/**
 * Saga 엔티티: 상태와 진행 상황을 DB에 저장
 */
@Entity
@Table(name = "order_sagas")
public class OrderSaga {
    @Id
    private String id;
    
    @Enumerated(EnumType.STRING)
    private SagaState state;
    
    private Long orderId;
    private Long paymentId;
    private Long reservationId;
    private Long shipmentId;
    
    @Column(columnDefinition = "TEXT")
    private String data;  // JSON으로 직렬화된 OrderRequest
    
    private String compensationReason;
    private String errorMessage;
    
    private LocalDateTime startedAt;
    private LocalDateTime updatedAt;
    
    // ... 생성자, getter/setter
}
```

```yaml
# ✅ 올바른 설정: Saga 상태 추적을 위한 DB 설정

spring:
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true
  
  datasource:
    url: jdbc:postgresql://saga-db:5432/saga_db
    username: ${DB_USER}
    password: ${DB_PASSWORD}

# Saga 실행 설정
app:
  saga:
    execution:
      # 비동기 실행: ThreadPool 크기
      thread-pool-size: 20
      # Saga 타임아웃: 1시간
      timeout-minutes: 60
      # Dead Saga 감지: 10분 이상 PENDING이면 실패 처리
      dead-saga-check-minutes: 10
```

---

## 🔬 내부 동작 원리 — Orchestrator 상태 머신 심층 분석

### Saga 상태 머신 다이어그램

```
┌─────────────────────────────────────────────────────────────────┐
│                   Orchestrator 상태 머신                         │
└─────────────────────────────────────────────────────────────────┘

                           ┌──────────────┐
                           │   PENDING    │ ← Saga 시작
                           └──────┬───────┘
                                  │
                    executeOrderCreation()
                                  │
                                  ▼
                          ┌──────────────────┐
                          │  ORDER_CREATED   │
                          └────────┬─────────┘
                                   │
                     executePayment() → 성공
                                   │
                                   ▼
                         ┌───────────────────────┐
                         │ PAYMENT_COMPLETED    │
                         └──────────┬────────────┘
                                    │
              executeInventoryReservation() → 성공
                                    │
                                    ▼
                          ┌──────────────────────┐
                          │ INVENTORY_RESERVED  │
                          └──────────┬───────────┘
                                     │
                 executeShipmentCreation() → 성공
                                     │
                                     ▼
                          ┌──────────────────────┐
                          │ SHIPMENT_CREATED    │
                          └──────────┬───────────┘
                                     │
                                     ▼
                          ┌──────────────────────┐
                          │  SAGA_COMPLETED     │ ✅ 성공
                          └──────────────────────┘

════════════════════════════════════════════════════════════════════

                      ❌ 장애 발생 시 보상 흐름

어느 단계에서든 예외 발생
        ↓
    ┌──────────────────────────┐
    │ COMPENSATION_STARTED     │
    └────────────┬─────────────┘
                 │
    (현재 단계부터 역순 보상)
                 │
        cancelShipment() if needed
                 │
                 ▼
         ┌─────────────────┐
         │ SHIPMENT_CANCELLED
         └────────┬────────┘
                  │
         releaseInventory() if needed
                  │
                  ▼
         ┌──────────────────┐
         │ INVENTORY_RELEASED│
         └────────┬─────────┘
                  │
          refundPayment() if needed
                  │
                  ▼
         ┌──────────────────┐
         │ PAYMENT_REFUNDED │
         └────────┬─────────┘
                  │
           cancelOrder() if needed
                  │
                  ▼
         ┌──────────────────┐
         │ ORDER_CANCELLED  │
         └────────┬─────────┘
                  │
                  ▼
         ┌──────────────────┐
         │  SAGA_FAILED     │ ❌ 실패
         └──────────────────┘
```

### Orchestrator의 책임 범위

```
┌─────────────────────────────────────────────────────────────────┐
│                    Orchestrator가 관리하는 것                    │
└─────────────────────────────────────────────────────────────────┘

✅ Orchestrator의 책임:
┌──────────────────────────────────────┐
│ 1. 전체 흐름 제어                    │
│    - PENDING → ORDER_CREATED →      │
│      PAYMENT_COMPLETED → ...         │
│                                      │
│ 2. 각 단계 간의 순서 보장             │
│    - 결제 전에 주문이 생성되어야 함   │
│    - 배송 전에 재고가 예약되어야 함   │
│                                      │
│ 3. 장애 시 보상 결정                │
│    - 어디까지 진행했는가?            │
│    - 어떤 보상이 필요한가?           │
│    - 보상 순서는?                    │
│                                      │
│ 4. 상태 추적 및 저장                │
│    - 현재 상태는?                    │
│    - 언제 실패했나?                  │
│    - 어떤 서비스가 응답했나?         │
└──────────────────────────────────────┘

❌ Orchestrator의 책임이 아닌 것 (각 서비스의 책임):
┌──────────────────────────────────────┐
│ 1. 개별 서비스 로직 구현              │
│    - 주문 생성 (OrderService)        │
│    - 결제 처리 (PaymentService)      │
│    - 재고 예약 (InventoryService)    │
│                                      │
│ 2. 로컬 데이터 일관성                │
│    - 각 서비스의 DB 트랜잭션        │
│    - 로컬 유효성 검사                │
│                                      │
│ 3. 보상 로직 구현                    │
│    - 이미 한 일을 취소하는 방법      │
│    - 보상이 멱등성 보장              │
│    - 보상 실패 시 대응               │
└──────────────────────────────────────┘
```

### Orchestrator 장애 복구

```
┌──────────────────────────────────────────────────────────────────┐
│              Orchestrator 자신이 장애를 만났을 때               │
└──────────────────────────────────────────────────────────────────┘

시나리오: Orchestrator가 PAYMENT_COMPLETED 상태를 DB에 저장하다가 장애

T1: executePayment() 성공
    → paymentClient.processPayment() OK (PaymentService 결제 완료)

T2: saga.setState(SagaState.PAYMENT_COMPLETED)
    → sagaRepository.save(saga)
    → [네트워크 오류] DB 저장 실패

T3: Orchestrator 프로세스 재시작

T4: Dead Saga Detector 실행 (스케줄러)
    → "10분 이상 PENDING 상태인 Saga는?"
    → 해당 Saga 찾음
    → 지난 로그 확인: "ORDER_CREATED까지는 완료"
    
T5: Orchestrator가 Saga 복구 (resumeSaga())
    → 현재 상태 = ORDER_CREATED
    → 다음 단계 = PAYMENT_COMPLETED 목표로 executePayment() 다시 실행
    
T6: PaymentService에 같은 요청이 다시 들어옴
    → ✅ Idempotency Key로 이미 처리된 것으로 인식
    → 중복 결제 방지
    → 상태 업데이트: PAYMENT_COMPLETED
    
T7: Saga 계속 진행: executeInventoryReservation()

✅ 해결책:
1. DB에 Saga 상태를 저장 → 복구 가능
2. 서비스들이 멱등성 보장 → 중복 요청 안전
3. Dead Saga Detector → 자동 감지 및 복구
4. Saga 타임아웃 → 무한 대기 방지
```

---

## 💻 Spring State Machine으로 구현하기

```java
// Spring State Machine 라이브러리 사용
@Configuration
@EnableStateMachine
public class OrderSagaStateMachineConfiguration 
        extends EnumStateMachineConfigurerAdapter<SagaState, SagaEvent> {
    
    @Override
    public void configure(StateMachineStateConfigurer<SagaState, SagaEvent> states)
            throws Exception {
        states
            .withStates()
                .initial(SagaState.PENDING)
                .states(EnumSet.allOf(SagaState.class))
                .end(SagaState.SAGA_COMPLETED)
                .end(SagaState.SAGA_FAILED);
    }
    
    @Override
    public void configure(StateMachineTransitionConfigurer<SagaState, SagaEvent> transitions)
            throws Exception {
        transitions
            .withExternal()
                .source(SagaState.PENDING)
                .target(SagaState.ORDER_CREATED)
                .event(SagaEvent.ORDER_CREATED)
                .action(orderCreationAction())
            .and()
            .withExternal()
                .source(SagaState.ORDER_CREATED)
                .target(SagaState.PAYMENT_COMPLETED)
                .event(SagaEvent.PAYMENT_COMPLETED)
                .action(paymentAction())
            .and()
            .withExternal()
                .source(SagaState.ORDER_CREATED)
                .target(SagaState.COMPENSATION_STARTED)
                .event(SagaEvent.PAYMENT_FAILED)
                .action(startCompensationAction());
    }
    
    @Bean
    public Action<SagaState, SagaEvent> orderCreationAction() {
        return context -> {
            // Saga 데이터 추출
            SagaContext sagaContext = context.getMessageHeader("sagaContext");
            // 주문 생성 로직
        };
    }
    
    @Bean
    public Action<SagaState, SagaEvent> paymentAction() {
        return context -> {
            // 결제 처리 로직
        };
    }
    
    @Bean
    public Action<SagaState, SagaEvent> startCompensationAction() {
        return context -> {
            // 보상 시작
        };
    }
}
```

---

## 📊 패턴 비교

| 항목 | Choreography | Orchestration |
|------|------|------|
| **흐름 제어** | 이벤트 기반 (자율 실행) | 중앙 제어 (명령 기반) |
| **결합도** | 낮음 | 높음 |
| **추적 난이도** | 어려움 | 쉬움 |
| **장애 원인 파악** | 분산 로그 필요 | 중앙 로그 충분 |
| **상태 관리** | 서비스별 로컬 | 중앙 Orchestrator |
| **조건부 로직** | 이벤트 핸들러에 분산 | Orchestrator에 집중 |
| **병렬 처리** | 자연스러움 | 명시적 제어 필요 |
| **새 서비스 추가** | 쉬움 | Orchestrator 수정 필요 |
| **구현 복잡도** | 높음 (멱등성) | 중간 |
| **리소스 사용** | 효율적 (비동기) | 적을 수 있음 |

---

## ⚖️ 트레이드오프

### Orchestration 선택 시
- ✅ **명확한 흐름**: Orchestrator에서 전체 로직을 한눈에 봄
- ✅ **쉬운 추적**: "지금 어디인가?"를 즉시 파악 가능
- ✅ **조건부 처리**: 복잡한 분기, 루프, 병렬 처리 간편
- ✅ **빠른 디버깅**: 중앙 로그에서 장애 원인 파악
- ✅ **상태 머신**: 명확한 상태 정의로 논리 오류 감소
- ❌ **강한 결합**: 새 서비스 추가 시 Orchestrator 수정
- ❌ **Orchestrator SPOF**: Orchestrator 장애 → 전체 Saga 중단
- ❌ **확장성 제한**: 수십 개 서비스의 경우 관리 복잡
- ❌ **기술 스택 제약**: Orchestrator와 서비스가 같은 언어/프레임워크 필요할 수 있음

---

## 📌 핵심 정리

✅ **Orchestrator**는 중앙에서 명시적으로 각 서비스에 "무엇을 하라"는 Command를 보내고, 응답을 받아 상태 머신을 따라 다음 단계를 결정합니다.

✅ **상태 머신**은 Saga의 진행 상황을 명확히 표현합니다:
- PENDING → ORDER_CREATED → PAYMENT_COMPLETED → INVENTORY_RESERVED → SHIPMENT_CREATED → SAGA_COMPLETED

✅ **보상 흐름**은 정방향의 역순으로 진행됩니다:
- SHIPMENT_CREATED → INVENTORY_RESERVED → PAYMENT_COMPLETED → ORDER_CREATED 순으로 취소

✅ **DB 기반 상태 저장**으로 Orchestrator 장애 시에도 복구 가능합니다. Dead Saga Detector가 미완료 Saga를 감지하고 재실행합니다.

✅ **트레이드오프**: 명확한 흐름 제어와 쉬운 디버깅의 대가로 강한 결합도와 확장성 제한이 따릅니다.

---

## 🤔 생각해볼 문제

**Q1.** Orchestrator 자신이 장애를 만났을 때 (예: 상태를 DB에 저장하다가 오류), Saga는 어느 상태에 있게 될까요? 이를 어떻게 복구할 것인가?

<details><summary>해설 보기</summary>
**시나리오**:
- executePayment() 성공 → PaymentService의 결제 처리 완료
- saga.setState(PAYMENT_COMPLETED)
- sagaRepository.save(saga) 시 DB 연결 오류

**결과**:
- PaymentService DB: Payment 레코드 생성됨 (결제 처리 완료)
- Orchestrator DB: 여전히 ORDER_CREATED 상태 (상태 업데이트 못함)
- Saga는 "중간 상태"에 고착됨

**복구 방법**:
1. **Dead Saga Detector** (스케줄러)
   - 주기적으로 (예: 5분마다) "10분 이상 변경 안 된 Saga" 감지
   - ORDER_CREATED → PAYMENT_COMPLETED로 진행해야 하는데 안 됨 감지

2. **Resume Logic**
   - Saga를 마지막 성공한 상태부터 재개
   - executePayment() 다시 실행
   - ✅ Idempotency Key로 중복 결제 방지
   - 상태 업데이트: PAYMENT_COMPLETED

3. **Manual Intervention**
   - 자동 복구 실패 시 관리자에게 알림
   - 관리자가 수동으로 상태 업데이트 또는 Saga 취소

**핵심**: DB에 상태를 저장하므로, 프로세스 재시작 후에도 어디까지 진행했는지 알 수 있습니다.
</details>

**Q2.** Orchestration에서 병렬 처리를 하려면 어떻게 해야 할까요? 예: 결제 + 재고 예약을 동시에 진행하되, 둘 다 성공해야 다음 단계로 진행

<details><summary>해설 보기</summary>
**시나리오**: 
- PAYMENT_COMPLETED와 INVENTORY_RESERVED를 동시에 실행 (순서 상관없음)
- 둘 다 완료 → BOTH_COMPLETED 상태로 전이
- 하나라도 실패 → COMPENSATION_STARTED

**구현 방법**:

```java
@Service
public class ParallelSagaOrchestrator {
    
    private final ExecutorService executor = Executors.newFixedThreadPool(4);
    
    private void executeParallelSteps(OrderSaga saga) {
        List<Future<Boolean>> futures = new ArrayList<>();
        
        // Step 1: 결제 (병렬 실행)
        futures.add(executor.submit(() -> executePayment(saga)));
        
        // Step 2: 재고 예약 (병렬 실행)
        futures.add(executor.submit(() -> executeInventoryReservation(saga)));
        
        // 둘 다 완료될 때까지 대기
        try {
            boolean allSucceeded = futures.stream()
                .allMatch(future -> {
                    try {
                        return future.get(30, TimeUnit.SECONDS);
                    } catch (Exception e) {
                        return false;
                    }
                });
            
            if (allSucceeded) {
                saga.setState(SagaState.PAYMENT_AND_INVENTORY_COMPLETED);
            } else {
                saga.setState(SagaState.COMPENSATION_STARTED);
            }
            
        } catch (Exception e) {
            saga.setState(SagaState.COMPENSATION_STARTED);
        }
        
        sagaRepository.save(saga);
        executeSaga(saga.getId());
    }
}
```

**상태 머신 확장**:
```
PAYMENT_COMPLETED ─┐
                   ├─→ PAYMENT_AND_INVENTORY_COMPLETED
INVENTORY_RESERVED ┘
                   ├─→ COMPENSATION_STARTED (하나라도 실패)
```

**Spring State Machine으로 표현**:
```java
transitions
    .withFork()  // 병렬 분기
        .source(SagaState.ORDER_CREATED)
        .target(SagaState.PAYMENT_PROCESSING)
        .target(SagaState.INVENTORY_PROCESSING)
    .and()
    .withJoin()  // 병렬 합치기
        .source(SagaState.PAYMENT_COMPLETED)
        .source(SagaState.INVENTORY_RESERVED)
        .target(SagaState.BOTH_COMPLETED);
```
</details>

**Q3.** Choreography와 Orchestration을 섞어 쓸 수 있을까요? 각각의 장점을 조합하려면?

<details><summary>해설 보기</summary>
**예**: 하이브리드 Saga 패턴

```
주문 생성 → [Orchestrator 제어 구간]
            ├─ 결제 처리
            ├─ 재고 예약
            └─ Shipment 생성 (PaymentCompletedEvent 발행)
                    │
                    └─→ [Choreography 구간]
                        ├─ NotificationService (주문 확인 이메일)
                        ├─ AnalyticsService (주문 통계)
                        └─ RecommendationService (추천 상품)
```

**구성**:
1. **핵심 비즈니스 로직** (Orchestration)
   - 주문 → 결제 → 재고 → 배송
   - 순서가 중요, 실패 시 보상 필수
   - 중앙 Orchestrator로 명시적 제어

2. **주변 서비스** (Choreography)
   - 알림, 분석, 추천 등
   - 순서가 중요하지 않음
   - 실패해도 주문 처리에는 영향 없음
   - 이벤트로 느슨하게 연결

**장점**:
- 핵심 비즈니스는 추적 가능 (Orchestration)
- 주변 기능은 확장성 있음 (Choreography)
- 각 부분에 최적의 패턴 적용

**구현**:
```java
private void executePayment(OrderSaga saga) {
    PaymentResponse response = paymentClient.processPayment(...);
    saga.setState(PAYMENT_COMPLETED);
    sagaRepository.save(saga);
    
    // ✅ PaymentCompletedEvent 발행
    // → NotificationService, AnalyticsService가 구독
    // → 이들은 독립적으로 실행 (Choreography)
    publishPaymentCompletedEvent(response);
}
```
</details>

---

<div align="center">

**[⬅️ 이전: Choreography Saga — 이벤트 기반 자율 실행](./02-choreography-saga.md)** | **[홈으로 🏠](../README.md)** | **[다음: Choreography vs Orchestration — 선택 기준 ➡️](./04-choreography-vs-orchestration.md)**

</div>

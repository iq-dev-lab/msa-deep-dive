# 05. 보상 트랜잭션 설계 — 멱등성과 취소 불가 단계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 보상 트랜잭션(Compensating Transaction)이 ACID 롤백과 다른 이유는?
- 어떤 비즈니스 작업은 취소할 수 없을까? 그렇다면 어떻게 대처할 것인가?
- 멱등성(Idempotency)을 보장하지 않으면 어떤 장애가 발생하는가?
- Idempotency Key 패턴으로 중복 처리를 어떻게 방지하는가?
- 보상 트랜잭션 자체가 실패했을 때 수동 개입은 어떻게 해야 하는가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

Saga 패턴의 핵심은 "분산된 여러 서비스의 작업을 조율하면서, 실패 시에는 일관성 있게 취소하는 것"입니다. 하지만 **하나의 로컬 트랜잭션을 롤백하는 것과는 다릅니다**. 

단일 DB의 트랜잭션에서는 ROLLBACK 명령어 하나로 모든 변경을 원래 상태로 돌릴 수 있습니다. 하지만 분산 시스템에서는:

1. **이미 외부 시스템에 영향을 미친 경우**: 결제가 이미 외부 PG사에 전송됐다면, "돌리기"가 아니라 "환불하기"로 보상해야 합니다.
2. **취소할 수 없는 작업**: 이메일을 이미 보냈다면? 고객에게 더 이상 어떻게 할 수 없습니다.
3. **부분적 성공**: 결제는 됐는데 재고 예약이 실패했다면, 결제는 "부분적으로" 성공한 상태입니다.

따라서 각 단계에 대해 "이것이 실패했을 때, 이미 진행된 것을 어떻게 취소할 것인가?"를 **미리 설계**해야 합니다. 이것이 **보상 트랜잭션 설계**의 핵심입니다.

또한 분산 환경의 네트워크 불안정성 때문에 **같은 명령이 여러 번 도착**할 수 있으므로, 멱등성 보장이 필수입니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ 잘못된 접근 1: 보상 트랜잭션 설계 없이 단순히 "역순 실행"
// 취소 불가능한 작업이 있거나, 순서가 맞지 않을 수 있음

@Service
public class NaiveCompensationSaga {
    
    @Transactional
    public void processOrder(OrderRequest request) throws Exception {
        String orderId = UUID.randomUUID().toString();
        
        try {
            // Step 1: 주문 생성
            Order order = createOrder(request, orderId);
            log.info("Order created: {}", orderId);
            
            // Step 2: 결제 처리
            Payment payment = processPayment(order);
            log.info("Payment processed: {}", payment.getId());
            
            // Step 3: 배송 생성
            Shipment shipment = createShipment(order);
            log.info("Shipment created: {}", shipment.getId());
            
        } catch (Exception e) {
            log.error("Error occurred, rolling back...");
            
            // ❌ 문제 1: 역순 실행이 보상이 아님
            // 단순히 order, payment, shipment 삭제?
            // 하지만 OrderService는 이미 주문을 생성했고,
            // 고객이 이미 주문 번호를 받았음!
            
            rollbackShipment(orderId);   // DB 삭제?
            rollbackPayment(orderId);    // 결제 취소?
            rollbackOrder(orderId);      // 주문 취소?
            
            throw e;
        }
    }
    
    // ❌ 문제 2: 멱등성 보장 없음
    // 결제 취소를 두 번 실행하면?
    private void rollbackPayment(String orderId) {
        Payment payment = paymentRepository.findByOrderId(orderId)
            .orElseThrow();
        
        // 외부 PG사에 "취소" 요청
        externalPGClient.cancel(payment.getId());
        
        // ❌ 같은 요청이 두 번 들어오면:
        // 첫 번째: 결제 취소 성공 ($100 환불)
        // 두 번째: 결제 취소 다시 시도 → $100 또 환불? (중복 환불!)
        
        payment.setStatus("CANCELLED");
        paymentRepository.save(payment);
    }
    
    // ❌ 문제 3: 취소 불가능한 경우를 고려하지 않음
    private void rollbackNotification(String orderId) {
        // 이미 고객에게 이메일을 보냈는데, 어떻게 "롤백"할 것인가?
        // notificationService.unsendEmail(orderId);  // 불가능!
        
        // 남은 선택지: 
        // 1. 무시하기 (고객이 받은 이메일은 그냥 두기)
        // 2. 추가 이메일 (사과 이메일 보내기)
        // 3. 수동 개입 (관리자가 처리)
    }
}

/*
❌ 심각한 문제들:
1. 역순 실행 ≠ 보상
   - 주문 생성 후 고객에게 확인 전달됨
   - 취소해도 고객은 이미 주문 번호를 알고 있음
   
2. 멱동성 없음 = 중복 장애
   - 환불을 두 번 실행 → 고객 계좌에 이중 환불
   - 재고 감소를 두 번 → 재고 음수?
   - 심각한 데이터 불일치
   
3. 취소 불가능한 작업 간과
   - 이메일, 메시지, 외부 API 호출 등
   - 이런 것들을 어떻게 보상할 것인가?
   
4. 보상 자체의 실패 고려 부족
   - 환불을 시도했는데 PG사 API가 응답 없음
   - 결과: 돈은 안 환불되는데 주문은 취소된 상태?
*/
```

```java
// ❌ 잘못된 접근 2: 멱등성을 제대로 구현하지 않은 결제 취소

@Service
public class PaymentServiceWithoutIdempotency {
    
    @Transactional
    public void refundPayment(Long paymentId) {
        Payment payment = paymentRepository.findById(paymentId)
            .orElseThrow();
        
        if (payment.getStatus().equals("COMPLETED")) {
            // ❌ 문제: 외부 PG사 호출과 DB 업데이트가 분리됨
            
            // T1: 외부 PG사에 환불 요청
            RefundResponse response = externalPGClient.refund(
                payment.getPgTransactionId(),
                payment.getAmount()
            );
            
            // T2: PG사는 환불 처리 완료 ($100 환불)
            // T3: 네트워크 오류로 response 수신 못함
            // T4: 자동 재시도 로직이 refundPayment 메서드를 다시 호출
            // T5: payment.getStatus() = "COMPLETED" (DB에는 아직 변경 안 됨)
            // T6: 또 다시 externalPGClient.refund() 호출
            // T7: 결과: 고객 계좌에서 $200 환불! (중복 환불)
            
            payment.setStatus("REFUNDED");
            paymentRepository.save(payment);
        }
    }
}
```

---

## ✨ 올바른 접근 (After)

```java
// ✅ 올바른 접근: 멱등성 보장 + 취소 불가능한 단계 식별

/**
 * Saga 보상 설계의 핵심 원칙:
 * 
 * 1. 각 단계마다 취소 가능 여부 판단
 * 2. 취소 불가능하면 Pivot Transaction(회전점) 표시
 * 3. 모든 보상은 멱등하게 구현
 * 4. 보상 실패 시 수동 개입 절차 정의
 */

@Entity
@Table(name = "saga_steps")
public class SagaStep {
    @Id
    private Long id;
    
    private String name;
    private String description;
    
    // Pivot Transaction 여부: true면 이 단계 이후는 보상 불가능 영역
    private Boolean isPivot;
    
    private String compensationLogic;  // 취소 방법 설명
    
    private LocalDateTime compensatedAt;
}

/**
 * Step 1: 취소 가능한 단계 → 표준 보상 트랜잭션
 */
@Service
@Slf4j
public class OrderCompensationService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private IdempotencyKeyRepository idempotencyKeyRepository;
    
    /**
     * 주문 생성 취소 (보상 트랜잭션)
     * 멱등성 보장: 같은 보상이 여러 번 실행되어도 한 번만 효과적
     */
    @Transactional
    public void compensateOrderCreation(Long orderId, String sagaId) {
        // Step 1: 멱등성 키 확인
        String idempotencyKey = generateCompensationKey("order.cancel", orderId, sagaId);
        
        if (idempotencyKeyRepository.existsByKey(idempotencyKey)) {
            log.info("Order cancellation already processed: {}", orderId);
            return;  // 이미 보상됨
        }
        
        try {
            // Step 2: 주문 취소 (DB 변경만)
            Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new EntityNotFoundException("Order not found"));
            
            order.setStatus("CANCELLED");
            order.setCancelledAt(LocalDateTime.now());
            order.setCancellationReason("Saga compensation initiated");
            orderRepository.save(order);
            
            // Step 3: 멱등성 키 기록 (이 트랜잭션과 함께 커밋)
            IdempotencyKey key = new IdempotencyKey(
                idempotencyKey,
                "OrderCompensation",
                orderId,
                "SUCCESS"
            );
            idempotencyKeyRepository.save(key);
            
            log.info("Order cancelled (compensation): {}", orderId);
            
        } catch (Exception e) {
            log.error("Order cancellation failed: {}", orderId, e);
            throw new CompensationException("Unable to cancel order", e);
        }
    }
}

/**
 * Step 2: 외부 시스템 호출 취소 (멱등성 필수)
 */
@Service
@Slf4j
public class PaymentCompensationService {
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Autowired
    private IdempotencyKeyRepository idempotencyKeyRepository;
    
    /**
     * 결제 환불 (보상 트랜잭션)
     * 멱등성 키로 중복 환불 방지
     */
    @Transactional
    public void compensatePayment(Long paymentId, String sagaId) {
        // Step 1: 멱등성 확인
        String idempotencyKey = generateCompensationKey("payment.refund", paymentId, sagaId);
        
        if (idempotencyKeyRepository.existsByKey(idempotencyKey)) {
            log.info("Payment refund already processed: {}", paymentId);
            return;
        }
        
        try {
            Payment payment = paymentRepository.findById(paymentId)
                .orElseThrow();
            
            // Step 2: 환불 요청 (외부 PG사)
            // ✅ 멱등성 키를 PG사에도 전달하여 중복 환불 방지
            RefundRequest refundRequest = RefundRequest.builder()
                .pgTransactionId(payment.getPgTransactionId())
                .amount(payment.getAmount())
                .idempotencyKey(idempotencyKey)  // PG사도 멱등성 보장
                .reason("Saga compensation initiated")
                .build();
            
            RefundResponse response = externalPGClient.refund(refundRequest);
            
            if (!response.isSuccess()) {
                throw new PaymentRefundException("Refund failed: " + response.getErrorMessage());
            }
            
            // Step 3: DB에 환불 기록 (PG사 응답 받은 후)
            payment.setStatus("REFUNDED");
            payment.setRefundTransactionId(response.getRefundTransactionId());
            payment.setRefundedAt(LocalDateTime.now());
            paymentRepository.save(payment);
            
            // Step 4: 멱등성 키 저장 (이 트랜잭션과 함께 커밋)
            IdempotencyKey key = new IdempotencyKey(
                idempotencyKey,
                "PaymentCompensation",
                paymentId,
                "SUCCESS"
            );
            idempotencyKeyRepository.save(key);
            
            log.info("Payment refunded (compensation): {} (Transaction: {})",
                paymentId, response.getRefundTransactionId());
            
        } catch (PaymentRefundException e) {
            log.error("Payment refund failed: {}", paymentId, e);
            throw new CompensationException("Unable to refund payment", e);
        }
    }
    
    private String generateCompensationKey(String action, Long resourceId, String sagaId) {
        return String.format("%s:%d:%s:%d",
            action,
            resourceId,
            sagaId,
            System.currentTimeMillis() / 1000);  // 타임스탬프 (1초 단위)
    }
}

/**
 * Step 3: 취소 불가능한 단계 (Pivot Transaction)
 */
@Service
@Slf4j
public class NotificationCompensationService {
    
    @Autowired
    private NotificationLogRepository notificationLogRepository;
    
    /**
     * 이메일 발송은 취소할 수 없음
     * 대신: 사과 이메일 발송 또는 수동 개입
     */
    @Transactional
    public void compensateEmailNotification(Long orderId, String sagaId) {
        // 이미 고객에게 발송된 이메일은 회수 불가능
        
        // 대체 방안 1: 사과 이메일 발송
        try {
            Customer customer = getCustomerForOrder(orderId);
            
            EmailTemplate apologyEmail = emailTemplateService.getTemplate("ORDER_CANCELLED_APOLOGY");
            emailService.sendAsync(
                customer.getEmail(),
                apologyEmail,
                Map.of("orderId", orderId, "sagaId", sagaId)
            );
            
            // 발송 로그 기록
            NotificationLog log = new NotificationLog(
                orderId,
                "EMAIL",
                "APOLOGY_SENT",
                "Original order was cancelled, apology email sent"
            );
            notificationLogRepository.save(log);
            
            log.info("Apology email sent for cancelled order: {}", orderId);
            
        } catch (Exception e) {
            log.error("Failed to send apology email: {}", orderId, e);
            
            // 대체 방안 2: 수동 개입 필요 (관리자 알림)
            notifyAdministrator(
                "EMAIL_COMPENSATION_FAILED",
                "Order " + orderId + " was cancelled but apology email failed to send. Manual action required."
            );
        }
    }
}

/**
 * Saga 보상 오케스트레이션
 */
@Service
@Slf4j
public class SagaCompensationOrchestrator {
    
    @Autowired
    private OrderCompensationService orderCompensation;
    
    @Autowired
    private PaymentCompensationService paymentCompensation;
    
    @Autowired
    private InventoryCompensationService inventoryCompensation;
    
    @Autowired
    private NotificationCompensationService notificationCompensation;
    
    /**
     * Saga 보상 실행
     * 원래 실행 순서의 역순으로 보상을 진행하되,
     * 각 단계별 의존성과 Pivot Transaction을 고려
     */
    public void executeCompensation(OrderSaga saga) {
        try {
            log.info("[Saga:{}] Starting compensation", saga.getId());
            
            // Pivot Transaction 확인:
            // - Notification(이메일) 이후는 보상 불가능
            // - 그 전까지만 자동 보상
            
            if (saga.hasShipmentCreated()) {
                // Step 1: 배송 취소 (취소 가능)
                inventoryCompensation.compensateShipment(saga.getShipmentId(), saga.getId());
            }
            
            if (saga.hasInventoryReserved()) {
                // Step 2: 재고 해제 (취소 가능)
                inventoryCompensation.compensateInventory(saga.getReservationId(), saga.getId());
            }
            
            if (saga.hasPaymentCompleted()) {
                // Step 3: 결제 환불 (취소 가능)
                paymentCompensation.compensatePayment(saga.getPaymentId(), saga.getId());
            }
            
            // Pivot: 이 지점부터는 보상 불가능 (또는 부분 보상만 가능)
            
            if (saga.hasOrderCreated()) {
                // Step 4: 주문 취소 (데이터만 변경)
                orderCompensation.compensateOrderCreation(saga.getOrderId(), saga.getId());
            }
            
            if (saga.hasNotificationSent()) {
                // Step 5: 알림 보상 (취소 불가능 → 대체 액션)
                notificationCompensation.compensateEmailNotification(saga.getOrderId(), saga.getId());
            }
            
            saga.setState(SagaState.SAGA_COMPENSATION_COMPLETED);
            sagaRepository.save(saga);
            
            log.info("[Saga:{}] Compensation completed successfully", saga.getId());
            
        } catch (CompensationException e) {
            log.error("[Saga:{}] Compensation failed", saga.getId(), e);
            saga.setState(SagaState.SAGA_COMPENSATION_FAILED);
            saga.setCompensationErrorMessage(e.getMessage());
            sagaRepository.save(saga);
            
            // 수동 개입 필요
            notifyAdministrator("SAGA_COMPENSATION_FAILED", saga.getId());
        }
    }
}
```

```yaml
# ✅ 올바른 설정: Idempotency Key를 위한 DB 스키마

spring:
  jpa:
    hibernate:
      ddl-auto: validate
  
  datasource:
    url: jdbc:postgresql://saga-db:5432/saga_db
    username: ${DB_USER}
    password: ${DB_PASSWORD}

# Idempotency Key 테이블 생성 쿼리:
# CREATE TABLE idempotency_keys (
#   key VARCHAR(255) PRIMARY KEY,
#   request_type VARCHAR(100) NOT NULL,
#   resource_id BIGINT NOT NULL,
#   result VARCHAR(10) NOT NULL,  -- SUCCESS, FAILED
#   created_at TIMESTAMP NOT NULL,
#   expires_at TIMESTAMP NOT NULL  -- TTL: 24시간 후 자동 삭제
# );

# CREATE INDEX idx_idempotency_keys_resource_id 
# ON idempotency_keys(resource_id);
```

---

## 🔬 내부 동작 원리 — 보상 트랜잭션의 특성

### 보상 트랜잭션 vs 롤백

```
┌────────────────────────────────────────────────────────────────┐
│         보상 트랜잭션 vs 데이터베이스 롤백                     │
└────────────────────────────────────────────────────────────────┘

║ 속성              │ DB 롤백         │ 보상 트랜잭션      ║
║────────────────────────────────────────────────────────────────║
║ 범위              │ 한 DB 내 원자   │ 여러 서비스 거쳐   ║
║ 타이밍            │ 즉시            │ 나중에 (비동기)    ║
║ 대상              │ 모든 변경       │ 특정 단계만        ║
║ 로직              │ 자동 (DBMS)    │ 수동 정의          ║
║ 외부 효과 처리    │ 불가능          │ 반영(환불 등)      ║
║ 멱등성            │ 자동            │ 명시적 보장 필요   ║

┌──────────────────────────────────────────────────────┐
│ DB 트랜잭션 롤백:                                   │
│                                                      │
│ BEGIN TRANSACTION                                    │
│   INSERT INTO orders ... (성공)                     │
│   INSERT INTO payments ... (실패)                   │
│ ROLLBACK                                             │
│                                                      │
│ 결과: orders와 payments 모두 롤백됨                 │
│      마치 아무것도 일어나지 않은 것처럼             │
└──────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│ 보상 트랜잭션:                                             │
│                                                            │
│ T1: INSERT INTO orders ... (성공, Order1 생성)            │
│ T2: CALL externalPaymentGateway.charge(...) (실패)        │
│ T3: Compensate (역순):                                     │
│   T3-1: CALL externalPaymentGateway.refund(...)           │
│          (외부 시스템에 환불 요청, 돈이 실제로 환불됨)  │
│   T3-2: UPDATE orders SET status = 'CANCELLED'            │
│          (Order1 상태 변경, 완전 삭제 아님)               │
│                                                            │
│ 결과: Order1은 CANCELLED 상태로 남음 (기록 유지)          │
│      고객 계좌에는 환불됨 (외부 시스템 반영)             │
└────────────────────────────────────────────────────────────┘
```

### Pivot Transaction (회전점 트랜잭션)

```
┌────────────────────────────────────────────────────────────────┐
│              Saga 진행 흐름에서 Pivot Transaction              │
└────────────────────────────────────────────────────────────────┘

정방향 (Forward) 실행:
┌──────────┬─────────┬────────────┬────────────┬──────────────┐
│ 주문생성 │ 결제    │ 재고예약   │ 배송생성   │ 알림발송     │
│ (가역)   │ (가역)  │ (가역)     │ (가역)     │ (불가역)     │
└──────────┴─────────┴────────────┴────────────┴──────────────┘
           │         │            │            │              │
           ▼         ▼            ▼            ▼              ▼
        DB 트랜잭션 진행...              Pivot Point
                                           ↓
                                   이메일 발송 (취소 불가)
                                   SMS 발송 (취소 불가)
                                   외부 API 호출 (원복 불가)

역방향 (Backward) 보상:
┌──────────┬─────────┬────────────┬────────────╱ ✗ ┐
│ 주문취소 │ 환불   │ 재고해제   │ 배송취소  │     │
│ (DB변경) │ (환불) │ (예약 해제)│ (취소)   │ 불가 │
└──────────┴─────────┴────────────┴──────────────╱  │
                                                  ← Pivot 이후는
                                                    보상 불가능
                                                    대신:
                                                    - 사과 이메일
                                                    - 수동 개입
```

### 멱등성 보장 메커니즘

```
┌────────────────────────────────────────────────────────────────┐
│              Idempotency Key를 통한 중복 방지                 │
└────────────────────────────────────────────────────────────────┘

시나리오: 결제 환불을 두 번 시도

T1: 환불 요청 도착
    idempotencyKey = "refund:payment123:saga456"
    
    ┌─ 멱등성 확인
    │  이미 처리된 요청인가?
    │  idempotencyRepository.existsByKey(idempotencyKey) → NO
    │
    ├─ 환불 실행
    │  externalPGClient.refund(paymentId, idempotencyKey)
    │  → PG사 DB에도 idempotencyKey 저장
    │  → 고객 계좌에서 $100 환불
    │
    └─ 멱등성 키 저장
       idempotencyRepository.save(new IdempotencyKey(...))
       → Saga DB에 기록

T2: 같은 환불 요청이 또 도착 (재시도 또는 중복 네트워크 전송)
    idempotencyKey = "refund:payment123:saga456" (동일)
    
    ┌─ 멱등성 확인
    │  idempotencyRepository.existsByKey(idempotencyKey) → YES!
    │  
    └─ 즉시 반환 (외부 PG사 호출 없음)
       log.info("Already processed")
       return  // 환불 2회 방지!

┌──────────────────────────────────────────────────────────┐
│ 결과:                                                    │
│ - 첫 번째 요청: 환불 성공 ($100 출금)                   │
│ - 두 번째 요청: 무시됨 (환불 안 됨)                     │
│ - 최종 결과: 환불 정확히 1회만 실행됨 ✅                │
└──────────────────────────────────────────────────────────┘
```

---

## 💻 멱등성 구현 패턴

```java
// Pattern 1: DB 기반 Idempotency Key

@Entity
@Table(name = "idempotency_keys")
public class IdempotencyKey {
    @Id
    private String key;
    
    private String operationType;
    private Long resourceId;
    private String result;  // SUCCESS, FAILED
    private String resultData;  // 결과 저장 (재시도 시 응답용)
    
    private LocalDateTime createdAt;
    private LocalDateTime expiresAt;  // TTL: 24시간
}

@Service
public class IdempotentPaymentService {
    
    public void refundPaymentWithIdempotency(Long paymentId, String sagaId) {
        String idempotencyKey = generateKey(paymentId, sagaId);
        
        // Step 1: 멱등성 확인
        Optional<IdempotencyKey> existing = 
            idempotencyKeyRepository.findById(idempotencyKey);
        
        if (existing.isPresent()) {
            // 이미 처리됨
            IdempotencyKey record = existing.get();
            if (record.getResult().equals("SUCCESS")) {
                log.info("Refund already successful: {}", idempotencyKey);
                // 필요시 resultData 응답
                return;
            } else {
                // 이전에 실패했으므로 다시 시도
                log.info("Previous attempt failed, retrying: {}", idempotencyKey);
            }
        }
        
        try {
            // Step 2: 실제 환불 처리
            RefundResponse response = externalPGClient.refund(
                paymentId,
                idempotencyKey  // PG사도 멱등성 보장
            );
            
            // Step 3: 성공 기록
            IdempotencyKey record = new IdempotencyKey();
            record.setKey(idempotencyKey);
            record.setOperationType("PAYMENT_REFUND");
            record.setResourceId(paymentId);
            record.setResult("SUCCESS");
            record.setResultData(response.toString());
            record.setCreatedAt(LocalDateTime.now());
            record.setExpiresAt(LocalDateTime.now().plusDays(1));
            
            idempotencyKeyRepository.save(record);
            
        } catch (Exception e) {
            // Step 4: 실패 기록 (다음 재시도를 위해)
            IdempotencyKey record = new IdempotencyKey();
            record.setKey(idempotencyKey);
            record.setOperationType("PAYMENT_REFUND");
            record.setResourceId(paymentId);
            record.setResult("FAILED");
            record.setResultData(e.getMessage());
            record.setCreatedAt(LocalDateTime.now());
            record.setExpiresAt(LocalDateTime.now().plusDays(1));
            
            idempotencyKeyRepository.save(record);
            
            throw new CompensationException("Refund failed", e);
        }
    }
}

// Pattern 2: 상태 기반 멱등성

@Entity
@Table(name = "payments")
public class Payment {
    @Id
    private Long id;
    
    private String status;  // COMPLETED, REFUNDING, REFUNDED, REFUND_FAILED
    
    // ✅ 상태로 멱등성 보장
    // REFUNDING 상태에서 또 요청 오면 → 이미 진행 중
    // REFUNDED 상태에서 또 요청 오면 → 이미 완료
    // REFUND_FAILED 상태에서 또 요청 오면 → 재시도 가능
}

@Service
public class PaymentServiceWithStateBasedIdempotency {
    
    public void refundPayment(Long paymentId) {
        Payment payment = paymentRepository.findById(paymentId)
            .orElseThrow();
        
        // Step 1: 상태 확인으로 멱등성 보장
        if (payment.getStatus().equals("REFUNDED")) {
            log.info("Payment already refunded: {}", paymentId);
            return;  // 이미 환불됨
        }
        
        if (payment.getStatus().equals("REFUNDING")) {
            log.info("Payment refund already in progress: {}", paymentId);
            return;  // 이미 진행 중
        }
        
        // Step 2: REFUNDING 상태로 전환
        payment.setStatus("REFUNDING");
        paymentRepository.save(payment);
        
        try {
            // Step 3: 환불 처리
            externalPGClient.refund(payment.getPgTransactionId());
            
            // Step 4: 성공
            payment.setStatus("REFUNDED");
            paymentRepository.save(payment);
            
        } catch (Exception e) {
            // Step 5: 실패
            payment.setStatus("REFUND_FAILED");
            payment.setRefundErrorMessage(e.getMessage());
            paymentRepository.save(payment);
            
            throw e;
        }
    }
}
```

---

## 📊 패턴 비교

| 항목 | DB 롤백 | 보상 트랜잭션 |
|------|------|------|
| **범위** | 한 DB 내 | 여러 서비스 |
| **실행 시점** | 즉시 | 나중에 (비동기) |
| **외부 효과** | 처리 불가 | 반영 (환불, 취소) |
| **멱등성** | 자동 | 명시적 필요 |
| **설계 복잡도** | 낮음 | 높음 |
| **매커니즘** | UNDO | COMPENSATE |
| **완전성** | 완벽 (원자성) | 부분적 (최종 일관성) |

---

## ⚖️ 트레이드오프

### 보상 트랜잭션의 과제
- ✅ **외부 시스템 처리**: 환불, 취소 등 실제 반영
- ✅ **비즈니스 로직 반영**: 각 도메인의 특성을 보상에 반영
- ✅ **부분 보상 가능**: 취소 가능한 부분만 취소, 불가능한 부분은 대체
- ❌ **복잡한 설계**: 각 단계별 보상 로직 정의 필요
- ❌ **멱등성 관리**: Idempotency Key 인프라 구축 필수
- ❌ **취소 불가능한 작업**: 이메일, 메시지 등을 어떻게 보상할 것인가
- ❌ **보상 실패 처리**: 보상 자체가 실패했을 때 수동 개입 필요

---

## 📌 핵심 정리

✅ **보상 트랜잭션**은 ACID 롤백이 아니라 "비즈니스 관점의 취소"입니다. 외부 시스템의 영향을 반영하면서 일관성을 복구합니다.

✅ **모든 보상은 멱등성을 보장**해야 합니다. Idempotency Key 패턴으로 같은 보상이 여러 번 실행되어도 정확히 한 번의 효과만 가집니다.

✅ **Pivot Transaction(회전점)**을 식별해야 합니다. 이 지점 이후는 자동 보상이 불가능하므로, 대체 액션(사과 이메일) 또는 수동 개입이 필요합니다.

✅ **취소 불가능한 작업**:
- 이메일 발송 → 사과 이메일 또는 알림
- SMS 발송 → 추가 통지
- 외부 API 호출 → 환불/취소만 가능
- 실시간 스트림 → 기록만 남김

✅ **보상 전략**:
1. 정방향 성공 필요
2. 역방향 실패 → 보상 시작
3. 역순 보상 실행 (각 단계별 멱등성 보장)
4. 보상 실패 → 수동 개입 및 관리자 알림

---

## 🤔 생각해볼 문제

**Q1.** "배송 완료"라는 단계는 보상이 가능할까요? 이미 고객에게 배송되었다면?

<details><summary>해설 보기</summary>
**답**: 배송은 **Pivot Transaction 이후**의 영역입니다.

**시나리오**:
1. 주문 생성 ✅
2. 결제 처리 ✅
3. 재고 예약 ✅
4. **배송 생성 ← 어느 순간부터는 고객이 배송 추적 중**
5. 배송 시작 → 택배사가 배송 중
6. **배송 완료** ← 고객이 상품을 받음!

**보상 가능 여부**:
- 배송 **생성** 직후 (택배사에 픽업 요청 전) → 보상 가능 (배송 취소)
- 배송 **시작** 후 (택배사가 픽업함) → 부분 보상 (배송 반품 요청)
- 배송 **완료** 후 (고객이 받음) → 보상 불가능 (환불/반품만 가능)

**보상 전략**:
```
정상 흐름: 배송 → 배송 완료 → 정산
실패 흐름: 정산 실패 → 보상 시작
  → 배송 단계로 역순 가기 → 하지만 이미 배송 완료?
  → Pivot Transaction 확인!
  → 배송 완료 후라면: 환불/반품 정책만 적용
     (배송 취소 아님, 고객이 반품 신청 필요)
```

**즉**: 배송 완료는 보상 대상이 아니라, **환불/반품 정책**에 따라 처리합니다.
</details>

**Q2.** Idempotency Key를 24시간 후에 자동 삭제하는데, 만약 그 후에 같은 요청이 또 들어온다면?

<details><summary>해설 보기</summary>
**시나리오**:
1. T1: 결제 환불 요청 → idempotencyKey 저장
2. T1+24시간: idempotencyKey 자동 삭제 (TTL)
3. T1+24시간+1분: 같은 환불 요청 또 도착

**문제**:
- Idempotency Key가 이미 삭제됨
- 새 요청으로 인식 → 또 다시 환불!
- 결과: 중복 환불

**해결 방법**:

**방법 1: 더 긴 TTL 설정**
```java
// 24시간 대신 7일로 설정
record.setExpiresAt(LocalDateTime.now().plusDays(7));
```
하지만 언제까지 보관해야 하는가? 영구?

**방법 2: 재요청 자체를 방지**
```java
// Saga 상태로 이미 "보상됨"을 판단
// idempotency Key 대신 Saga 상태 확인
if (saga.getState() == SagaState.COMPENSATION_COMPLETED) {
    return;  // 이미 보상됨
}
```
Saga 상태는 영구 보관되므로 문제 없음.

**방법 3: 하이브리드**
```java
// 24시간: idempotency Key로 빠른 응답
// 그 이후: Saga 상태로 확인
if (idempotencyKeyRepository.existsByKey(key)) {
    return idempotencyKeyRepository.getResult(key);
}
// idempotency Key가 없으면 Saga 상태 확인
Saga saga = sagaRepository.findById(sagaId);
if (saga.getState() == SagaState.COMPENSATION_COMPLETED) {
    return;  // 이미 보상됨 (24시간 후에도)
}
```

**권장**: **방법 3 하이브리드**가 가장 실용적입니다.
- 최근 24시간: 빠른 Idempotency Key 조회
- 24시간 이상: Saga 상태로 확인
- 저장소 비용과 안정성의 균형 유지
</details>

**Q3.** 보상 트랜잭션 자체가 실패했을 때 (예: PG사에 환불 요청 실패), 재시도는 언제 어떻게 해야 할까요?

<details><summary>해설 보기</summary>
**종류별 재시도 전략**:

**1. 일시적 오류 (네트워크 불안정)**
```java
try {
    externalPGClient.refund(paymentId, idempotencyKey);
} catch (TemporaryException e) {  // 타임아웃, 503 등
    // 즉시 재시도 (exponential backoff)
    Thread.sleep(1000);  // 1초 대기
    externalPGClient.refund(paymentId, idempotencyKey);
}
```

**2. 영구적 오류 (계좌 없음, 금액 초과 등)**
```java
catch (PermanentException e) {
    // 재시도 불가능 → 수동 개입
    log.error("Cannot refund (permanent error): {}", e.getMessage());
    saga.setState(SagaState.COMPENSATION_FAILED);
    notifyAdministrator("REFUND_FAILED", saga.getId());
}
```

**3. 상태 불명 (응답 없음)**
```java
catch (TimeoutException e) {
    // 실제 환불됐는지 아닌지 모름 → 상태 확인
    PaymentStatus status = externalPGClient.queryStatus(paymentId);
    if (status.isRefunded()) {
        // 환불됨 → 상태만 업데이트
        payment.setStatus("REFUNDED");
    } else {
        // 환불 안 됨 → 재시도
        retryRefund(paymentId);
    }
}
```

**권장 재시도 전략**:
```yaml
# 설정
compensation:
  retry:
    max-attempts: 3
    initial-delay-ms: 1000      # 1초
    max-delay-ms: 60000         # 60초
    backoff-multiplier: 2       # exponential
    
    # 재시도 스케줄
    # 1차 실패 → 1초 후 재시도
    # 2차 실패 → 2초 후 재시도
    # 3차 실패 → 관리자 알림 & 수동 개입
```

**수동 개입 절차**:
```
1. Saga 상태: COMPENSATION_FAILED
2. 관리자 대시보드에 경고
3. 관리자 조회:
   - "이 Saga는 보상 실패함"
   - "PG사 환불이 정말로 안 됐는가?"
4. 관리자 액션:
   - 옵션 1: "재시도" 버튼 클릭 → 보상 다시 실행
   - 옵션 2: "강제 완료" → Saga를 COMPENSATION_COMPLETED로 표시
   - 옵션 3: "수동 환불" → 관리자가 직접 PG사에 전화
```

**자동 재시도 스케줄러**:
```java
@Service
public class FailedCompensationRetryScheduler {
    
    @Scheduled(fixedRate = 300000)  // 5분마다
    public void retryFailedCompensations() {
        List<OrderSaga> failedSagas = sagaRepository
            .findByState(SagaState.COMPENSATION_FAILED);
        
        for (OrderSaga saga : failedSagas) {
            if (saga.getRetryCount() < 3) {
                try {
                    sagaCompensationOrchestrator.executeCompensation(saga);
                    saga.incrementRetryCount();
                    sagaRepository.save(saga);
                } catch (Exception e) {
                    log.warn("Retry failed for saga: {}", saga.getId(), e);
                }
            } else {
                // 3회 재시도 실패 → 수동 개입 필요
                notifyAdministrator("MAX_RETRY_EXCEEDED", saga.getId());
            }
        }
    }
}
```
</details>

---

<div align="center">

**[⬅️ 이전: Choreography vs Orchestration — 선택 기준](./04-choreography-vs-orchestration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Saga 실패 처리와 모니터링 ➡️](./06-saga-failure-and-monitoring.md)**

</div>

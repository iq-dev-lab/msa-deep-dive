# 01. 분산 트랜잭션의 문제 — 2PC와 Saga

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 2PC(Two-Phase Commit)가 MSA 환경에서 왜 실용적으로 쓸 수 없는가?
- 분산 시스템에서 "모든 참여자가 응답해야만 진행"이라는 조건이 가용성을 해치는 이유는?
- Saga 패턴은 2PC의 어떤 문제를 해결하고, 어떤 새로운 도전을 만드는가?
- CAP Theorem에서 Saga는 어느 쪽을 선택하는 패턴인가?
- 보상 트랜잭션(Compensating Transaction)이 롤백과 다른 이유는?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

분산 마이크로서비스 환경에서 여러 서비스에 걸친 업무 로직을 실행할 때, 일관성(Consistency)을 어떻게 보장할 것인가는 아키텍처의 핵심 문제입니다. 전통적인 단일 데이터베이스 환경에서는 한 번의 ACID 트랜잭션으로 해결했던 문제를 이제는 네트워크 상의 여러 독립적인 서비스 경계를 넘어서 해결해야 합니다.

2PC(Two-Phase Commit)는 분산 데이터베이스 환경에서 수십 년 동안 검증된 프로토콜이지만, 마이크로서비스 아키텍처의 특성(느슨한 결합, 높은 가용성 요구)과 현대적인 클라우드 환경(네트워크 장애의 불가피성, 다양한 기술 스택)에서는 충분하지 않습니다. Saga 패턴은 이러한 제약을 인정하고, 각 서비스의 로컬 트랜잭션을 활용하면서도 전체 비즈니스 일관성을 달성하는 실용적인 접근 방식입니다.

분산 트랜잭션 처리 방식을 올바르게 이해하지 못하면, 데이터 불일치, 데드락, 성능 저하, 또는 급작스러운 장애로 인한 수백 건의 고아 레코드(orphaned records)가 발생할 수 있습니다. 따라서 각 패턴의 동작 원리, 장단점, 그리고 MSA 환경에서의 적용 시기를 명확히 해야 합니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ 잘못된 접근: 2PC를 MSA에 강제로 적용하려는 시도
// 다중 데이터베이스 트랜잭션 코디네이터를 사용하는 경우

@Service
public class OrderService {
    
    @Transactional  // 단일 @Transactional로 여러 서비스를 묶으려는 시도
    public void createOrderWithPaymentAndInventory(OrderRequest request) {
        // 이 트랜잭션이 OrderDB, PaymentDB, InventoryDB 세 개의 데이터베이스를
        // 묶으려고 시도 — 실제로는 불가능하거나 매우 위험함
        
        // Phase 1: Prepare — 모든 DB가 준비 완료를 응답해야 함
        Order order = orderRepository.save(new Order(request));
        
        // PaymentService의 데이터베이스에 직접 접근 (강결합!)
        paymentRepository.save(new Payment(order.getId(), request.getAmount()));
        
        // InventoryService의 데이터베이스에 직접 접근 (강결합!)
        inventoryRepository.decreaseStock(request.getProductId(), request.getQuantity());
        
        // Phase 2: Commit — 모든 DB가 동시에 커밋되어야 함
        // 하지만 이 중 하나라도 느리면 전체가 블로킹됨
    }
}

/*
실제 문제점:
1. 코디네이터 노드 장애 → 모든 참여 DB가 무한정 락 상태
2. PaymentService 노드가 응답 없음 → 전체 주문 처리 중단
3. 네트워크 지연 → 모든 트랜잭션이 대기, 동시성 저하
4. 2PC 타임아웃 → 부분 커밋 상태 (데이터 불일치)
5. 가용성 저하 → 가장 느린 서비스가 전체 성능 좌우
*/
```

```yaml
# ❌ 잘못된 설정: 코디네이터 기반 2PC 의존
# 여러 데이터베이스를 하나의 글로벌 트랜잭션으로 묶으려는 시도

spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
  datasource:
    # 글로벌 트랜잭션 코디네이터 사용 시도
    jndi-name: java:/TransactionManager
    # ← 이 방식은 온프레미스 환경에서도 문제 많음
    # ← 클라우드 MSA 환경에서는 매우 위험함

# 결과:
# - 코디네이터 SPOF(Single Point of Failure)
# - 네트워크 분할(Network Partition) 시 시스템 다운
# - 기업 내 여러 팀의 독립적 DB를 강제로 결합
```

---

## ✨ 올바른 접근 (After)

```java
// ✅ 올바른 접근: Saga 패턴으로 각 서비스의 로컬 트랜잭션만 사용
// 보상 트랜잭션(Compensating Transaction)으로 실패 처리

@Service
public class OrderSagaService {
    
    private final OrderRepository orderRepository;
    private final PaymentServiceClient paymentClient;
    private final InventoryServiceClient inventoryClient;
    private final SagaLogRepository sagaLogRepository;
    
    /**
     * 주문 생성 Saga 실행
     * 각 단계는 독립적인 로컬 트랜잭션으로 실행
     * 장애 발생 시 보상 트랜잭션으로 취소
     */
    public void executeOrderSaga(OrderRequest request) throws SagaException {
        String sagaId = UUID.randomUUID().toString();
        SagaLog sagaLog = new SagaLog(sagaId, "ORDER_SAGA_START");
        sagaLogRepository.save(sagaLog);
        
        try {
            // Step 1: 주문 생성 (OrderService의 로컬 트랜잭션)
            Order order = createOrderLocally(request, sagaId);
            recordSagaStep(sagaId, "ORDER_CREATED", "success", order.getId());
            
            // Step 2: 결제 처리 (PaymentService의 로컬 트랜잭션)
            try {
                PaymentResult payment = paymentClient.processPayment(
                    new PaymentRequest(order.getId(), request.getAmount())
                );
                recordSagaStep(sagaId, "PAYMENT_PROCESSED", "success", payment.getId());
            } catch (PaymentException e) {
                // 결제 실패 시 보상 트랜잭션: 주문 취소
                compensateOrder(order.getId(), sagaId);
                recordSagaStep(sagaId, "ORDER_CANCELLED", "compensation", order.getId());
                throw new SagaException("Payment failed, order cancelled", e);
            }
            
            // Step 3: 재고 예약 (InventoryService의 로컬 트랜잭션)
            try {
                InventoryResult inventory = inventoryClient.reserveStock(
                    new InventoryRequest(request.getProductId(), request.getQuantity())
                );
                recordSagaStep(sagaId, "INVENTORY_RESERVED", "success", inventory.getId());
            } catch (InventoryException e) {
                // 재고 부족 시 보상 트랜잭션: 결제 환불 + 주문 취소
                paymentClient.refundPayment(new RefundRequest(order.getId()));
                compensateOrder(order.getId(), sagaId);
                recordSagaStep(sagaId, "REFUND_PROCESSED", "compensation", order.getId());
                recordSagaStep(sagaId, "ORDER_CANCELLED", "compensation", order.getId());
                throw new SagaException("Inventory unavailable, refund processed", e);
            }
            
            // 모든 단계 완료
            recordSagaStep(sagaId, "ORDER_SAGA_COMPLETED", "success", order.getId());
            
        } catch (Exception e) {
            recordSagaStep(sagaId, "ORDER_SAGA_FAILED", "error", null);
            throw new SagaException("Saga execution failed", e);
        }
    }
    
    @Transactional  // 로컬 트랜잭션만 사용
    private Order createOrderLocally(OrderRequest request, String sagaId) {
        Order order = new Order(
            request.getCustomerId(),
            request.getProductId(),
            request.getQuantity(),
            request.getAmount(),
            sagaId
        );
        return orderRepository.save(order);
    }
    
    @Transactional  // 보상 트랜잭션: 주문 취소
    private void compensateOrder(Long orderId, String sagaId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new EntityNotFoundException("Order not found"));
        order.setStatus("CANCELLED");
        order.setCancelledAt(LocalDateTime.now());
        orderRepository.save(order);
    }
    
    private void recordSagaStep(String sagaId, String step, String status, Object data) {
        SagaLog log = new SagaLog(sagaId, step);
        log.setStatus(status);
        log.setData(data != null ? data.toString() : null);
        log.setTimestamp(LocalDateTime.now());
        sagaLogRepository.save(log);
    }
}

/*
장점:
1. 각 서비스가 독립적으로 실행 → 높은 가용성
2. 네트워크 장애가 전체를 막지 않음 → 복원력 증가
3. 각 서비스는 로컬 트랜잭션만 관리 → 복잡도 감소
4. 보상 트랜잭션으로 실패 처리 → 기업 로직에 맞춤
*/
```

```yaml
# ✅ 올바른 설정: 각 서비스가 독립적인 DB를 가짐
# Saga를 통해 느슨한 결합 유지

spring:
  application:
    name: order-service
  datasource:
    url: jdbc:postgresql://order-db:5432/order_db  # 독립적인 DB
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate  # 스키마 관리는 서비스별로

# 다른 서비스(PaymentService)의 DB는 다음과 같음:
# spring:
#   datasource:
#     url: jdbc:postgresql://payment-db:5432/payment_db

# 서비스 간 통신은 HTTP/gRPC/메시지 브로커로만 진행
# 2PC 코디네이터 불필요
```

---

## 🔬 내부 동작 원리 — 2PC vs Saga 심층 분석

### 2PC(Two-Phase Commit) 프로토콜의 동작

```
┌─────────────────────────────────────────────────────────────────────┐
│                      2PC 타임라인                                    │
└─────────────────────────────────────────────────────────────────────┘

시각       Coordinator        Participant1(Payment)  Participant2(Inventory)
────────────────────────────────────────────────────────────────────────
T0         시작 신호
           ─────────────────────────────────────────────────────────────→

           ┌─ Phase 1: Prepare (준비 단계) ─┐

T1         Prepare Request
           ─────────────────→  ✓ Lock 획득
                               ✓ 금액 재확인
                               ← YES (준비 완료)
           
T2         Prepare Request
           ─────────────────────────────────→  ✓ Lock 획득
                                               ✓ 재고 재확인
                                               ← YES (준비 완료)

T3         ✓ 모든 참여자로부터
           ✓ YES 응답 수신
           
           ┌─ Phase 2: Commit (커밋 단계) ─┐

T4         Commit Request
           ─────────────────→  ✓ 실제 결제 처리
                               ✓ Lock 해제
                               ✓ Log 기록
                               ← ACK (완료)

T5         Commit Request
           ─────────────────────────────────→  ✓ 실제 재고 감소
                                               ✓ Lock 해제
                                               ✓ Log 기록
                                               ← ACK (완료)

T6         거래 완료

────────────────────────────────────────────────────────────────────────

❌ 문제 시나리오:

시각       Coordinator        Participant1(Payment)  Participant2(Inventory)
────────────────────────────────────────────────────────────────────────
T1-T2      Prepare 완료       ✓ Lock 획득             ✓ Lock 획득
           
T3         ...

T4         Commit Request 전송 중...  [네트워크 장애 발생]
           
T4-T5      Response 대기     ⏳ Lock 유지 (무한정)   ⏳ Lock 유지 (무한정)
           
           결과: DEADLOCK!
           - 결제는 처리됨 (Lock 해제 못함)
           - 재고는 예약됨 (Lock 해제 못함)
           - Coordinator도 다운됨 (복구 불가)
           - 수동 개입 필요 (DBA가 DB 직접 수정)
```

### Saga 패턴의 동작

```
┌──────────────────────────────────────────────────────────────────────┐
│                      Saga 타임라인 (보상 Saga)                      │
└──────────────────────────────────────────────────────────────────────┘

시각    Orchestrator    OrderService      PaymentService   InventoryService
───────────────────────────────────────────────────────────────────────────
T0      Saga 시작
        
T1      주문 생성 요청
        ────────────────→  ✓ Order 생성
                           ✓ Commit
                           ✓ Event 발행
                           ← Response

T2      결제 처리 요청
        ──────────────────────────────→  ✓ Payment 처리
                                        ✓ Commit
                                        ✓ Event 발행
                                        ← Response

T3      재고 예약 요청
        ─────────────────────────────────────────────→  ✓ Stock 예약
                                                       ✓ Commit
                                                       ✓ Event 발행
                                                       ← Response

T4      모든 단계 완료

───────────────────────────────────────────────────────────────────────────

❌ 장애 발생 시나리오:

시각    Orchestrator    OrderService      PaymentService   InventoryService
───────────────────────────────────────────────────────────────────────────
T1      주문 생성 요청
        ────────────────→  ✓ Order 생성
                           
T2      결제 처리 요청
        ──────────────────────────────→  ✓ Payment 처리

T3      재고 예약 요청
        ─────────────────────────────────────────────→  ❌ Stock 부족!
                                                       ← Error Response

T4      ✓ Error 수신
        보상 Saga 시작:
        
        환불 요청
        ──────────────────────────────→  ✓ Refund 처리
                                        ✓ Commit
                                        ← Response

T5      주문 취소 요청
        ────────────────→  ✓ Order 취소
                           ✓ Commit
                           ← Response

T6      보상 완료, Saga 종료
        (각 서비스는 Lock을 유지하지 않음 → 높은 가용성)

────────────────────────────────────────────────────────────────────────
✅ 장점:
- T1-T2-T3 각 단계가 독립적으로 커밋됨 → Lock 시간 최소
- T3의 장애가 T1, T2의 영향을 주지 않음
- 보상 트랜잭션도 로컬 트랜잭션 → 단순하고 복원력 있음
```

### CAP Theorem 관점에서의 선택

```
┌────────────────────────────────────────────────────────────────┐
│              CAP Theorem: 세 가지 선택                        │
└────────────────────────────────────────────────────────────────┘

         ┌─────────────────────┐
         │   Consistency (C)   │  모든 노드가 같은 데이터
         │ (강한 일관성)       │
         └──────────┬──────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
    ┌─────────────────────────────────┐
    │   2PC 선택: C + A                │
    │   - 모든 참여자 응답 필요        │
    │   - 하나 실패 → 전체 실패        │
    │   - 결과: 낮은 가용성 (A 포기)   │
    └─────────────────────────────────┘
    
    ┌─────────────────────────────────┐
    │   Saga 선택: A + P               │
    │   (결과적 일관성)                │
    │   - 각 서비스 독립 커밋           │
    │   - 강한 일관성 대신 복원력       │
    │   - 결과: 높은 가용성             │
    └─────────────────────────────────┘
        │           │           │
        │           │           └──────────────────┐
        │           └──────────────────┐           │
        ▼                              ▼           ▼
 ┌─────────────────────┐    ┌─────────────────────┐
 │  Partition (P)      │    │  Availability (A)   │
 │ (분할 허용)         │    │  (높은 가용성)      │
 └─────────────────────┘    └─────────────────────┘
```

---

## 💻 2PC와 Saga의 구체적 코드 비교

### 데이터 상태 추적

```java
// 2PC: 코디네이터가 모든 상태를 중앙에서 관리
@Entity
@Table(name = "global_transactions")
public class GlobalTransaction {
    @Id
    private String transactionId;
    @Enumerated(EnumType.STRING)
    private TransactionState state;  // PREPARING, PREPARED, COMMITTING, COMMITTED
    private LocalDateTime startTime;
    private LocalDateTime preparedTime;
    private LocalDateTime commitTime;
    
    // 문제: 코디네이터 장애 시 이 상태를 복구할 방법 없음
}

// Saga: 각 단계를 SagaLog로 기록
@Entity
@Table(name = "saga_logs")
public class SagaLog {
    @Id
    private Long id;
    private String sagaId;
    private String step;              // ORDER_CREATED, PAYMENT_PROCESSED, etc.
    @Enumerated(EnumType.STRING)
    private StepStatus status;        // SUCCESS, FAILED, COMPENSATED
    private String resultData;
    private LocalDateTime timestamp;
    
    // 장점: 각 서비스의 로컬 DB에 기록되므로 복구 가능
    //      비즈니스 로직에 맞춘 단계 정의 가능
}
```

---

## 📊 패턴 비교

| 항목 | 2PC | Saga |
|------|-----|------|
| **결합도** | 강결합 (코디네이터 중심) | 느슨한 결합 (독립적 실행) |
| **Lock 시간** | 긴 (Phase 1-2 동안 계속) | 짧음 (각 단계 로컬 트랜잭션) |
| **가용성** | 낮음 (모든 참여자 필요) | 높음 (일부 장애 허용) |
| **네트워크 장애 대응** | 취약 (코디네이터 SPOF) | 강함 (보상 트랜잭션으로 복구) |
| **일관성 모델** | 강한 일관성 (C) | 결과적 일관성 (Eventually Consistent) |
| **디버깅 난이도** | 쉬움 (중앙 추적) | 어려움 (분산 로그) |
| **보상 트랜잭션** | Rollback (자동) | Compensation (수동 정의) |
| **시스템 복잡도** | 낮음 (프로토콜 표준) | 높음 (비즈니스 로직 반영) |
| **클라우드 적합성** | 낮음 | 높음 (마이크로서비스 친화) |
| **구현 도구** | DB 연동 (JTA, XA) | 메시지 브로커, Orchestrator |

---

## ⚖️ 트레이드오프

### 2PC 선택 시
- ✅ **강한 일관성**: 모든 업데이트가 동시에 반영되거나 모두 실패
- ✅ **단순한 복구**: Rollback 로직이 간단 (DB가 자동 처리)
- ✅ **추적의 용이성**: 중앙 코디네이터에서 전체 흐름 파악 가능
- ❌ **가용성 저하**: 느린 참여자 하나가 전체를 블로킹
- ❌ **확장성 제한**: 참여자 수가 많아지면 성능 급락
- ❌ **클라우드 부적합**: 네트워크 불안정성에 취약

### Saga 선택 시
- ✅ **높은 가용성**: 부분 장애를 허용하고 계속 진행
- ✅ **우수한 확장성**: 서비스/단계 추가가 용이
- ✅ **클라우드 친화**: 독립적 배포, 다양한 기술 스택 지원
- ✅ **비즈니스 로직 반영**: 각 서비스의 특성에 맞춘 보상 가능
- ❌ **결과적 일관성**: 중간 상태에서 데이터 불일치 가능
- ❌ **복잡한 보상**: 모든 단계마다 보상 로직 작성 필요
- ❌ **디버깅 어려움**: 분산 환경에서 장애 원인 파악 복잡

---

## 📌 핵심 정리

✅ **2PC(Two-Phase Commit)**는 강한 일관성을 보장하지만, MSA 환경에서는 코디네이터의 SPOF(Single Point of Failure), 높은 결합도, 낮은 가용성 때문에 실용적이지 않습니다.

✅ **Saga 패턴**은 각 서비스의 로컬 트랜잭션을 활용하여 느슨한 결합을 유지하면서 비즈니스 일관성을 달성합니다.

✅ **Saga의 두 가지 구현 방식**
- **Choreography**: 이벤트 기반, 자율 실행, 느슨한 결합 (흐름 추적 어려움)
- **Orchestration**: 중앙 오케스트레이터, 명시적 제어, 추적 용이 (결합도 높음)

✅ **보상 트랜잭션(Compensating Transaction)**은 실패 시 "롤백"이 아니라 "취소"를 수행합니다. 모든 단계가 취소 가능해야 하며, 멱등성을 보장해야 합니다.

✅ **CAP Theorem의 관점**: 2PC는 C + A를 선택 (P 포기 불가)하므로 실패하고, Saga는 A + P를 선택하여 E(결과적 일관성)을 실현합니다.

---

## 🤔 생각해볼 문제

**Q1.** 다음 비즈니스 요구사항에서 2PC와 Saga 중 어느 것을 선택해야 할까요? "결제가 성공했으면 반드시 주문이 생성되어야 하고, 반대도 마찬가지다. 즉, 어느 한쪽이라도 실패했으면 다른 쪽도 실패해야 한다."

<details><summary>해설 보기</summary>
이것이 바로 분산 트랜잭션의 핵심 요구사항입니다. 

**Saga의 관점**: "모두 성공하거나 모두 보상하기"입니다. 
- Order 생성 성공 → Payment 처리 실패 → Order 취소 보상 (결과: 둘 다 없는 상태)
- Order 생성 성공 → Payment 처리 성공 → Inventory 부족 → Payment 환불 + Order 취소 (결과: 전부 취소)

즉, Saga는 **최종 일관성(Final Consistency)**을 보장합니다. 중간 상태(Order 생성만 된 상태)는 잠시 존재할 수 있지만, 결국 모두 성공하거나 모두 취소됩니다.

**결론**: Saga로 충분합니다. 2PC는 이 요구사항을 더 빨리 달성하지만, 네트워크 불안정 환경에서 오히려 더 위험합니다.
</details>

**Q2.** 보상 트랜잭션(Compensating Transaction)이 항상 가능할까요? 불가능한 경우의 예를 들어보세요.

<details><summary>해설 보기</summary>
일부 단계는 보상이 불가능합니다. 이를 **Pivot Transaction(회전점 트랜잭션)**이라고 부릅니다:

- **불가능한 경우들**:
  1. "이메일 발송 완료" → 취소 불가능 (발송된 이메일 회수 불가)
  2. "문자 메시지 전송" → 취소 불가능 (전송된 메시지 삭제 불가)
  3. "외부 PG에 결제 요청 이미 전달됨" → Rollback 불가능 (환불만 가능)

**해결 방법**:
- **Pivot Transaction 뒤는 보상 불가 영역**: 이 지점 이후의 실패는 수동 개입 필요
- **Pivot Transaction 이전까지만 자동 보상**: "주문 생성" "결제" 등은 자동 보상, 이후 "배송" "통보"는 수동 개입

따라서 Saga 설계 시 Pivot Transaction을 명확히 파악하는 것이 중요합니다.
</details>

**Q3.** Saga가 "각 서비스의 로컬 트랜잭션만 사용"한다면, 중간 상태(Order 생성됨, Payment 아직 미처리)에서 고객이 주문을 조회하면 어떻게 보일까요?

<details><summary>해설 보기</summary>
**현재 상태**: Order 테이블에는 "PENDING" 상태로 주문이 저장됨, Payment 테이블에는 아직 기록 없음

**고객이 본 화면**:
- Order API → "주문 생성됨, 결제 처리 중입니다"
- Payment API → "결제 정보 없음"
- 또는 Order의 status = "AWAITING_PAYMENT"로 표시

**이 상태에서의 문제점**:
1. "결제 중인가?", "결제됨?" 상태가 명확하지 않음
2. 고객이 같은 주문을 또 생성할 수 있음 (API 멱등성 관리 필요)
3. 다른 서비스도 이 Order를 볼 수 있음 (중간 상태 노출)

**해결 방법**:
- **상태 명확화**: Order.status = "PENDING", "PAYMENT_PROCESSING", "COMPLETED", "FAILED" 등 세분화
- **멱등성 보장**: 같은 요청에는 같은 응답 (요청 ID로 중복 방지)
- **상태 조회 API**: "이 주문은 현재 어느 단계인가?" 조회 가능하게 설계
- **타임아웃**: 일정 시간 PENDING이면 자동 취소 후 보상

즉, Saga 도입 시 **중간 상태 관리가 필수**입니다.
</details>

---

<div align="center">

**[⬅️ 이전: 서비스 간 외래키 — 물리적 무결성 없이 일관성 보장](../data-management/06-cross-service-foreign-keys.md)** | **[홈으로 🏠](../README.md)** | **[다음: Choreography Saga — 이벤트 기반 자율 실행 ➡️](./02-choreography-saga.md)**

</div>

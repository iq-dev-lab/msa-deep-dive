# 04. Choreography vs Orchestration — 선택 기준

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Choreography와 Orchestration의 근본적인 차이는 무엇인가?
- 내 시스템에 Choreography가 적합한가, 아니면 Orchestration이 필요한가?
- 서비스 수, Saga 단계 수에 따라 선택 기준이 달라지는가?
- 각 패턴의 장단점을 정량적으로 비교할 수 있는가?
- 처음에는 Choreography로 시작했는데 나중에 Orchestration으로 변경해야 할 경우는?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

Choreography와 Orchestration 중 어느 것을 선택하느냐는 시스템 아키텍처의 근본적인 방향을 결정합니다. 단순히 "이 패턴이 더 좋다"는 식의 판단은 위험하며, 비즈니스 요구사항, 시스템 복잡도, 팀 역량, 개발 단계 등 여러 요소를 균형있게 고려해야 합니다.

처음부터 잘못된 선택을 하면, 나중에 변경하기 매우 어렵습니다. Choreography로 시작한 시스템을 Orchestration으로 바꾸려면, 이미 메시지 브로커에 익숙해진 팀의 사고방식부터 코드 구조까지 모두 바꿔야 합니다. 따라서 **초기 선택이 매우 중요**합니다.

또한 실무에서는 순수한 Choreography나 Orchestration만 사용하는 경우는 드물고, 대부분 **하이브리드 방식**을 취합니다. 핵심 비즈니스 흐름은 Orchestration으로 제어하고, 주변 기능은 Choreography로 연결하는 식입니다. 따라서 각 패턴의 특성을 정확히 이해하고 상황에 맞게 조합할 수 있어야 합니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ 잘못된 접근 1: 모든 Saga를 Choreography로만 처리하려는 시도
// 서비스가 늘어나고 단계가 복잡해지면서 흐름 파악 불가능

@Service
public class OrderEventHandler {
    
    @KafkaListener(topics = "order-events")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Step 1: 주문 생성 처리
        Order order = createOrder(event);
        
        // 자동으로 PaymentService가 이벤트를 받을 것이라고 가정
        publishOrderProcessedEvent(order);
    }
}

@Service
public class PaymentEventHandler {
    
    @KafkaListener(topics = "order-processed-events")
    public void handleOrderProcessed(OrderProcessedEvent event) {
        // Step 2: 결제 처리
        Payment payment = processPayment(event);
        
        // 자동으로 InventoryService가 이벤트를 받을 것이라고 가정
        publishPaymentCompletedEvent(payment);
    }
}

@Service
public class InventoryEventHandler {
    
    @KafkaListener(topics = "payment-completed-events")
    public void handlePaymentCompleted(PaymentCompletedEvent event) {
        // Step 3: 재고 예약
        Inventory inventory = reserveInventory(event);
        
        publishInventoryReservedEvent(inventory);
    }
}

// 더 추가됨... ShippingEventHandler, NotificationEventHandler, ...

/*
❌ 문제점:
T1: 새로운 요구사항 발생
    "결제 완료 후 고객에게 이메일을 보내되, 
     고객의 VIP 등급에 따라 결제 수수료 할인을 해야 한다"

T2: 코드 변경 필요
    - OrderEventHandler에는 영향 없음
    - PaymentEventHandler에 VIP 로직 추가?
    - NotificationHandler도 PaymentCompleted를 구독?
    - 두 서비스의 순서는?
    - 둘 다 실패하면 보상은?
    
T3: 흐름이 복잡해짐 (이벤트 체인이 트리 구조로)
    OrderCreated
      ├─ PaymentService → PaymentCompleted
      │   ├─ InventoryService → InventoryReserved
      │   │   ├─ ShippingService → ShipmentCreated
      │   │   │   ├─ NotificationService (이메일)
      │   │   │   └─ ReviewService (후기 요청)
      │   │   └─ BillingService (청구)
      │   ├─ VIPDiscountService (VIP 처리)  ← 언제 실행?
      │   └─ AuditService (감시)
      └─ RejectNotificationService (거부 알림)  ← 언제?
      
T4: 결과
    - 어떤 이벤트를 어떤 서비스가 구독해야 하는지 불명확
    - 이벤트 순서가 뒤바뀌면 논리 오류 (결제 전에 할인?)
    - 보상 흐름을 정의할 수 없음 (이벤트 체인이 너무 복잡)
    - 새 서비스 추가 시 기존 코드가 영향을 받을 수 있음 (강결합 위험)
    - 전체 흐름을 한눈에 파악하는 문서 만들기 어려움
*/
```

```java
// ❌ 잘못된 접근 2: 간단한 Saga도 Orchestration으로 처리하려는 시도
// 불필요한 복잡도 추가, 개발 속도 저하

@Service
public class SimpleOrderOrchestrator {
    
    // 이메일 발송 같은 단순한 작업도 Orchestrator에서 제어
    public void executeEmailNotificationSaga(OrderRequest request) {
        String sagaId = UUID.randomUUID().toString();
        OrderSaga saga = new OrderSaga(sagaId);
        saga.setState(SagaState.PENDING);
        sagaRepository.save(saga);
        
        // Step 1: 이메일 발송 준비
        try {
            EmailRequest emailRequest = new EmailRequest(
                request.getCustomerEmail(),
                "주문 확인"
            );
            emailService.sendEmail(emailRequest);
            
            saga.setState(SagaState.EMAIL_SENT);
            sagaRepository.save(saga);
            
            // Step 2: 사용자 알림 전송
            pushService.sendNotification(request.getCustomerId(), "이메일 발송됨");
            
            saga.setState(SagaState.NOTIFICATION_SENT);
            sagaRepository.save(saga);
            
            saga.setState(SagaState.SAGA_COMPLETED);
            sagaRepository.save(saga);
            
        } catch (Exception e) {
            // 보상 시작... (매우 복잡)
        }
    }
}

/*
❌ 문제점:
- Orchestrator와 DB 테이블이 계속 증가
- 단순한 이메일 발송을 위해 3단계 상태 머신 필요?
- 보상? 이메일 발송은 취소할 수 없는데...
- 개발 속도 저하: 간단한 기능에 Saga 추가하는데 5배 시간 소요
- 팀 인지 부담: Saga 개념을 모르는 신입이 이해하기 어려움
*/
```

---

## ✨ 올바른 접근 (After)

```java
// ✅ 올바른 접근: 복잡도와 중요도에 따라 적절한 패턴 선택

/**
 * 의사결정 트리:
 * 
 * 1. Saga가 필요한가? (분산 트랜잭션)
 *    ├─ NO → 단순한 API 호출로 충분
 *    └─ YES → 다음 질문으로
 * 
 * 2. 단계가 많고(5+) 복잡한가?
 *    ├─ YES → Orchestration 권장
 *    │        (명확한 흐름 제어, 조건부 로직, 병렬처리 필요)
 *    └─ NO → 다음 질문으로
 * 
 * 3. 각 단계가 독립적이고 순서가 중요하지 않은가?
 *    ├─ YES → Choreography 권장
 *    │        (느슨한 결합, 간단한 흐름)
 *    └─ NO → Orchestration 권장
 *           (명시적 순서 제어 필요)
 */

// Case 1: 단순한 Saga → Choreography 사용
@Service
public class SimpleOrderChoreography {
    
    @KafkaListener(topics = "order-created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // 간단한 단계: 주문 생성 → 결제 → 재고 (3단계, 순서 중요)
        // 하지만 각 단계가 독립적이고, 조건부 로직 없음
        Payment payment = processPayment(event);
        publishPaymentCompletedEvent(payment);
    }
    
    @KafkaListener(topics = "payment-completed")
    public void handlePaymentCompleted(PaymentCompletedEvent event) {
        Inventory inventory = reserveInventory(event);
        publishInventoryReservedEvent(inventory);
    }
    
    // Choreography는 이 정도가 최대
    // 5단계를 넘어가면 흐름 파악이 어려움
}

// Case 2: 복잡한 Saga → Orchestration 사용
@Service
public class ComplexOrderOrchestration {
    
    /**
     * 복잡한 흐름:
     * 1. 주문 생성
     * 2. 결제 처리
     * 3-1. VIP 고객이면 할인 적용 (조건부)
     * 3-2. 일반 고객이면 기본 가격 (조건부)
     * 4. 재고 예약
     * 5-1. 재고 충분하면 배송 생성 (조건부)
     * 5-2. 재고 부족하면 보상 시작 (조건부)
     * 6. 배송 및 이메일 발송
     * 7-1. 성공하면 정산 (조건부)
     * 7-2. 실패하면 환불 (조건부)
     */
    
    public void startComplexOrderSaga(OrderRequest request) {
        String sagaId = UUID.randomUUID().toString();
        OrderSaga saga = new OrderSaga(sagaId);
        saga.setData(request);
        sagaRepository.save(saga);
        
        executeSaga(sagaId);
    }
    
    private void executeSaga(String sagaId) {
        OrderSaga saga = sagaRepository.findById(sagaId).orElseThrow();
        
        switch (saga.getState()) {
            case PENDING:
                createOrder(saga);
                break;
                
            case ORDER_CREATED:
                processPayment(saga);
                break;
                
            case PAYMENT_COMPLETED:
                // 조건부 로직: VIP 확인 후 할인 적용
                CustomerType customerType = getCustomerType(saga.getCustomerId());
                if (customerType == CustomerType.VIP) {
                    applyVIPDiscount(saga);  // 별도 단계
                } else {
                    saga.setState(SagaState.DISCOUNT_EVALUATED);
                    sagaRepository.save(saga);
                }
                reserveInventory(saga);
                break;
                
            case DISCOUNT_EVALUATED:
                reserveInventory(saga);
                break;
                
            case INVENTORY_RESERVED:
                // 조건부 로직: 재고 충분 확인
                if (saga.getReservationId() != null) {
                    createShipment(saga);
                } else {
                    saga.setState(SagaState.COMPENSATION_STARTED);
                    sagaRepository.save(saga);
                    executeCompensation(saga);
                }
                break;
                
            // ... 더 많은 단계들
        }
    }
}

// Case 3: 하이브리드 접근 (권장)
@Service
public class HybridSagaApproach {
    
    /**
     * 핵심 비즈니스 로직 (Orchestration)
     * - 주문 생성
     * - 결제 처리
     * - 재고 예약
     * 
     * 주변 기능 (Choreography)
     * - 이메일 발송
     * - 푸시 알림
     * - 분석 데이터 수집
     */
    
    // 핵심: Orchestrator가 명시적으로 제어
    public void startOrderSaga(OrderRequest request) {
        String sagaId = UUID.randomUUID().toString();
        OrderSaga saga = new OrderSaga(sagaId);
        sagaRepository.save(saga);
        
        // 주문 생성
        Order order = createOrder(request, sagaId);
        
        // 결제 처리
        try {
            Payment payment = processPayment(order, sagaId);
            
            // 재고 예약
            Inventory inventory = reserveInventory(order, sagaId);
            
            // 성공 → OrderCompletedEvent 발행
            OrderCompletedEvent completedEvent = new OrderCompletedEvent(
                order.getId(),
                payment.getId(),
                inventory.getId(),
                sagaId
            );
            publishEvent(completedEvent);  // 주변 기능이 구독
            
        } catch (Exception e) {
            // 실패 → OrderFailedEvent 발행
            OrderFailedEvent failedEvent = new OrderFailedEvent(
                order.getId(),
                e.getMessage(),
                sagaId
            );
            publishEvent(failedEvent);  // 주변 기능이 구독
        }
    }
    
    // 주변 기능: Choreography로 독립적 처리
    @KafkaListener(topics = "order-completed")
    public void handleOrderCompleted(OrderCompletedEvent event) {
        // 이메일 발송 (독립적)
        sendOrderConfirmationEmail(event);
        
        // 푸시 알림 (독립적)
        sendPushNotification(event);
        
        // 분석 (독립적)
        recordOrderAnalytics(event);
        
        // 실패해도 주문 처리에는 영향 없음
    }
    
    @KafkaListener(topics = "order-failed")
    public void handleOrderFailed(OrderFailedEvent event) {
        // 고객에게 실패 알림
        sendFailureNotification(event);
        
        // 관리자에게 알림
        notifyAdministrator(event);
    }
}
```

---

## 🔬 내부 동작 원리 — 선택 기준의 정량적 분석

### 복잡도 비교: 단계 수와 이해도의 관계

```
┌──────────────────────────────────────────────────────────────┐
│         Saga 단계 수에 따른 패턴 선택 가이드                 │
└──────────────────────────────────────────────────────────────┘

이해도
  │
100%│   ┌─────── Orchestration 명확
    │   │        (흐름이 한눈에 보임)
    │   │
 80%├───┼─────── 혼합 구간
    │   │        (Choreography 흐름 파악 어려워짐)
    │   │
 60%├───┼─────┬─ Choreography 한계
    │   │     │  (단계마다 다른 파일에 분산)
    │   │     │
    │   │     │
 40%├───┤     └─ 개발 속도 저하
    │   │        (메인테넌스 어려움)
    │
 20%├───┴─ 완전 혼란
    │      (흐름 파악 불가)
    │
  0%└───────────────────────────────────────────────
      1   2   3   4   5   6   7   8   9  10+  Saga 단계 수

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

단계 1-3:    ✅ Choreography 충분
             (각 단계가 명확, 이벤트 체인이 선형)
             
단계 4-5:    ⚠️  선택 필요
             - 조건부 로직 없음 → Choreography OK
             - 조건부 로직 있음 → Orchestration 권장
             
단계 6+:     ❌ Orchestration 필수
             (Choreography는 흐름 추적 거의 불가)
```

### 서비스 간 의존성 복잡도

```
┌──────────────────────────────────────────────────────────────┐
│                   서비스 수와 의존성                         │
└──────────────────────────────────────────────────────────────┘

Choreography 의존성 그래프 (서비스 4개):

    OrderService ───→ Event1 ───→ PaymentService ───→ Event2 ───→ InventoryService ───→ Event3 ───→ ShippingService

    ✅ 장점: 각 서비스가 이벤트만 알면 됨 (느슨한 결합)
    ❌ 단점: 전체 흐름을 파악하려면 4개 파일 모두 봐야 함


Orchestration 의존성 그래프 (서비스 4개):

                    ┌─→ PaymentService
                    │
    Orchestrator ───├─→ InventoryService
                    │
                    └─→ ShippingService

    ✅ 장점: Orchestrator 하나만 보면 전체 흐름 파악 (명확)
    ❌ 단점: Orchestrator가 모든 서비스를 알아야 함 (강결합)


더 복잡한 경우 (조건부 로직, 병렬처리):

    Choreography는 이벤트 체인이 복잡한 트리 구조로 변함
    → 흐름 파악 불가능
    
    Orchestration은 switch/if 문으로 표현 가능
    → 명확하게 제어 가능
```

### 성능 특성 비교

```
┌──────────────────────────────────────────────────────────────┐
│              처리 시간과 리소스 사용                         │
└──────────────────────────────────────────────────────────────┘

Choreography (4단계, 각 100ms):
┌─────────┬─────────┬─────────┬─────────┐
│ Order   │Payment  │Inventory│Shipping │
└────┬────┴────┬────┴────┬────┴────┬────┘
     Event1     Event2     Event3     Event4
     
     └─────────────────────────────────→ 총 400ms+ (순차)
     
비동기 처리 가능하므로 병렬성 있음, 하지만 Kafka 지연 추가


Orchestration (4단계, 각 100ms):
     
Request →[Orchestrator]→ Service1, Service2, Service3, Service4 (병렬 가능)
     
    └─ 총 100-200ms (병렬 처리 시)
    
하지만 Orchestrator가 각각의 응답을 기다려야 함 (블로킹)


결론:
- Choreography: 높은 처리량, 낮은 응답 시간 (비동기)
- Orchestration: 낮은 처리량, 제어된 응답 시간 (동기/대기)
```

---

## 💻 선택 기준 체크리스트

```java
// 패턴 선택을 돕는 의사결정 코드

public class SagaPatternSelector {
    
    public enum SelectedPattern {
        CHOREOGRAPHY,
        ORCHESTRATION,
        HYBRID
    }
    
    public SelectedPattern selectPattern(SagaCharacteristics saga) {
        int score = 0;
        
        // 1. 단계 수 (점수: 0-30)
        score += calculateStepComplexityScore(saga.getStepCount());
        
        // 2. 조건부 로직 (점수: 0-20)
        if (saga.hasConditionalLogic()) {
            score += 20;  // Orchestration에 유리
        }
        
        // 3. 병렬 처리 필요 (점수: 0-15)
        if (saga.hasParallelSteps()) {
            score += 15;  // Orchestration에 유리
        }
        
        // 4. 서비스 간 느슨한 결합 필요 (점수: 0-20)
        if (saga.needsLooseCoupling()) {
            score -= 15;  // Choreography에 유리
        }
        
        // 5. 흐름 추적의 중요성 (점수: 0-15)
        if (saga.requiresDetailedTracking()) {
            score += 15;  // Orchestration에 유리
        }
        
        // 점수 해석
        if (score <= 20) {
            return SelectedPattern.CHOREOGRAPHY;
        } else if (score >= 50) {
            return SelectedPattern.ORCHESTRATION;
        } else {
            return SelectedPattern.HYBRID;
        }
    }
    
    private int calculateStepComplexityScore(int stepCount) {
        // 단계가 많을수록 Orchestration 점수 증가
        return Math.min(stepCount * 3, 30);  // 최대 30점
    }
}

// 사용 예
SagaCharacteristics orderSaga = new SagaCharacteristics();
orderSaga.setStepCount(3);  // 단계 3개
orderSaga.setHasConditionalLogic(false);  // 조건부 로직 없음
orderSaga.setHasParallelSteps(false);  // 병렬 처리 없음
orderSaga.setNeedsLooseCoupling(true);  // 느슨한 결합 원함

SelectedPattern pattern = selector.selectPattern(orderSaga);
// 결과: CHOREOGRAPHY (조건부 없고, 단계 적음)
```

---

## 📊 패턴 비교

| 항목 | Choreography | Orchestration | 하이브리드 |
|------|------|------|------|
| **흐름 명확도** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **구현 난이도** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **추적 난이도** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **결합도** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **확장성** | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| **병렬 처리** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **적합한 단계 수** | 1-4 | 4+ | 혼합 |
| **적합한 조건부** | 없음 | 많음 | 혼합 |
| **팀 학습곡선** | 낮음 | 높음 | 중간 |

---

## ⚖️ 트레이드오프 분석

### Choreography 선택 시
- ✅ **느슨한 결합**: 서비스별 독립 배포 가능
- ✅ **높은 가용성**: 일부 서비스 지연이 다른 부분 영향 없음
- ✅ **빠른 개발**: 간단한 이벤트 기반 구현
- ✅ **쉬운 확장**: 새 서비스 추가 용이 (기존 코드 수정 불필요)
- ❌ **흐름 파악 어려움**: 전체 흐름이 분산된 이벤트 핸들러에 흩어짐
- ❌ **복잡한 보상**: 보상 체인이 얽혀서 이해 어려움
- ❌ **멱등성 관리**: 모든 핸들러가 멱등성 보장해야 함
- ❌ **버그 추적 어려움**: 분산 환경에서 장애 원인 파악 복잡

### Orchestration 선택 시
- ✅ **명확한 흐름**: 한 곳(Orchestrator)에서 전체 로직 파악
- ✅ **쉬운 조건부 처리**: if/switch로 복잡한 로직 표현
- ✅ **병렬 처리**: 여러 서비스를 동시에 실행 가능
- ✅ **빠른 디버깅**: 중앙 로그에서 장애 원인 즉시 파악
- ✅ **상태 머신**: 상태 정의로 논리 오류 감소
- ❌ **강한 결합**: Orchestrator가 모든 서비스 알아야 함
- ❌ **확장성 제한**: 새 서비스 추가 시 Orchestrator 수정 필요
- ❌ **Orchestrator SPOF**: Orchestrator 장애 → 전체 Saga 중단
- ❌ **높은 개발 비용**: 상태 머신, 보상 로직 등 구현 복잡

### 하이브리드 선택 시
- ✅ **각 부분에 최적 패턴**: 핵심은 Orchestration, 주변은 Choreography
- ✅ **균형 있는 설계**: 명확성과 유연성의 조화
- ✅ **점진적 확장**: 처음엔 Choreography, 복잡해지면 Orchestration 추가
- ❌ **관리 복잡도**: 두 패턴을 모두 이해해야 함
- ❌ **일관성 유지**: 어디까지 Orchestration, 어디서 Choreography?의 판단 필요
- ❌ **팀 역량 요구**: 높은 수준의 아키텍처 이해 필요

---

## 📌 핵심 정리

✅ **Choreography**는 각 서비스가 이벤트에 반응하여 자율적으로 실행하는 패턴입니다. 느슨한 결합과 높은 확장성을 제공하지만, 단계가 많거나 복잡한 로직에서는 흐름 추적이 어렵습니다.

✅ **Orchestration**은 중앙의 Orchestrator가 명시적으로 각 서비스를 제어하는 패턴입니다. 명확한 흐름과 쉬운 디버깅을 제공하지만, 강한 결합과 확장성 제한이 따릅니다.

✅ **선택 기준**:
- **단계 1-3개, 조건부 없음**: Choreography
- **단계 4-5개, 조건부 있음**: 선택 필요 (일반적으로 Orchestration)
- **단계 6개 이상, 복잡한 로직**: Orchestration
- **혼합 흐름**: Hybrid (핵심=Orchestration, 주변=Choreography)

✅ **실전 권장사항**:
1. **초기 단계**: Choreography로 빠르게 시작
2. **복잡해질 때**: Orchestration으로 단계적 전환
3. **성숙 단계**: Hybrid 모델로 균형 있게 설계
4. **문서화**: 선택한 패턴과 그 이유를 명확히 기록

✅ **팀의 역량에 따라**:
- 초급 팀: Choreography (구현이 단순)
- 중급 팀: Hybrid (선택과 조합 능력)
- 고급 팀: Orchestration + Choreography (상황별 최적화)

---

## 🤔 생각해볼 문제

**Q1.** Saga의 단계가 처음에는 3개였는데, 요구사항이 늘어나면서 8개가 되었다면? Choreography에서 Orchestration으로 마이그레이션하려면 어떤 단계를 거쳐야 할까?

<details><summary>해설 보기</summary>
**마이그레이션 전략**:

1. **Phase 1: 현황 파악**
   - 기존 Choreography의 모든 이벤트 흐름 문서화
   - 각 단계 간의 의존성 파악
   - 실패 케이스별 보상 흐름 정리

2. **Phase 2: Orchestrator 설계**
   - 8개 단계를 상태 머신으로 모델링
   - Orchestrator DB 스키마 설계
   - 조건부 로직 명시

3. **Phase 3: 점진적 마이그레이션**
   ```
   방식 1: 이벤트 → Orchestrator → 이벤트 (중간 계층 추가)
   - 기존 이벤트는 유지
   - 새 Orchestrator가 이벤트를 수신
   - Orchestrator가 다음 이벤트 발행
   - 사실상 Orchestration으로 전환
   
   방식 2: 새 Saga부터 Orchestration 적용
   - 기존 Choreography는 유지
   - 새로운 Saga는 Orchestration으로
   - 시간을 들여 완전히 전환
   ```

4. **Phase 4: 테스트 및 검증**
   - 모든 단계 조합에 대한 테스트
   - 보상 흐름 검증
   - 성능 비교 (응답 시간, 처리량)

5. **Phase 5: 배포 및 모니터링**
   - 카나리 배포 (신규 주문은 Orchestration)
   - 기존 주문은 Choreography로 계속 처리
   - 메트릭 모니터링

**핵심**: 한 번에 모든 Saga를 바꾸는 것은 위험하므로, 신규부터 적용하고 점진적으로 기존 것을 마이그레이션하는 것이 안전합니다.
</details>

**Q2.** "주문 → 결제 → 배송"은 Choreography로 충분한데, 별도로 "매일 자정에 정산 처리"라는 배치 작업이 있다면? 이를 Saga에 포함시켜야 할까?

<details><summary>해설 보기</summary>
**답**: 이것은 Saga가 아니라 **별도의 배치 프로세스**로 분리해야 합니다.

**이유**:
1. **Saga의 정의**: 분산 트랜잭션을 처리하기 위한 패턴
   - 여러 서비스의 **동기적** 작업을 조율
   - 실시간 요청에 대한 응답

2. **정산 처리**: 배치 작업
   - 매일 자정에 **배치**로 실행
   - 주문과는 느슨하게 결합
   - 실시간 응답 불필요

**올바른 설계**:
```
[Order Saga] (실시간)
├─ Order 생성
├─ Payment 처리
└─ Shipping 생성

[Settlement Batch] (배치, 스케줄러)
├─ 어제의 모든 주문 조회
├─ 각 주문별 정산 금액 계산
├─ 파트너사에 송금
└─ 정산 로그 기록
```

**아키텍처**:
```java
// 실시간 Saga
@Service
public class OrderChoreography {
    // Order → Payment → Shipping
}

// 배치 처리 (별도)
@Service
public class SettlementBatchService {
    @Scheduled(cron = "0 0 * * *")  // 매일 자정
    public void executeSettlement() {
        // 배치 로직
    }
}
```

**연결 방식**:
- Order Saga 완료 시 Settlement이 이미 DB에 기록되었으므로
- Settlement Batch는 단순히 그 기록을 조회해서 처리

**즉**: Saga와 Batch는 별도의 스코프이며, Saga는 실시간 요청 처리, Batch는 집계/정산 작업으로 분리합니다.
</details>

**Q3.** Orchestration을 사용 중인데, 갑자기 "우리 비즈니스의 일부를 외부 회사에 아웃소싱하게 됐다. 새로운 외부 서비스가 필요하다"는 요구사항이 발생했다. 기존 Orchestrator를 수정해야 할까?

<details><summary>해설 보기</summary>
**답**: 상황에 따라 다릅니다.

**Case 1: 핵심 업무 흐름의 일부**
예: 배송 대행사로 변경 (주문 → 결제 → [배송 서비스 변경])

```
기존: OrderSagaOrchestrator가 ShippingService 호출
      → 새 서비스로 변경 (ExternalShippingService)
      → Orchestrator 코드 수정 (switch문에 조건 추가)
      → 기존 고객과 신규 고객 경로 분기
```

**Case 2: 부가 기능 추가**
예: 외부 배송사는 추가 기능만 (배송 추적, 보험 등)

```
기존: Order Saga 완료 후 Event 발행
      → 새로운 리스너: ExternalShippingEnhancementService
      → Choreography 방식으로 부가 기능 추가
      → Orchestrator 수정 불필요
```

**올바른 선택 기준**:
- **핵심 흐름 변경** → Orchestrator 수정 (필요하면 Orchestration 유지)
- **부가 기능 추가** → Event 기반 Choreography 추가 (Orchestrator 건드리지 않음)

**권장 아키텍처**:
```
[OrderSagaOrchestrator] (핵심)
├─ 주문 생성
├─ 결제 처리
└─ [배송 추상화] ← 어떤 배송사든 인터페이스로 추상화
    ├─ InternalShippingService
    └─ ExternalShippingService

[Event-based Enhancement] (부가기능)
└─ OrderCompletedEvent 구독
    ├─ NotificationService
    ├─ AnalyticsService
    └─ ExternalServiceEnhancement (신규)
```

**핵심**: 핵심 흐름은 Orchestrator에서 명확히, 부가기능은 Event로 느슨하게 연결하면 변화에 유연하게 대응 가능합니다.
</details>

---

<div align="center">

**[⬅️ 이전: Orchestration Saga — 중앙 오케스트레이터 설계](./03-orchestration-saga.md)** | **[홈으로 🏠](../README.md)** | **[다음: 보상 트랜잭션 설계 — 멱등성과 취소 불가 단계 ➡️](./05-compensating-transaction.md)**

</div>

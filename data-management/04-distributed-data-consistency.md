# 04. 분산 데이터 일관성 — ACID vs BASE

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- ACID와 BASE의 근본적인 차이는 무엇인가?
- CAP Theorem에서 왜 항상 3가지를 모두 가질 수 없는가?
- Eventual Consistency가 의미하는 바와 실제 구현은?
- 강한 일관성이 필수인 경우와 허용 가능한 경우는?
- 충돌 해결(Conflict Resolution) 전략의 구현은?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

**모놀리스 환경에서 데이터 일관성은 단순했습니다.** 한 데이터베이스가 ACID 트랜잭션으로 모든 것을 보장했습니다. 하나의 금융 거래도 원자적(Atomic)으로 처리되고, 일관성(Consistency)도 보장되고, 격리성(Isolation)도 완벽했습니다.

**하지만 마이크로서비스 환경은 다릅니다.** 여러 서비스가 여러 데이터베이스에 데이터를 저장합니다. 주문을 생성할 때:
1. 주문 서비스에 주문 저장
2. 결제 서비스에 결제 정보 저장
3. 재고 서비스에서 재고 차감
4. 배송 서비스에 배송 작업 생성

이 모든 작업이 하나의 트랜잭션으로 묶일 수 없습니다. 네트워크 지연, 서비스 장애, 동시성 문제로 인해 일부는 성공하고 일부는 실패할 수 있습니다.

**따라서 MSA는 ACID를 포기하고 BASE를 받아들여야 합니다.** 이는 단순히 "데이터가 일관성이 없어도 된다"는 의미가 아니라, "일관성을 다르게 정의한다"는 의미입니다. 이를 정확히 이해하고 구현해야 MSA 환경에서 안정적인 시스템을 만들 수 있습니다.

---

## 😱 흔한 실수 (Before — ACID 강요)

```java
// BEFORE: MSA 환경에서도 ACID를 강요하려는 시도

@Service
public class OrderServiceWithACID {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private PaymentServiceClient paymentServiceClient;
    
    @Autowired
    private InventoryServiceClient inventoryServiceClient;
    
    // 분산 트랜잭션 시도 (2PC - Two-Phase Commit)
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1단계: 주문 생성
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        order.setTotalAmount(request.getTotalAmount());
        Order savedOrder = orderRepository.save(order);
        
        try {
            // 2단계: 결제 요청 (분산 트랜잭션)
            PaymentResponse payment = paymentServiceClient
                .processPayment(request.getCustomerId(), request.getTotalAmount());
            
            if (!payment.isSuccessful()) {
                throw new PaymentFailedException("결제 실패");
            }
            
            // 3단계: 재고 차감 (분산 트랜잭션)
            InventoryResponse inventory = inventoryServiceClient
                .reduceInventory(request.getProductId(), request.getQuantity());
            
            if (!inventory.isSuccessful()) {
                throw new InsufficientInventoryException("재고 부족");
            }
            
            // 모든 작업 완료 (커밋)
            return savedOrder;
        } catch (Exception e) {
            // 문제: 주문은 이미 저장됨 (롤백 불가)
            // 결제는 실패 (부분 실패 상태)
            // 재고는 차감되지 않음
            // 시스템이 불일치 상태에 빠짐
            
            throw new OrderCreationException("주문 생성 실패", e);
        }
    }
}

// 문제점 분석
/*
시나리오: 결제는 성공했는데 재고 서비스가 다운됨

1. 주문 저장: ✅ (PostgreSQL 트랜잭션)
2. 결제: ✅ (결제 서비스가 처리)
3. 재고: ❌ (재고 서비스 다운)

결과:
- 고객은 결제되었다 (결제 서비스 데이터)
- 상품은 배송되지 않을 것 (재고 차감 안 됨)
- 상품은 여전히 재고에 있다 (재고 서비스)
- 데이터 불일치!

이 불일치를 해결할 방법이 없음
(각각 다른 DB, 다른 서비스이므로 롤백 불가)
*/
```

2PC (Two-Phase Commit) 시도의 문제:

```
Prepare Phase (1단계):
┌──────────────────────────────────────────────┐
│ Coordinator (Order Service)                  │
│ "모든 서비스가 커밋 준비 가능한가?"           │
└──────────────────────────────────────────────┘
    │              │                │
    ▼              ▼                ▼
Payment Service  Inventory Service  Shipping Service
"예, 준비됨"      "예, 준비됨"      "아니오, 다운됨"
    │              │                │
    └──────────────┴────────────────┘
                   │
                   ▼
모두 "준비됨"이어야만 Commit Phase로 진행
하나라도 "아니오"면 모두 롤백

문제:
❌ 결제 서비스가 이미 돈을 받음 (롤백 불가)
❌ 하드웨어 장애로 어떤 서비스 응답 없음 (무한 대기)
❌ 성능 저하 (Locking으로 인한 병목)
```

---

## ✨ 올바른 접근 (After — BASE와 Eventual Consistency)

```java
// AFTER: BASE (Basically Available, Soft state, Eventual consistency)

// 1단계: 이벤트 기반 아키텍처로 전환
@Service
public class OrderServiceWithEventSourcing {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private EventStore eventStore;
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1단계: 주문 생성 (로컬 DB만, ACID 보장)
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        order.setTotalAmount(request.getTotalAmount());
        order.setStatus("PENDING");
        Order savedOrder = orderRepository.save(order);
        
        // 2단계: 이벤트 저장 (원자적)
        OrderCreatedEvent event = new OrderCreatedEvent(
            savedOrder.getId(),
            request.getCustomerId(),
            request.getTotalAmount(),
            request.getProductId(),
            request.getQuantity()
        );
        eventStore.save(event);
        
        // 3단계: 이벤트 발행 (비동기)
        // 이제 각 서비스가 비동기로 구독
        eventPublisher.publishEvent(event);
        
        // 즉시 반환 (주문 생성됨)
        return savedOrder;
        // 결제, 재고는 나중에 비동기로 처리됨
    }
}

// 2단계: 각 서비스가 독립적으로 이벤트 처리
@Service
public class PaymentEventListener {
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Autowired
    private PaymentServiceClient paymentServiceClient;
    
    @EventListener
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        // 결제 서비스가 비동기로 결제 처리
        try {
            PaymentResponse response = paymentServiceClient
                .processPayment(event.getCustomerId(), event.getTotalAmount());
            
            Payment payment = new Payment();
            payment.setOrderId(event.getOrderId());
            payment.setAmount(event.getTotalAmount());
            payment.setStatus("COMPLETED");
            paymentRepository.save(payment);
        } catch (PaymentException e) {
            // 결제 실패: 보상 트랜잭션 실행
            eventPublisher.publishEvent(
                new PaymentFailedEvent(event.getOrderId())
            );
        }
    }
}

@Service
public class InventoryEventListener {
    
    @Autowired
    private InventoryServiceClient inventoryServiceClient;
    
    @Autowired
    private InventoryRepository inventoryRepository;
    
    @EventListener
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        // 재고 서비스가 비동기로 재고 차감
        try {
            InventoryResponse response = inventoryServiceClient
                .reduceInventory(event.getProductId(), event.getQuantity());
            
            Inventory inv = inventoryRepository.findByProductId(event.getProductId());
            inv.reduceStock(event.getQuantity());
            inventoryRepository.save(inv);
        } catch (InsufficientInventoryException e) {
            // 재고 부족: 보상 트랜잭션
            eventPublisher.publishEvent(
                new InventoryReservationFailedEvent(event.getOrderId())
            );
        }
    }
}

// 3단계: 보상 트랜잭션 (Saga 패턴)
@Service
public class OrderSagaOrchestrator {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private PaymentServiceClient paymentServiceClient;
    
    @EventListener
    public void onPaymentFailed(PaymentFailedEvent event) {
        // 결제 실패 → 주문 취소
        Order order = orderRepository.findById(event.getOrderId()).orElseThrow();
        order.setStatus("PAYMENT_FAILED");
        orderRepository.save(order);
        
        // 이미 차감된 재고를 복구 (보상)
        eventPublisher.publishEvent(
            new RestoreInventoryEvent(event.getOrderId())
        );
    }
    
    @EventListener
    public void onInventoryReservationFailed(InventoryReservationFailedEvent event) {
        // 재고 부족 → 주문 취소
        Order order = orderRepository.findById(event.getOrderId()).orElseThrow();
        order.setStatus("OUT_OF_STOCK");
        orderRepository.save(order);
        
        // 이미 처리된 결제를 환불 (보상)
        eventPublisher.publishEvent(
            new RefundPaymentEvent(event.getOrderId())
        );
    }
}

// 최종 일관성 달성
/*
시간 흐름:

t=0: 주문 생성 요청
    └─> 주문 저장 (PENDING) ✅

t=1: OrderCreatedEvent 발행
    ├─> PaymentEventListener 구독
    └─> InventoryEventListener 구독

t=2~10: 비동기 처리 진행 중
    ├─ 결제 처리 중...
    └─ 재고 차감 중...

t=11: 모두 완료
    ├─ 주문: CONFIRMED ✅
    ├─ 결제: COMPLETED ✅
    └─ 재고: REDUCED ✅

최종 일관성 달성: 모든 서비스의 데이터가 일치

장점:
✅ 일부 서비스 장애 시에도 주문은 생성됨
✅ 각 서비스 독립 처리 (느슨한 결합)
✅ 높은 가용성 (Availability)
✅ 자동 복구 (보상 트랜잭션)
*/
```

---

## 🔬 내부 동작 원리 — CAP Theorem과 트레이드오프

### CAP Theorem 실제 의미

```
  ┌─────────────────────────────────────────┐
  │        분산 시스템의 3가지 특성          │
  └─────────────────────────────────────────┘
  
  C (Consistency)      A (Availability)       P (Partition Tolerance)
  강한 일관성           높은 가용성             네트워크 장애 대응
  (모든 읽기가          (항상 응답)            (분산 시스템 필수)
   최신 데이터)
      │                    │                       │
      │                    │                       │
      └────────────┬───────┴───────────┬───────────┘
                   │                   │
                   ├─ CP (일관성+장애)  │
                   │  └─ 일관성 보장하려면
                   │     일부 노드 요청 거부
                   │     (강한 일관성)
                   │
                   └─ AP (가용성+장애)  
                      └─ 가용성 보장하려면
                         최종 일관성 허용
                         (Eventual Consistency)

현실: P(네트워크 장애)는 반드시 발생
→ C와 A 중 하나를 선택해야 함
```

### ACID vs BASE 비교

```
ACID (원자성, 일관성, 격리성, 지속성):
├─ Atomicity: 모두 성공하거나 모두 실패
│  └─ "모든 계좌 이체가 원자적으로 처리"
├─ Consistency: 규칙을 위반할 수 없음
│  └─ "외래키 제약 조건은 항상 유효"
├─ Isolation: 동시 트랜잭션이 서로 간섭 없음
│  └─ "A의 쓰기가 B의 읽기를 블로킹"
└─ Durability: 커밋된 데이터는 영구적
   └─ "전원 꺼져도 데이터 복구됨"

BASE (기본 가용성, 소프트 상태, 최종 일관성):
├─ Basically Available: 항상 응답
│  └─ "조금 오래된 데이터라도 응답"
├─ Soft state: 상태가 일시적으로 불일치
│  └─ "캐시와 DB가 잠시 다를 수 있음"
└─ Eventual Consistency: 결국 일치
   └─ "시간이 지나면 모든 복제본이 일치"

사용 시나리오:
ACID → 금융 거래 (일관성 필수)
BASE → 소셜 미디어 좋아요 (약간 지연 OK)
```

### Eventual Consistency의 구현 수준

```
Level 1: 즉시 일관성 (모놀리스 ACID)
시간: 0ms
사용자 A: 계좌 1,000,000원
사용자 B: 보기 → 1,000,000원 ✅

Level 2: 약한 일관성 (캐시)
시간: ~10ms (캐시 무효화)
사용자 A: 계좌 1,000,000원 (DB 갱신)
사용자 B: 보기 → 990,000원 (캐시에서 읽음, 이전 값)
시간: 10ms 후
사용자 B: 다시 보기 → 1,000,000원 ✅

Level 3: 최종 일관성 (이벤트 기반 비동기)
시간: ~100~1000ms
사용자 A: 계좌 1,000,000원 (DB 갱신, 이벤트 발행)
사용자 B: 보기 → 990,000원 (아직 동기화 안 됨)
시간: 500ms 후
사용자 B: 다시 보기 → 1,000,000원 ✅

Level 4: 약한 일관성 (최종 불일치)
시간: ~minutes~hours (멀티 데이터센터)
DC1 (미국): 계좌 1,000,000원
DC2 (유럽): 보기 → 990,000원 (아직 복제 안 됨)
시간: 30분 후
DC2: 다시 보기 → 1,000,000원 ✅
```

### 충돌 해결 전략 (Conflict Resolution)

```
시나리오: 분산 환경에서 같은 데이터를 동시에 수정

사용자: 계좌 잔액 1,000,000원
- 지점 A에서 500,000원 출금
- 지점 B에서 300,000원 출금
동시에 발생!

┌─────────────────────────────────────────────────┐
│              충돌 해결 전략                       │
└─────────────────────────────────────────────────┘

1️⃣ Last Write Wins (LWW):
   └─ 가장 최근 쓰기 우선
   
   시간:      0ms           100ms           200ms
   지점 A:   -500,000      (쓰기 완료)
   지점 B:              -300,000           (쓰기 완료)
   
   최종: 700,000원 (300,000원 출금만 반영)
   문제: 500,000원 출금은 사라짐 ❌

2️⃣ First Write Wins (FWW):
   └─ 첫 쓰기 우선 (나머지는 경고 또는 거부)
   
   최종: 500,000원 (첫 출금만 반영)
   문제: 300,000원 출금은 실패 ❌

3️⃣ Vector Clock / Causal Ordering:
   └─ 인과 관계 추적으로 충돌 감지
   
   지점 A: [A:1, B:0] - A 첫 쓰기
   지점 B: [A:0, B:1] - B 첫 쓰기
   
   → 동시 쓰기 감지! (인과 관계 없음)
   → 애플리케이션 레벨에서 병합 처리

4️⃣ CRDTs (Conflict-free Replicated Data Types):
   └─ 충돌 없이 자동 병합 가능한 데이터 구조
   
   G-Counter (수치 누적):
   지점 A: +500,000
   지점 B: +300,000
   
   병합: 정렬된 로그로 누적
   최종 잔액: 1,000,000 - 500,000 - 300,000 = 200,000 ✅

5️⃣ 멀티 마스터 복제:
   └─ 모든 변경을 기록하고 충돌만 해결
   
   Changelog:
   ├─ [t=100] 지점 A: -500,000
   ├─ [t=100] 지점 B: -300,000
   ├─ [t=200] 충돌 감지
   └─ [t=201] 사용자에게 선택 요청
      "어느 출금을 취소할까요?"
```

---

## 💻 Java 실전 코드

```java
// Eventual Consistency 구현: 재시도와 타임아웃
@Service
public class DistributedTransactionService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Autowired
    private CircuitBreaker circuitBreaker;
    
    private static final int MAX_RETRIES = 3;
    private static final long RETRY_DELAY_MS = 1000;
    
    public void processDistributedTransaction(Order order) {
        // 1단계: 로컬 트랜잭션 (ACID)
        Order savedOrder = saveOrderLocally(order);
        
        // 2단계: 원격 서비스 호출 (Eventual Consistency)
        processPaymentWithRetry(order.getCustomerId(), order.getTotalAmount());
        
        // 3단계: 재고 차감 (Eventual Consistency)
        reduceInventoryWithRetry(order.getProductId(), order.getQuantity());
    }
    
    private void processPaymentWithRetry(Long customerId, BigDecimal amount) {
        int attempts = 0;
        
        while (attempts < MAX_RETRIES) {
            try {
                PaymentResponse response = circuitBreaker.executeSupplier(
                    () -> restTemplate.postForObject(
                        "http://payment-service/api/payments",
                        new PaymentRequest(customerId, amount),
                        PaymentResponse.class
                    )
                );
                
                if (response.isSuccessful()) {
                    return;  // 성공
                } else {
                    throw new PaymentException("결제 실패: " + response.getMessage());
                }
            } catch (Exception e) {
                attempts++;
                
                if (attempts >= MAX_RETRIES) {
                    // 최종 실패: 보상 트랜잭션 시작
                    handlePaymentFailure(customerId, amount);
                    throw new PaymentRetryExhaustedException(e);
                }
                
                log.warn("결제 재시도 {}/{}: {}", attempts, MAX_RETRIES, e.getMessage());
                
                try {
                    Thread.sleep(RETRY_DELAY_MS * (long) Math.pow(2, attempts));
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException(ie);
                }
            }
        }
    }
    
    private void handlePaymentFailure(Long customerId, BigDecimal amount) {
        // 보상 로직: 환불, 주문 취소 등
        log.error("결제 최종 실패, 보상 시작: customerId={}, amount={}", 
                  customerId, amount);
        
        // 데이터베이스에 실패 기록
        PaymentFailureRecord record = new PaymentFailureRecord();
        record.setCustomerId(customerId);
        record.setAmount(amount);
        record.setStatus("PENDING_COMPENSATION");
        paymentFailureRepository.save(record);
    }
}

// CAP Theorem 선택 구현
@Service
public class ConsistencyLevelStrategy {
    
    // CP 전략: 일관성 우선 (금융)
    @Service
    public static class StrongConsistencyService {
        
        @Autowired
        private OrderRepository orderRepository;
        
        @Transactional(isolation = Isolation.SERIALIZABLE)
        public void transferMoney(Long fromAccount, Long toAccount, BigDecimal amount) {
            // 강한 격리 수준 (직렬화)
            // 모든 쓰기가 순차적으로 처리됨
            // 성능: 낮음, 일관성: 높음
            
            Account from = orderRepository.findByIdWithLock(fromAccount);
            Account to = orderRepository.findByIdWithLock(toAccount);
            
            if (from.getBalance().compareTo(amount) < 0) {
                throw new InsufficientFundsException();
            }
            
            from.setBalance(from.getBalance().subtract(amount));
            to.setBalance(to.getBalance().add(amount));
            
            orderRepository.save(from);
            orderRepository.save(to);
        }
    }
    
    // AP 전략: 가용성 우선 (소셜 미디어)
    @Service
    public static class HighAvailabilityService {
        
        @Autowired
        private PostRepository postRepository;
        
        public void likePost(Long postId) {
            // 비동기, 최종 일관성
            // 성능: 높음, 일관성: 약함
            
            Post post = postRepository.findById(postId)
                .orElseThrow(PostNotFoundException::new);
            
            // 로컬 증가 (즉시 응답)
            post.incrementLikeCount();
            postRepository.save(post);
            
            // 분산 캐시에도 비동기로 업데이트
            asyncUpdateCache(postId, post.getLikeCount());
        }
        
        @Async
        private void asyncUpdateCache(Long postId, Long likeCount) {
            redisCache.set("post:likes:" + postId, likeCount);
        }
    }
}

// Vector Clock을 사용한 인과 관계 추적
@Data
public class VectorClock {
    private Map<String, Long> clock;
    
    public static VectorClock create(String nodeId) {
        VectorClock vc = new VectorClock();
        vc.clock = new ConcurrentHashMap<>();
        vc.clock.put(nodeId, 1L);
        return vc;
    }
    
    // 이벤트 발생 시 증가
    public void increment(String nodeId) {
        clock.put(nodeId, clock.getOrDefault(nodeId, 0L) + 1);
    }
    
    // 다른 Vector Clock과 병합 (최대값)
    public void merge(VectorClock other) {
        other.clock.forEach((nodeId, value) -> {
            clock.put(nodeId, Math.max(clock.getOrDefault(nodeId, 0L), value));
        });
    }
    
    // 인과 관계 확인
    public boolean happensBefore(VectorClock other) {
        boolean hasSmaller = false;
        
        for (String nodeId : clock.keySet()) {
            Long thisVal = clock.get(nodeId);
            Long otherVal = other.clock.getOrDefault(nodeId, 0L);
            
            if (thisVal > otherVal) return false;
            if (thisVal < otherVal) hasSmaller = true;
        }
        
        return hasSmaller;
    }
    
    // 동시 실행 확인
    public boolean isConcurrent(VectorClock other) {
        return !happensBefore(other) && !other.happensBefore(this);
    }
}

// CRDT (G-Counter) - 충돌 없는 카운터
@Data
public class GCounter {
    private Map<String, Long> values;  // nodeId -> count
    
    public void increment(String nodeId) {
        values.put(nodeId, values.getOrDefault(nodeId, 0L) + 1);
    }
    
    // 두 CRDT 병합 (항상 가능)
    public GCounter merge(GCounter other) {
        GCounter merged = new GCounter();
        merged.values = new HashMap<>(this.values);
        
        other.values.forEach((nodeId, value) -> {
            merged.values.put(nodeId, 
                Math.max(merged.values.getOrDefault(nodeId, 0L), value));
        });
        
        return merged;
    }
    
    // 전체 값 계산
    public long value() {
        return values.values().stream()
            .mapToLong(Long::longValue)
            .sum();
    }
}
```

---

## 📊 ACID vs BASE 비교

| 항목 | ACID | BASE |
|------|------|------|
| **일관성** | 강함 (즉시) | 약함 (최종) |
| **가용성** | 낮음 (일부 거부 가능) | 높음 (항상 응답) |
| **분할 허용도** | 낮음 (강한 일관성) | 높음 (최종 일관성) |
| **응답 시간** | 느림 (대기) | 빠름 (즉시) |
| **트랜잭션** | 지원 (원자성) | 미지원 (이벤트) |
| **사용 사례** | 금융 | 소셜 미디어 |
| **스케일링** | 어려움 | 쉬움 |

---

## ⚖️ 트레이드오프

```
강한 일관성 vs 높은 가용성
├─ ACID (강한 일관성):
│  ├─ ✅ 데이터 신뢰도 높음
│  ├─ ✅ 논리적 정확성 보장
│  ├─ ❌ 장애 시 서비스 거부
│  └─ ❌ 확장성 제한
│
└─ BASE (높은 가용성):
   ├─ ✅ 항상 응답 (장애 내성)
   ├─ ✅ 높은 확장성
   ├─ ❌ 약간의 불일치 가능
   └─ ❌ 프로그래밍 복잡도 증가

선택 기준:
├─ 금융 거래 → ACID (일관성 필수)
├─ 주문 시스템 → BASE (약간의 지연 OK)
├─ 소셜 미디어 → BASE (실시간 필수 아님)
└─ 재고 → 중간 수준 (과도한 중복 판매만 방지)

충돌 해결 비용:
├─ LWW: 간단하지만 데이터 손실 가능
├─ Vector Clock: 정확하지만 복잡
├─ CRDT: 자동 병합 가능하지만 특정 자료형만
└─ 애플리케이션: 유연하지만 비즈니스 로직 필요
```

---

## 📌 핵심 정리

```
✅ ACID vs BASE:
  - ACID: 강한 일관성, 낮은 가용성 (모놀리스)
  - BASE: 높은 가용성, 최종 일관성 (분산 시스템)

✅ CAP Theorem:
  - Consistency, Availability, Partition tolerance 중 2개만 가능
  - P(네트워크 장애)는 필수 → C와 A 선택
  - CP: 금융 (일관성 우선)
  - AP: 소셜 미디어 (가용성 우선)

✅ Eventual Consistency:
  - 최종적으로 모든 복제본이 일치
  - 시간이 필요 (밀리초~초)
  - 이벤트 기반 아키텍처로 구현

✅ 구현 방법:
  - 이벤트 소싱: 모든 변경을 이벤트로 기록
  - 보상 트랜잭션 (Saga): 실패 시 역순 처리
  - 재시도: 일시적 장애 대응
  - Circuit Breaker: 계단식 장애 방지

✅ 충돌 해결:
  - LWW (Last Write Wins): 간단, 데이터 손실 위험
  - Vector Clock: 인과 관계 추적
  - CRDT: 자동 병합 (특정 타입만)
  - 애플리케이션: 비즈니스 규칙 기반

✅ 선택 기준:
  - 강한 일관성 필수: 금융 거래, 고가 상품
  - 최종 일관성 허용: 주문, 재고, 소셜
  - 실시간 일관성: 검색 인덱스 (최대 1초 지연)
```

---

## 🤔 생각해볼 문제

**Q1.** 고객이 계좌에서 100만원을 출금합니다. 동시에 배우자의 계좌로 50만원이 입금됩니다. 시스템이 최종 일관성을 사용하는 경우, 어떤 문제가 발생할 수 있을까요?

<details>
<summary>해설 보기</summary>

문제 시나리오 (Eventual Consistency):

t=0: 
- 계좌 A: 100만원
- 계좌 B: 50만원
(총: 150만원)

t=1: 출금 요청
- 계좌 A에서 100만원 출금 = 0원
- 계좌 총액: 50만원

t=2: 입금 요청
- 계좌 B에 50만원 입금 = 100만원
- 계좌 총액: 100만원

실제 발생 가능한 시나리오 (최종 일관성의 위험):

t=0: 계좌 A: 100만원

t=1ms: 거래 1: A에서 -100만원 (기록됨)
      거래 2: B에서 +50만원 (기록됨)

t=100ms: 데이터 동기화 중...
        A의 변경: 다른 DC로 복제 중
        B의 변경: 다른 DC로 복제 중

t=200ms: 
- DC1에서: A=0, B=100 (합계: 100만원) ✅
- DC2에서: A=100, B=50 (합계: 150만원) ❌

→ 일시적으로 돈이 생김!

t=300ms: 최종 동기화
- 양쪽 모두: A=0, B=100 (합계: 100만원) ✅

결론: **금융 거래는 최종 일관성으로 불가능**
→ CP 선택 필수 (강한 일관성 + 장애 격리)
→ 분산 트랜잭션 (2PC 또는 Saga) 필요

</details>

**Q2.** 이커머스 시스템에서 최종 일관성을 사용합니다. 상품 A의 재고는 실제로 10개인데, 주문 시스템은 아직 15개라고 생각합니다. 3명의 고객이 각각 6개씩 주문했습니다. 어떤 문제가 발생할까요?

<details>
<summary>해설 보기</summary>

Overselling 문제:

t=0: 
- 재고 시스템: 상품 A = 10개
- 주문 시스템: 상품 A = 15개 (아직 동기화 안 됨)

t=1ms:
- 고객 1: 6개 주문 (가능, 15-6=9)
- 고객 2: 6개 주문 (가능, 9-6=3)
- 고객 3: 6개 주문 (불가능, 3<6 → 거부)

결과: 고객 1,2만 주문 성공 (12개)
하지만 실제 재고: 10개 (부족 2개)

→ 약속한 배송 일정 지킬 수 없음
→ 비용 증가 (긴급 구매, 일부 환불)

해결책:
1. **과매각 허용**:
   - 추가 공급 계획 (긴급 주문 가능)
   - 부분 배송 허용

2. **보정 재고**:
   - 실제 재고보다 적게 표시
   - 안전 재고 = 실제 - 최대 주문 지연시간의 판매량

3. **강한 일관성 (재고)**:
   - 주문 시 재고 서비스에 직접 조회
   - API Composition (약간의 지연)

4. **하이브리드**:
   - 보통은 캐시된 재고 사용 (빠름)
   - 재고 부족 시 실시간 확인

결론: 재고는 **약간의 강한 일관성** 필요
→ 즉시 일관성은 아니지만, 너무 큰 차이는 위험

</details>

**Q3.** Vector Clock을 사용하여 두 데이터센터의 동시 쓰기를 감지했습니다. 충돌이 발생했습니다. 이를 자동으로 해결할 수 있을까요?

<details>
<summary>해설 보기</summary>

Vector Clock의 한계:

VC_A = [A:2, B:0]  (DC1에서 2번 쓰기)
VC_B = [A:0, B:2]  (DC2에서 2번 쓰기)

→ 동시 쓰기 감지! (neither happens-before)

문제: 이게 **어떤 종류의 충돌**인지 알 수 없음

시나리오 1: 가격 업데이트
- DC1: 상품 가격 → 10,000원
- DC2: 상품 가격 → 15,000원
→ 어느 것을 선택해야 하나?

시나리오 2: 카운터 증가
- DC1: 좋아요 +1
- DC2: 좋아요 +1
→ 병합 결과: 좋아요 +2 (자동 가능)

결론: **Vector Clock은 충돌만 감지, 해결하지 않음**

자동 해결 가능한 경우:
- CRDT (누적 연산): G-Counter, OR-Set
- 멱등한 연산: SET (마지막 값)
- 스칼라 연산: LWW (Last Write Wins)

자동 해결 불가능한 경우:
- 비즈니스 로직이 필요한 충돌
- 선택이 필요한 경우

→ 이 경우 사용자나 관리자가 선택해야 함

구현:
```java
public void resolveConflict(DataValue local, DataValue remote, VectorClock vcLocal, VectorClock vcRemote) {
    if (vcLocal.isConcurrent(vcRemote)) {
        // 동시 쓰기 충돌
        if (isMergeableOperation(local)) {
            // CRDT 타입: 자동 병합
            return merge(local, remote);
        } else {
            // 비즈니스 로직 필요
            notifyAdminForConflictResolution(local, remote);
        }
    }
}
```

</details>

---

<div align="center">

**[⬅️ 이전: Join 없는 데이터 조회 전략 — API Composition, CQRS, 복제](./03-join-free-query-strategies.md)** | **[홈으로 🏠](../README.md)** | **[다음: 데이터 이관 전략 — 모놀리스 DB에서 서비스별 DB로 ➡️](./05-data-migration-strategies.md)**

</div>

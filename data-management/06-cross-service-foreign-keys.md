# 06. 서비스 간 외래키 — 물리적 무결성 없이 일관성 보장

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 왜 서비스 간 물리적 외래키는 사용하면 안 되는가?
- 서비스 간 참조 무결성을 어떻게 보장하는가?
- 이벤트 기반 보상 트랜잭션(Saga)의 구현 방법은?
- 고아 데이터(Orphan Data)는 언제 발생하고 어떻게 처리하는가?
- 서비스 간 ID 참조 패턴의 모범 사례는?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

**모놀리스 환경에서 참조 무결성은 데이터베이스가 보장합니다.**

```sql
-- 모놀리스: 강한 외래키 제약
ALTER TABLE orders 
ADD CONSTRAINT fk_customer 
FOREIGN KEY (customer_id) REFERENCES customers(customer_id);

-- 고객을 삭제하려고 하면?
DELETE FROM customers WHERE customer_id = 1;
-- → ERROR: Cannot delete customer (외래키 제약 위반)
```

**하지만 마이크로서비스에서는 다릅니다.** 주문(Order)과 고객(Customer)이 다른 DB에 있으면:
- 물리적 FK를 만들 수 없음 (크로스 DB)
- FK를 무시하고 삭제하면? → 고아 데이터 발생
- 두 서비스의 일관성이 깨짐

**따라서 애플리케이션 레벨에서 참조 무결성을 관리해야 합니다.** 이를 통해:
- 독립적 스케일링 가능
- 서비스 배포 순서 자유도
- 하지만 더 복잡한 로직 필요

---

## 😱 흔한 실수 (Before — 불가능한 FK)

```java
// BEFORE: 서비스 간 물리적 FK 시도 (작동하지 않음)

@Entity
@Table(name = "orders")
public class Order {
    @Id
    private Long orderId;
    
    @Column(name = "customer_id")
    private Long customerId;
    
    // ❌ 이 FK는 작동하지 않음
    // customers 테이블은 다른 DB에 있기 때문
    @ManyToOne
    @JoinColumn(name = "customer_id", 
                referencedColumnName = "customer_id",
                foreignKey = @ForeignKey(name = "fk_order_customer"))
    private Customer customer;
}

// 또 다른 문제: 논리적 불일치 발생

@Service
public class OrderServiceWithoutConstraints {
    
    @Autowired
    private OrderRepository orderRepository;
    
    public void createOrder(Long customerId, OrderRequest request) {
        // 문제: customerId가 정말 존재하는가?
        // 확인할 방법이 없음 (다른 DB이므로)
        
        Order order = new Order();
        order.setCustomerId(customerId);  // 존재하지 않는 customerId 가능!
        orderRepository.save(order);
    }
}

// 고아 데이터 발생 시나리오

@Service
public class CustomerServiceDangerous {
    
    public void deleteCustomer(Long customerId) {
        // customers 테이블에서 고객 삭제
        customerRepository.deleteById(customerId);
        // → 이 고객의 모든 주문이 고아 데이터가 됨!
        
        // 주문 서비스는 customerId 참조가 깨짐
        // 하지만 알 방법이 없음
    }
}

// 결과: 
/*
주문 서비스 DB:
┌───────────────────┐
│ orders 테이블      │
├───────────────────┤
│ order_id │ cust_id │
├──────────┼────────┤
│    1     │  100   │
│    2     │  200   │
│    3     │  999   │ ← 고아 데이터
└───────────────────┘

고객 서비스 DB:
┌──────────────────────┐
│ customers 테이블     │
├──────────────────────┤
│ customer_id │ name   │
├─────────────┼────────┤
│    100      │ Alice  │
│    200      │ Bob    │
│    999      │ (없음) │ ← 삭제됨
└──────────────────────┘

주문 3을 조회하려고 하면?
- 고객 999 정보를 요청 → 없음
- 주문은 고객 정보 없이 표시 불가
- 애플리케이션 오류 발생
*/
```

---

## ✨ 올바른 접근 (After — 애플리케이션 레벨 참조 무결성)

### 전략 1: 이벤트 기반 보상 트랜잭션 (Saga)

```java
// 1단계: 주문 생성 (일관성 보장)

@Service
public class OrderCreationSaga {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private CustomerServiceClient customerServiceClient;
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public Order createOrder(CreateOrderCommand command) {
        // Step 1: 고객 존재 여부 확인
        try {
            CustomerDTO customer = customerServiceClient
                .getCustomer(command.getCustomerId());
            
            if (customer == null) {
                throw new CustomerNotFoundException(command.getCustomerId());
            }
        } catch (CustomerServiceException e) {
            // 고객 서비스가 응답하지 않음
            // → 고객이 존재한다고 가정하고 진행 (또는 거부)
            // 더 안전: 일단 거부
            throw new OrderCreationException("고객 서비스 불가: " + e.getMessage());
        }
        
        // Step 2: 주문 생성
        Order order = new Order();
        order.setCustomerId(command.getCustomerId());
        order.setTotalAmount(command.getTotalAmount());
        order.setStatus("PENDING");
        Order savedOrder = orderRepository.save(order);
        
        // Step 3: 이벤트 발행 (결제, 배송 등)
        eventPublisher.publishEvent(
            new OrderCreatedEvent(
                savedOrder.getId(),
                command.getCustomerId(),
                command.getTotalAmount()
            )
        );
        
        return savedOrder;
    }
}

// 2단계: 고객 삭제 시 보상 처리

@Service
public class CustomerDeleteSaga {
    
    @Autowired
    private CustomerRepository customerRepository;
    
    @Autowired
    private OrderServiceClient orderServiceClient;
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public void deleteCustomer(Long customerId) {
        // Step 1: 고객이 정말 존재하는가?
        Customer customer = customerRepository
            .findById(customerId)
            .orElseThrow(() -> new CustomerNotFoundException(customerId));
        
        // Step 2: 진행 중인 주문이 있는가?
        try {
            List<OrderDTO> activeOrders = orderServiceClient
                .getOrdersByCustomer(customerId);
            
            if (!activeOrders.isEmpty()) {
                // 주문이 있으면 삭제 불가
                throw new CustomerHasActiveOrdersException(customerId);
            }
        } catch (OrderServiceException e) {
            // 주문 서비스가 응답 없음
            // → 안전을 위해 삭제 거부
            throw new CustomerDeletionException("주문 서비스 확인 불가: " + e.getMessage());
        }
        
        // Step 3: 고객 삭제
        customerRepository.deleteById(customerId);
        
        // Step 4: 이벤트 발행 (다른 서비스들이 정리)
        eventPublisher.publishEvent(
            new CustomerDeletedEvent(customerId)
        );
    }
}

// 3단계: 이벤트 구독으로 서비스 간 일관성 유지

@Service
public class OrderServiceCustomerSubscriber {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @EventListener
    @Transactional
    public void onCustomerDeleted(CustomerDeletedEvent event) {
        // 고객이 삭제됨 → 이 고객의 완료된 주문은 유지
        // 하지만 미완료 주문은 취소
        List<Order> orders = orderRepository
            .findByCustomerIdAndStatusIn(
                event.getCustomerId(),
                Arrays.asList("PENDING", "PROCESSING")
            );
        
        for (Order order : orders) {
            order.setStatus("CANCELLED");
            order.setReason("Customer deleted");
            orderRepository.save(order);
            
            // 환불 처리 등...
            publishCompensationEvent(order);
        }
    }
}
```

### 전략 2: 스냅샷 저장 (참조 복제)

```java
// 전략: 주문 생성 시 고객 정보를 스냅샷으로 저장

@Entity
@Table(name = "orders")
public class Order {
    @Id
    private Long orderId;
    
    // 고객 ID (참조만)
    @Column(name = "customer_id")
    private Long customerId;
    
    // 고객 정보 스냅샷 (복제)
    @Embedded
    private CustomerSnapshot customerSnapshot;
    
    private BigDecimal totalAmount;
    private LocalDateTime createdAt;
}

@Embeddable
public class CustomerSnapshot {
    private String customerName;
    private String email;
    private String phone;
    private LocalDateTime snapshotAt;
}

@Service
public class OrderServiceWithSnapshot {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private CustomerServiceClient customerServiceClient;
    
    @Transactional
    public Order createOrder(Long customerId, OrderRequest request) {
        // Step 1: 고객 정보 조회
        CustomerDTO customer = customerServiceClient
            .getCustomer(customerId);
        
        if (customer == null) {
            throw new CustomerNotFoundException(customerId);
        }
        
        // Step 2: 주문 + 스냅샷 저장
        Order order = new Order();
        order.setCustomerId(customerId);
        order.setTotalAmount(request.getTotalAmount());
        order.setCreatedAt(LocalDateTime.now());
        
        // 고객 정보 스냅샷 저장
        CustomerSnapshot snapshot = new CustomerSnapshot();
        snapshot.setCustomerName(customer.getName());
        snapshot.setEmail(customer.getEmail());
        snapshot.setPhone(customer.getPhone());
        snapshot.setSnapshotAt(LocalDateTime.now());
        
        order.setCustomerSnapshot(snapshot);
        
        Order savedOrder = orderRepository.save(order);
        
        return savedOrder;
    }
    
    // 장점: 고객이 나중에 삭제되어도 주문 조회 가능
    public OrderDetailDTO getOrderDetail(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        
        // 고객 정보는 스냅샷에서
        CustomerSnapshot snapshot = order.getCustomerSnapshot();
        
        // 고객 현재 정보는 별도로 조회 (optional)
        CustomerDTO currentCustomer = null;
        try {
            currentCustomer = customerServiceClient
                .getCustomer(order.getCustomerId());
        } catch (CustomerNotFoundException e) {
            // 고객이 삭제됨: 스냅샷만 표시
            log.warn("고객 정보 없음 (삭제됨): {}", order.getCustomerId());
        }
        
        return new OrderDetailDTO(order, snapshot, currentCustomer);
    }
}

/*
스냅샷의 장점:
✅ 고객 삭제 후에도 주문 조회 가능
✅ 가독성 좋음 (주문 조회 = 모든 정보 포함)
✅ 성능 좋음 (JOIN 불필요)

스냅샷의 단점:
❌ 고객 정보가 오래될 수 있음 (고객이 이름 변경)
❌ 메모리 오버헤드 (같은 정보 저장)
❌ 동기화 문제 (수정 시 모든 주문 갱신? 안 함?)
*/
```

### 전략 3: 논리적 참조 + 검증

```java
// 전략: 계속 고객 서비스에서 조회하되, 없으면 우아하게 처리

@Service
public class OrderServiceWithValidation {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private CustomerServiceClient customerServiceClient;
    
    @Transactional
    public Order createOrder(Long customerId, OrderRequest request) {
        // 참조 검증: 고객이 존재하는가?
        validateCustomerExists(customerId);
        
        // 주문 생성
        Order order = new Order();
        order.setCustomerId(customerId);
        order.setTotalAmount(request.getTotalAmount());
        return orderRepository.save(order);
    }
    
    private void validateCustomerExists(Long customerId) {
        try {
            CustomerDTO customer = customerServiceClient
                .getCustomer(customerId);
            
            if (customer == null) {
                throw new CustomerNotFoundException(customerId);
            }
        } catch (CustomerServiceUnavailableException e) {
            // 고객 서비스가 다운됨
            // 옵션 1: 주문 생성 거부 (엄격함)
            throw new OrderCreationException("고객 서비스 불가");
            
            // 옵션 2: 계속 진행 (베타) - 나중에 검증
            // log.warn("고객 서비스 다운, 주문은 일단 생성: {}", customerId);
        }
    }
    
    // 고아 데이터 검증: 정기적으로 실행
    @Scheduled(cron = "0 2 * * *")  // 매일 새벽 2시
    public void validateOrphanData() {
        List<Order> allOrders = orderRepository.findAll();
        
        for (Order order : allOrders) {
            try {
                CustomerDTO customer = customerServiceClient
                    .getCustomer(order.getCustomerId());
                
                if (customer == null) {
                    // 고아 데이터 발견!
                    handleOrphanOrder(order);
                }
            } catch (CustomerServiceException e) {
                // 고객 서비스 장애: 나중에 다시 확인
                log.warn("고객 서비스 오류로 검증 불가: orderId={}", 
                         order.getId());
            }
        }
    }
    
    private void handleOrphanOrder(Order orphanOrder) {
        // 고아 데이터 처리:
        // 옵션 1: 주문 취소
        if (orphanOrder.getStatus().equals("PENDING")) {
            orphanOrder.setStatus("CANCELLED");
            orphanOrder.setReason("Customer deleted");
            orderRepository.save(orphanOrder);
        }
        
        // 옵션 2: 알림
        log.error("고아 데이터 발견: orderId={}, deletedCustomerId={}", 
                  orphanOrder.getId(), 
                  orphanOrder.getCustomerId());
        
        // 옵션 3: 관리자에게 통보
        notifyAdminOfOrphanData(orphanOrder);
    }
}
```

---

## 🔬 내부 동작 원리 — 참조 무결성 보장 메커니즘

```
모놀리스 (물리적 FK):
┌─────────────────────────────┐
│      customers 테이블        │
├─────────────────────────────┤
│ customer_id (PK)            │
│ name                        │
└─────────────────────────────┘
         ▲
         │ (FK 제약)
         │ (1:N)
         │
┌─────────────────────────────┐
│      orders 테이블          │
├─────────────────────────────┤
│ order_id (PK)               │
│ customer_id (FK) ←─────┐    │
│ amount                  │    │
└─────────────────────────┴────┘

고객 삭제 시도:
DELETE FROM customers WHERE customer_id = 1;
→ ERROR: Foreign key constraint violation
→ 고아 데이터 불가능 ✅

마이크로서비스 (애플리케이션 레벨):
┌──────────────────────────┐
│ 고객 서비스               │
│ ┌──────────────────────┐ │
│ │ customers DB         │ │
│ │ customer_id: 1       │ │
│ └──────────────────────┘ │
└──────────────────────────┘
         │
         │ (이벤트 또는 API)
         │
┌──────────────────────────┐
│ 주문 서비스               │
│ ┌──────────────────────┐ │
│ │ orders DB            │ │
│ │ customer_id: 1 (FK?) │ │
│ └──────────────────────┘ │
└──────────────────────────┘

고객 삭제 시도:
DELETE FROM customers WHERE customer_id = 1;
→ 삭제 가능! (다른 DB이므로)
→ 주문 서비스는 customer_id 1 참조 무결성 깨짐 ❌

해결책:
1. 이벤트 기반: 삭제 전 주문 서비스에 통보
2. 스냅샷: 참조 ID + 필요한 데이터 저장
3. 검증: 정기적으로 고아 데이터 확인
```

### Saga 패턴의 보상 트랜잭션

```
시나리오: 주문 생성 → 결제 → 배송

순방향 (Happy Path):
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Order Service│     │PaymentService│     │ShipmentService
└──────────────┘     └──────────────┘     └──────────────┘
     │                     │                     │
     │ OrderCreated        │                     │
     ├────────────────────>│                     │
     │                     │ PaymentProcessed    │
     │                     ├────────────────────>│
     │                     │                     │ ShipmentCreated
     │                     │                     │
     └─────── All Success ────────────────────>│

역방향 (보상, 장애 발생):
결제 서비스가 다운된 경우:

┌──────────────┐     ┌──────────────┐     
│ Order Service│     │PaymentService│     
└──────────────┘     └──────────────┘    

1. OrderCreated 이벤트 발행
2. PaymentService가 응답 없음 (다운)
3. 타임아웃: OrderService가 감지
4. PaymentFailed 이벤트 발행
5. OrderService가 보상: 주문 취소
   ┌─────────────────────────────────┐
   │ Order 상태: CANCELLED           │
   │ Reason: Payment service failed  │
   └─────────────────────────────────┘

결과: 주문은 생성되지 않음 (논리적 일관성)
```

---

## 💻 Java 실전 코드 — 고아 데이터 정리

```java
// 고아 데이터 감지 및 처리 자동화

@Entity
@Table(name = "orders")
public class Order {
    @Id
    private Long orderId;
    
    @Column(name = "customer_id")
    private Long customerId;
    
    @Column(name = "reference_valid")
    private Boolean referenceValid;  // 참조가 유효한가?
    
    @Column(name = "reference_checked_at")
    private LocalDateTime referenceCheckedAt;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
}

@Service
public class OrphanDataManager {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private CustomerServiceClient customerServiceClient;
    
    @Autowired
    private OrphanDataRepository orphanRepository;
    
    // 1. 고아 데이터 감지
    @Scheduled(cron = "0 2 * * *")
    public void detectOrphanData() {
        log.info("고아 데이터 감지 시작...");
        
        // 최근에 검증하지 않은 주문들
        List<Order> ordersToCheck = orderRepository
            .findByReferenceCheckedAtBeforeOrReferenceCheckedAtNull(
                LocalDateTime.now().minusHours(24),
                PageRequest.of(0, 1000)
            );
        
        for (Order order : ordersToCheck) {
            try {
                CustomerDTO customer = customerServiceClient
                    .getCustomer(order.getCustomerId());
                
                if (customer == null) {
                    // 고아 데이터 발견!
                    order.setReferenceValid(false);
                    orderRepository.save(order);
                    
                    // 별도 추적
                    OrphanDataRecord record = new OrphanDataRecord();
                    record.setOrderId(order.getId());
                    record.setDeletedCustomerId(order.getCustomerId());
                    record.setDetectedAt(LocalDateTime.now());
                    record.setStatus(OrphanDataStatus.PENDING);
                    orphanRepository.save(record);
                    
                    log.warn("고아 데이터 발견: orderId={}", order.getId());
                } else {
                    // 참조 유효함
                    order.setReferenceValid(true);
                    orderRepository.save(order);
                }
            } catch (Exception e) {
                log.error("고객 서비스 확인 실패: orderId={}", order.getId(), e);
            }
            
            order.setReferenceCheckedAt(LocalDateTime.now());
            orderRepository.save(order);
        }
    }
    
    // 2. 고아 데이터 정리
    @Scheduled(cron = "0 3 * * *")
    public void cleanupOrphanData() {
        List<OrphanDataRecord> pendingRecords = orphanRepository
            .findByStatus(OrphanDataStatus.PENDING);
        
        for (OrphanDataRecord record : pendingRecords) {
            Order order = orderRepository.findById(record.getOrderId())
                .orElse(null);
            
            if (order == null) continue;
            
            switch (order.getStatus()) {
                case PENDING:
                case PROCESSING:
                    // 진행 중인 주문: 취소
                    cancelOrphanOrder(order, record);
                    break;
                    
                case COMPLETED:
                case SHIPPED:
                    // 완료된 주문: 보관만 (기록 유지)
                    record.setStatus(OrphanDataStatus.ARCHIVED);
                    orphanRepository.save(record);
                    break;
                    
                case CANCELLED:
                    // 이미 취소됨: 삭제 가능
                    record.setStatus(OrphanDataStatus.DELETED);
                    orphanRepository.save(record);
                    break;
            }
        }
    }
    
    private void cancelOrphanOrder(Order order, OrphanDataRecord record) {
        order.setStatus(OrderStatus.CANCELLED);
        order.setCancellationReason("Customer no longer exists");
        orderRepository.save(order);
        
        // 환불 처리
        try {
            refundPayment(order.getId());
        } catch (Exception e) {
            log.error("환불 처리 실패: orderId={}", order.getId(), e);
        }
        
        // 기록 업데이트
        record.setStatus(OrphanDataStatus.RESOLVED);
        record.setResolvedAt(LocalDateTime.now());
        orphanRepository.save(record);
    }
}

// 3. 참조 무결성 검증 API
@RestController
@RequestMapping("/api/admin/data-integrity")
public class DataIntegrityController {
    
    @Autowired
    private OrphanDataManager orphanDataManager;
    
    @PostMapping("/check-references")
    public ResponseEntity<?> checkReferences() {
        orphanDataManager.detectOrphanData();
        return ResponseEntity.ok("검증 시작");
    }
    
    @PostMapping("/cleanup-orphans")
    public ResponseEntity<?> cleanupOrphans() {
        orphanDataManager.cleanupOrphanData();
        return ResponseEntity.ok("정리 시작");
    }
    
    @GetMapping("/orphan-data")
    public List<OrphanDataRecord> getOrphanData() {
        return orphanDataManager.getAllOrphanRecords();
    }
}

// 4. 비동기 참조 검증 (Hot Path 영향 최소화)
@Service
public class AsyncReferenceValidator {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private CustomerServiceClient customerServiceClient;
    
    // 주문 조회 시: 참조 검증은 비동기
    public OrderDTO getOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        
        // 비동기: 백그라운드에서 검증
        validateReferenceAsync(order);
        
        return orderToDTO(order);
    }
    
    @Async
    private void validateReferenceAsync(Order order) {
        try {
            CustomerDTO customer = customerServiceClient
                .getCustomer(order.getCustomerId());
            
            boolean isValid = customer != null;
            order.setReferenceValid(isValid);
            order.setReferenceCheckedAt(LocalDateTime.now());
            
            if (!isValid) {
                // 고아 데이터 감지: 로그 및 모니터링
                log.warn("고아 데이터 감지 (비동기): orderId={}", order.getId());
            }
            
            orderRepository.save(order);
        } catch (Exception e) {
            log.warn("참조 검증 실패 (비동기): orderId={}", order.getId(), e);
        }
    }
}
```

---

## 📊 참조 무결성 전략 비교

| 전략 | 물리적 FK | 이벤트 기반 | 스냅샷 | 논리적 검증 |
|------|----------|----------|--------|-----------|
| **고아 데이터 방지** | ✅ DB 강제 | ✅ 이벤트 통보 | ❌ 부분 | ✅ 정기 검증 |
| **조회 성능** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **구현 복잡도** | 낮음 | 높음 | 중간 | 중간 |
| **참조 최신도** | 즉시 | 이벤트 지연 | 오래됨 | 최신 |
| **서비스 독립성** | ❌ 강결합 | ✅ 독립 | ✅ 독립 | ✅ 독립 |
| **데이터 중복** | 없음 | 없음 | 있음 | 없음 |
| **실패 대응** | DB 거부 | 보상 | 우아한 처리 | 정정 |

---

## ⚖️ 트레이드오프

```
참조 무결성 수준 vs 시스템 복잡도
├─ 물리적 FK (모놀리스):
│  ├─ ✅ 100% 무결성 (DB 강제)
│  ├─ ✅ 구현 단순 (DB 기능)
│  ├─ ❌ 서비스 결합 (불가능)
│  └─ ❌ MSA 용도 부적합
│
├─ 이벤트 기반 (최적):
│  ├─ ✅ 높은 무결성 (이벤트 통보)
│  ├─ ✅ 서비스 독립 (느슨한 결합)
│  ├─ ❌ 복잡한 구현 (Saga)
│  └─ ❌ 최종 일관성 (지연)
│
├─ 스냅샷 (빠름):
│  ├─ ✅ 빠른 조회 (스냅샷)
│  ├─ ✅ 고아 데이터 내성 (데이터 있음)
│  ├─ ❌ 데이터 중복 (메모리)
│  └─ ❌ 동기화 이슈 (수정 시)
│
└─ 논리적 검증 (유연):
   ├─ ✅ 유연한 처리 (우아한 실패)
   ├─ ✅ 가능한 조합 (다른 전략과)
   ├─ ❌ 고아 데이터 가능 (처리 필요)
   └─ ❌ 모니터링 필수 (자동 아님)

선택 기준:
├─ 강한 일관성 필수 → 이벤트 기반
├─ 빠른 조회 필수 → 스냅샷
├─ 유연한 실패 처리 → 논리적 검증
└─ 최적 → 이벤트 + 스냅샷 + 검증 조합
```

---

## 📌 핵심 정리

```
✅ 물리적 FK는 불가능:
  - 서로 다른 데이터베이스 간 제약 불가
  - 서비스 간 강결합 (배포 순서 의존)
  - MSA 원칙 위반

✅ 애플리케이션 레벨 참조 무결성:
  1. 이벤트 기반 (Saga): 생성/삭제 시 이벤트로 동기화
  2. 스냅샷: 필요한 데이터 미리 저장
  3. 논리적 검증: API로 존재 여부 확인

✅ 고아 데이터 처리:
  - 감지: 정기적 검증 배치
  - 분류: 진행 중? 완료? 취소?
  - 처리: 상태별 보정 (취소, 환불 등)
  - 모니터링: 알림 및 통계

✅ Saga 패턴:
  - 순방향: 각 서비스가 순차 처리
  - 역방향: 실패 시 보상 트랜잭션
  - 보상: 이전 단계 역순 취소

✅ 참조 무결성 보장 수준:
  - DB 강제: 불가능 (물리적 FK)
  - 이벤트 통보: 높음 (Saga 패턴)
  - 정기 검증: 중간 (배치)
  - 최종 일관성: 약함 (비동기)

✅ 모범 사례:
  - 참조 검증: 주문 생성 시 필수
  - 고아 감지: 자동화 배치
  - 보상 처리: 명확한 롤백 로직
  - 모니터링: 고아 데이터 통계 추적
```

---

## 🤔 생각해볼 문제

**Q1.** 고객을 삭제하기 전에 "진행 중인 주문 확인" API를 호출했습니다. 응답은 "주문 없음"이었습니다. 고객을 삭제하려고 하는 그 순간, 다른 클라이언트가 같은 고객을 위해 새 주문을 생성했습니다. 이 Race Condition을 어떻게 방지할까요?

<details>
<summary>해설 보기</summary>

Classic Race Condition 시나리오:

```
Thread 1 (고객 삭제)           Thread 2 (주문 생성)
│                              │
├─ checkOrders(customerId)     │
│  └─ 응답: 없음 ✅            │
│                              ├─ createOrder(customerId)
│                              │  └─ 고객 검증: ✅
│                              │  └─ 저장: ✅
│                              │
├─ deleteCustomer(customerId)  │
│  └─ DELETE 실행 ✅           │
│
결과: 새 주문이 있는데 고객이 삭제됨 ❌
```

해결책:

1. **Pessimistic Locking** (비관적 잠금):
   ```java
   @Transactional
   public void deleteCustomer(Long customerId) {
       // 고객을 X-Lock으로 잠금 (쓰기 잠금)
       Customer customer = customerRepository
           .findByIdWithXLock(customerId);  // SELECT FOR UPDATE
       
       // 다른 트랜잭션은 이 고객에 접근 불가
       // 주문 생성도 블로킹됨
       
       List<Order> activeOrders = orderRepository
           .findByCustomerId(customerId);
       
       if (!activeOrders.isEmpty()) {
           throw new CustomerHasOrdersException();
       }
       
       customerRepository.delete(customer);
       // 트랜잭션 커밋: 잠금 해제
   }
   
   // 주문 생성도 수정
   @Transactional
   public Order createOrder(Long customerId, OrderRequest request) {
       // 고객을 S-Lock으로 잠금 (읽기 잠금)
       Customer customer = customerRepository
           .findByIdWithSLock(customerId);  // SELECT FOR SHARE
       
       if (customer == null) {
           throw new CustomerNotFoundException();
       }
       
       // 이때 deleteCustomer가 X-Lock을 기다리고 있으면
       // 여기서 블로킹됨
       // 또는 이미 X-Lock 중이면 즉시 실패
       
       Order order = createOrderEntity(customerId, request);
       return orderRepository.save(order);
   }
   ```

2. **Optimistic Locking** (낙관적 잠금):
   ```java
   @Entity
   public class Customer {
       @Id
       private Long customerId;
       
       @Version  // 낙관적 잠금 버전
       private Long version;
       
       private String name;
       private String status;  // ACTIVE, DELETED
   }
   
   @Transactional
   public void deleteCustomer(Long customerId) {
       Customer customer = customerRepository.findById(customerId)
           .orElseThrow();
       
       // 버전 확인: 다른 트랜잭션이 수정했는가?
       customer.setStatus("DELETED");
       customerRepository.save(customer);  // version++
   }
   
   @Transactional
   public Order createOrder(Long customerId, OrderRequest request) {
       Customer customer = customerRepository.findById(customerId)
           .orElseThrow();
       
       if (customer.getStatus().equals("DELETED")) {
           throw new CustomerDeletedException();
       }
       
       Order order = createOrderEntity(customerId, request);
       return orderRepository.save(order);
   }
   ```

3. **이벤트 기반 순차 처리**:
   ```java
   // 고객 삭제 전 모든 진행 중인 주문 대기
   @Transactional
   public void deleteCustomer(Long customerId) {
       // 1단계: 이벤트 발행
       eventPublisher.publishEvent(
           new CustomerDeletionRequestedEvent(customerId)
       );
       
       // 2단계: 모든 주문 서비스의 완료를 기다림
       // (또는 타임아웃)
       awaitOrderServiceAcknowledgement(customerId);
       
       // 3단계: 고객 삭제
       customerRepository.deleteById(customerId);
   }
   ```

결론: **Locking이 가장 간단**
→ DB가 자동으로 순차성 보장
→ Application에서 처리하면 복잡도 증가

</details>

**Q2.** 주문에 고객 정보 스냅샷을 저장했는데, 고객이 이름을 "Alice"에서 "Alice Johnson"으로 변경했습니다. 기존 주문들의 스냅샷도 갱신해야 할까요?

<details>
<summary>해설 보기</summary>

스냅샷 갱신 문제:

선택지:

1. **갱신하지 않음** (보통):
   ```
   주문 1 (과거): "Alice"로 표시
   주문 2 (최신): "Alice Johnson"으로 표시
   
   → 과거 주문은 당시 고객명으로 표시
   → 역사적 정확성 (감사, 보고 용도)
   ✅ 저장공간 절감
   ❌ 고객명이 일관성 없음
   ```

2. **모두 갱신** (위험):
   ```
   모든 주문의 스냅샷 갱신
   
   → 리소스 많음 (UPDATE N개 행)
   → 갱신 중 불일치 가능성
   ❌ 감사 추적 손상
   ```

3. **하이브리드** (권장):
   ```
   - 스냅샷: 저장 유지 (감사용)
   - 별도 필드: 현재 고객명 (표시용)
   
   @Embeddable
   public class OrderCustomerInfo {
       // 스냅샷: 주문 시점의 정보
       private String snapshotName;  // "Alice"
       
       // 또는 API에서 조회
       private transient CustomerDTO currentInfo;  // "Alice Johnson"
   }
   
   @Transactional(readOnly = true)
   public OrderDetailDTO getOrderDetail(Long orderId) {
       Order order = orderRepository.findById(orderId);
       
       // 표시: 현재 고객 정보
       CustomerDTO currentCustomer = customerServiceClient
           .getCustomer(order.getCustomerId());
       
       // 또는 스냅샷 (고객 삭제 시)
       CustomerDTO displayCustomer = currentCustomer != null
           ? currentCustomer
           : order.getCustomerSnapshot();
       
       return new OrderDetailDTO(order, displayCustomer);
   }
   ```

결론: **스냅샷은 "기록"이므로 변경하지 않기**
→ 현재 정보가 필요하면 API 조회
→ 고객 서비스와 별도 동기화 불필요

</details>

**Q3.** Saga 패턴에서 다음과 같이 실패했습니다:
1. 주문 생성: ✅
2. 결제 처리: ✅
3. 배송 예약: ❌ (timeout)

보상 트랜잭션은:
1. 주문 취소: ✅
2. 결제 환불: ✅
3. 배송 취소: ❌ (timeout)

배송 취소가 실패했으면 어떻게 해야 할까요?

<details>
<summary>해설 보기</summary>

부분 실패한 보상 트랜잭션의 문제:

상태:
- 주문: CANCELLED ✅
- 결제: REFUNDED ✅
- 배송: PENDING (취소 시도 실패)

문제: 배송이 여전히 예약 상태
→ 창고에서 상품이 준비될 수 있음
→ 배송 서비스는 모름 (취소 메시지 못 받음)

해결책:

1. **재시도 (Retry)**:
   ```java
   @Service
   public class SagaCompensationHandler {
       
       private static final int MAX_RETRIES = 3;
       
       public void compensateShipment(Long orderId) {
           int attempts = 0;
           
           while (attempts < MAX_RETRIES) {
               try {
                   shipmentServiceClient.cancelReservation(orderId);
                   log.info("배송 취소 성공: {}", orderId);
                   return;
               } catch (ShipmentServiceTimeoutException e) {
                   attempts++;
                   if (attempts >= MAX_RETRIES) {
                       // 최종 실패
                       handleCompensationFailure(orderId);
                   } else {
                       Thread.sleep(1000 * attempts);
                   }
               }
           }
       }
   }
   ```

2. **수동 개입 (Manual Compensation)**:
   ```java
   private void handleCompensationFailure(Long orderId) {
       // DB에 기록
       FailedCompensation record = new FailedCompensation();
       record.setOrderId(orderId);
       record.setServiceName("ShipmentService");
       record.setReason("Timeout - 배송 취소 실패");
       record.setStatus(CompensationStatus.PENDING_MANUAL);
       failedCompensationRepository.save(record);
       
       // 관리자에게 알림
       notifyAdminOfFailedCompensation(record);
       // "orderId 123의 배송 예약을 수동으로 취소하세요"
   }
   ```

3. **보상의 보상 (Nested Saga)**:
   ```java
   // 배송 취소 실패 → 다른 방법으로 처리
   private void handleCompensationFailure(Long orderId) {
       try {
           // 방법 1: 배송 서비스 연락처 제공
           shipmentServiceClient.getContactInfo()
               .sendManualCancellationRequest(orderId);
       } catch (Exception e) {
           // 방법 2: 배송료 환불
           refundShippingFee(orderId);
           
           // 방법 3: 배송 예약을 "고객 취소 요청" 상태로
           shipmentServiceClient.markAsCancelledByCustomer(orderId);
       }
   }
   ```

결론: **Saga는 최종적 일관성만 보증**
→ 모든 단계가 성공한다는 보장은 없음
→ 실패한 보상은 모니터링 필요
→ 관리자 개입 가능한 구조 필요

</details>

---

<div align="center">

**[⬅️ 이전: 데이터 이관 전략 — 모놀리스 DB에서 서비스별 DB로](./05-data-migration-strategies.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — 분산 트랜잭션의 문제 — 2PC와 Saga ➡️](../saga/01-distributed-transaction-problem.md)**

</div>

# 03. Join 없는 데이터 조회 전략 — API Composition, CQRS, 복제

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Database per Service에서 JOIN이 불가능할 때 어떻게 데이터를 조회하는가?
- API Composition의 장단점은 무엇이고, N+1 문제는 어떻게 해결하는가?
- CQRS 패턴이 JOIN 문제를 어떻게 해결하는가?
- 데이터 복제(Replication) 전략의 구현과 일관성 관리는?
- 각 전략을 선택하는 기준은 무엇인가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

**Database per Service 원칙이 가져온 딜레마**: 각 서비스가 자신의 DB를 가지면 독립성은 보장되지만, 여러 서비스의 데이터가 필요한 조회는 어떻게 해야 할까요?

모놀리스 환경에서는 간단합니다: `SELECT o.*, s.* FROM orders o JOIN shipments s ON o.id = s.order_id`. 그런데 마이크로서비스 환경에서는 `orders` 테이블과 `shipments` 테이블이 완전히 다른 데이터베이스에 있습니다. **물리적 JOIN이 불가능합니다.**

이 문제를 해결하는 방법은 3가지입니다:
1. **API Composition**: 여러 서비스 API를 호출해서 애플리케이션 레벨에서 데이터 조합
2. **CQRS**: 쓰기는 원본 서비스, 읽기는 특화된 읽기 모델에서
3. **데이터 복제**: 필요한 데이터를 자신의 DB에 미리 복사

각 방법은 서로 다른 트레이드오프를 가집니다. 이를 정확히 이해하고 상황에 맞는 전략을 선택해야 합니다.

---

## 😱 흔한 실수 (Before — 불가능한 JOIN 시도)

```java
// BEFORE: Database per Service에서 JOIN 시도 (작동하지 않음)

@Service
public class OrderDetailService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;  // orders_db 연결
    
    public OrderDetailDTO getOrderDetail(Long orderId) {
        // 이 SQL 쿼리는 작동하지 않음
        // orders_db에는 shipments 테이블이 없기 때문
        try {
            String sql = "SELECT o.order_id, o.customer_id, o.total_amount, " +
                         "       s.shipment_id, s.status, s.estimated_delivery " +
                         "FROM orders o " +
                         "LEFT JOIN shipments s ON o.order_id = s.order_id " +
                         "WHERE o.order_id = ?";
            
            return jdbcTemplate.queryForObject(sql, new Object[]{orderId}, 
                new OrderDetailRowMapper());
        } catch (DataAccessException e) {
            // "shipments" table not found 예외 발생
            // shipments는 다른 DB에 있음!
            throw new RuntimeException("JOIN 실패: shipments 테이블을 찾을 수 없음");
        }
    }
}

// 또 다른 잘못된 시도: Foreign Key 사용
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private Long orderId;
    
    @ManyToOne
    @JoinColumn(name = "shipment_id", 
                referencedColumnName = "shipment_id",
                foreignKey = @ForeignKey(name = "fk_orders_shipments"))
    // 이 FK는 작동하지 않음 (다른 DB 서버!)
    private Shipment shipment;
}

// 결과: 
// ❌ 쿼리 실행 실패
// ❌ 애플리케이션 로딩 실패 (Entity 검증 단계에서)
// ❌ 서비스 간 강한 결합
```

이 상황에서의 문제:
- 물리적으로 불가능한 JOIN 시도
- 컴파일은 되지만 런타임에 실패
- 다른 팀의 DB 구조에 의존

---

## ✨ 올바른 접근 (After — 3가지 전략)

### 전략 1: API Composition (애플리케이션 레벨 조합)

```java
// AFTER: API Composition을 사용한 조회

@Service
public class OrderDetailServiceComposition {
    
    @Autowired
    private OrderServiceClient orderServiceClient;
    
    @Autowired
    private ShipmentServiceClient shipmentServiceClient;
    
    @Autowired
    private PaymentServiceClient paymentServiceClient;
    
    public OrderDetailDTO getOrderDetail(Long orderId) {
        // 1단계: 주문 정보 조회
        OrderDTO order = orderServiceClient.getOrder(orderId);
        
        if (order == null) {
            throw new OrderNotFoundException(orderId);
        }
        
        // 2단계: 배송 정보 조회
        ShipmentDTO shipment = null;
        try {
            shipment = shipmentServiceClient.getShipmentByOrderId(orderId);
        } catch (ShipmentServiceException e) {
            // 배송 서비스 장애 시에도 주문 정보는 제공
            log.warn("배송 정보 조회 실패: " + e.getMessage());
        }
        
        // 3단계: 결제 정보 조회
        PaymentDTO payment = null;
        try {
            payment = paymentServiceClient.getPaymentByOrderId(orderId);
        } catch (PaymentServiceException e) {
            log.warn("결제 정보 조회 실패: " + e.getMessage());
        }
        
        // 4단계: 데이터 조합 (애플리케이션 레벨)
        return new OrderDetailDTO(
            order.getOrderId(),
            order.getCustomerId(),
            order.getTotalAmount(),
            shipment,
            payment
        );
    }
}

// 문제점: N+1 문제 (1개 주문 + N개 항목 = N+1번 조회)
@Service
public class OrderListServiceComposition {
    
    @Autowired
    private OrderServiceClient orderServiceClient;
    
    @Autowired
    private ShipmentServiceClient shipmentServiceClient;
    
    // 주문 목록 조회 (매우 비효율적)
    public List<OrderDetailDTO> getOrdersWithShipments(Long customerId) {
        // 1단계: 고객의 모든 주문 조회
        List<OrderDTO> orders = orderServiceClient.getOrdersByCustomer(customerId);
        // SQL: SELECT * FROM orders WHERE customer_id = ? (10개 주문)
        
        // 2단계: 각 주문마다 배송 정보 조회 (10번 API 호출)
        List<OrderDetailDTO> result = new ArrayList<>();
        for (OrderDTO order : orders) {
            ShipmentDTO shipment = shipmentServiceClient
                .getShipmentByOrderId(order.getOrderId());
            // HTTP: GET /shipments/order/1, /shipments/order/2, ... /shipments/order/10
            // 총 10번의 네트워크 요청!
            
            result.add(new OrderDetailDTO(order, shipment));
        }
        
        return result;
        // 총 요청: 1 (주문 목록) + 10 (각 배송 정보) = 11 번
        // 단일 JOIN으로는 1번이면 되는데!
    }
}

// N+1 최적화: 병렬 처리
@Service
public class OrderListServiceOptimized {
    
    @Autowired
    private OrderServiceClient orderServiceClient;
    
    @Autowired
    private ShipmentServiceClient shipmentServiceClient;
    
    public List<OrderDetailDTO> getOrdersWithShipments(Long customerId) {
        // 1단계: 주문 조회
        List<OrderDTO> orders = orderServiceClient.getOrdersByCustomer(customerId);
        
        // 2단계: 배송 정보 배치로 조회 (최적화)
        List<Long> orderIds = orders.stream()
            .map(OrderDTO::getOrderId)
            .collect(Collectors.toList());
        
        Map<Long, ShipmentDTO> shipments = shipmentServiceClient
            .getShipmentsByOrderIds(orderIds);  // 10개를 한 번에 조회
        
        // 3단계: 조합
        return orders.stream()
            .map(order -> new OrderDetailDTO(order, shipments.get(order.getOrderId())))
            .collect(Collectors.toList());
        // 총 요청: 1 (주문) + 1 (배송 배치) = 2번
    }
}
```

### 전략 2: CQRS (Command Query Responsibility Segregation)

```java
// CQRS: 쓰기는 원본, 읽기는 특화된 모델

// 1단계: 쓰기 모델 (PostgreSQL - 주문 서비스)
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private Long orderId;
    
    @Version
    private Long version;
    
    private Long customerId;
    private BigDecimal totalAmount;
    
    @OneToMany(cascade = CascadeType.ALL)
    private List<OrderLineItem> lineItems;
}

// 2단계: 쓰기 이벤트 발행
@Service
@Transactional
public class OrderCommandService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public void createOrder(CreateOrderCommand command) {
        Order order = new Order();
        order.setCustomerId(command.getCustomerId());
        order.setTotalAmount(command.getTotalAmount());
        order.setLineItems(command.getLineItems());
        
        Order savedOrder = orderRepository.save(order);
        
        // 이벤트 발행 (읽기 모델 업데이트용)
        eventPublisher.publishEvent(
            new OrderCreatedEvent(
                savedOrder.getOrderId(),
                savedOrder.getCustomerId(),
                savedOrder.getTotalAmount()
            )
        );
    }
}

// 3단계: 이벤트 구독으로 읽기 모델 생성 (Elasticsearch)
@Service
public class OrderReadModelProjection {
    
    @Autowired
    private OrderReadRepository orderReadRepository;  // Elasticsearch
    
    @Autowired
    private ShipmentServiceClient shipmentServiceClient;
    
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // 쓰기 모델에서 배송 정보를 조회
        ShipmentDTO shipment = shipmentServiceClient
            .getShipmentByOrderId(event.getOrderId());
        
        // 읽기 모델에 저장 (JOIN된 형태)
        OrderReadModel readModel = new OrderReadModel(
            event.getOrderId(),
            event.getCustomerId(),
            event.getTotalAmount(),
            shipment  // 미리 JOIN된 데이터
        );
        
        orderReadRepository.save(readModel);
    }
}

// 4단계: 읽기 (Elasticsearch에서 빠르게 조회)
@Service
public class OrderQueryService {
    
    @Autowired
    private OrderReadRepository orderReadRepository;
    
    public OrderDetailDTO getOrderDetail(Long orderId) {
        // Elasticsearch에서 조회 (이미 JOIN된 데이터)
        return orderReadRepository.findById(orderId);
        // SQL JOIN과 같은 성능, 하지만 빠른 검색 가능
    }
    
    public List<OrderDetailDTO> getOrdersByCustomer(Long customerId) {
        // 복잡한 쿼리도 Elasticsearch로 빠르게 처리
        return orderReadRepository.findByCustomerId(customerId);
    }
}

// Elasticsearch 읽기 모델 (이미 JOIN된 형태)
@Document(indexName = "orders")
public class OrderReadModel {
    @Id
    private Long orderId;
    
    @Field(type = FieldType.Text)
    private Long customerId;
    
    @Field(type = FieldType.Double)
    private BigDecimal totalAmount;
    
    // 배송 정보 (원래는 다른 DB에 있지만, 읽기 모델에는 포함)
    @Field(type = FieldType.Object)
    private ShipmentDTO shipment;
    
    // 결제 정보도 포함 가능
    @Field(type = FieldType.Object)
    private PaymentDTO payment;
}
```

### 전략 3: 데이터 복제 (Data Replication)

```java
// 데이터 복제: 필요한 데이터를 자신의 DB에 복사

// 시나리오: 주문 조회 시 고객 정보가 필요 (고객 서비스가 느린 경우)

// 1단계: 주문 서비스의 로컬 고객 정보 스냅샷
@Entity
@Table(name = "customer_snapshot")
public class CustomerSnapshot {
    @Id
    private Long customerId;
    
    private String customerName;
    private String email;
    private String phone;
    
    // 언제 마지막으로 동기화했는가
    @Column(name = "synced_at")
    private LocalDateTime syncedAt;
}

// 2단계: 고객 업데이트 이벤트 구독
@Service
public class CustomerSnapshotProjection {
    
    @Autowired
    private CustomerSnapshotRepository snapshotRepository;
    
    // 고객 서비스가 발행한 이벤트 구독
    @EventListener
    @Transactional
    public void onCustomerUpdated(CustomerUpdatedEvent event) {
        // 자신의 DB에 고객 정보 저장 (스냅샷)
        CustomerSnapshot snapshot = new CustomerSnapshot();
        snapshot.setCustomerId(event.getCustomerId());
        snapshot.setCustomerName(event.getCustomerName());
        snapshot.setEmail(event.getEmail());
        snapshot.setPhone(event.getPhone());
        snapshot.setSyncedAt(LocalDateTime.now());
        
        snapshotRepository.save(snapshot);
    }
    
    // 초기 동기화 (또는 주기적 재동기화)
    @Scheduled(fixedDelay = 3600000)  // 1시간마다
    public void syncCustomerData() {
        List<CustomerDTO> allCustomers = customerServiceClient.getAllCustomers();
        
        for (CustomerDTO customer : allCustomers) {
            CustomerSnapshot snapshot = new CustomerSnapshot();
            snapshot.setCustomerId(customer.getId());
            snapshot.setCustomerName(customer.getName());
            snapshot.setEmail(customer.getEmail());
            snapshot.setPhone(customer.getPhone());
            snapshot.setSyncedAt(LocalDateTime.now());
            
            snapshotRepository.saveOrUpdate(snapshot);
        }
    }
}

// 3단계: 조회 시 로컬 스냅샷 사용
@Service
public class OrderDetailServiceReplicated {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private CustomerSnapshotRepository snapshotRepository;
    
    public OrderDetailDTO getOrderDetail(Long orderId) {
        // 주문 조회 (로컬 DB)
        Order order = orderRepository.findById(orderId).orElseThrow();
        
        // 고객 정보 조회 (로컬 스냅샷, 빠름)
        CustomerSnapshot customer = snapshotRepository
            .findById(order.getCustomerId()).orElse(null);
        
        // 조합 (JOIN 대신 메모리에서)
        return new OrderDetailDTO(order, customer);
        // 매우 빠름 (모두 로컬 DB)
        // 하지만 고객 정보는 약간 오래될 수 있음 (Eventual Consistency)
    }
}

// 문제: 고객 정보가 삭제된 경우?
@Service
public class ConsistencyCheckService {
    
    @Autowired
    private CustomerSnapshotRepository snapshotRepository;
    
    @Autowired
    private CustomerServiceClient customerServiceClient;
    
    // 고아 데이터 검증 (Orphan Data)
    @Scheduled(fixedDelay = 86400000)  // 매일
    public void validateCustomerSnapshots() {
        List<CustomerSnapshot> snapshots = snapshotRepository.findAll();
        
        for (CustomerSnapshot snapshot : snapshots) {
            try {
                // 원본 서비스에 고객이 존재하는가?
                CustomerDTO original = customerServiceClient
                    .getCustomer(snapshot.getCustomerId());
                
                if (original == null) {
                    // 고객이 삭제되었음 (스냅샷만 남음)
                    snapshotRepository.delete(snapshot);
                }
            } catch (CustomerNotFoundException e) {
                // 고객 삭제됨 (이벤트를 받지 못한 경우)
                snapshotRepository.delete(snapshot);
            }
        }
    }
}
```

---

## 🔬 내부 동작 원리 — 3가지 전략의 흐름

### API Composition 흐름

```
클라이언트 요청: GET /orders/123/detail
         │
         ▼
┌──────────────────────────────┐
│  OrderDetailService          │
│  (Orchestrator Pattern)      │
└──────────────────────────────┘
    │              │                 │
    │ (병렬 호출)  │                 │
    ▼              ▼                 ▼
┌─────────┐  ┌──────────┐  ┌────────────┐
│ Order   │  │ Shipment │  │ Payment    │
│ Service │  │ Service  │  │ Service    │
└─────────┘  └──────────┘  └────────────┘
    │              │                 │
    │ (응답 수집)  │                 │
    └──────────────┴─────────────────┘
                   │
                   ▼ (데이터 조합)
         OrderDetailDTO
         │
         └─> 클라이언트

장점:
✅ 각 서비스 독립성 (각각 진화 가능)
✅ 구현 단순 (REST API 호출)
✅ 실시간 데이터 (최신 상태)

단점:
❌ N+1 문제 (많은 목록 조회 시)
❌ 네트워크 오버헤드 (3개 API 호출)
❌ 한 서비스 느리면 전체 느림
❌ 부분 실패 처리 복잡 (어떤 서비스 데이터는 있고 없고)
```

### CQRS 흐름

```
쓰기 흐름:
┌─────────┐
│ 주문    │
│ 생성    │
└─────────┘
    │
    ▼
┌─────────────────────────┐
│ Order Service (쓰기)     │
│ PostgreSQL              │
└─────────────────────────┘
    │
    │ OrderCreatedEvent 발행
    ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Shipment     │  │ Payment      │  │ Analytics    │
│ 서비스 구독  │  │ 서비스 구독  │  │ 서비스 구독  │
└──────────────┘  └──────────────┘  └──────────────┘

읽기 흐름:
┌──────────────┐
│ 주문 상세    │
│ 조회 요청    │
└──────────────┘
    │
    ▼ (읽기 모델에서 조회)
┌─────────────────────────┐
│ OrderReadModel          │
│ Elasticsearch           │
│ (이미 JOIN된 데이터)    │
│ {                       │
│   order: {...},         │
│   shipment: {...},      │
│   payment: {...}        │
│ }                       │
└─────────────────────────┘
    │
    ▼
  응답 (매우 빠름)

장점:
✅ 읽기 성능 최적 (인덱스된 데이터)
✅ 복잡한 쿼리 가능 (읽기 모델 특화)
✅ 쓰기와 읽기 독립적 스케일
✅ 비동기 업데이트 (이벤트 기반)

단점:
❌ 최종 일관성 (쓰기 후 읽기 모델 업데이트 지연)
❌ 구현 복잡 (이벤트, 이벤트 핸들러)
❌ 읽기 모델과 쓰기 모델 불일치 가능성
❌ 이벤트 중복 처리 문제 (멱등성 필요)
```

### 데이터 복제 흐름

```
초기 동기화:
고객 서비스
    │
    ├─ 모든 고객 데이터
    │
    ▼
주문 서비스
    └─ customer_snapshot 테이블

이벤트 기반 동기화:
고객 서비스
    │
    │ CustomerUpdatedEvent
    ▼
주문 서비스 (이벤트 구독)
    │
    └─ customer_snapshot 업데이트

조회 시:
GET /orders/123/detail
    │
    ▼
SELECT o.*, cs.* FROM orders o
JOIN customer_snapshot cs ON o.customer_id = cs.customer_id

(모두 로컬 DB, 매우 빠름)

장점:
✅ 조회 성능 (JOIN 가능, 로컬 DB)
✅ 구현 단순 (로컬 JOIN)
✅ 네트워크 호출 없음
✅ 기존 RDBMS 스킬 활용

단점:
❌ 데이터 중복 (스토리지 비용)
❌ 동기화 관리 필요 (이벤트 구독)
❌ 일관성 문제 (고아 데이터)
❌ 대용량 데이터는 스냅샷 어려움
```

---

## 💻 Java 실전 코드

```java
// API Composition 최적화: 동시 조회
@Service
public class OrderDetailServiceAsync {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Autowired
    private AsyncRestTemplate asyncRestTemplate;
    
    public OrderDetailDTO getOrderDetailAsync(Long orderId) {
        // 비동기 병렬 호출
        ListenableFuture<OrderDTO> orderFuture = 
            asyncRestTemplate.getForEntity(
                "http://order-service/api/orders/" + orderId,
                OrderDTO.class
            ).map(HttpEntity::getBody);
        
        ListenableFuture<ShipmentDTO> shipmentFuture = 
            asyncRestTemplate.getForEntity(
                "http://shipment-service/api/shipments/order/" + orderId,
                ShipmentDTO.class
            ).map(HttpEntity::getBody);
        
        ListenableFuture<PaymentDTO> paymentFuture = 
            asyncRestTemplate.getForEntity(
                "http://payment-service/api/payments/order/" + orderId,
                PaymentDTO.class
            ).map(HttpEntity::getBody);
        
        // 모두 완료될 때까지 대기
        ListenableFuture<OrderDetailDTO> detailFuture = 
            Futures.transform(
                Futures.allAsList(orderFuture, shipmentFuture, paymentFuture),
                results -> {
                    OrderDTO order = (OrderDTO) results.get(0);
                    ShipmentDTO shipment = (ShipmentDTO) results.get(1);
                    PaymentDTO payment = (PaymentDTO) results.get(2);
                    return new OrderDetailDTO(order, shipment, payment);
                }
            );
        
        return detailFuture.get();
    }
}

// CQRS 읽기 모델 업데이트 (멱등성)
@Service
public class OrderReadModelProjectionIdempotent {
    
    @Autowired
    private OrderReadRepository readRepository;
    
    @EventListener
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        // 멱등성: 같은 이벤트를 여러 번 처리해도 안전
        OrderReadModel existing = readRepository.findById(event.getOrderId())
            .orElse(null);
        
        if (existing != null) {
            // 이미 처리한 이벤트 (중복)
            if (existing.getSyncedAt().isAfter(event.getEventTime())) {
                // 더 최신 정보가 있으므로 무시
                return;
            }
        }
        
        OrderReadModel readModel = new OrderReadModel();
        readModel.setOrderId(event.getOrderId());
        readModel.setCustomerId(event.getCustomerId());
        readModel.setTotalAmount(event.getTotalAmount());
        readModel.setSyncedAt(event.getEventTime());
        
        readRepository.save(readModel);  // INSERT OR UPDATE
    }
}

// 데이터 복제: 부분 실패 처리
@Service
public class CustomerSnapshotServiceResilient {
    
    @Autowired
    private CustomerSnapshotRepository snapshotRepository;
    
    @Autowired
    private CustomerServiceClient customerServiceClient;
    
    @EventListener
    @Transactional
    public void onCustomerUpdated(CustomerUpdatedEvent event) {
        try {
            // 원본 서비스에서 최신 데이터 조회
            CustomerDTO latest = customerServiceClient
                .getCustomer(event.getCustomerId());
            
            // 스냅샷 저장
            CustomerSnapshot snapshot = new CustomerSnapshot();
            snapshot.setCustomerId(latest.getId());
            snapshot.setCustomerName(latest.getName());
            snapshot.setEmail(latest.getEmail());
            snapshot.setSyncedAt(LocalDateTime.now());
            
            snapshotRepository.save(snapshot);
        } catch (CustomerServiceUnavailableException e) {
            // 고객 서비스가 다운된 경우
            // 이벤트의 정보로만 스냅샷 업데이트
            CustomerSnapshot snapshot = snapshotRepository
                .findById(event.getCustomerId())
                .orElse(new CustomerSnapshot());
            
            snapshot.setCustomerId(event.getCustomerId());
            snapshot.setCustomerName(event.getCustomerName());
            snapshot.setEmail(event.getEmail());
            snapshot.setSyncedAt(LocalDateTime.now());
            snapshot.setFromEvent(true);  // 이벤트 데이터 플래그
            
            snapshotRepository.save(snapshot);
            
            log.warn("고객 서비스 장애, 이벤트 데이터로 동기화: " + event.getCustomerId());
        }
    }
}
```

---

## 📊 3가지 전략 비교

| 항목 | API Composition | CQRS | 데이터 복제 |
|------|-----------------|------|-----------|
| **구현 복잡도** | 낮음 | 높음 | 중간 |
| **조회 성능** | 중간 (N+1) | 높음 (인덱스) | 높음 (로컬 JOIN) |
| **데이터 신선도** | 실시간 | 약간 지연 (비동기) | 약간 지연 (동기화) |
| **네트워크** | 많은 호출 | 적음 (읽기만) | 없음 (로컬) |
| **일관성** | 강함 | 최종 (Eventual) | 최종 (Eventual) |
| **확장성** | 중간 | 높음 (읽기 특화) | 중간 |
| **저장소** | 없음 (API만) | 높음 (읽기 모델) | 높음 (스냅샷) |
| **장애 격리** | 약함 (한 서비스 느리면 전체) | 강함 (읽기는 독립) | 강함 (로컬 DB) |

---

## ⚖️ 트레이드오프

```
조회 성능 vs 구현 복잡도
├─ API Composition
│  ├─ ✅ 구현 단순 (REST 호출)
│  ├─ ✅ 데이터 신선도
│  ├─ ❌ N+1 성능 문제
│  └─ ❌ 네트워크 오버헤드
│
├─ CQRS
│  ├─ ✅ 읽기 성능 최적
│  ├─ ✅ 복잡한 쿼리 가능
│  ├─ ❌ 최종 일관성
│  └─ ❌ 구현 복잡
│
└─ 데이터 복제
   ├─ ✅ 로컬 JOIN (빠름)
   ├─ ✅ 구현 중간
   ├─ ❌ 스토리지 중복
   └─ ❌ 동기화 관리

선택 기준:
├─ 단일 서비스 조회 → API Composition (단순)
├─ 복잡한 검색 쿼리 → CQRS (Elasticsearch)
├─ 자주 사용되는 데이터 → 데이터 복제 (로컬 JOIN)
├─ 높은 성능 요구 → CQRS 또는 복제
└─ 강한 일관성 필수 → API Composition
```

---

## 📌 핵심 정리

```
✅ Database per Service에서 JOIN은 불가능:
  - 물리적 FK 사용 불가
  - 크로스 DB JOIN 불가
  - 다른 방법 필요

✅ 3가지 해결 전략:
  1. API Composition
     - 여러 서비스 API 호출 후 애플리케이션에서 조합
     - 단순하지만 N+1 문제
  2. CQRS
     - 쓰기: 원본 DB, 읽기: 특화된 읽기 모델
     - 복잡하지만 최고의 읽기 성능
  3. 데이터 복제
     - 필요한 데이터 스냅샷으로 저장
     - 로컬 JOIN 가능, 동기화 관리 필요

✅ 각 전략의 특성:
  - API: 실시간, 구현 단순, N+1 문제
  - CQRS: 지연, 구현 복잡, 최적 성능
  - 복제: 지연, 중간 복잡도, 빠른 조회

✅ 선택 기준:
  - 빠른 응답 필요 → CQRS 또는 복제
  - 강한 일관성 필수 → API Composition
  - 자주 접근하는 데이터 → 복제
  - 복잡한 검색 → CQRS
```

---

## 🤔 생각해볼 문제

**Q1.** API Composition에서 주문 목록 조회 시 각 주문의 배송 상태가 필요합니다. 10개 주문이 있다면 총 몇 번의 API 호출이 필요하고, 이를 N+1 최적화로 줄이려면 어떻게 해야 할까요?

<details>
<summary>해설 보기</summary>

N+1 문제:
- 기본: 1 (주문 목록) + 10 (각 배송 조회) = 11번
- 최악: 각 주문 + 각 배송 + 각 결제 = 1 + 10 + 10 = 21번

N+1 최적화:
1. **배치 조회 API 추가**:
   ```
   GET /shipments?orderIds=1,2,3,...,10
   ```
   - 10개를 한 번에 조회
   - 호출: 1 + 1 = 2번

2. **응답 캐싱**:
   ```
   @Cacheable(value = "shipment", key = "#orderId")
   Shipment getShipment(Long orderId)
   ```
   - 같은 주문 조회 시 캐시 사용
   - 반복 조회 줄임

3. **필드 선택**:
   ```
   GET /orders?fields=id,customerId&include=shipment
   ```
   - 불필요한 필드 제외
   - 네트워크 트래픽 감소

4. **비동기 병렬 호출**:
   - 10번을 순차가 아니라 병렬 실행
   - 응답 시간은 1번 시간의 10배 아님

결론: API 설계 단계에서 배치 조회를 지원하는 것이 가장 효과적입니다.

</details>

**Q2.** CQRS를 사용하는 환경에서 읽기 모델을 업데이트하는 이벤트 핸들러가 중복으로 실행되었습니다 (같은 이벤트 2번). 읽기 모델이 잘못된 데이터를 가질 가능성은?

<details>
<summary>해설 보기</summary>

CQRS의 멱등성 문제:

이벤트 중복 실행 시나리오:
1. OrderCreatedEvent 발행
2. OrderReadModelProjection 구독, 읽기 모델 업데이트
3. 네트워크 지연으로 ACK 전송 실패
4. 메시지 브로커가 이벤트 재전송
5. 다시 읽기 모델 업데이트 (중복)

멱등성 보장 방법:
```java
@EventListener
@Transactional
public void onOrderCreated(OrderCreatedEvent event) {
    // 방법 1: 이벤트 ID 기반 중복 제거
    if (eventIdStore.has(event.getEventId())) {
        return;  // 이미 처리함
    }
    
    // 처리
    OrderReadModel model = createModel(event);
    readRepository.save(model);
    
    // 처리 기록
    eventIdStore.mark(event.getEventId());
}

// 방법 2: 데이터베이스 유니크 제약
@Entity
@Table(uniqueConstraints = {
    @UniqueConstraint(columnNames = {"orderId", "version"})
})
public class OrderReadModel {
    private Long orderId;
    private Long version;  // 이벤트 버전
}
```

결론: CQRS에서는 읽기 모델이 **멱등한(Idempotent)** 업데이트를 받아야 합니다. 같은 이벤트를 여러 번 처리해도 결과가 같아야 합니다.

</details>

**Q3.** 데이터 복제 전략에서 고객 서비스가 다운되었을 때, 주문 서비스의 customer_snapshot은 오래된 데이터입니다. 주문 조회 시 최신 데이터를 표시하려면 어떻게 해야 할까요?

<details>
<summary>해설 보기</summary>

복제 데이터의 신선도 문제:

상황:
- customer_snapshot: 2024년 1월의 고객 정보
- 고객 서비스: 다운 (최신 정보에 접근 불가)
- 사용자: "최신 정보를 보고 싶다"

해결책:

1. **Stale-While-Revalidate 패턴**:
   ```java
   public OrderDetailDTO getOrderDetail(Long orderId) {
       // 1단계: 로컬 스냅샷 (빠른 응답)
       OrderDetailDTO detail = getFromSnapshot(orderId);
       
       // 2단계: 백그라운드에서 최신 데이터 업데이트 시도
       customerServiceClient.getCustomerAsync(detail.getCustomerId())
           .thenAccept(latest -> updateSnapshot(latest))
           .exceptionally(ex -> {
               // 고객 서비스 장애 시 로컬 데이터로 응답
               return null;
           });
       
       return detail;  // 즉시 응답 (스냅샷 사용)
   }
   ```

2. **데이터 신선도 표시**:
   ```java
   @Data
   public class OrderDetailDTO {
       // ...
       private LocalDateTime customerDataUpdatedAt;
       private boolean customerDataStale;  // 24시간 이상 오래됨
   }
   
   // 프론트엔드는 stale 데이터를 표시
   // "이 정보는 1월 15일 기준입니다"
   ```

3. **Fallback 전략**:
   ```java
   public OrderDetailDTO getOrderDetail(Long orderId) {
       OrderDetailDTO detail = getFromSnapshot(orderId);
       
       try {
           // 고객 서비스에서 최신 데이터 조회
           CustomerDTO latest = customerServiceClient.getCustomer(detail.getCustomerId());
           detail.setCustomer(latest);  // 최신 데이터로 교체
       } catch (ServiceUnavailableException e) {
           // 고객 서비스 장애: 스냅샷 데이터 사용
           // 프론트엔드에 경고 표시
           detail.setCustomerDataWarning("고객 정보가 오래되었을 수 있습니다");
       }
       
       return detail;
   }
   ```

결론: 복제 데이터는 신뢰할 수 없을 수 있으므로:
- 신선도 메타데이터 추가
- 가능하면 실시간 데이터 조회 시도
- 실패 시 스냅샷 데이터 + 경고 표시

</details>

---

<div align="center">

**[⬅️ 이전: Polyglot Persistence — 서비스별 DB 기술 선택](./02-polyglot-persistence.md)** | **[홈으로 🏠](../README.md)** | **[다음: 분산 데이터 일관성 — ACID vs BASE ➡️](./04-distributed-data-consistency.md)**

</div>

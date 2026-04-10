# 05. 데이터 이관 전략 — 모놀리스 DB에서 서비스별 DB로

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 모놀리스 시스템을 마이크로서비스로 마이그레이션할 때 DB를 어떻게 분리하는가?
- Strangler Fig 패턴이 데이터 이관에 어떻게 적용되는가?
- 이중 쓰기(Dual Write) 전략의 위험과 해결 방법은?
- Kafka Outbox Pattern으로 원자성을 어떻게 보장하는가?
- Shadow Mode를 사용한 데이터 검증 방법은?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

**대부분의 마이크로서비스 여정은 모놀리스에서 시작됩니다.** 초기 스타트업은 빠른 개발을 위해 모든 코드를 한 곳에 모으고, 하나의 데이터베이스에 모든 데이터를 저장합니다. 하지만 회사가 성장하면서 코드베이스가 커지고, DB도 복잡해지고, 팀도 늘어나고, 배포 위험도 증가합니다.

**그래서 마이크로서비스로 전환하기로 결정합니다.** 하지만 **"오늘은 모놀리스, 내일은 마이크로서비스"는 불가능합니다.** 이미 있는 고객의 데이터를 잃을 수 없고, 서비스 중단도 없어야 합니다. 라이브 환경에서 데이터를 조용히 분리해야 합니다.

이것이 **데이터 이관 전략**의 핵심입니다. Strangler Fig 패턴으로 점진적으로 전환하면서, 이중 쓰기, Outbox Pattern, Shadow Mode 등을 통해 데이터 일관성을 유지해야 합니다. 이 모든 기술을 정확히 이해하고 순서대로 적용해야 안전합니다.

---

## 😱 흔한 실수 (Before — 무계획 이관)

```java
// BEFORE: 위험한 데이터 이관 시도

@Service
public class OrderServiceUnsafeMigration {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;  // 모놀리스 DB (PostgreSQL 단일)
    
    @Autowired
    private NewOrderRepository newOrderRepository;  // 새 마이크로서비스 DB
    
    // 😱 위험: 프로덕션에서 직접 데이터 복사
    public void migrateOrdersUnsafe() {
        // Step 1: 기존 DB에서 모든 주문 읽기
        String sql = "SELECT * FROM orders";
        List<Order> oldOrders = jdbcTemplate.query(sql, new OrderRowMapper());
        
        // Step 2: 새 DB에 쓰기 (바로!)
        for (Order order : oldOrders) {
            newOrderRepository.save(order);  // 즉시 저장
        }
        
        // 문제 발생:
        // ❌ 마이그레이션 중에 새 주문이 생기면?
        // ❌ 쓰기 중 네트워크 장애 → 부분 이관
        // ❌ 데이터 검증이 없음 → 일관성 보장 불가
        // ❌ 롤백 불가 (새 DB에는 쓰기했으니)
        // ❌ 기존 애플리케이션도 계속 돌고 있음 (중복/누락 위험)
    }
}

// 또 다른 위험한 시도: 이중 쓰기 없이 바로 전환
@Service
public class OrderServiceDirectCutover {
    
    @Autowired
    private OrderRepository oldOrderRepository;     // 기존 DB
    
    @Autowired
    private NewOrderRepository newOrderRepository;  // 신규 DB
    
    private boolean USE_NEW_DB = false;  // 언제 true로 전환할지 모름
    
    public Order createOrder(OrderRequest request) {
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        order.setTotalAmount(request.getTotalAmount());
        
        if (USE_NEW_DB) {
            // 신규 DB만 사용
            return newOrderRepository.save(order);
        } else {
            // 기존 DB 사용
            return oldOrderRepository.save(order);
        }
        
        // 문제:
        // ❌ 전환 순간 데이터 불일치 (양쪽 다 다른 상태)
        // ❌ 이전 데이터를 조회할 때 기존 DB를 봐야 하나? 신규 DB를 봐야 하나?
        // ❌ 조인이 불가능 (데이터가 양쪽에 분산)
        // ❌ 빅뱅 전환 (모두 깨질 위험)
    }
}

// 위험한 이중 쓰기: 원자성 없음
@Service
public class OrderServiceNaiveDualWrite {
    
    @Autowired
    private OrderRepository oldDb;
    
    @Autowired
    private OrderRepository newDb;
    
    public Order createOrder(OrderRequest request) {
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        
        // 이중 쓰기 시도 (원자성 없음)
        Order oldResult = oldDb.save(order);
        
        try {
            Order newResult = newDb.save(order);  // 실패 가능성
        } catch (Exception e) {
            // 문제: 기존 DB에는 저장됐는데 신규 DB에는 실패
            // → 데이터 불일치! 롤백도 불가능
            log.error("신규 DB 쓰기 실패: 기존 DB와 불일치!");
        }
        
        return oldResult;
    }
}

// 위험: 실패 가능성
/*
시나리오: 신규 DB가 느린 경우

기존 DB: INSERT → 즉시 완료
신규 DB: INSERT → 타임아웃 (10초) → 실패

결과:
- 기존 DB: order_id = 1 저장됨
- 신규 DB: 저장 실패

조회:
- 기존 앱에서 order 1 조회: ✅
- 신규 앱에서 order 1 조회: ❌ (없음)

→ 서비스 간 데이터 불일치!
*/
```

---

## ✨ 올바른 접근 (After — Strangler Fig + Outbox Pattern)

### 단계 1: 라우팅 레이어 추가 (Strangler Fig)

```java
// Strangler Fig Pattern: 점진적 전환

@Service
public class OrderServiceStranglerFig {
    
    @Autowired
    private OrderRepository oldDb;  // 기존 모놀리스 DB
    
    @Autowired
    private OrderRepository newDb;  // 신규 마이크로서비스 DB
    
    @Autowired
    private FeatureToggleService featureToggle;
    
    // 읽기: 점진적으로 신규 DB로 전환
    public Order getOrder(Long orderId) {
        if (featureToggle.isEnabled("use-new-order-db")) {
            // 50%는 신규 DB에서 읽기 (카나리 배포 같은 느낌)
            return newDb.findById(orderId).orElseThrow();
        } else {
            // 50%는 기존 DB에서 읽기
            return oldDb.findById(orderId).orElseThrow();
        }
    }
    
    // 쓰기: 항상 양쪽 다 (이중 쓰기)
    @Transactional
    public Order createOrder(OrderRequest request) {
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        order.setTotalAmount(request.getTotalAmount());
        
        // 기존 DB에 저장 (원본)
        Order savedOrder = oldDb.save(order);
        
        // 신규 DB에도 저장 (복제)
        try {
            newDb.save(order);
        } catch (Exception e) {
            // 실패 시에도 기존 DB는 동작하므로 서비스 지속
            log.warn("신규 DB 저장 실패: " + e.getMessage());
            // 나중에 재시도하거나 배치로 동기화
        }
        
        return savedOrder;
    }
}

// Feature Toggle로 점진적 전환
/*
Day 1: 쓰기 시작 (기존 + 신규), 읽기는 기존만
   기존 DB: 새 주문 저장
   신규 DB: 새 주문 복사 시작

Day 7: 신규 DB에 일주일 데이터 축적
   데이터 검증 시작

Day 14: 읽기를 10%만 신규 DB에서
   신규 DB 안정성 확인

Day 21: 읽기를 50% 신규 DB에서
   양쪽 성능 비교

Day 28: 읽기를 90% 신규 DB에서
   기존 DB가 장애 시 대체 확인

Day 35: 읽기 100% 신규 DB에서
   기존 DB 제거 가능
*/
```

### 단계 2: Outbox Pattern으로 원자성 보장

```java
// Outbox Pattern: 이중 쓰기의 원자성 문제 해결

@Entity
@Table(name = "orders")
public class Order {
    @Id
    private Long orderId;
    private Long customerId;
    private BigDecimal totalAmount;
}

@Entity
@Table(name = "outbox")  // Outbox 테이블 (같은 DB)
public class OutboxEvent {
    @Id
    @GeneratedValue
    private Long eventId;
    
    private String eventType;
    private String payload;
    private LocalDateTime createdAt;
    
    @Enumerated(EnumType.STRING)
    private OutboxStatus status;  // PENDING, PUBLISHED, FAILED
}

@Service
public class OrderServiceWithOutbox {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private OutboxRepository outboxRepository;
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // Step 1: 주문 저장 + Outbox 이벤트 저장 (같은 트랜잭션)
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        order.setTotalAmount(request.getTotalAmount());
        Order savedOrder = orderRepository.save(order);
        
        // Step 2: 이벤트를 Outbox에 기록 (같은 트랜잭션)
        OutboxEvent event = new OutboxEvent();
        event.setEventType("OrderCreated");
        event.setPayload(objectToJson(savedOrder));
        event.setStatus(OutboxStatus.PENDING);
        event.setCreatedAt(LocalDateTime.now());
        outboxRepository.save(event);  // 원자적으로 저장됨
        
        return savedOrder;
    }
}

// 별도 스레드: Outbox Poller (주기적으로 실행)
@Service
public class OutboxPoller {
    
    @Autowired
    private OutboxRepository outboxRepository;
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Scheduled(fixedDelay = 1000)  // 1초마다 실행
    public void pollAndPublish() {
        // PENDING 상태인 모든 이벤트 조회
        List<OutboxEvent> pendingEvents = outboxRepository
            .findByStatus(OutboxStatus.PENDING);
        
        for (OutboxEvent event : pendingEvents) {
            try {
                // Kafka에 발행
                kafkaTemplate.send("order-events", 
                    event.getEventId().toString(), 
                    event.getPayload());
                
                // 발행 완료 표시
                event.setStatus(OutboxStatus.PUBLISHED);
                outboxRepository.save(event);
            } catch (Exception e) {
                // 실패: 다음 폴링 때 재시도
                event.setStatus(OutboxStatus.FAILED);
                outboxRepository.save(event);
            }
        }
    }
}

// 신규 DB를 업데이트하는 서비스
@Service
public class NewOrderDBUpdater {
    
    @Autowired
    private NewOrderRepository newOrderRepository;
    
    @KafkaListener(topics = "order-events")
    public void onOrderCreated(String eventPayload) {
        // Kafka에서 이벤트를 받아 신규 DB에 저장
        Order order = jsonToObject(eventPayload, Order.class);
        newOrderRepository.save(order);
    }
}

// Outbox Pattern의 보장
/*
장점:
✅ 원자성: Order + OutboxEvent를 같은 트랜잭션에서 저장
✅ 보증 전달: Outbox Poller가 반복 실행하므로 결국 전달됨
✅ 중복 제거: 같은 이벤트가 여러 번 보내지더라도 멱등 처리 가능

시나리오: 네트워크 장애로 Kafka 발행 실패
t=0: Order 저장 + OutboxEvent(PENDING) 저장 ✅
t=1: Kafka 발행 시도 → 네트워크 오류
t=2: OutboxEvent 상태 = PENDING (변경 안 됨)
t=3~: Outbox Poller 계속 시도
t=100: 네트워크 복구 → Kafka 발행 성공
t=101: OutboxEvent 상태 = PUBLISHED ✅

→ 데이터 손실 없음!
*/
```

### 단계 3: Shadow Mode로 검증

```java
// Shadow Mode: 신규 DB의 결과를 검증하지만 실제 응답에는 사용하지 않음

@Service
public class OrderServiceShadowMode {
    
    @Autowired
    private OrderRepository oldDb;
    
    @Autowired
    private OrderRepository newDb;
    
    @Autowired
    private DataConsistencyValidator validator;
    
    @Autowired
    private ShadowModeMetrics metrics;
    
    public List<Order> getOrdersByCustomer(Long customerId) {
        // 1단계: 기존 DB에서 조회 (실제 응답)
        List<Order> oldResult = oldDb.findByCustomerId(customerId);
        
        // 2단계: 신규 DB에서도 조회 (비동기, 결과는 버림)
        asyncQueryAndCompare(customerId, oldResult);
        
        // 3단계: 기존 DB 결과 반환
        return oldResult;
    }
    
    private void asyncQueryAndCompare(Long customerId, List<Order> expectedResult) {
        CompletableFuture.runAsync(() -> {
            try {
                // 신규 DB에서 조회
                List<Order> newResult = newDb.findByCustomerId(customerId);
                
                // 데이터 비교
                ValidationResult result = validator.compare(expectedResult, newResult);
                
                if (!result.isConsistent()) {
                    // 불일치 감지
                    metrics.recordInconsistency(customerId, result);
                    log.warn("데이터 불일치 감지: customerId={}, " +
                            "oldCount={}, newCount={}, diff={}",
                        customerId, 
                        expectedResult.size(),
                        newResult.size(),
                        result.getDifferences());
                } else {
                    metrics.recordConsistency(customerId);
                }
            } catch (Exception e) {
                // 신규 DB 장애는 무시 (실제 응답에 영향 없음)
                metrics.recordError(customerId, e);
            }
        });
    }
}

// 데이터 불일치 감지 및 모니터링
@Service
public class DataConsistencyValidator {
    
    public ValidationResult compare(List<Order> expected, List<Order> actual) {
        ValidationResult result = new ValidationResult();
        
        // 1. 개수 비교
        if (expected.size() != actual.size()) {
            result.addDifference(
                "COUNT_MISMATCH",
                "Expected: " + expected.size() + ", Actual: " + actual.size()
            );
        }
        
        // 2. 순서대로 비교
        for (int i = 0; i < Math.min(expected.size(), actual.size()); i++) {
            Order exp = expected.get(i);
            Order act = actual.get(i);
            
            if (!exp.getOrderId().equals(act.getOrderId())) {
                result.addDifference(
                    "ID_MISMATCH",
                    "Index: " + i + ", Expected: " + exp.getOrderId() + 
                    ", Actual: " + act.getOrderId()
                );
            }
            
            if (!exp.getTotalAmount().equals(act.getTotalAmount())) {
                result.addDifference(
                    "AMOUNT_MISMATCH",
                    "OrderId: " + exp.getOrderId() + 
                    ", Expected: " + exp.getTotalAmount() + 
                    ", Actual: " + act.getTotalAmount()
                );
            }
        }
        
        return result;
    }
}

// Shadow Mode를 통한 점진적 신뢰도 확인
/*
Week 1: Shadow Mode 시작
        신규 DB: 데이터 계속 축적 중
        모니터링: 불일치 없음 ✅

Week 2: 약간의 불일치 발견
        원인: 타임존 문제 (날짜 저장이 다름)
        수정: 신규 DB에서 타임존 처리 추가

Week 3: 모든 쿼리에서 100% 일치
        신뢰도: 매우 높음
        다음 단계로 진행 가능

Week 4: 읽기 10%를 신규 DB로 전환
        실제 사용자 피드백 수집
*/
```

---

## 🔬 내부 동작 원리 — Strangler Fig 마이그레이션 전체 흐름

```
Phase 1: 준비 (동시 운영 시작)
┌──────────────────────────────────────┐
│        모놀리스 (기존 DB)              │
│  ┌────────────────────────────────┐  │
│  │ orders, shipments, payments... │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
         │
         │ Outbox Pattern 추가
         ├─ Outbox 테이블 추가
         ├─ Outbox Poller 시작
         └─ Kafka 토픽 설정
         │
         ▼
┌──────────────────────────────────────┐
│      신규 마이크로서비스 (신규 DB)     │
│  ┌────────────────────────────────┐  │
│  │ order_service_db (비어있음)     │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘

Phase 2: 데이터 복제 (이중 쓰기 + 배치 이관)
┌──────────────────────────────────────┐
│        모놀리스 (기존 DB)              │
│  ┌────────────────────────────────┐  │
│  │ orders (기존 데이터 + 신규) ✅  │  │
│  │ outbox (이벤트 기록)            │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
         │
         ├─ Dual Write (신규 주문)
         ├─ Batch Sync (기존 주문)
         │
         ▼
┌──────────────────────────────────────┐
│      신규 마이크로서비스 (신규 DB)     │
│  ┌────────────────────────────────┐  │
│  │ orders (복제됨) + Outbox 구독  │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘

Phase 3: 검증 (Shadow Mode)
모든 읽기 요청:
  1. 기존 DB에서 조회 → 응답 반환
  2. 신규 DB에서 조회 (비동기) → 검증만
  
검증 통과 비율:
  Week 1: 98% (타임존 문제)
  Week 2: 99.9% (수정됨)
  Week 3: 100% ✅

Phase 4: 읽기 전환 (카나리 배포)
  Day 1: 5%는 신규 DB에서 읽기
  Day 2: 10%는 신규 DB에서 읽기
  Day 3: 25%는 신규 DB에서 읽기
  Day 4: 50%는 신규 DB에서 읽기 (AB 테스트)
  Day 5: 90%는 신규 DB에서 읽기
  Day 6: 100%는 신규 DB에서 읽기 ✅

Phase 5: 쓰기 전환
모든 신규 요청:
  1. 신규 DB에 쓰기 (원본)
  2. 기존 DB에는 Outbox로만 동기화 (선택사항)

Phase 6: 기존 시스템 제거
기존 DB:
  ├─ Backup 저장
  ├─ Archive (1년)
  └─ 최종 제거
```

---

## 💻 Java 실전 코드 — 완전한 이관 전략

```java
// 1. Outbox Pattern 구현
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long eventId;
    
    @Column(nullable = false)
    private String aggregateId;
    
    @Column(nullable = false)
    private String eventType;
    
    @Lob
    @Column(nullable = false)
    private String payload;
    
    @Enumerated(EnumType.STRING)
    private OutboxEventStatus status;
    
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @Column
    private LocalDateTime publishedAt;
}

@Service
@Transactional
public class OrderServiceWithOutbox {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private OutboxEventRepository outboxRepository;
    
    public Order createOrder(CreateOrderCommand command) {
        // Step 1: 도메인 로직 실행
        Order order = Order.create(
            command.getCustomerId(),
            command.getLineItems()
        );
        
        // Step 2: Order + OutboxEvent를 같은 트랜잭션에 저장
        Order savedOrder = orderRepository.save(order);
        
        // Step 3: 이벤트를 Outbox에 기록
        OutboxEvent event = new OutboxEvent();
        event.setAggregateId(savedOrder.getId().toString());
        event.setEventType("OrderCreatedEvent");
        event.setPayload(serializeToJson(new OrderCreatedEvent(
            savedOrder.getId(),
            savedOrder.getCustomerId(),
            savedOrder.getTotalAmount()
        )));
        event.setStatus(OutboxEventStatus.PENDING);
        event.setCreatedAt(LocalDateTime.now());
        
        outboxRepository.save(event);
        // 여기서 트랜잭션 커밋 → Order + OutboxEvent 모두 저장됨
        
        return savedOrder;
    }
}

// 2. Outbox Poller (별도 스케줄 태스크)
@Service
public class OutboxEventPoller {
    
    @Autowired
    private OutboxEventRepository outboxRepository;
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    private static final int BATCH_SIZE = 100;
    private static final long RETRY_DELAY_MS = 5000;
    
    @Scheduled(fixedDelay = 1000)
    public void pollAndPublish() {
        try {
            List<OutboxEvent> pendingEvents = outboxRepository
                .findByStatusOrderByCreatedAtAsc(
                    OutboxEventStatus.PENDING,
                    PageRequest.of(0, BATCH_SIZE)
                );
            
            for (OutboxEvent event : pendingEvents) {
                publishEvent(event);
            }
        } catch (Exception e) {
            log.error("Outbox 폴링 중 오류 발생", e);
        }
    }
    
    private void publishEvent(OutboxEvent event) {
        try {
            // Kafka에 발행
            SendResult<String, String> result = kafkaTemplate
                .send("order-events", 
                    event.getAggregateId(),
                    event.getPayload())
                .get();
            
            // 발행 성공
            event.setStatus(OutboxEventStatus.PUBLISHED);
            event.setPublishedAt(LocalDateTime.now());
            outboxRepository.save(event);
            
        } catch (Exception e) {
            // 발행 실패 (네트워크, Kafka 다운)
            // 상태를 변경하지 않음 → 다음 폴링 때 재시도
            log.warn("Outbox 이벤트 발행 실패: eventId={}", 
                event.getEventId(), e);
        }
    }
}

// 3. 신규 DB를 업데이트하는 서비스
@Service
public class NewOrderServiceDatabaseUpdater {
    
    @Autowired
    private NewOrderRepository newOrderRepository;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @KafkaListener(topics = "order-events", 
                   groupId = "new-order-service",
                   containerFactory = "kafkaListenerContainerFactory")
    @Transactional
    public void onOrderEvent(String payload) {
        try {
            Map<String, Object> event = objectMapper
                .readValue(payload, new TypeReference<>() {});
            
            String eventType = (String) event.get("eventType");
            
            switch (eventType) {
                case "OrderCreatedEvent":
                    handleOrderCreated(event);
                    break;
                case "OrderCancelledEvent":
                    handleOrderCancelled(event);
                    break;
            }
        } catch (JsonProcessingException e) {
            log.error("이벤트 파싱 실패: {}", payload, e);
        }
    }
    
    private void handleOrderCreated(Map<String, Object> event) {
        Order order = new Order();
        order.setId(Long.parseLong(event.get("orderId").toString()));
        order.setCustomerId(Long.parseLong(event.get("customerId").toString()));
        order.setTotalAmount(new BigDecimal(event.get("totalAmount").toString()));
        
        newOrderRepository.save(order);
    }
    
    private void handleOrderCancelled(Map<String, Object> event) {
        Long orderId = Long.parseLong(event.get("orderId").toString());
        Order order = newOrderRepository.findById(orderId).orElse(null);
        
        if (order != null) {
            order.setStatus("CANCELLED");
            newOrderRepository.save(order);
        }
    }
}

// 4. Shadow Mode 검증
@Service
public class ShadowModeQueryValidator {
    
    @Autowired
    private OrderRepository oldOrderRepository;
    
    @Autowired
    private NewOrderRepository newOrderRepository;
    
    @Autowired
    private DataConsistencyMetrics metrics;
    
    public List<Order> getOrdersByCustomer(Long customerId) {
        // 1단계: 기존 DB에서 조회 (실제 응답)
        List<Order> result = oldOrderRepository.findByCustomerId(customerId);
        
        // 2단계: Shadow Mode - 신규 DB에서도 조회 후 검증
        validateAgainstNewDb(customerId, result);
        
        return result;
    }
    
    @Async
    private void validateAgainstNewDb(Long customerId, List<Order> expectedResult) {
        try {
            List<Order> newResult = newOrderRepository.findByCustomerId(customerId);
            
            if (expectedResult.size() != newResult.size()) {
                metrics.recordMismatch(
                    "SIZE_MISMATCH",
                    customerId,
                    expectedResult.size(),
                    newResult.size()
                );
                return;
            }
            
            for (int i = 0; i < expectedResult.size(); i++) {
                Order exp = expectedResult.get(i);
                Order act = newResult.get(i);
                
                if (!exp.getId().equals(act.getId()) ||
                    !exp.getTotalAmount().equals(act.getTotalAmount())) {
                    
                    metrics.recordMismatch(
                        "DATA_MISMATCH",
                        customerId,
                        exp.toString(),
                        act.toString()
                    );
                    return;
                }
            }
            
            metrics.recordMatch(customerId);
            
        } catch (Exception e) {
            // 신규 DB 장애는 무시 (실제 응답에 영향 없음)
            log.warn("Shadow Mode 검증 중 오류: customerId={}", customerId, e);
            metrics.recordError(customerId, e);
        }
    }
}

// 5. 배치 이관 (기존 데이터 초기화)
@Service
public class DataMigrationBatch {
    
    @Autowired
    private OrderRepository oldOrderRepository;
    
    @Autowired
    private NewOrderRepository newOrderRepository;
    
    private static final int BATCH_SIZE = 1000;
    
    public void migrateExistingOrders(Long startId) {
        Pageable pageable = PageRequest.of(0, BATCH_SIZE);
        Page<Order> page = oldOrderRepository.findByIdGreaterThan(startId, pageable);
        
        while (page.hasContent()) {
            List<Order> orders = page.getContent();
            
            // 신규 DB에 저장
            newOrderRepository.saveAll(orders);
            
            log.info("마이그레이션 진행: {} ~ {} 완료",
                orders.get(0).getId(),
                orders.get(orders.size() - 1).getId());
            
            if (page.hasNext()) {
                pageable = pageable.next();
                page = oldOrderRepository.findByIdGreaterThan(startId, pageable);
            } else {
                break;
            }
        }
    }
}

// 6. Feature Toggle으로 읽기 전환 관리
@Service
public class OrderReadRouting {
    
    @Autowired
    private OrderRepository oldOrderRepository;
    
    @Autowired
    private NewOrderRepository newOrderRepository;
    
    @Autowired
    private FeatureToggleClient featureToggle;
    
    public Order getOrder(Long orderId) {
        // Feature Toggle으로 확률적 라우팅
        int percentage = featureToggle.getPercentage("use-new-order-db");
        
        if (shouldUseNewDb(percentage)) {
            return newOrderRepository.findById(orderId)
                .orElse(oldOrderRepository.findById(orderId).orElse(null));
        } else {
            return oldOrderRepository.findById(orderId).orElse(null);
        }
    }
    
    private boolean shouldUseNewDb(int percentage) {
        // customerId 해시로 일관된 라우팅
        return (System.nanoTime() % 100) < percentage;
    }
}
```

---

## 📊 마이그레이션 단계별 체크리스트

| 단계 | 기존 DB 읽기 | 신규 DB 읽기 | 기존 DB 쓰기 | 신규 DB 쓰기 | 기간 |
|------|-----------|-----------|-----------|-----------|------|
| **1. 준비** | 100% | - | 100% | - | 1주 |
| **2. 이중 쓰기** | 100% | - | 100% | 100% | 2주 |
| **3. 배치 이관** | 100% | 데이터 축적 중 | 100% | 100% | 1주 |
| **4. Shadow Mode** | 100% | 검증만 | 100% | 100% | 2주 |
| **5. 읽기 카나리** | 90% | 10% | 100% | 100% | 1주 |
| **6. 읽기 50%** | 50% | 50% | 100% | 100% | 3일 |
| **7. 읽기 100%** | - | 100% | 100% | 100% | 1주 |
| **8. 쓰기 전환** | - | 100% | - | 100% | 2일 |
| **9. 정리** | Archive | - | - | - | 지속 |

---

## ⚖️ 트레이드오프

```
안전성 vs 속도
├─ 빠른 이관 (Big Bang):
│  ├─ ✅ 빠름 (1주)
│  ├─ ❌ 위험도 높음 (장애 발생 시 롤백 불가)
│  └─ ❌ 검증 시간 부족
│
└─ 점진적 이관 (Strangler Fig):
   ├─ ✅ 안전함 (여러 체크포인트)
   ├─ ✅ 실시간 검증
   ├─ ❌ 느림 (6~8주)
   └─ ❌ 양쪽 시스템 유지 비용

데이터 일관성 보장 수준
├─ 이중 쓰기 (나이브):
│  ├─ ✅ 구현 단순
│  ├─ ❌ 원자성 없음 (부분 실패 가능)
│  └─ ❌ 재시도 복잡
│
└─ Outbox Pattern:
   ├─ ✅ 원자성 보장
   ├─ ✅ 자동 재시도
   ├─ ❌ 구현 복잡
   └─ ❌ Kafka 의존성

검증 방법 선택
├─ 수동 검증:
│  ├─ ✅ 100% 정확
│  ├─ ❌ 느림 (많은 시간 필요)
│  └─ ❌ 휴먼 에러 가능
│
└─ Shadow Mode:
   ├─ ✅ 자동화됨
   ├─ ✅ 빠름 (병렬 처리)
   ├─ ❌ 샘플링만 검증 (일부 누락 가능)
   └─ ❌ 모니터링 필요
```

---

## 📌 핵심 정리

```
✅ 데이터 이관 원칙:
  - Big Bang 마이그레이션은 피할 것
  - Strangler Fig: 점진적, 롤백 가능
  - Shadow Mode: 신규 시스템 검증
  - Feature Toggle: 단계적 전환

✅ Strangler Fig 단계:
  1. Outbox Pattern 추가 (원자성)
  2. 이중 쓰기 시작 (신규 DB에 복제)
  3. 배치 이관 (기존 데이터 초기화)
  4. Shadow Mode 검증 (자동 비교)
  5. 읽기 카나리 (5% → 10% → 50%)
  6. 읽기 100% 전환
  7. 쓰기 전환
  8. 기존 시스템 제거 (Archive 후)

✅ Outbox Pattern의 보장:
  - 원자성: Order + OutboxEvent 같은 트랜잭션
  - 보증 전달: Poller가 반복 시도
  - 멱등성: 중복 발행해도 안전

✅ 검증 자동화:
  - Shadow Mode: 비동기 비교
  - 메트릭: 불일치 감지 및 모니터링
  - 알림: 불일치 시 자동 통보

✅ 위험 관리:
  - Feature Toggle: 즉시 롤백 가능
  - 카나리 배포: 영향 범위 제한
  - Backup: 각 단계마다 저장
```

---

## 🤔 생각해볼 문제

**Q1.** Outbox Pattern에서 Kafka 발행은 성공했는데, 데이터베이스 업데이트 직후 애플리케이션이 크래시했습니다. 신규 DB에는 같은 데이터가 저장될까요?

<details>
<summary>해설 보기</summary>

시나리오 분석:

t=0: 
- Outbox 데이터: {eventId: 1, status: PENDING}
- Kafka: 준비됨

t=1: OutboxPoller 시작
- Kafka 발행 요청

t=2: 발행 성공
- Kafka: 메시지 저장됨
- 상태: PUBLISHED로 변경하려는 중...

t=3: 데이터베이스 업데이트 중
- UPDATE outbox SET status='PUBLISHED' WHERE eventId=1...

t=4: 애플리케이션 크래시!
- 업데이트 미완료
- Outbox 상태: 여전히 PENDING

t=5: 애플리케이션 재시작
- OutboxPoller 재시작
- PENDING 이벤트 감지 (eventId: 1)
- Kafka에 다시 발행

t=6: Kafka 수신 (이미 저장된 데이터)
- 신규 DB 이벤트 리스너: 메시지 수신
- Order 저장... (이미 있나?)

결과:
- **멱등성 처리가 있으면 ✅ 안전**:
  ```java
  Order existing = newOrderRepository.findById(orderId);
  if (existing != null) {
      // 이미 있으면 무시
      return;
  }
  ```

- **멱등성 처리가 없으면 ❌ 중복 저장**:
  - 같은 주문이 2번 저장됨
  - 데이터 불일치

결론: Outbox Pattern은 **이벤트 발행 보증**만 제공합니다.
**신규 DB가 멱등하게 처리**해야 중복 저장을 방지합니다.

</details>

**Q2.** Shadow Mode 검증 중에 기존 DB와 신규 DB의 데이터가 10%만 불일치합니다. 이 상태에서 읽기를 50%로 전환해도 될까요?

<details>
<summary>해설 보기</summary>

위험한 상황:

10% 불일치 의미:
- 1,000개 주문 중 100개는 기존 DB와 신규 DB가 다름
- 고객이 읽을 때 50% 확률로 잘못된 데이터를 볼 수 있음

시나리오:
고객 A가 주문 상세 조회:
- 기존 DB: 100만원 (정확)
- 신규 DB: 없거나 50만원 (오류)

만약 신규 DB에서 조회하면:
- 데이터 손실 또는 불정확한 정보 제공
- 고객 불만족

권장사항:
1. **불일치 원인 파악**:
   ```
   10% 불일치의 원인:
   - 데이터 타입 차이 (int vs long)?
   - 타임존 문제?
   - 조인 누락?
   - 일부 레코드만 동기화 안 됨?
   ```

2. **근본 원인 해결 후 재검증**:
   - 신규 DB 수정
   - 배치 재동기화
   - 불일치율 0% 달성

3. **불일치 패턴 분석**:
   ```java
   // 어떤 레코드가 안 맞는가?
   List<Order> mismatches = validator.findMismatches();
   mismatches.stream()
       .collect(Collectors.groupingBy(Order::getStatus))
       // 특정 상태의 주문만 안 맞는가?
       
       .collect(Collectors.groupingBy(Order::getCreatedDate))
       // 특정 시간대의 주문만 안 맞는가?
   ```

4. **불일치가 무시해도 되는 것인지 확인**:
   - 선택사항 필드인가? (문제 없음)
   - 중요 필드인가? (해결 필요)
   - 비즈니스에 영향이 있나?

결론: **0% 불일치 달성 후에 읽기 전환**
→ 데이터 무결성이 최우선

</details>

**Q3.** Outbox Pattern을 사용하고 있는데, Outbox 테이블이 계속 커지고 있습니다 (5GB). 문제가 될까요?

<details>
<summary>해설 보기</summary>

Outbox 테이블 증가의 문제:

원인:
- PUBLISHED 이벤트를 제거하지 않음
- 매일 발행되는 이벤트: 수백만 건
- 저장만 하고 정리하지 않음

문제:
1. **디스크 용량 증가**:
   - 매월 1GB씩 증가
   - 1년에 12GB (스토리지 비용)

2. **쿼리 성능 저하**:
   ```sql
   SELECT * FROM outbox WHERE status = 'PENDING'
   -- 수백만 건을 스캔 후 PENDING만 필터링
   -- 인덱스가 있어도 테이블 크기가 크면 느림
   ```

3. **백업 시간 증가**:
   - 대용량 테이블 백업에 오래 걸림

해결책:

1. **정리 정책 (Cleanup Policy)**:
   ```java
   @Scheduled(cron = "0 2 * * *")  // 매일 새벽 2시
   public void cleanupPublishedEvents() {
       // 30일 이상 된 PUBLISHED 이벤트 삭제
       LocalDateTime cutoff = LocalDateTime.now().minusDays(30);
       outboxRepository.deleteByStatusAndCreatedAtBefore(
           OutboxEventStatus.PUBLISHED,
           cutoff
       );
   }
   ```

2. **아카이빙**:
   ```java
   @Scheduled(cron = "0 3 * * 0")  // 주 1회
   public void archivePublishedEvents() {
       // 1년 이상 된 이벤트 별도 테이블로 이동
       List<OutboxEvent> archived = outboxRepository
           .findByStatusAndCreatedAtBefore(
               OutboxEventStatus.PUBLISHED,
               LocalDateTime.now().minusYears(1)
           );
       outboxArchiveRepository.saveAll(archived);
       outboxRepository.deleteAll(archived);
   }
   ```

3. **파티셔닝**:
   - 월별 파티션으로 나누기
   - 오래된 파티션 드롭

4. **별도 이벤트 저장소**:
   - Outbox: 즉시 필요한 이벤트만 (PENDING)
   - Event Store: 장기 보관 (PUBLISHED)

결론: **정기적 정리 정책 필요**
→ PUBLISHED 이벤트는 불필요 (이미 처리됨)
→ 저장하려면 별도 테이블로 아카이빙

</details>

---

<div align="center">

**[⬅️ 이전: 분산 데이터 일관성 — ACID vs BASE](./04-distributed-data-consistency.md)** | **[홈으로 🏠](../README.md)** | **[다음: 서비스 간 외래키 — 물리적 무결성 없이 일관성 보장 ➡️](./06-cross-service-foreign-keys.md)**

</div>

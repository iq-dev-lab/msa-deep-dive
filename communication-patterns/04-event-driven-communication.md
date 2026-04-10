# 04. 이벤트 기반 통신 — Kafka와 결합도 분리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 발행자가 구독자를 모르는 이벤트 기반 통신이 결합도를 낮추는 메커니즘은?
- Kafka의 Topic, Partition, Consumer Group, Offset의 역할과 상호 관계는?
- 이벤트 스키마가 진화할 때 Backward/Forward Compatibility를 어떻게 보장하고, Avro Schema Registry는 왜 필요한가?
- Topic 설계 시 "서비스당 1개 토픽" vs "이벤트 유형당 1개 토픽"의 트레이드오프는?
- Outbox Pattern이 데이터베이스 트랜잭션과 메시지 발행을 어떻게 원자적으로 처리하는가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

이벤트 기반 통신은 MSA의 근본적인 느슨한 결합을 가능하게 합니다. 동기 호출과 달리 **발행자는 누가 메시지를 받을지, 구독자는 누가 보낼지 알 필요가 없습니다**. 이는 마이크로서비스 팀 간 의존성을 획기적으로 줄입니다.

예를 들어 주문 완료 이벤트가 발행되면, 현재는 "알림 발송", "배송 준비", "분석" 서비스가 구독하지만, 나중에 "추천" 서비스나 "회계" 서비스가 추가되어도 **Order Service의 코드는 수정할 필요가 없습니다**. 새로운 팀은 이벤트만 구독하면 됩니다.

그러나 이벤트 기반 아키텍처는 **Eventual Consistency, 메시지 손실, 중복 처리** 등의 복잡한 문제를 동시에 안겨줍니다. Kafka는 이런 문제들을 해결하도록 설계되었으며:

1. **Durability**: 브로커에 메시지를 저장하여 구독자 장애 시에도 복구 가능
2. **Partitioning**: 토픽을 여러 파티션으로 분산하여 처리량 확장
3. **Consumer Group**: 여러 구독자가 자신의 offset을 독립적으로 관리
4. **Replication**: 여러 브로커에 복제하여 고가용성 보장

MSA에서 이런 특성들을 활용하는 것이 시스템의 확장성, 가용성, 복원력을 결정합니다.

---

## 😱 흔한 실수 (Before — 잘못된 이벤트 발행)

```java
// ❌ 안티패턴 1: 이벤트 발행 후 실패 처리 안 함
@Service
public class OrderService {
    @Autowired private OrderRepository orderRepository;
    @Autowired private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. DB에 주문 저장 (성공)
        Order order = orderRepository.save(new Order(request));
        
        // 2. 이벤트 발행 (실패할 수 있음!)
        try {
            kafkaTemplate.send("order-events", order.getId().toString(),
                new OrderCreatedEvent(order.getId(), order.getTotalAmount()));
        } catch (Exception e) {
            // ❌ 문제: 주문은 저장되었지만 이벤트는 발행 안 됨
            // 결과: 알림 미발송, 배송 미준비, 분석 데이터 누락
            logger.error("이벤트 발행 실패", e);
            // 조용히 넘어감 → 데이터 불일치
        }
        
        return order;
    }
}

// ❌ 안티패턴 2: 중복 처리 미보장
@Service
public class NotificationService {
    
    @KafkaListener(topics = "order-events", groupId = "notification-service")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Kafka 장애로 같은 메시지가 2번 전송되었다면?
        // 
        // 1번째 처리:
        //   emailService.send(order.getCustomer().getEmail(), "주문 완료")
        //   → Success, offset commit
        //
        // 2번째 처리 (재전송):
        //   emailService.send(order.getCustomer().getEmail(), "주문 완료")
        //   → 이메일 2번 발송 💥
        
        emailService.send(event.getCustomerId(), "주문이 완료되었습니다");
        // 멱등성 없음 → 중복 실행 위험
    }
}

// ❌ 안티패턴 3: 스키마 진화 미고려
// v1.0 이벤트
public class OrderCreatedEvent {
    public Long orderId;
    public BigDecimal totalAmount;
    // 필드: 2개
}

// v2.0 이벤트 (새 필드 추가)
public class OrderCreatedEvent {
    public Long orderId;
    public BigDecimal totalAmount;
    public String currency;      // ← 새로 추가 (필수)
    // 필드: 3개
}

// v2.0 Producer가 메시지 발행:
// { "orderId": 123, "totalAmount": 50000, "currency": "KRW" }

// v1.0 Consumer가 수신:
// 필드 currency 무시하지만, 코드가 null 체크 없으면 NPE 발생
deserializer.deserialize(data); // currency 필드 null → Exception
```

---

## ✨ 올바른 접근 (After — Kafka + Outbox Pattern)

```java
// ✅ 개선 1: Outbox Pattern으로 원자성 보장
@Entity
@Table(name = "orders")
public class Order {
    @Id private Long id;
    @Column private String customerId;
    @Column private BigDecimal totalAmount;
    @Column private String status;
    
    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private List<OrderEvent> events = new ArrayList<>();
    
    public void addEvent(OrderEvent event) {
        this.events.add(event);
    }
}

// Outbox 테이블
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "aggregate_type")
    private String aggregateType;    // "Order"
    
    @Column(name = "aggregate_id")
    private Long aggregateId;        // order.id
    
    @Column(name = "event_type")
    private String eventType;        // "OrderCreated"
    
    @Column(name = "event_payload", columnDefinition = "TEXT")
    private String eventPayload;     // JSON 형식
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "published_at")
    private LocalDateTime publishedAt; // null = 미발행
}

@Service
public class OrderService {
    @Autowired private OrderRepository orderRepository;
    @Autowired private OutboxEventRepository outboxRepository;
    
    // ✅ 핵심: DB 트랜잭션 내에서 이벤트도 저장
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. 주문 저장
        Order order = orderRepository.save(new Order(request));
        
        // 2. 같은 트랜잭션 내에서 Outbox에 이벤트 저장
        OutboxEvent outboxEvent = new OutboxEvent();
        outboxEvent.setAggregateType("Order");
        outboxEvent.setAggregateId(order.getId());
        outboxEvent.setEventType("OrderCreated");
        outboxEvent.setEventPayload(objectMapper.writeValueAsString(
            new OrderCreatedEvent(order.getId(), order.getTotalAmount())));
        outboxEvent.setCreatedAt(LocalDateTime.now());
        
        outboxRepository.save(outboxEvent);
        
        // 트랜잭션 커밋 시:
        // - ORDER 행 저장
        // - OUTBOX_EVENTS 행 저장
        // 모두 성공 또는 모두 실패 (원자성)
        
        return order;
    }
}

// ✅ 별도 스레드: Outbox 폴러
@Component
public class OutboxPoller {
    
    @Autowired private OutboxEventRepository outboxRepository;
    @Autowired private KafkaTemplate<String, String> kafkaTemplate;
    
    @Scheduled(fixedRate = 1000) // 1초마다 실행
    public void pollAndPublish() {
        // 미발행 이벤트 조회
        List<OutboxEvent> unpublished = outboxRepository
            .findByPublishedAtIsNull();
        
        for (OutboxEvent event : unpublished) {
            try {
                // Kafka에 발행
                kafkaTemplate.send(
                    "order-events",
                    event.getAggregateId().toString(),
                    event.getEventPayload());
                
                // 발행 표시 (멱등성: 같은 이벤트 재발행 안 함)
                event.setPublishedAt(LocalDateTime.now());
                outboxRepository.save(event);
                
            } catch (Exception e) {
                // 실패한 이벤트는 다음 폴링 때 재시도
                logger.warn("이벤트 발행 실패: {}", event.getId());
            }
        }
    }
}

// ✅ 개선 2: Avro Schema Registry로 스키마 관리
@Configuration
public class KafkaConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, 
            "localhost:9092");
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
            StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
            ConfluentAvroSerializer.class);  // Avro 직렬화
        configProps.put(AbstractKafkaSchemaSerDeConfig.SCHEMA_REGISTRY_URL_CONFIG,
            "http://localhost:8081");
        
        return new DefaultProducerFactory<>(configProps);
    }
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,
            "localhost:9092");
        configProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
            StringDeserializer.class);
        configProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
            ConfluentAvroDeserializer.class);  // Avro 역직렬화
        configProps.put(AbstractKafkaSchemaSerDeConfig.SCHEMA_REGISTRY_URL_CONFIG,
            "http://localhost:8081");
        
        return new DefaultConsumerFactory<>(configProps);
    }
}

// Avro 스키마 정의 (v2.0 - 하위호환)
/*
{
  "type": "record",
  "namespace": "com.example.order.event",
  "name": "OrderCreatedEvent",
  "fields": [
    {
      "name": "orderId",
      "type": "long"
    },
    {
      "name": "totalAmount",
      "type": "double"
    },
    {
      "name": "currency",
      "type": ["null", "string"],  // v2.0 추가, 선택적
      "default": null
    },
    {
      "name": "customerId",
      "type": ["null", "string"],  // v2.0 추가, 선택적
      "default": null
    }
  ]
}
*/

// ✅ 개선 3: 멱등성 보장하는 Consumer
@Component
public class NotificationService {
    
    @Autowired private EmailService emailService;
    @Autowired private IdempotencyStore idempotencyStore;
    
    @KafkaListener(topics = "order-events", groupId = "notification-service")
    public void handleOrderCreated(
            OrderCreatedEvent event,
            @Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String messageKey,
            @Header(KafkaHeaders.OFFSET) long offset) {
        
        // 멱등성 키: topic + partition + offset
        String idempotencyKey = String.format("order-events-%d-%d", 
            0, offset); // partition 0, offset
        
        // 중복 처리 확인
        if (idempotencyStore.exists(idempotencyKey)) {
            logger.info("중복 메시지 무시: {}", idempotencyKey);
            return;
        }
        
        try {
            // 비즈니스 로직
            emailService.sendOrderConfirmation(
                event.getCustomerId(),
                event.getOrderId(),
                event.getTotalAmount());
            
            // 성공 후 멱등성 키 저장
            idempotencyStore.put(idempotencyKey);
            
        } catch (Exception e) {
            // 실패 시: 예외 발생 → Kafka 자동 재시도
            throw new RuntimeException("알림 발송 실패", e);
        }
    }
}

// ✅ 개선 4: Topic 설계 - 도메인별 이벤트 토픽
/*
Topic 설계 원칙:

1. 서비스별 이벤트 토픽 (권장)
   - order-events    (Order Service가 발행)
   - payment-events  (Payment Service가 발행)
   - shipping-events (Shipping Service가 발행)
   
   장점:
   ✅ 발행자 명확
   ✅ Consumer Group 관리 간단
   ✅ 정책 (파티션, 보관기간) 별도 설정 가능
   
2. 도메인 이벤트 토픽
   - order-lifecycle (OrderCreated, OrderShipped, OrderCanceled)
   - user-lifecycle (UserRegistered, UserVerified, UserDeleted)
   
   장점:
   ✅ 관련 이벤트 한 곳에 모음
   ✅ 도메인 전용 Consumer 그룹 가능
*/
```

---

## 🔬 내부 동작 원리 — Kafka 아키텍처와 일관성

### 1. Kafka 토폴로지: Topic, Partition, Replica

```
        Producer                   Consumer
          ↓                           ↑
     ┌─────────────────────────────────────┐
     │     BROKER 0                        │
     ├─────────────────────────────────────┤
     │ Topic: order-events                 │
     │  ├─ Partition 0 (Leader)            │
     │  │  [0,1,2,3,4,5] ← offset          │
     │  │  Replica on Broker 1             │
     │  │                                  │
     │  ├─ Partition 1 (Follower)          │
     │  │  [0,1,2] ← offset                │
     │  │  Replica on Broker 2             │
     │  │                                  │
     │  └─ Partition 2 (Follower)          │
     │     [0,1,2,3] ← offset              │
     └─────────────────────────────────────┘

Producer 쓰기:
1. 파티션 선택 (key 기반 또는 라운드로빈)
2. Leader 브로커에 메시지 전송
3. Leader가 follower에 복제
4. In-Sync Replica (ISR) 확인 후 응답

Consumer 읽기:
1. Consumer Group에 가입 (e.g., "notification-service")
2. 파티션 할당 (파티션마다 1 Consumer)
3. Offset 추적 (마지막으로 읽은 위치)
4. 새 메시지 폴링 (최신 offset부터)
```

### 2. Offset 관리와 Consumer Group

```
Topic: order-events (3 파티션)

  Partition 0        Partition 1        Partition 2
  ─────────────      ─────────────      ─────────────
  [msg0,msg1,...]    [msg0,msg1,...]    [msg0,msg1,...]
       ↑ Offset 5     ↑ Offset 3        ↑ Offset 4

Consumer Group: "notification-service"
┌──────────────────────────┐
│ notification-service-1   │  → P0, offset=5 (다음 msg는 6)
├──────────────────────────┤
│ notification-service-2   │  → P1, offset=3
├──────────────────────────┤
│ notification-service-3   │  → P2, offset=4
└──────────────────────────┘

장점:
✅ 병렬 처리: 3개 파티션을 3개 Consumer가 동시 처리
✅ 확장: Consumer 4개 추가 시 자동 리밸런싱
✅ 재처리: offset을 이전 시점으로 이동 → 재처리

예: shipping-service Consumer Group
┌──────────────────────────┐
│ shipping-service-1       │  → P0, offset=2 (다른 진행도)
├──────────────────────────┤
│ shipping-service-2       │  → P1, offset=7
├──────────────────────────┤
│ shipping-service-3       │  → P2, offset=9
└──────────────────────────┘
(각 Consumer Group이 독립적 offset 유지)
```

### 3. Backward & Forward Compatibility

```
Avro 스키마 진화:

v1.0:
{
  "name": "OrderCreatedEvent",
  "fields": [
    {"name": "orderId", "type": "long"},
    {"name": "totalAmount", "type": "double"}
  ]
}

v2.0 (필드 추가, 선택적):
{
  "name": "OrderCreatedEvent",
  "fields": [
    {"name": "orderId", "type": "long"},
    {"name": "totalAmount", "type": "double"},
    {"name": "currency", "type": ["null", "string"], "default": null}
  ]
}

시나리오 1: v1.0 Producer → v2.0 Consumer
Producer: { "orderId": 123, "totalAmount": 50000 }
Consumer: currency 필드 없음 → default (null) 사용
Result: 정상 처리 ✅ (Forward Compatibility)

시나리오 2: v2.0 Producer → v1.0 Consumer
Producer: { "orderId": 123, "totalAmount": 50000, "currency": "KRW" }
Consumer: currency 필드 무시 (알려지지 않은 필드)
Result: 정상 처리 ✅ (Backward Compatibility)

Schema Registry가 하는 일:
1. 새 스키마 등록 시 호환성 검증
2. 호환 불가능 스키마 등록 거부 (배포 전에!)
3. 메시지 직렬화/역직렬화 시 스키마 ID 자동 포함
4. 컨슈머가 올바른 스키마 버전 자동 선택
```

### 4. Outbox Pattern의 일관성

```
문제: 데이터베이스와 Kafka 동기화

❌ WITHOUT Outbox Pattern (2-Phase Commit 불가)
┌─────────────────────────────┐
│ DB 트랜잭션 시작            │
├─────────────────────────────┤
│ 1. ORDER 행 INSERT           │
│    → 성공                   │
│ 2. Kafka 메시지 발행         │
│    → 네트워크 오류 💥        │
├─────────────────────────────┤
│ DB 롤백 (ORDER 없음)        │
│ Kafka 메시지 발행됨 (고아)   │
│ → 데이터 불일치!             │
└─────────────────────────────┘

✅ WITH Outbox Pattern
┌─────────────────────────────────────┐
│ DB 트랜잭션 시작                    │
├─────────────────────────────────────┤
│ 1. ORDER 행 INSERT                  │
│    → 성공                          │
│ 2. OUTBOX_EVENTS 행 INSERT          │
│    (이벤트를 DB에 저장)             │
│    → 성공                          │
│ 3. 트랜잭션 커밋                    │
│    → 모두 성공 ✅                   │
├─────────────────────────────────────┤
│ 별도 프로세스 (비동기)              │
│ 1. OUTBOX_EVENTS에서 미발행 조회    │
│ 2. Kafka에 발행                     │
│ 3. published_at 업데이트            │
│ (실패 시 다음 폴링에 재시도)        │
└─────────────────────────────────────┘

결과: 항상 일관성 유지! ✅
```

---

## 💻 코드 예제 — Kafka 실전

### application.yml 설정
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      # 발행자 설정
      acks: all                      # 모든 ISR 레플리카 확인
      retries: 3
      properties:
        linger.ms: 10                # 10ms 동안 배치 대기
        batch.size: 16384            # 배치 크기
        compression.type: snappy     # 압축
        enable.idempotence: true     # 멱등성 (0.11+)
    
    consumer:
      # 구독자 설정
      bootstrap-servers: localhost:9092
      group-id: order-service
      auto-offset-reset: earliest    # 초기 offset
      enable-auto-commit: false      # 수동 커밋
      max-poll-records: 100          # 한 번에 최대 100개
      session-timeout-ms: 30000      # 세션 타임아웃
      
  # Avro Schema Registry
    properties:
      schema.registry.url: http://localhost:8081
```

### Producer 구현
```java
@Component
public class OrderEventProducer {
    
    @Autowired private KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;
    @Autowired private ObjectMapper objectMapper;
    
    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotalAmount(),
            order.getCurrency());
        
        // Topic: order-events
        // Key: orderId (같은 주문의 이벤트는 같은 파티션으로)
        // Value: Avro 직렬화
        ListenableFuture<SendResult<String, OrderCreatedEvent>> future =
            kafkaTemplate.send("order-events", 
                order.getId().toString(), 
                event);
        
        future.addCallback(
            // Success
            result -> logger.info("주문 이벤트 발행: {}", order.getId()),
            // Failure
            ex -> logger.error("주문 이벤트 발행 실패: {}", order.getId(), ex)
        );
    }
}
```

### Consumer 구현 (멱등성)
```java
@Component
public class OrderEventListener {
    
    @Autowired private ShippingService shippingService;
    @Autowired private IdempotencyStore idempotencyStore;
    @Autowired private KafkaTemplate<String, ?> kafkaTemplate;
    
    @KafkaListener(
        topics = "order-events",
        groupId = "shipping-service",
        containerFactory = "kafkaListenerContainerFactory")
    public void onOrderCreated(
            OrderCreatedEvent event,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment acknowledgment) {
        
        // 멱등성 키 생성
        String idempotencyKey = String.format("%s-%d-%d", 
            topic, partition, offset);
        
        // 중복 메시지 확인
        if (idempotencyStore.exists(idempotencyKey)) {
            logger.info("중복 메시지 무시: {}", idempotencyKey);
            acknowledgment.acknowledge(); // offset 커밋
            return;
        }
        
        try {
            // 배송 준비
            shippingService.prepareShipment(event.getOrderId());
            
            // 멱등성 키 저장 (중복 방지)
            idempotencyStore.save(idempotencyKey);
            
            // 수동 offset 커밋 (성공 시에만)
            acknowledgment.acknowledge();
            
            logger.info("배송 준비 완료: {}", event.getOrderId());
            
        } catch (Exception e) {
            // 실패 시: offset 커밋 안 함
            // → Kafka가 자동으로 같은 메시지 재전송
            logger.error("배송 준비 실패: {}", event.getOrderId(), e);
            throw new RuntimeException(e);
        }
    }
}

// 멱등성 저장소 (Redis)
@Component
public class IdempotencyStore {
    
    @Autowired private RedisTemplate<String, String> redisTemplate;
    
    private static final String PREFIX = "idempotency:";
    private static final Duration TTL = Duration.ofHours(24);
    
    public boolean exists(String key) {
        return Boolean.TRUE.equals(
            redisTemplate.hasKey(PREFIX + key));
    }
    
    public void save(String key) {
        redisTemplate.opsForValue().set(
            PREFIX + key, "processed", TTL);
    }
}
```

### Topic 생성 및 관리
```bash
# 1. Topic 생성 (3 파티션, 복제 3)
kafka-topics --create \
  --topic order-events \
  --partitions 3 \
  --replication-factor 3 \
  --config retention.ms=604800000 \  # 7일 보관
  --config compression.type=snappy \
  --bootstrap-server localhost:9092

# 2. Consumer Group 상태 확인
kafka-consumer-groups --list \
  --bootstrap-server localhost:9092

kafka-consumer-groups --describe \
  --group shipping-service \
  --bootstrap-server localhost:9092

# Output:
# GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# shipping-serv   order-events    0          125             150             25
# shipping-serv   order-events    1          98              98              0
# shipping-serv   order-events    2          312             320             8

# 3. Offset 초기화 (재처리용)
kafka-consumer-groups --reset-offsets \
  --to-earliest \
  --group shipping-service \
  --topic order-events \
  --execute \
  --bootstrap-server localhost:9092
```

---

## 📊 패턴 비교 — Topic 설계 전략

| 기준 | 서비스당 1개 토픽 | 이벤트 타입당 1개 토픽 |
|------|-----------------|-------------------|
| **Topic명** | order-events, payment-events | order-created, payment-captured |
| **발행자** | Order Service (명확) | 여러 서비스 가능 |
| **Consumer** | 도메인 기반 | 이벤트 기반 |
| **파티셔닝** | orderId로 순서 보장 | 불명확 |
| **확장성** | 서비스 추가 시 새 토픽 | 이벤트 추가 시 새 토픽 |
| **관리 복잡도** | 낮음 | 높음 |
| **권장** | ✅ MSA | ⚠️ 신중히 |

---

## ⚖️ 트레이드오프

```
이벤트 기반 아키텍처의 이점과 비용:

Eventual Consistency
✅ 높은 가용성 (한 서비스 장애 → 다른 서비스 영향 없음)
✅ 확장성 (Consumer 추가로 처리량 확장)
✅ 느슨한 결합 (발행자-구독자 독립)

❌ 복잡성 (일관성 보장이 어려움)
❌ 디버깅 (비동기 추적 어려움)
❌ 지연 (나중에 처리, 실시간성 낮음)

────────────────────────────────────────

Kafka 운영 비용:
✅ 높은 처리량 (수백만 msg/sec)
✅ 낮은 지연 (ms 단위)
✅ 내구성 (메시지 손실 없음)

❌ 인프라 (Zookeeper + Broker 관리)
❌ 디스크 (메시지 저장 공간)
❌ 네트워크 (높은 대역폭 사용)

────────────────────────────────────────

Outbox Pattern의 비용:
✅ 원자성 (DB와 메시지 동기화)
✅ 재시도 가능 (Kafka 네트워크 오류 자동 복구)

❌ 데이터 중복 (Outbox 테이블 유지)
❌ 폴링 오버헤드 (주기적 쿼리)
❌ 지연 (폴링 주기만큼 발행 지연)

최적화: 폴링 1초, 배치 100개 → 오버헤드 최소화
```

---

## 📌 핵심 정리

```
✅ 이벤트 기반 통신이 결합도를 낮추는 메커니즘

1. 느슨한 결합
   - Publisher: "OrderCreatedEvent 발행" (누가 받든 상관 없음)
   - Subscriber: 토픽 구독 (누가 발행했든 상관 없음)
   - 새 서비스 추가 = 이벤트만 구독 (Publisher 코드 변경 없음)

2. 독립적 확장
   - Consumer 추가 → Kafka가 파티션 자동 재분배
   - Producer 처리량 증가 → Kafka 브로커 추가
   - 서비스 간 의존성 없음

3. 느슨한 시간적 결합
   - Subscriber가 오프라인 → 메시지 저장됨
   - 나중에 복구 → 저장된 메시지부터 처리

✅ Kafka 토폴로지 이해

Topic = 데이터 저장소
├─ Partition (병렬 처리)
│  ├─ Replica (고가용성)
│  └─ Offset (재처리)
└─ Consumer Group (독립적 offset)

Performance:
- 파티션 수 = Consumer 병렬도
- Replica factor = 가용성 (3 권장)
- Retention = 저장 기간

✅ Schema Registry의 역할

1. 버전 관리
   v1.0 → v2.0 (필드 추가, 선택적)
   → Backward/Forward Compatibility 자동 검증

2. 스키마 ID 자동 관리
   메시지 = [SchemaID][Payload]
   Consumer가 올바른 버전으로 역직렬화

3. Breaking Change 감지
   배포 전: 새 스키마 등록 → 호환성 검증 → 거부/승인

✅ Outbox Pattern = 원자성 보장

1. 메시지 = 데이터의 일부 (같은 트랜잭션)
2. DB 커밋 = 메시지 저장
3. 폴러 프로세스 = 비동기 발행
4. 재시도 = 자동 (폴링 반복)

결과: DB와 Kafka 항상 동기화 ✅

✅ Consumer 멱등성 구현

1. 멱등성 키 = 토픽 + 파티션 + 오프셋
2. 중복 체크 = Redis (빠른 조회)
3. 성공 후만 offset 커밋
4. 실패 시 = 예외 발생 → Kafka 재전송

결과: 중복 실행 불가능 ✅

✅ Topic 설계 원칙

서비스당 1개 토픽 (권장):
- order-events (Order Service)
- payment-events (Payment Service)
- shipping-events (Shipping Service)

장점:
✅ 발행자 명확
✅ 파티셔닝 전략 단순 (orderId로 순서 보장)
✅ Consumer Group 관리 용이
✅ 정책 별도 설정 가능
```

---

## 🤔 생각해볼 문제

**Q1.** Order Service가 OrderCreatedEvent를 발행했는데, Notification Service의 Consumer가 처리 중 Exception을 던집니다. Kafka가 자동으로 어떻게 처리하고, 무엇을 주의해야 할까?

<details>
<summary>해설 보기</summary>

**Kafka Consumer의 자동 재시도 메커니즘:**

```
상황:
KafkaListener 메서드에서 Exception 발생
→ Kafka가 offset 커밋 안 함 (enable-auto-commit=false)
→ 다음 폴링 때 같은 메시지 재전송

시각화:
[메시지 수신] → [처리 실패] → [Exception]
                              ↓
                       [Offset 커밋 안 함]
                              ↓
                       [1초 후 재시도]
                              ↓
                       [처리 실패] → [Exponential backoff]
```

**문제점과 해결:**

```java
// ❌ 무한 루프 위험
@KafkaListener(topics = "order-events")
public void handleOrder(OrderEvent event) {
    // 항상 실패하는 로직 (버그)
    throw new RuntimeException("계속 실패!");
    
    // 결과:
    // 계속 재시도 (3분? 1시간?)
    // → 다른 메시지는 처리 안 됨 (HOL 블로킹)
    // → 시스템 과부하
}

// ✅ 개선: DLQ (Dead Letter Queue)로 이동
@KafkaListener(topics = "order-events")
public void handleOrder(
        OrderEvent event,
        KafkaOperations kafkaOperations) {
    
    int maxRetries = 3;
    int retryCount = 0;
    
    while (retryCount < maxRetries) {
        try {
            processOrder(event);
            return; // 성공 → 루프 탈출
        } catch (TemporaryException e) {
            retryCount++;
            if (retryCount < maxRetries) {
                Thread.sleep(1000 * retryCount); // 지수 백오프
            }
        } catch (PermanentException e) {
            // 복구 불가능한 에러 → DLQ로
            kafkaOperations.send(
                "order-events-dlq",  // Dead Letter Queue
                event);
            logger.error("DLQ로 이동: {}", event);
            return;
        }
    }
    
    // 최종 실패 → DLQ
    kafkaOperations.send("order-events-dlq", event);
}

// DLQ 처리 (별도 프로세스)
@KafkaListener(topics = "order-events-dlq")
public void handleDLQ(OrderEvent event) {
    // 1. 실패 로그 저장
    dlqRepository.save(new DLQMessage(event));
    
    // 2. 알림 (관리자에게 알림)
    alertService.notifyDLQ("order-events", event);
    
    // 3. 수동 개입 (이 부분은 사람이 처리)
    logger.warn("DLQ 메시지, 수동 처리 필요: {}", event);
}
```

**핵심 포인트:**
1. Exception 발생 → offset 커밋 안 됨 → 자동 재시도
2. 같은 메시지 반복 처리 → 멱등성 필수
3. 영구 실패 → DLQ로 이동 (수동 처리)

</details>

**Q2.** Outbox Pattern에서 폴러가 1초마다 실행될 때, 2시간 동안 같은 이벤트가 1000번 발행된다면? 비용은?

<details>
<summary>해설 보기</summary>

**Outbox Pattern의 중복 발행 위험:**

```
시나리오:
이벤트 1개가 published_at 업데이트 실패

while (true) {
    unpublished = findByPublishedAtIsNull()  // 같은 이벤트 반복 조회
    // [이벤트 1]
    
    send(event1) → Kafka (성공)
    update(event1.publishedAt) → DB (실패!)
    
    // 1초 후
    unpublished = findByPublishedAtIsNull()  // [이벤트 1] 또 조회!
    send(event1) → Kafka (중복!)
    ...
    // 반복
}

결과: 2시간 = 7200초 → 같은 이벤트 7200회 발행!
```

**문제점:**

```
Kafka 레벨:
✗ 메시지 중복 7200배
✗ 네트워크 대역폭 낭비
✗ 디스크 사용량 급증

Consumer 레벨:
✗ 이벤트 처리 7200번 (멱등성 있으면 OK)
✗ 데이터베이스 부하
✗ 알림 7200번 발송 (사용자 피해)
```

**해결 방법:**

```java
// ✅ 방법 1: 발행 성공까지 원자적으로
@Transactional
public void publishEvent(OutboxEvent event) {
    try {
        // 1. 발행
        kafkaTemplate.send(topic, key, event.getPayload());
        
        // 2. 같은 트랜잭션 내에서 업데이트
        event.setPublishedAt(now());
        outboxRepository.save(event);
        
        // 트랜잭션 커밋 (모두 성공)
    } catch (Exception e) {
        // 실패 → 롤백 (업데이트 안 됨)
        // → 다음 폴링에 재시도
    }
}

// ✅ 방법 2: 로크 (한 번에 하나만)
@Scheduled(fixedRate = 1000)
public void pollAndPublish() {
    List<OutboxEvent> unpublished = outboxRepository
        .findByPublishedAtIsNullForUpdate();  // 행 잠금
    
    for (OutboxEvent event : unpublished) {
        if (event.getLock() == null) {  // 다른 폴러가 처리 중이 아님
            event.setLock(UUID.randomUUID());
            
            try {
                kafkaTemplate.send(...);
                event.setPublishedAt(now());
                outboxRepository.save(event);
            } finally {
                event.setLock(null);
            }
        }
    }
}

// ✅ 방법 3: CDC (Change Data Capture)
// 대안: 데이터베이스 변경 로그 감시 (Debezium)
// - 더 효율적
// - 폴링 오버헤드 없음
// - 지연 최소화
```

**권장사항:**

```
작은 시스템 (QPS < 1000):
→ Outbox Polling (간단)
→ 1초 주기 (충분)

중간 시스템 (1000 < QPS < 10000):
→ Batch 폴링 + Lock
→ 100ms 주기 (빠른 재시도)

대규모 시스템 (QPS > 10000):
→ CDC (Debezium)
→ 거의 실시간
→ 폴링 오버헤드 제거
```

</details>

**Q3.** Backward Compatibility와 Forward Compatibility의 차이는? Avro 스키마에서 어떻게 구분하는가?

<details>
<summary>해설 보기</summary>

**용어 정의:**

```
Backward Compatibility (구버전 클라이언트가 신버전 데이터 읽기)
v1.0 Consumer ← v2.0 Producer

Forward Compatibility (신버전 클라이언트가 구버전 데이터 읽기)
v2.0 Consumer ← v1.0 Producer
```

**Avro 스키마 진화 규칙:**

```
v1.0 스키마:
{
  "fields": [
    {"name": "orderId", "type": "long"},
    {"name": "totalAmount", "type": "double"}
  ]
}

────────────────────────────────────────────────────

v2.0: 필드 추가 (선택적 - Backward & Forward 호환)
{
  "fields": [
    {"name": "orderId", "type": "long"},
    {"name": "totalAmount", "type": "double"},
    {"name": "currency", 
     "type": ["null", "string"],  // union 타입 (선택적)
     "default": null}              // 기본값 필수!
  ]
}

Backward (v1 consumer ← v2 producer):
Producer: { "orderId": 123, "totalAmount": 50000, "currency": "KRW" }
Consumer (v1): currency 필드 무시 ✅
Result: { "orderId": 123, "totalAmount": 50000 }

Forward (v2 consumer ← v1 producer):
Producer: { "orderId": 123, "totalAmount": 50000 }
Consumer (v2): currency 필드 = default (null) ✅
Result: { "orderId": 123, "totalAmount": 50000, "currency": null }

────────────────────────────────────────────────────

❌ Breaking Change (선택적이 아님):
{
  "fields": [
    {"name": "orderId", "type": "long"},
    {"name": "totalAmount", "type": "double"},
    {"name": "currency", 
     "type": "string",  // 필수!
     // "default" 없음 → v1은 처리 불가
    }
  ]
}

v1 Consumer ← v2 Producer: 
Producer: { "orderId": 123, "totalAmount": 50000, "currency": "KRW" }
Consumer (v1): currency 필드가 뭐지? 알 수 없음 ❌
Error: 스키마 미스매치

Schema Registry: 이 새 스키마는 호환 불가능!
→ 등록 거부 ✋ (배포 전 차단!)
```

**규칙 정리:**

```
✅ Backward 호환 (v1 Consumer 보호)
- 필드 추가 (선택적): "type": ["null", "T"], "default": null
- 필드 제거: OK (old consumer가 무시)
- 필드 순서 변경: OK (필드 ID 기반)

✅ Forward 호환 (v2 Consumer 보호)
- 필드 추가 (선택적): 기본값 필수
- 필드 제거: OK (new consumer가 무시)
- 필드 이름 변경: OK (필드 ID 기반, 별칭 설정)

❌ Both 호환 불가
- 필드 추가 (필수): 기본값 없음
- 필드 타입 변경: long → string
- 필드 삭제 (기존 필드 ID 재사용): Avro에서 금지

Schema Registry 확인:
kafka-schema-registry-client \
  --subject order-value \
  --compatibility BACKWARD_TRANSITIVE  # v1, v2, v3 모두 호환
```

**결론:**

모든 필드 추가는 선택적으로 (기본값 포함)
→ Backward & Forward 모두 호환
→ Kafka + MSA의 표준 방식

</details>

---

<div align="center">

**[⬅️ 이전: gRPC 완전 분해 — Protocol Buffers와 HTTP/2](./03-grpc-deep-dive.md)** | **[홈으로 🏠](../README.md)** | **[다음: API Composition 패턴 — 여러 서비스 데이터 조합 ➡️](./05-api-composition.md)**

</div>

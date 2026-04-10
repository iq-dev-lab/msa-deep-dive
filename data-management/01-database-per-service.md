# 01. Database per Service 원칙 — 왜 DB를 분리해야 하는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- MSA에서 DB를 서비스마다 분리하는 핵심 이유는 무엇인가?
- Shared Database 안티패턴이 실제로 어떤 문제를 야기하는가?
- Database per Service로 전환 시 발생하는 조인, 트랜잭션 문제를 어떻게 해결하는가?
- 스키마 분리와 서버 분리 중 어떤 분리 수준을 선택해야 하는가?
- 비용 vs 격리 트레이드오프를 어떻게 평가해야 하는가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

마이크로서비스 아키텍처의 핵심은 **서비스의 독립성과 자율성**입니다. 여러 팀이 각각의 서비스를 독립적으로 개발, 배포, 운영해야 한다는 의미입니다. 이것이 가능하려면 데이터도 독립적으로 관리되어야 합니다. Shared Database는 이 독립성을 근본적으로 훼손합니다.

**Shared Database의 문제점을 구체적으로 생각해보세요.** 주문 서비스와 배송 서비스가 동일한 데이터베이스를 공유한다면, 주문 서비스 팀이 `orders` 테이블에 새로운 컬럼을 추가하거나 스키마를 변경할 때 배송 서비스가 영향을 받습니다. 배포 일정을 조율해야 하고, 한 팀의 버그는 다른 팀의 서비스까지 장애를 일으킵니다. 이는 MSA의 핵심 이점인 **독립적 배포, 빠른 혁신, 장애 격리**를 모두 제거합니다.

**Database per Service 원칙은 조직 문제를 기술로 해결하는 방식입니다.** DB를 분리하면 기술적으로 강제하게 되므로, 팀이 제아무리 긴급해도 다른 팀의 DB 스키마를 함부로 수정할 수 없습니다. 대신 공식적인 API와 메시지를 통해 통신해야 합니다. 이는 느슨한 결합 상태를 유지하게 해줍니다.

---

## 😱 흔한 실수 (Before — Shared Database 안티패턴)

```sql
-- BEFORE: 모든 서비스가 동일한 데이터베이스를 공유
-- Database: shared_database

-- 주문 서비스 테이블
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    total_amount DECIMAL(10, 2),
    created_at TIMESTAMP
);

-- 배송 서비스 테이블
CREATE TABLE shipments (
    shipment_id INT PRIMARY KEY,
    order_id INT,
    warehouse_id INT,
    status VARCHAR(50),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)  -- 직접 물리적 FK
);

-- 결제 서비스 테이블
CREATE TABLE payments (
    payment_id INT PRIMARY KEY,
    order_id INT,
    amount DECIMAL(10, 2),
    status VARCHAR(50),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)  -- 직접 물리적 FK
);

-- 주문 서비스 팀이 새로운 기능을 위해 스키마 변경
ALTER TABLE orders ADD COLUMN subscription_id INT;
-- 문제: 다른 서비스들도 즉시 배포해야 함 (코드 변경 필요)
-- 문제: 배송 서비스가 준비 안 됐으면 주문 서비스 배포 블로킹
-- 문제: 변경의 영향 범위가 불분명 (모든 서비스 점검 필요)

-- 결제 서비스 팀이 쿼리 성능을 위해 인덱스 추가
CREATE INDEX idx_orders_customer ON orders(customer_id);
-- 문제: 누가 언제 왜 만들었는지 불명확
-- 문제: 주문 서비스 팀이 이 인덱스가 필요한지 모름
-- 문제: DB 변경 권한이 복잡해짐 (누가 변경할 수 있나?)

-- 급한 버그 수정 발생
UPDATE orders SET total_amount = total_amount * 1.1 WHERE customer_id = 5;
-- 문제: 이 쿼리를 실행할 권한이 누구인가?
-- 문제: 다른 팀의 로직과 충돌하지는 않나?
-- 문제: 회귀 테스트가 필요한 범위가 불명확
```

이 상황에서는:
- **스키마 변경 병목**: 한 팀의 변경이 다른 팀을 블로킹
- **배포 조율 필요**: 여러 팀의 배포를 동기화해야 함
- **장애 전파**: 배송 서비스의 느린 쿼리가 주문 서비스의 성능까지 영향
- **데이터 소유권 불명확**: 누가 어떤 컬럼을 관리하는가?

---

## ✨ 올바른 접근 (After — Database per Service)

```sql
-- AFTER: 각 서비스가 독립적인 데이터베이스를 소유
-- 주문 서비스 - orders_db (주문 서비스 팀 소유)
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    total_amount DECIMAL(10, 2),
    status VARCHAR(50),
    subscription_id INT,  -- 주문 서비스 팀이 자유롭게 추가 가능
    created_at TIMESTAMP
);

-- 배송 서비스 - shipments_db (배송 서비스 팀 소유)
CREATE TABLE shipments (
    shipment_id INT PRIMARY KEY,
    order_id INT,  -- 주문 서비스의 ID를 참조하지만 FK 없음
    warehouse_id INT,
    status VARCHAR(50),
    expected_delivery_date DATE
    -- 중요: orders 테이블을 JOIN할 수 없음 (독립적 DB)
);

-- 결제 서비스 - payments_db (결제 서비스 팀 소유)
CREATE TABLE payments (
    payment_id INT PRIMARY KEY,
    order_id INT,  -- 주문 서비스의 ID를 참조하지만 FK 없음
    amount DECIMAL(10, 2),
    status VARCHAR(50),
    payment_method VARCHAR(50)
);

-- 각 서비스는 독립적으로 스키마 변경 가능
-- 주문 서비스 - 새 기능을 위해 스키마 변경 (다른 팀 영향 없음)
ALTER TABLE orders ADD COLUMN subscription_id INT;
-- 장점: 주문 서비스 팀만 배포하면 됨
-- 장점: 배송/결제 서비스는 영향 없음
-- 장점: 변경 권한이 명확함 (주문 서비스 팀이 소유)

-- 배송 서비스 - 자신의 DB에만 영향
ALTER TABLE shipments ADD COLUMN tracking_number VARCHAR(100);
-- 다른 팀은 전혀 영향 받지 않음

-- 주문 조회 시 배송 정보가 필요? → API Composition 또는 CQRS 사용
-- (다음 장 참조)
```

이 구조에서는:
- **독립적 스키마 변경**: 각 팀이 자신의 DB는 완전히 제어
- **독립적 배포**: 다른 팀의 일정에 영향 받지 않음
- **장애 격리**: 배송 DB 장애가 주문 서비스 성능에 영향 없음
- **데이터 소유권 명확**: 각 서비스가 자신의 데이터만 책임

---

## 🔬 내부 동작 원리 — Database per Service 구현 수준

### 1. 스키마 분리 (Schema-per-service)

```
┌─────────────────────────────────────────┐
│  공유 데이터베이스 서버                 │
├─────────────────────────────────────────┤
│ Schema: orders_schema                    │
│  ├─ orders 테이블                       │
│  └─ orders_temp 테이블                  │
├─────────────────────────────────────────┤
│ Schema: shipments_schema                 │
│  ├─ shipments 테이블                    │
│  └─ shipments_history 테이블            │
├─────────────────────────────────────────┤
│ Schema: payments_schema                  │
│  ├─ payments 테이블                     │
│  └─ refunds 테이블                      │
└─────────────────────────────────────────┘
```

**스키마 분리의 장점:**
- 비용 효율적 (DB 서버 1개)
- 중앙화된 백업/모니터링
- 스키마 수준의 격리 (물리적 FK 불가)

**스키마 분리의 문제:**
- 누군가 잘못 크로스 스키마 JOIN을 작성할 수 있음 (기술적 강제 아님)
- DB 리소스 경합 (한 서비스의 무거운 쿼리가 다른 서비스 영향)
- 권한 관리 복잡 (DB 유저별 스키마 접근 제어 필요)

### 2. 서버 분리 (Database-per-service)

```
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ 주문 서비스      │    │ 배송 서비스      │    │ 결제 서비스      │
│ (orders_db)      │    │ (shipments_db)   │    │ (payments_db)    │
├──────────────────┤    ├──────────────────┤    ├──────────────────┤
│  PostgreSQL      │    │  MySQL           │    │  PostgreSQL      │
│  (port 5432)     │    │  (port 3306)     │    │  (port 5433)     │
├──────────────────┤    ├──────────────────┤    ├──────────────────┤
│ orders 테이블    │    │ shipments 테이블 │    │ payments 테이블  │
└──────────────────┘    └──────────────────┘    └──────────────────┘
```

**서버 분리의 장점:**
- 완전한 기술적 격리 (크로스 스키마 JOIN 불가능)
- 독립적 리소스 관리 (각 서비스가 자신의 메모리, 디스크, CPU 보유)
- 각 서비스가 최적의 DB 기술 선택 가능 (Polyglot Persistence)

**서버 분리의 문제:**
- 높은 운영 비용 (DB 서버 여러 개 유지)
- 복잡한 백업/복구 전략
- 네트워크 오버헤드 (원격 DB 연결)

### 3. 실전: 하이브리드 접근

```
┌─────────────────────────────────────┐      ┌──────────────┐
│  Primary DB Server                  │      │ Cache Server │
│  (Cost-sensitive 서비스들)          │      │  (Redis)     │
├─────────────────────────────────────┤      └──────────────┘
│ Schema: auth_schema (인증)           │
│ Schema: inventory_schema (재고)      │
│ Schema: notification_schema (알림)   │
└─────────────────────────────────────┘
         ↑
    스키마 분리
    (비용 효율)

┌──────────────────────┐     ┌──────────────────────┐
│ 주문 DB (PostgreSQL) │     │ 검색 DB (Elastic)    │
└──────────────────────┘     └──────────────────────┘
         ↑                              ↑
    서버 분리                       서버 분리
    (성능/격리 중요)               (Polyglot)
```

**결정 기준:**
- **스키마 분리 적합**: 비용 우선, 장애 격리 필요 없음, 리소스 공유 가능
- **서버 분리 적합**: 장애 격리 중요, 독립적 스케일링 필요, 기술 선택 자유도 중요

---

## 💻 Java 실전 코드 — Database per Service 구현

```java
// OrderService - 주문 서비스 (자신의 DB만 접근)
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;  // orders_db만 접근
    
    @Autowired
    private RestTemplate restTemplate;  // 다른 서비스는 API로 호출
    
    public OrderResponse createOrder(OrderRequest request) {
        // 1. 주문 저장 (자신의 DB)
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        order.setTotalAmount(request.getTotalAmount());
        order.setStatus("PENDING");
        Order savedOrder = orderRepository.save(order);
        
        // 2. 배송 정보가 필요? → 배송 서비스 API 호출 (DB 직접 접근 X)
        ShipmentInfo shipmentInfo = null;
        try {
            shipmentInfo = restTemplate.getForObject(
                "http://shipment-service/api/shipments/" + savedOrder.getOrderId(),
                ShipmentInfo.class
            );
        } catch (Exception e) {
            // 배송 서비스 장애 시에도 주문은 정상 생성됨 (결합도 낮음)
            log.warn("배송 서비스 조회 실패: " + e.getMessage());
        }
        
        return new OrderResponse(savedOrder, shipmentInfo);
    }
    
    // 주문만 변경 (다른 서비스에 영향 없음)
    public void updateOrderStatus(Long orderId, String status) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.setStatus(status);
        orderRepository.save(order);
        
        // 상태 변경 알림 (Event-driven, 메시지 큐 사용)
        eventPublisher.publishEvent(new OrderStatusChanged(orderId, status));
    }
}

// ShipmentService - 배송 서비스 (자신의 DB만 접근)
@Service
public class ShipmentService {
    
    @Autowired
    private ShipmentRepository shipmentRepository;  // shipments_db만 접근
    
    // 주문 DB에는 접근하지 않음 (API로 주문 정보 받음)
    public ShipmentInfo getShipmentInfo(Long orderId) {
        // orderId는 외부에서 받은 참조값일 뿐 (FK 아님)
        // shipments_db에서 order_id를 검색
        Shipment shipment = shipmentRepository.findByOrderId(orderId);
        if (shipment == null) {
            throw new ShipmentNotFoundException(orderId);
        }
        return shipment.toResponse();
    }
}

// PaymentService 설정 예시
@Service
public class PaymentService {
    
    @Autowired
    private PaymentRepository paymentRepository;  // payments_db만 접근
    
    public void processPayment(PaymentRequest request) {
        Payment payment = new Payment();
        payment.setOrderId(request.getOrderId());
        payment.setAmount(request.getAmount());
        payment.setStatus("PROCESSING");
        paymentRepository.save(payment);
        
        // 주문 정보가 필요? → 주문 서비스 API 호출
        // 절대 orders_db에 직접 접근하지 않음
        Order order = restTemplate.getForObject(
            "http://order-service/api/orders/" + request.getOrderId(),
            Order.class
        );
    }
}

// 스키마 분리 설정 (application.yml)
spring:
  datasource:
    # 스키마별로 분리된 DB (같은 DB 서버 내)
    auth:
      url: jdbc:postgresql://db.local:5432/shared_db
      username: auth_user
      password: ${AUTH_DB_PASSWORD}
    orders:
      url: jdbc:postgresql://db.local:5432/shared_db
      username: orders_user
      password: ${ORDERS_DB_PASSWORD}
    
  jpa:
    properties:
      hibernate:
        default_schema: orders_schema  # 기본 스키마

// 또는 서버 분리 설정 (application.yml)
spring:
  datasource:
    primary:
      url: jdbc:postgresql://orders-db.local:5432/orders_db
      username: ${ORDERS_DB_USER}
    shipment:
      url: jdbc:mysql://shipments-db.local:3306/shipments_db
      username: ${SHIPMENTS_DB_USER}
    payment:
      url: jdbc:postgresql://payments-db.local:5432/payments_db
      username: ${PAYMENTS_DB_USER}
```

---

## 📊 스키마 분리 vs 서버 분리 비교

| 항목 | 스키마 분리 | 서버 분리 |
|------|-----------|---------|
| **비용** | 낮음 (1개 DB 서버) | 높음 (다중 서버) |
| **기술적 강제** | 약함 (개발자 실수 가능) | 강함 (물리적으로 불가능) |
| **리소스 격리** | 약함 (CPU/메모리 공유) | 강함 (독립적 리소스) |
| **장애 격리** | 약함 (한 서비스 과부하 → 전체 영향) | 강함 (장애 격리) |
| **Polyglot 지원** | 어려움 (같은 DB 기술) | 쉬움 (각 서비스 기술 선택) |
| **백업/복구** | 중앙화 (단순) | 분산 (복잡) |
| **권한 관리** | 복잡 (스키마별 접근 제어) | 단순 (서버별 접근 제어) |
| **네트워크** | 로컬 (빠름) | 원격 (레이턴시 추가) |
| **스케일링** | 모든 서비스가 같은 리소스 경합 | 각 서비스 독립적 스케일 |

---

## ⚖️ 트레이드오프

```
비용 vs 격리
├─ 낮은 비용 (스키마 분리)
│  ├─ ✅ 인프라 비용 절감
│  ├─ ✅ 운영 인원 절감
│  └─ ❌ 기술적 강제 약함
│
└─ 높은 격리 (서버 분리)
   ├─ ✅ 강한 장애 격리
   ├─ ✅ 독립적 스케일링
   ├─ ✅ 기술 자유도
   └─ ❌ 운영 비용 증가

초기 단계: 스키마 분리로 시작 → 비용 효율
성장 단계: 서버 분리로 전환 → 장애 격리 중요
성숙 단계: 하이브리드 (중요한 서비스만 분리)

조인 없음으로 인한 복잡성 증가
├─ API Composition (N+1 문제)
├─ CQRS 읽기 모델 (운영 복잡도)
└─ 데이터 복제 (동기화 이슈)

강한 일관성 vs 최종 일관성
├─ ACID (Shared DB) - 강하지만 확장 어려움
└─ BASE (Distributed DB) - 약하지만 확장 용이
```

---

## 📌 핵심 정리

```
✅ Database per Service의 핵심 원칙:
  - 각 마이크로서비스는 자신의 독립적인 데이터베이스를 소유
  - 다른 서비스의 DB에 직접 접근하지 않음
  - 서비스 간 통신은 API 또는 메시지로만 진행

✅ Shared Database 안티패턴의 문제:
  - 스키마 변경이 병목 (여러 팀 조율 필요)
  - 배포 일정 동기화 강제 (독립 배포 불가)
  - 장애 격리 불가 (한 팀의 버그 → 전체 장애)

✅ 구현 방법:
  - 스키마 분리: 비용 효율, 기술적 강제 약함
  - 서버 분리: 비용 높음, 기술적 강제 강함
  - 초기에는 스키마 분리로 시작, 필요시 서버 분리 전환

✅ 대가:
  - 조인이 불가능 → API Composition 또는 CQRS로 해결
  - 분산 트랜잭션 필요 → Saga 패턴 사용
  - 최종 일관성 허용 (ACID → BASE)

✅ Database per Service가 해결하는 것:
  - 서비스 팀의 독립성 보장
  - 독립적 배포 가능
  - 장애 격리 (구현 수준에 따라)
  - 기술 자유도 (Polyglot Persistence)
```

---

## 🤔 생각해볼 문제

**Q1.** 현재 주문(orders), 배송(shipments), 결제(payments) 서비스가 모두 shared_db 데이터베이스의 다른 스키마를 사용하고 있습니다. 주문 서비스가 고객 정보를 표시하기 위해 `SELECT o.*, c.* FROM orders o JOIN customers c ON o.customer_id = c.customer_id` 쿼리를 작성했다면, 고객 정보 서비스가 고객 스키마 구조를 변경할 때 어떤 문제가 발생할까요?

<details>
<summary>해설 보기</summary>

이 쿼리는 주문 서비스가 고객 정보 서비스의 데이터베이스 스키마에 강한 결합을 만듭니다. 고객 정보 서비스가 컬럼 이름을 변경하거나 구조를 변경하면, 주문 서비스 코드도 즉시 수정해야 합니다. 특히:

1. **배포 동기화 필요**: 스키마 변경 후 쿼리 변경 후 배포 (여러 팀 조율)
2. **런타임 에러**: 변경 전에 주문 서비스가 배포되면 NULL 또는 예외 발생
3. **독립 배포 불가**: Database per Service의 핵심 이점 상실

정답: 이런 직접 JOIN은 금지되어야 합니다. 대신:
- 주문 조회 API는 주문 정보만 반환
- 고객 정보가 필요하면 클라이언트가 주문 서비스 + 고객 서비스 API를 각각 호출
- 또는 주문 생성 시 고객 정보를 스냅샷으로 저장 (별도 섹션 참조)

</details>

**Q2.** 초기 스타트업에서는 스키마 분리로 시작했는데, 회사가 성장하면서 데이터베이스 서버가 과부하 상태입니다. 한 서비스의 느린 쿼리가 전체 서비스 성능에 영향을 줍니다. 서버 분리로의 마이그레이션 전략은 어떻게 될까요?

<details>
<summary>해설 보기</summary>

스키마 분리에서 서버 분리로 마이그레이션하는 단계:

1. **영향도 분석**:
   - 어떤 서비스의 쿼리가 가장 무거운가?
   - 다른 서비스와의 직접 JOIN이 있는가?

2. **점진적 분리** (Strangler Fig 패턴):
   - 가장 무거운 서비스를 먼저 분리 (예: 검색 서비스)
   - 리전 라우팅 로직 추가 (어느 DB에서 조회할지)
   - 데이터 동기화 (이전 스키마에서 새 서버로 복제)

3. **검증 단계**:
   - 샤도우 모드: 새 서버에서도 쿼리 실행, 결과 비교
   - 일정 기간 동시 운영
   - 이상 없을 때 라우팅 전환

4. **점진적 완전 분리**:
   - 핵심 서비스부터 하나씩 분리
   - 비용과 이점 재평가
   - 필수 서비스만 분리, 나머지는 스키마 분리 유지 (비용 최적화)

</details>

**Q3.** 어떤 팀이 실수로 다른 서비스의 스키마에 직접 접근하는 코드를 작성했습니다. 코드 리뷰에서도 통과했습니다. 조직 수준에서 이를 방지할 수 있는 방법은 무엇일까요?

<details>
<summary>해설 보기</summary>

기술적 강제가 최선의 방법입니다:

1. **스키마 분리 → 서버 분리로 전환**:
   - 더 이상 크로스 스키마 JOIN이 가능하지 않음
   - DB 연결 문자열에 접근 권한이 다름

2. **데이터베이스 접근 제어**:
   - 각 서비스에 해당 스키마/DB에만 접근 권한 부여
   - 다른 스키마 접근 시도 → 데이터베이스 거부 (SQL 예외)
   - 접근 시도 로그 자동 수집

3. **코드 리뷰 자동화**:
   - 정적 분석 도구: 다른 서비스 DB 접근 감지
   - 테스트: 런타임에 크로스 서비스 DB 접근 시도 → 테스트 실패
   - CI/CD 통합: merge 전 자동 검사

4. **조직 문화**:
   - "DB 접근은 API로만" 정책 명문화
   - 코드 리뷰 체크리스트에 포함
   - 아키텍처 가이드 공유

결론: **기술적 강제 > 프로세스 강제** (프로세스는 실수하기 쉬움)

</details>

---

<div align="center">

**[⬅️ 이전: Chapter 2 — GraphQL Federation — 분산 스키마 통합](../communication-patterns/07-graphql-federation.md)** | **[홈으로 🏠](../README.md)** | **[다음: Polyglot Persistence — 서비스별 DB 기술 선택 ➡️](./02-polyglot-persistence.md)**

</div>

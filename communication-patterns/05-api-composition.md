# 05. API Composition 패턴 — 여러 서비스 데이터 조합

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 관계형 데이터베이스의 JOIN을 마이크로서비스에서 할 수 없는 이유와, 그 대신 API Composition이 필요한 이유는?
- API Gateway, BFF(Backend for Frontend), 클라이언트 측 조합의 3가지 패턴을 어떤 상황에 사용할까?
- 한 서비스가 실패했을 때 부분 응답 vs fallback vs timeout 중 어느 전략을 선택해야 할까?
- N+1 서비스 호출 문제를 CompletableFuture로 어떻게 병렬화하고, 성능을 몇 배 개선할까?
- BFF 패턴에서 모바일용 BFF vs 웹용 BFF를 분리하는 기준은?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

마이크로서비스 아키텍처에서는 **관계형 데이터베이스의 JOIN이 불가능**합니다. 각 서비스가 자신의 데이터베이스를 소유하기 때문입니다. 예를 들어 "사용자 정보 + 주문 내역 + 배송 상태"를 조합해서 반환해야 할 때, 이를 단일 서비스에서는 처리할 수 없습니다.

대신 클라이언트(또는 API Gateway, BFF)가 **여러 서비스를 순차적으로 또는 병렬로 호출하여 데이터를 조합**해야 합니다. 이 과정에서 발생하는 문제들:

1. **N+1 호출**: "사용자 100명의 주문 정보" 조회 시 1번(사용자) + 100번(각 주문) = 101번 호출
2. **부분 장애**: User Service는 정상이지만 Inventory Service가 느리면 전체 응답이 느려짐
3. **지연 누적**: 5개 서비스를 순차 호출 시 500ms * 5 = 2500ms
4. **데이터 불일치**: 조합 과정에서 일부 데이터만 최신 버전

API Composition 패턴은 이런 문제들을 **전략적으로 조합하고 병렬화하여 응답 시간을 최소화**하면서 **일관성과 복원력을 보장**합니다.

---

## 😱 흔한 실수 (Before — 순차 호출과 N+1 문제)

```java
// ❌ 안티패턴 1: 순차 호출로 인한 지연 누적
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired private OrderService orderService;
    @Autowired private UserService userService;
    @Autowired private InventoryService inventoryService;
    @Autowired private ShippingService shippingService;
    @Autowired private PaymentService paymentService;
    
    @GetMapping("/{id}")
    public ResponseEntity<OrderDetailResponse> getOrderDetail(
            @PathVariable Long id) {
        
        long startTime = System.currentTimeMillis();
        
        // 1. 주문 조회 (200ms)
        Order order = orderService.getOrder(id);
        
        // 2. 사용자 조회 (100ms) - 순차 호출
        User user = userService.getUser(order.getUserId());
        
        // 3. 배송 정보 조회 (150ms) - 순차 호출
        ShippingInfo shipping = shippingService.getShipping(id);
        
        // 4. 결제 정보 조회 (120ms) - 순차 호출
        PaymentInfo payment = paymentService.getPayment(order.getPaymentId());
        
        // 5. 재고 상태 조회 (180ms) - 순차 호출
        for (OrderItem item : order.getItems()) {
            InventoryStatus inv = inventoryService.getStatus(item.getProductId());
            item.setInventoryStatus(inv);
        }
        
        // 총 소요 시간: 200 + 100 + 150 + 120 + 180 = 750ms ❌
        // 사용자는 0.75초 기다려야 함 (부정적 UX)
        
        long elapsed = System.currentTimeMillis() - startTime;
        logger.info("주문 상세 조회: {}ms", elapsed);
        
        return ResponseEntity.ok(new OrderDetailResponse(
            order, user, shipping, payment));
    }
}

// ❌ 안티패턴 2: N+1 서비스 호출 문제
@RestController
@RequestMapping("/api/users")
public class UserOrderController {
    
    @Autowired private UserService userService;
    @Autowired private OrderService orderService;  // 주문 서비스
    
    @GetMapping("/{id}/orders-with-details")
    public ResponseEntity<UserWithOrdersResponse> getUserWithOrders(
            @PathVariable Long id) {
        
        // 1. 사용자 조회 (1회)
        User user = userService.getUser(id);
        
        // 2. 사용자의 주문 목록 조회 (1회)
        List<Order> orders = orderService.getOrdersByUserId(id); // 50개
        
        // 3. 각 주문의 상세 정보 조회 (N회 = 50회!) ❌
        for (Order order : orders) {
            OrderDetail detail = orderService.getOrderDetail(order.getId());
            // 재고 정보도?
            for (OrderItem item : order.getItems()) {
                InventoryStatus inv = inventoryService.getStatus(item.getProductId());
                // 배송 정보도?
                ShippingInfo shipping = shippingService.getShipping(order.getId());
            }
        }
        
        // 호출 수: 1(사용자) + 1(주문목록) + 50(주문상세) + 
        //          (항목수 * 재고) + (배송)
        // 최악: 200+ 호출!
        // 각 호출 100ms → 총 20초 ❌
        
        return ResponseEntity.ok(new UserWithOrdersResponse(user, orders));
    }
}

// ❌ 안티패턴 3: 일부 서비스 장애 시 전체 장애
@GetMapping("/{id}")
public ResponseEntity<OrderDetailResponse> getOrderDetail(
        @PathVariable Long id) {
    
    try {
        Order order = orderService.getOrder(id);
        User user = userService.getUser(order.getUserId());
        ShippingInfo shipping = shippingService.getShipping(id);
        PaymentInfo payment = paymentService.getPayment(order.getPaymentId());
        
        return ResponseEntity.ok(new OrderDetailResponse(
            order, user, shipping, payment));
        
    } catch (ServiceException e) {
        // ShippingService가 느리면 (타임아웃)
        // → 전체 요청 실패 ❌
        // 사용자가 받는 에러: "서비스 이용 불가"
        // 실제 문제: 배송 정보만 제공 불가 (주문, 사용자, 결제는 정상)
        
        return ResponseEntity.status(503).build();
    }
}
```

---

## ✨ 올바른 접근 (After — 병렬 호출 + 부분 응답)

```java
// ✅ 개선 1: CompletableFuture로 병렬 호출
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired private OrderService orderService;
    @Autowired private UserService userService;
    @Autowired private InventoryService inventoryService;
    @Autowired private ShippingService shippingService;
    @Autowired private PaymentService paymentService;
    
    @Autowired
    @Qualifier("compositionThreadPool")
    private ExecutorService executorService;
    
    @GetMapping("/{id}")
    public ResponseEntity<OrderDetailResponse> getOrderDetail(
            @PathVariable Long id) {
        
        long startTime = System.currentTimeMillis();
        
        try {
            // 1. 주문 조회 (필수)
            Order order = orderService.getOrder(id);
            
            // 2. 병렬로 여러 서비스 호출 (비동기)
            CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(
                () -> userService.getUser(order.getUserId()),
                executorService)
                .exceptionally(ex -> {
                    logger.warn("사용자 정보 조회 실패: {}", order.getUserId());
                    return null; // 부분 응답 허용
                });
            
            CompletableFuture<ShippingInfo> shippingFuture = 
                CompletableFuture.supplyAsync(
                () -> shippingService.getShipping(id),
                executorService)
                .withTimeout(Duration.ofSeconds(3)) // 3초 타임아웃
                .exceptionally(ex -> null);
            
            CompletableFuture<PaymentInfo> paymentFuture = 
                CompletableFuture.supplyAsync(
                () -> paymentService.getPayment(order.getPaymentId()),
                executorService)
                .exceptionally(ex -> null);
            
            // 3. 재고 정보는 배치로 조회 (N개 호출 → 1개 호출)
            List<Long> productIds = order.getItems().stream()
                .map(OrderItem::getProductId)
                .collect(Collectors.toList());
            
            CompletableFuture<Map<Long, InventoryStatus>> inventoryFuture =
                CompletableFuture.supplyAsync(
                () -> inventoryService.getStatuses(productIds), // 배치 조회!
                executorService)
                .exceptionally(ex -> new HashMap<>());
            
            // 4. 모든 비동기 작업 완료 대기 (병렬 처리)
            CompletableFuture.allOf(
                userFuture, shippingFuture, 
                paymentFuture, inventoryFuture)
                .join();
            
            // 5. 결과 조합
            User user = userFuture.join();
            ShippingInfo shipping = shippingFuture.join();
            PaymentInfo payment = paymentFuture.join();
            Map<Long, InventoryStatus> inventoryMap = inventoryFuture.join();
            
            // 재고 정보 주문 항목에 매핑
            order.getItems().forEach(item -> 
                item.setInventoryStatus(inventoryMap.get(item.getProductId())));
            
            long elapsed = System.currentTimeMillis() - startTime;
            
            // 순차: 750ms → 병렬: ~250ms (max 요청) ✅
            // 가장 오래 걸린 요청(200ms) 기준으로 완료
            logger.info("주문 상세 조회 (병렬): {}ms", elapsed);
            
            return ResponseEntity.ok(new OrderDetailResponse(
                order, user, shipping, payment));
                
        } catch (TimeoutException e) {
            logger.error("요청 타임아웃: {}", id);
            return ResponseEntity.status(504).build();
        }
    }
}

// ✅ 개선 2: 부분 응답 + Fallback
@GetMapping("/{id}")
public ResponseEntity<OrderDetailResponse> getOrderDetailWithFallback(
        @PathVariable Long id) {
    
    Order order = orderService.getOrder(id);
    
    // User 정보 - 캐시 fallback
    User user = userService.getUser(order.getUserId());
    if (user == null) {
        user = userCache.get(order.getUserId()); // 캐시된 이전 정보
    }
    
    // Shipping 정보 - 기본값 fallback
    ShippingInfo shipping = null;
    try {
        shipping = shippingService.getShipping(id);
    } catch (TimeoutException e) {
        // 타임아웃 시 기본값 반환 (배송 상태 조회 중)
        shipping = ShippingInfo.unknown();
    }
    
    // Payment 정보 - 최소 정보만 반환
    PaymentInfo payment = null;
    try {
        payment = paymentService.getPayment(order.getPaymentId());
    } catch (ServiceException e) {
        // 결제 정보 없음 (보안상 민감)
        payment = null;
    }
    
    // 부분 응답도 괜찮음 (일부 필드 null)
    OrderDetailResponse response = new OrderDetailResponse(
        order, user, shipping, payment);
    
    // 응답 상태: 200 OK (부분 성공)
    // 클라이언트: 받은 정보로 UI 구성, 누락 정보는 보류
    return ResponseEntity.ok(response);
}

// ✅ 개선 3: BFF 패턴 - 모바일용 최적화
@RestController
@RequestMapping("/api/mobile/orders")
public class MobileOrderBFF {
    
    @Autowired private OrderServiceClient orderClient;
    @Autowired private UserServiceClient userClient;
    
    @GetMapping("/{id}")
    public ResponseEntity<MobileOrderResponse> getMobileOrder(
            @PathVariable Long id) {
        
        // 모바일 클라이언트 특화:
        // 1. 최소 데이터만 반환 (대역폭 절약)
        // 2. 그림 요청 최소화
        // 3. 빠른 응답 (< 500ms)
        
        Order order = orderClient.getOrder(id);
        User user = userClient.getUser(order.getUserId());
        
        // 모바일용으로 축소된 응답
        return ResponseEntity.ok(new MobileOrderResponse(
            order.getId(),
            order.getTotalAmount(),
            user.getName(),
            user.getPhoneNumber(), // 연락처만
            order.getStatus()
            // 상세 정보(배송, 결제)는 제외 → 다른 엔드포인트에서 조회
        ));
    }
}

// ✅ 개선 4: 웹용 BFF
@RestController
@RequestMapping("/api/web/orders")
public class WebOrderBFF {
    
    @Autowired private OrderServiceClient orderClient;
    @Autowired private UserServiceClient userClient;
    @Autowired private ShippingServiceClient shippingClient;
    @Autowired private PaymentServiceClient paymentClient;
    
    @GetMapping("/{id}")
    public ResponseEntity<WebOrderResponse> getWebOrder(
            @PathVariable Long id) {
        
        // 웹 클라이언트 특화:
        // 1. 상세 정보 모두 반환
        // 2. 계산된 필드 추가 (총액, 남은 일수 등)
        // 3. 넉넉한 타임아웃
        
        Order order = orderClient.getOrder(id);
        User user = userClient.getUser(order.getUserId());
        ShippingInfo shipping = shippingClient.getShipping(id);
        PaymentInfo payment = paymentClient.getPayment(order.getPaymentId());
        
        // 웹용 상세 응답
        return ResponseEntity.ok(new WebOrderResponse(
            order.getId(),
            order.getTotalAmount(),
            order.getTax(),           // 웹에서는 세금 정보 필요
            order.getShippingCost(),
            user.getName(),
            user.getEmail(),
            user.getPhoneNumber(),
            user.getAddress(),        // 주소 상세
            shipping.getStatus(),
            shipping.getTrackingNumber(),
            shipping.getEstimatedDelivery(),
            payment.getMethod(),
            payment.getLastDigits(),
            calculateRemainingDays(order.getCreatedAt())
        ));
    }
}

// ✅ 개선 5: API Gateway 조합 패턴
@Configuration
public class ApiGatewayComposition {
    
    // Spring Cloud Gateway로 구성
    @Bean
    public RouteLocator compositionRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            // /api/orders/{id} 요청 시 여러 서비스 조합
            .route("order-composition", r -> r
                .path("/api/orders/{id}")
                .filters(f -> f
                    // 1. Order Service 호출
                    .addRequestHeader("X-Service", "order-service")
                    // 2. 응답에 User 정보 추가
                    .modifyResponseBody(
                        String.class, ComposedOrderResponse.class,
                        (exchange, body) -> {
                            // 응답 본문에서 userId 추출
                            // User Service 호출
                            // 결과 조합
                            return Mono.just(composedResponse);
                        }
                    )
                )
                .uri("http://order-service")
            )
            .build();
    }
}
```

---

## 🔬 내부 동작 원리 — 성능 최적화와 일관성

### 1. 순차 vs 병렬 호출의 성능

```
순차 호출 (Sequential):
[User Service] (100ms)
                ↓
            [Inventory Service] (150ms)
                                ↓
                            [Shipping Service] (120ms)
                                                 ↓
                                         [Payment Service] (80ms)
────────────────────────────────────────────────────────
총 시간: 100 + 150 + 120 + 80 = 450ms

병렬 호출 (Parallel with CompletableFuture):
[User Service] (100ms)     ┐
[Inventory Service] (150ms)├─ 동시 실행
[Shipping Service] (120ms) │
[Payment Service] (80ms)   ┘
────────────────────────────────────────────────────────
총 시간: max(100, 150, 120, 80) = 150ms

성능 향상: 450ms → 150ms (3배!)

구현:
CompletableFuture.allOf(
    supplyAsync(() -> userService.get(id)),
    supplyAsync(() -> inventoryService.get(id)),
    supplyAsync(() -> shippingService.get(id)),
    supplyAsync(() -> paymentService.get(id))
).join();
```

### 2. N+1 문제의 해결

```
문제: 사용자 100명의 주문 정보 조회

❌ N+1 호출 패턴:
1회: 사용자 100명 목록 조회
100회: 각 사용자의 주문 조회 (for 루프)
────────────────────────────────
총: 101회

각 호출 20ms → 2020ms ❌

✅ 배치 조회 패턴:
1회: 사용자 100명 목록 조회
1회: 100명의 주문을 한 번에 조회 (IN 쿼리)
────────────────────────────────
총: 2회

각 호출 20ms → 40ms ✅

구현 (서비스 메서드):
// UserService
public List<User> getUsersByIds(List<Long> ids) {
    return users.stream()
        .filter(u -> ids.contains(u.getId()))
        .collect(toList());
}

// OrderService - 배치 조회 엔드포인트
public Map<Long, List<Order>> getOrdersByUserIds(List<Long> userIds) {
    return orders.stream()
        .filter(o -> userIds.contains(o.getUserId()))
        .collect(Collectors.groupingBy(Order::getUserId));
}

// Composition 레이어
List<User> users = userService.getUsers(); // 100명
List<Long> userIds = users.stream()
    .map(User::getId)
    .collect(toList());

Map<Long, List<Order>> ordersByUser = 
    orderService.getOrdersByUserIds(userIds); // 배치!

// 메모리에서 조합
users.forEach(user -> 
    user.setOrders(ordersByUser.getOrDefault(user.getId(), [])));
```

### 3. 부분 장애 대응 전략

```
3개 서비스 호출 시나리오:

             ┌─→ Service A (정상, 100ms)
Client ──────┼─→ Service B (느림, 타임아웃 3초)
             └─→ Service C (장애, 500ms 에러)

전략 1: All-or-Nothing
- 하나 실패 → 전체 실패 (500ms)
- 장점: 일관성
- 단점: 가용성 낮음

전략 2: Partial Success
- 가능한 정보만 반환 (max 100ms)
- A: 성공 ✓
- B: 타임아웃 ✗ → null 또는 기본값
- C: 실패 ✗ → null 또는 캐시값
- 응답: { A: data, B: null, C: cached }
- 장점: 높은 가용성
- 단점: 완전하지 않은 데이터

전략 3: Fallback Chain
- Primary: Service B (3초 타임아웃)
  → 실패
- Secondary: Cache
  → 최근 데이터 반환 (30분 전)
- Tertiary: Default
  → "조회 중" 상태
- 장점: 최선의 데이터 제공
- 단점: 구현 복잡

코드:
CompletableFuture.supplyAsync(() -> serviceB.get())
    .completeOnTimeout(null, 3, TimeUnit.SECONDS)
    .exceptionally(ex -> {
        // 캐시 시도
        return cache.get(key);
    })
    .exceptionally(ex -> {
        // 기본값
        return default Value;
    })
```

### 4. API Composition 패턴 비교

```
                API Gateway        BFF            Client-Side
┌─────────────┬──────────────┬──────────────┬──────────────┐
│ 위치        │ 중앙         │ 서비스별    │ 클라이언트    │
├─────────────┼──────────────┼──────────────┼──────────────┤
│ 복잡도      │ 높음         │ 중간         │ 낮음          │
│             │              │              │               │
│ 유지보수    │ 병목(변경)   │ 분산(유연)  │ 클라이언트   │
│ 비용        │              │              │ 책임          │
├─────────────┼──────────────┼──────────────┼──────────────┤
│ 캐싱        │ 중앙(효율)   │ 로컬(최적)  │ 로컬(최소)   │
├─────────────┼──────────────┼──────────────┼──────────────┤
│ 보안        │ 중앙제어     │ 분산제어    │ 클라이언트   │
├─────────────┼──────────────┼──────────────┼──────────────┤
│ 사용 사례   │ 공개 API    │ 특화된 클라 │ 간단한      │
│             │ (인증 중앙)  │ (모바일,    │ 조합        │
│             │              │  웹)        │              │
└─────────────┴──────────────┴──────────────┴──────────────┘

권장:
- 복잡한 인증: API Gateway (JWT 검증 중앙화)
- 여러 클라이언트: BFF (모바일/웹 분리)
- 단순 조합: Client-Side (프론트엔드)
```

---

## 💻 코드 예제 — API Composition 실전

### CompletableFuture 유틸리티
```java
@Component
public class CompositionHelper {
    
    @Autowired
    @Qualifier("compositionThreadPool")
    private ExecutorService executor;
    
    /**
     * 여러 비동기 작업을 병렬로 실행하고 결과를 조합
     */
    public <T1, T2, T3, R> R compose(
            Supplier<T1> supplier1,
            Supplier<T2> supplier2,
            Supplier<T3> supplier3,
            ThreeArgFunction<T1, T2, T3, R> combiner) {
        
        CompletableFuture<T1> future1 = 
            CompletableFuture.supplyAsync(supplier1, executor)
                .exceptionally(ex -> {
                    logger.warn("작업 1 실패", ex);
                    return null;
                });
        
        CompletableFuture<T2> future2 = 
            CompletableFuture.supplyAsync(supplier2, executor)
                .exceptionally(ex -> {
                    logger.warn("작업 2 실패", ex);
                    return null;
                });
        
        CompletableFuture<T3> future3 = 
            CompletableFuture.supplyAsync(supplier3, executor)
                .exceptionally(ex -> {
                    logger.warn("작업 3 실패", ex);
                    return null;
                });
        
        return CompletableFuture.allOf(future1, future2, future3)
            .thenApply(v -> combiner.apply(
                future1.join(),
                future2.join(),
                future3.join()))
            .join();
    }
    
    @FunctionalInterface
    public interface ThreeArgFunction<T1, T2, T3, R> {
        R apply(T1 t1, T2 t2, T3 t3);
    }
}

// 사용 예
@Service
public class OrderCompositionService {
    
    @Autowired private CompositionHelper compositionHelper;
    @Autowired private OrderClient orderClient;
    @Autowired private UserClient userClient;
    @Autowired private ShippingClient shippingClient;
    
    public OrderDetail getOrderDetail(Long orderId) {
        return compositionHelper.compose(
            () -> orderClient.getOrder(orderId),
            () -> userClient.getUser(...),
            () -> shippingClient.getShipping(orderId),
            (order, user, shipping) -> 
                new OrderDetail(order, user, shipping)
        );
    }
}
```

### BFF 구현 예제
```java
// 모바일용 BFF
@RestController
@RequestMapping("/api/bff/mobile")
public class MobileOrderBFF {
    
    @Autowired private RestTemplate restTemplate;
    
    @GetMapping("/orders/{id}")
    public ResponseEntity<MobileOrderDto> getOrder(@PathVariable Long id) {
        // 최소 정보만 병렬로 조회
        CompletableFuture<Order> orderFuture = 
            CompletableFuture.supplyAsync(
                () -> restTemplate.getForObject(
                    "http://order-service/orders/" + id, Order.class));
        
        CompletableFuture<User> userFuture = 
            CompletableFuture.supplyAsync(
                () -> {
                    Order order = orderFuture.join();
                    return restTemplate.getForObject(
                        "http://user-service/users/" + order.getUserId(),
                        User.class);
                });
        
        CompletableFuture.allOf(orderFuture, userFuture).join();
        
        Order order = orderFuture.join();
        User user = userFuture.join();
        
        // 모바일 최적 응답 (텍스트만)
        return ResponseEntity.ok(new MobileOrderDto(
            order.getId(),
            order.getTotalAmount(),
            order.getStatus(),
            user.getName()
        ));
    }
}

// 웹용 BFF
@RestController
@RequestMapping("/api/bff/web")
public class WebOrderBFF {
    
    @Autowired private RestTemplate restTemplate;
    
    @GetMapping("/orders/{id}")
    public ResponseEntity<WebOrderDto> getOrder(@PathVariable Long id) {
        // 모든 정보를 병렬로 조회
        CompletableFuture<Order> orderFuture = ...;
        CompletableFuture<User> userFuture = ...;
        CompletableFuture<ShippingInfo> shippingFuture = ...;
        CompletableFuture<PaymentInfo> paymentFuture = ...;
        
        CompletableFuture.allOf(
            orderFuture, userFuture, 
            shippingFuture, paymentFuture).join();
        
        // 웹 상세 응답
        return ResponseEntity.ok(new WebOrderDto(
            order, user, shipping, payment,
            calculateMetadata(order)
        ));
    }
    
    private OrderMetadata calculateMetadata(Order order) {
        // 웹용 추가 계산
        return new OrderMetadata(
            order.getTax(),
            calculateRemainingDays(order.getCreatedAt()),
            estimateDeliveryDate(order.getShippingAddress())
        );
    }
}
```

### 타임아웃과 부분 응답
```java
@Service
public class ResilientCompositionService {
    
    @Autowired private RestTemplate restTemplate;
    @Autowired private CacheManager cacheManager;
    
    public OrderWithUser getOrderWithFallback(Long orderId) {
        Order order = restTemplate.getForObject(
            "http://order-service/orders/" + orderId,
            Order.class);
        
        User user = null;
        try {
            // 3초 타임아웃으로 사용자 정보 조회
            user = RestTemplate로 3초 제한 조회();
        } catch (ResourceAccessException e) {
            logger.warn("사용자 정보 조회 실패 (타임아웃), 캐시 사용");
            // 캐시에서 이전 정보 가져오기
            Cache cache = cacheManager.getCache("users");
            if (cache != null) {
                Cache.ValueWrapper wrapper = 
                    cache.get(order.getUserId());
                if (wrapper != null) {
                    user = (User) wrapper.get();
                }
            }
        }
        
        // user가 null일 수 있음 (부분 응답)
        return new OrderWithUser(order, user);
    }
}
```

---

## 📊 패턴 비교 — API Composition 방식

| 구분 | API Gateway | BFF | Client-Side |
|------|-----------|-----|-----------|
| **구현 위치** | 중앙 (게이트웨이) | 서비스별 | 클라이언트 |
| **조합 로직** | 중앙집중식 | 분산 | 클라이언트 |
| **캐싱** | 중앙 (효율) | 로컬 (최적) | 로컬 (최소) |
| **보안** | 중앙 제어 | 서비스별 | 클라이언트 |
| **의존성 관리** | 병목 위험 | 독립적 | 느슨함 |
| **복잡도** | 높음 | 중간 | 낮음 |
| **규모 확장** | 어려움 | 쉬움 | 가능 |
| **권장 사례** | 공개 API, 인증 중앙 | 멀티 클라이언트 | 프론트엔드 |

---

## ⚖️ 트레이드오프

```
병렬 호출의 성능 vs 복잡도:

병렬 호출 (CompletableFuture)
✅ 성능: 3배 이상 향상 (450ms → 150ms)
✅ 응답 시간 예측 가능 (max 서비스)

❌ 복잡도: 비동기 코드 (디버깅 어려움)
❌ 메모리: 동시 요청 처리 (스레드풀 관리)
❌ 에러 처리: 부분 실패 대응 필요

────────────────────────────────────────

부분 응답 vs 일관성:

부분 응답 (일부 null 허용)
✅ 가용성: 한 서비스 장애 → 나머지 정보 제공
✅ 사용자 경험: "뭔가라도 보여주기"

❌ 복잡도: 클라이언트 null 처리
❌ 일관성: 버전 차이 가능성

완전 응답 (모두 필수)
✅ 일관성: 항상 완전한 데이터
✅ 예측성: 데이터 형식 일정

❌ 가용성: 한 서비스 장애 → 전체 실패
❌ 사용자 경험: "서비스 이용 불가"

────────────────────────────────────────

N+1 배치 조회 최적화:

배치 조회 (권장)
- 100명 사용자 → 100명 주문 배치 조회
- 호출: 2회 (1회 + 1회 배치)
- 시간: 40ms

N+1 루프 조회
- 100명 사용자 → 각각 주문 조회
- 호출: 101회 (1회 + 100회)
- 시간: 2020ms

비용: 50배 성능 향상
```

---

## 📌 핵심 정리

```
✅ API Composition의 핵심 원리

1. JOIN 불가능 (각 서비스 독립 DB)
   → 클라이언트 또는 중간층이 조합

2. 병렬 호출로 성능 최적화
   순차: A(100ms) + B(150ms) + C(120ms) = 370ms
   병렬: max(100, 150, 120) = 150ms
   → 2.5배 향상

3. 부분 응답으로 가용성 향상
   - 일부 서비스 느림/장애 → 나머지 정보는 반환
   - fallback (캐시, 기본값) 활용

✅ 3가지 Composition 패턴

1. API Gateway
   - 중앙 조합 (공개 API)
   - 장점: 인증 중앙화, 일관된 정책
   - 단점: 병목, 높은 복잡도
   
2. BFF (Backend for Frontend)
   - 클라이언트별 분산 조합 (모바일/웹)
   - 장점: 클라이언트 최적화, 독립 배포
   - 단점: 코드 중복, 관리 분산
   
3. Client-Side
   - 클라이언트가 직접 조합 (단순 조합)
   - 장점: 간단, 백엔드 부하 적음
   - 단점: 보안 노출, 네트워크 오버헤드

✅ 성능 최적화 전략

1. 병렬 호출
   CompletableFuture.allOf()
   → 가장 긴 요청 시간으로 완료

2. 배치 조회 (N+1 해결)
   IN 쿼리로 100개를 1회에 조회
   → 101회 → 2회

3. 타임아웃 설정
   - Critical Path: 3초
   - Optional: 1초
   → 느린 서비스가 전체 지연 방지

4. Fallback 전략
   Primary → Cache → Default Value
   → 최상의 가용성

✅ 부분 응답 설계

```json
// 완전 응답 (모두 필수)
{
  "orderId": 123,
  "user": {...},
  "shipping": {...},
  "payment": {...}
}

// 부분 응답 (일부 null 허용)
{
  "orderId": 123,
  "user": null,        // 조회 실패
  "shipping": {...},
  "payment": null      // 타임아웃
}
```

✅ BFF 패턴 분리 기준

모바일 BFF:
- 최소 필드만 (대역폭)
- 빠른 응답 (< 500ms)
- 텍스트 중심

웹 BFF:
- 모든 필드 (상세)
- 넉넉한 타임아웃
- 계산 필드 추가

구분하는 이유:
- 네트워크 대역폭 (모바일 < 웹)
- 화면 크기 (모바일 작음)
- 사용 패턴 (모바일 빠른 스크롤)
```

---

## 🤔 생각해볼 문제

**Q1.** 사용자 100명의 주문 목록과 각 주문의 상세 정보를 한 번에 조회해야 합니다. N+1 문제를 해결하려면?

<details>
<summary>해설 보기</summary>

**N+1 문제:**

```
❌ 나쁜 방식 (101회 호출):
1회: getUserWithOrders() → 100명 사용자
100회: 각 사용자의 주문 상세 조회

총: 101회

✅ 좋은 방식 (2회 호출):
1회: getUsers() → 100명 사용자
1회: getOrdersByUserIds([100명ID]) → 모든 주문
     → Map<UserId, List<Orders>>

구현:
```

```java
@Service
public class UserOrderCompositionService {
    
    @Autowired private UserServiceClient userClient;
    @Autowired private OrderServiceClient orderClient;
    @Autowired private ExecutorService executor;
    
    public List<UserWithOrders> getUsersWithOrders() {
        // 1. 모든 사용자 조회 (병렬)
        CompletableFuture<List<User>> usersFuture = 
            CompletableFuture.supplyAsync(
                () -> userClient.getAllUsers(),
                executor);
        
        // 2. 배치: 모든 사용자의 주문 한 번에 조회 (병렬)
        CompletableFuture<Map<Long, List<Order>>> ordersFuture =
            usersFuture.thenApplyAsync(users -> {
                List<Long> userIds = users.stream()
                    .map(User::getId)
                    .collect(toList());
                
                // 배치 조회 (1회!)
                return orderClient.getOrdersByUserIds(userIds);
            }, executor);
        
        // 3. 조합
        return CompletableFuture.allOf(usersFuture, ordersFuture)
            .thenApply(v -> {
                List<User> users = usersFuture.join();
                Map<Long, List<Order>> ordersMap = ordersFuture.join();
                
                return users.stream()
                    .map(user -> new UserWithOrders(
                        user,
                        ordersMap.getOrDefault(user.getId(), [])
                    ))
                    .collect(toList());
            })
            .join();
    }
}

// OrderService 배치 엔드포인트
@RestController
@RequestMapping("/orders")
public class OrderBatchController {
    
    @PostMapping("/batch/by-user-ids")
    public ResponseEntity<Map<Long, List<Order>>> 
            getOrdersByUserIds(@RequestBody List<Long> userIds) {
        
        // 데이터베이스 IN 쿼리로 최적화
        List<Order> orders = orderRepository
            .findAllByUserIdIn(userIds);  // SQL: IN (?, ?, ...)
        
        // 메모리에서 grouping
        Map<Long, List<Order>> result = orders.stream()
            .collect(Collectors.groupingBy(Order::getUserId));
        
        return ResponseEntity.ok(result);
    }
}
```

**성능 비교:**
- ❌ N+1: 101회 × 20ms = 2020ms
- ✅ 배치: 2회 × 20ms = 40ms
- **50배 향상!**

</details>

**Q2.** API Gateway, BFF, Client-Side 중 어느 패턴을 선택할까? (공개 REST API)

<details>
<summary>해설 보기</summary>

**공개 API의 특성:**

```
클라이언트 다양성:
- 웹 브라우저 (JavaScript)
- 모바일 앱 (iOS, Android)
- 데스크톱 앱
- 서드파티 개발자
- 공개 설명서 필요

보안:
- 인증 (OAuth, JWT)
- 권한 (RBAC)
- Rate Limiting
- API Key 관리
```

**선택: API Gateway가 최적**

```java
@Configuration
public class PublicApiGateway {
    
    // 1. 중앙 인증 (모든 요청 검증)
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) {
        http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .requestMatchers("/api/v1/**").authenticated()
            .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults()));
        return http.build();
    }
    
    // 2. 중앙 Rate Limiting
    @Bean
    public RateLimitFilter rateLimitFilter() {
        return new RateLimitFilter();
    }
    
    // 3. 통일된 응답 포맷
    @RestControllerAdvice
    public class ApiGatewayExceptionHandler {
        
        @ExceptionHandler(ServiceException.class)
        public ResponseEntity<ApiErrorResponse> handleServiceException(
                ServiceException e) {
            return ResponseEntity.status(e.getStatusCode())
                .body(new ApiErrorResponse(
                    e.getErrorCode(),
                    e.getMessage(),
                    e.getDetails()
                ));
        }
    }
    
    // 4. 버전 관리
    @RestController
    @RequestMapping("/api/v1/orders")
    public class PublicOrderController {
        
        @GetMapping("/{id}")
        @ApiOperation("주문 조회 (v1.0)")
        public ResponseEntity<OrderApiV1> getOrder(
                @PathVariable Long id,
                @RequestHeader(value = "Accept", 
                    defaultValue = "application/json") String accept) {
            // Composition 로직
            Order order = orderService.getOrder(id);
            User user = userService.getUser(order.getUserId());
            
            // 공개 API용 응답 (민감 정보 제거)
            return ResponseEntity.ok(new OrderApiV1(
                order.getId(),
                order.getTotalAmount(),
                order.getStatus(),
                user.getName(),
                user.getEmail()
                // 주소, 결제 정보 제외
            ));
        }
    }
}

// API 문서화 (OpenAPI)
@Configuration
public class OpenApiConfig {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("공개 REST API")
                .version("1.0.0")
                .description("Order Management API"))
            .servers(Arrays.asList(
                new Server().url("https://api.example.com")
                    .description("프로덕션"),
                new Server().url("https://sandbox.example.com")
                    .description("테스트")
            ));
    }
}
```

**다른 패턴은 부적절한 이유:**

- BFF: 특정 클라이언트 최적화 (공개는 다양)
- Client-Side: 보안 노출 (인증 정보 노출)

</details>

**Q3.** 응답 시간이 중요한 상황에서 부분 응답 vs 캐시 fallback 중 어느 것이 낫나?

<details>
<summary>해설 보기</summary>

**시나리오: User Service가 느림 (5초)**

```
요구사항: 응답 시간 1초 이내

❌ 부분 응답 (user = null)
응답 속도: 200ms (빠름)
데이터: 주문 정보는 있음, 사용자는 없음
UI: "주문자 정보 로드 중" 표시

❌ 캐시 fallback
응답 속도: 200ms (빠름)
데이터: 주문 정보 + 사용자 정보 (30분 전 버전)
UI: 최신인 줄 알고 표시

선택: 컨텍스트에 따라 다름
```

**데이터 특성별 선택:**

```
1. 자주 변하지 않는 정보 (사용자 프로필)
   → 캐시 fallback (오래된 데이터도 괜찮음)
   
   구현:
   User user = null;
   try {
       user = userService.getUser(userId);
   } catch (TimeoutException) {
       user = cache.get(userId);  // 30분 전 데이터
   }

2. 실시간이어야 하는 정보 (배송 상태)
   → 부분 응답 (null 허용)
   
   구현:
   ShippingStatus status = null;
   try {
       status = shippingService.getStatus(orderId);
   } catch (TimeoutException) {
       // null 반환 → UI에서 "조회 중" 표시
   }

3. 핵심 정보 (주문 정보)
   → 동기 필수 (타임아웃 길게)
   
   구현:
   Order order = orderService.getOrder(orderId);
   // 5초 타임아웃, 실패하면 에러 반환
```

**최적 전략: Fallback Chain**

```java
@Service
public class ResilientOrderService {
    
    public OrderDetail getOrderDetail(Long orderId) {
        Order order = orderService.getOrder(orderId);
        
        // User: 캐시 fallback (자주 안 변함)
        User user = getWithCacheFallback(
            () -> userService.getUser(order.getUserId()),
            Duration.ofSeconds(3),
            Duration.ofMinutes(30));
        
        // Shipping: 부분 응답 (실시간 중요)
        ShippingStatus shipping = getWithTimeout(
            () -> shippingService.getStatus(orderId),
            Duration.ofSeconds(2));  // null 허용
        
        // Payment: 동기 필수 (중요)
        PaymentInfo payment = paymentService.getPayment(
            order.getPaymentId());  // 타임아웃 길게
        
        return new OrderDetail(order, user, shipping, payment);
    }
    
    private <T> T getWithCacheFallback(
            Supplier<T> supplier,
            Duration timeout,
            Duration cacheTTL) {
        try {
            return supplier.get();  // 최신 데이터 시도
        } catch (TimeoutException e) {
            // 캐시에서 가져오기
            return cache.getOrElse(key, () -> {
                logger.warn("캐시도 없음, null 반환");
                return null;
            });
        }
    }
    
    private <T> T getWithTimeout(
            Supplier<T> supplier,
            Duration timeout) {
        try {
            return supplier.get();
        } catch (TimeoutException e) {
            logger.warn("타임아웃, null 반환");
            return null;  // 부분 응답
        }
    }
}
```

**결론: 데이터의 "실시간성" 요구도에 따라 결정**

- High: 부분 응답 (null 허용, 실시간 업데이트)
- Medium: 캐시 fallback (약간 오래된 데이터)
- Low: 동기 호출 (일관성 우선)

</details>

---

<div align="center">

**[⬅️ 이전: 이벤트 기반 통신 — Kafka와 결합도 분리](./04-event-driven-communication.md)** | **[홈으로 🏠](../README.md)** | **[다음: Service Mesh — Envoy Sidecar와 mTLS ➡️](./06-service-mesh.md)**

</div>

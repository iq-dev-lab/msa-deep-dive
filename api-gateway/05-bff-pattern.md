# 05. BFF 패턴 — 클라이언트별 최적화 API 레이어

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 왜 웹, 모바일, 서드파티 클라이언트가 같은 API를 사용할 수 없는가?
- BFF(Backend For Frontend)가 API Gateway와 어떻게 다른가?
- 모바일 BFF와 웹 BFF를 분리해야 하는 이유는?
- GraphQL을 BFF로 사용했을 때의 이점과 주의점은?
- BFF 팀 소유권 문제를 어떻게 해결할 수 있는가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

MSA 환경에서 마이크로서비스는 자신의 도메인에만 집중하여 설계됩니다. 예를 들어:

- **user-service**: 사용자 정보만 제공
- **order-service**: 주문 정보만 제공
- **payment-service**: 결제 정보만 제공

하지만 클라이언트는 이들을 **다양한 방식으로 조합**하여 화면을 구성합니다:

```
웹 대시보드: [사용자 정보] + [최근 주문 5개] + [결제 예정금액]
모바일 앱:  [사용자 이름] + [주문 상태] (간단하게)
관리자 CLI:  [모든 데이터] (상세하게)
서드파티 API: [공개 데이터만]
```

각 클라이언트가 **공통 API (모놀리식)**에 의존하면:
1. **언더페칭**: 필요한 데이터를 받으려고 10개 엔드포인트 호출
2. **오버페칭**: 불필요한 데이터까지 받아 네트워크 낭비
3. **느린 응답**: 느린 서비스 하나가 전체 응답 지연

이를 해결하는 것이 **BFF(Backend For Frontend)**입니다.

---

## 😱 흔한 실수 (Before)

### 오류 1: 모든 클라이언트를 위한 공통 API

```java
// UserController.java - 모든 정보를 전부 반환
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {
    
    private final UserService userService;
    private final OrderService orderService;
    
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable String id) {
        User user = userService.getUserById(id);
        
        // ❌ 모든 필드를 항상 포함
        UserResponse response = UserResponse.builder()
                .id(user.getId())
                .email(user.getEmail())
                .name(user.getName())
                .phone(user.getPhone())
                .address(user.getAddress())
                .city(user.getCity())
                .zipCode(user.getZipCode())
                .country(user.getCountry())
                .birthDate(user.getBirthDate())
                .joinDate(user.getJoinDate())
                .lastLoginDate(user.getLastLoginDate())
                .preferredLanguage(user.getPreferredLanguage())
                .marketingConsent(user.isMarketingConsent())
                .recentOrders(orderService.getRecentOrders(id, 100))  // ❌ 항상 100개
                .allInvoices(userService.getAllInvoices(id))          // ❌ 모든 송장
                .payments(userService.getPaymentHistory(id))          // ❌ 전체 결제 이력
                .build();
        
        return ResponseEntity.ok(response);
    }
}
```

**응답 예시:**
```json
{
  "id": "user123",
  "email": "user@example.com",
  "name": "홍길동",
  "phone": "010-1234-5678",
  "address": "서울시 강남구...",
  // ... 50개 필드
  "recentOrders": [... 100개 주문],
  "allInvoices": [... 1000개 송장],
  "payments": [... 500개 결제 기록]
}
```

**문제점:**
```
모바일 앱: [이름] 하나만 필요 → 너무 큰 응답 (1MB+)
웹사이트: [이름, 최근 5개 주문] 필요 → 100개 주문 모두 다운로드
CLI 관리자: [모든 데이터] → 그나마 맞음

→ 네트워크 낭비, 느린 응답, 불필요한 데이터베이스 쿼리
```

### 오류 2: 클라이언트가 여러 서비스 직접 호출

```typescript
// mobile-app.ts - ❌ 클라이언트가 여러 서비스 조율
async function getUserDashboard(userId: string) {
    const [userResponse, ordersResponse, paymentsResponse] = 
        await Promise.all([
            fetch(`http://user-service/api/users/${userId}`),
            fetch(`http://order-service/api/users/${userId}/orders`),
            fetch(`http://payment-service/api/users/${userId}/payments`)
        ]);
    
    // ❌ 클라이언트가 데이터 조합
    const user = await userResponse.json();
    const orders = await ordersResponse.json();
    const payments = await paymentsResponse.json();
    
    // ❌ 클라이언트에서 필터링
    const recentOrders = orders.slice(0, 5);
    const thisMonthPayments = payments.filter(p => 
        p.date.getMonth() === new Date().getMonth()
    );
    
    return {
        name: user.name,
        recentOrders,
        thisMonthPayments
    };
}
```

**문제점:**
- 클라이언트가 내부 마이크로서비스 구조를 알아야 함
- API 변경 → 모든 클라이언트 업데이트
- 네트워크 왕복 3번 (지연 누적)
- 클라이언트 로직 복잡 (비즈니스 로직 중복)

---

## ✨ 올바른 접근 (After)

### BFF 아키텍처

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Web App     │  │ Mobile App   │  │ Admin CLI    │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                  │
       │                 │                  │
       ▼                 ▼                  ▼
┌─────────────┐  ┌──────────────┐  ┌──────────────┐
│ Web BFF     │  │ Mobile BFF   │  │ Admin BFF    │
│             │  │              │  │              │
│ - 풍부한    │  │ - 최소 데이터│  │ - 모든 데이터│
│   응답      │  │ - 빠른 응답  │  │ - 상세 정보  │
│ - GraphQL   │  │ - REST       │  │ - REST      │
└──────┬──────┘  └──────┬───────┘  └──────┬───────┘
       │                │                 │
       └────────────────┼─────────────────┘
                        │
                        ▼
        ┌───────────────────────────────┐
        │ API Gateway                   │
        │ (인증, Rate Limiting, 라우팅) │
        └───────────┬───────────────────┘
                    │
        ┌───────────┼───────────┬──────────────┐
        │           │           │              │
        ▼           ▼           ▼              ▼
   ┌─────────┐ ┌────────┐ ┌──────────┐ ┌─────────┐
   │ User    │ │ Order  │ │ Payment  │ │Inventory│
   │Service  │ │Service │ │Service   │ │Service  │
   └─────────┘ └────────┘ └──────────┘ └─────────┘
```

### Web BFF 구현 (GraphQL)

```java
// GraphQL Schema (WebBffService)
// GraphQL은 클라이언트가 필요한 필드만 요청 가능

@Component
public class UserGraphQLQueryResolver implements GraphQLQueryResolver {
    
    private final UserServiceClient userClient;
    private final OrderServiceClient orderClient;
    
    /**
     * Query: {
     *   user(id: "user123") {
     *     id
     *     name
     *     recentOrders(limit: 5) {
     *       id
     *       total
     *     }
     *   }
     * }
     */
    public User user(String id) {
        return userClient.getUser(id);
    }
}

@Component
public class UserGraphQLResolver implements GraphQLResolver<User> {
    
    private final OrderServiceClient orderClient;
    
    /**
     * 지연 로딩: User 객체가 요청될 때만 주문 조회
     */
    public List<Order> recentOrders(User user, 
                                    @GraphQLArgument(name = "limit", 
                                                     defaultValue = "10") int limit) {
        return orderClient.getUserOrders(user.getId(), limit);
    }
}

@RestController
@RequestMapping("/graphql")
@RequiredArgsConstructor
public class GraphQLController {
    
    private final GraphQLProvider graphQLProvider;
    
    @PostMapping
    public ResponseEntity<Map<String, Object>> graphql(
            @RequestBody GraphQLRequest request) {
        
        ExecutionInput executionInput = ExecutionInput.newExecutionInput()
                .query(request.getQuery())
                .variables(request.getVariables())
                .build();
        
        ExecutionResult result = graphQLProvider.getGraphQL()
                .execute(executionInput);
        
        return ResponseEntity.ok(result.toSpecification());
    }
}
```

**GraphQL 사용 예시:**

```graphql
# Web App이 필요한 데이터만 요청
query GetUserDashboard($id: String!) {
  user(id: $id) {
    id
    name
    email
    recentOrders(limit: 5) {
      id
      total
      createdAt
    }
  }
}
```

**응답:**
```json
{
  "data": {
    "user": {
      "id": "user123",
      "name": "홍길동",
      "email": "hong@example.com",
      "recentOrders": [
        {"id": "order1", "total": 50000, "createdAt": "2026-04-10"},
        {"id": "order2", "total": 30000, "createdAt": "2026-04-09"}
      ]
    }
  }
}
```

### Mobile BFF 구현 (REST, 최소 데이터)

```java
// MobileBffController.java - 모바일 최적화
@RestController
@RequestMapping("/api/v1/mobile")
@RequiredArgsConstructor
public class MobileBffController {
    
    private final UserServiceClient userClient;
    private final OrderServiceClient orderClient;
    
    /**
     * 모바일: 이름과 최근 주문 상태만
     * 응답 크기: 5KB
     */
    @GetMapping("/users/{id}")
    public ResponseEntity<MobileUserResponse> getUser(@PathVariable String id) {
        User user = userClient.getUser(id);
        List<Order> orders = orderClient.getUserOrders(id, 3);
        
        MobileUserResponse response = MobileUserResponse.builder()
                .name(user.getName())
                .avatar(user.getAvatarUrl())
                .recentOrders(orders.stream()
                        .map(order -> new MobileOrderSummary(
                                order.getId(),
                                order.getStatus()
                        ))
                        .collect(Collectors.toList()))
                .build();
        
        return ResponseEntity.ok(response)
                .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES))
                .build();
    }
    
    @Data
    @AllArgsConstructor
    public static class MobileUserResponse {
        private String name;
        private String avatar;
        private List<MobileOrderSummary> recentOrders;
    }
    
    @Data
    public static class MobileOrderSummary {
        private String id;
        private String status;  // "PENDING", "SHIPPED", "DELIVERED"
    }
}
```

**모바일 응답 예시:**
```json
{
  "name": "홍길동",
  "avatar": "https://...",
  "recentOrders": [
    {"id": "order1", "status": "SHIPPED"},
    {"id": "order2", "status": "DELIVERED"},
    {"id": "order3", "status": "PENDING"}
  ]
}
```

### 병렬 요청으로 응답 시간 최적화

```java
// BFFService.java - 마이크로서비스 병렬 호출
@Service
@RequiredArgsConstructor
public class BFFService {
    
    private final UserServiceClient userClient;
    private final OrderServiceClient orderClient;
    private final PaymentServiceClient paymentClient;
    
    /**
     * 3개 서비스를 병렬로 호출
     * 총 시간: max(300ms, 200ms, 150ms) = 300ms
     * (순차: 650ms)
     */
    public Mono<DashboardResponse> getUserDashboard(String userId) {
        // 3개 호출을 동시에 실행
        Mono<User> userMono = userClient.getUser(userId);
        Mono<List<Order>> ordersMono = orderClient.getRecentOrders(userId, 5);
        Mono<PaymentInfo> paymentMono = paymentClient.getPaymentInfo(userId);
        
        return Mono.zip(userMono, ordersMono, paymentMono)
                .map(tuple -> {
                    User user = tuple.getT1();
                    List<Order> orders = tuple.getT2();
                    PaymentInfo payment = tuple.getT3();
                    
                    return DashboardResponse.builder()
                            .userName(user.getName())
                            .recentOrders(orders)
                            .outstandingAmount(payment.getOutstanding())
                            .build();
                })
                .timeout(Duration.ofSeconds(5))  // 타임아웃 설정
                .onErrorResume(ex -> 
                    // 하나가 실패해도 부분 응답 반환
                    Mono.just(createPartialResponse(userId, ex))
                );
    }
    
    private DashboardResponse createPartialResponse(String userId, Throwable error) {
        return DashboardResponse.builder()
                .error("부분 데이터만 로드됨: " + error.getMessage())
                .build();
    }
}
```

---

## 🔬 내부 동작 원리 — BFF의 데이터 최적화

### 언더페칭 vs 오버페칭

```
시나리오: 모바일 앱이 사용자 이름과 최근 주문 3개만 필요

[공통 API 방식 - 오버페칭]
GET /api/users/user123
┌─────────────────────────────────────┐
│ 응답 크기: 2MB                      │
│ - User 정보: 50개 필드              │
│ - 모든 주문: 1000개                 │
│ - 모든 결제: 500개                  │
│                                     │
│ 필요한 것: [이름, 최근 3개 주문]   │
│ 낭비된 것: 2MB - 5KB = 약 99.75%   │
└─────────────────────────────────────┘

[다중 요청 방식 - 언더페칭 + 네트워크 왕복]
GET /api/users/user123
  → 응답: 1MB (여전히 많음)
  → Wait: 100ms

GET /api/users/user123/orders
  → 응답: 500KB
  → Wait: 100ms

총 시간: 200ms
총 용량: 1.5MB


[BFF 방식 - 최적화]
GET /bff/mobile/users/user123
┌─────────────────────────────────────┐
│ 응답 크기: 5KB                      │
│ {                                   │
│   "name": "홍길동",                │
│   "recentOrders": [                │
│     {"id": "o1", "status": "..."}  │
│   ]                                 │
│ }                                   │
└─────────────────────────────────────┘

시간: 150ms (병렬 처리)
용량: 5KB (400배 감소!)
```

### GraphQL의 선택적 필드 조회

```graphql
# 예제 1: 최소 데이터 (모바일)
query MobileUser {
  user(id: "user123") {
    name
  }
}
응답 크기: 100 bytes
네트워크 시간: ~30ms

# 예제 2: 중간 데이터 (웹)
query WebUser {
  user(id: "user123") {
    id
    name
    email
    recentOrders(limit: 5) {
      id
      total
    }
  }
}
응답 크기: 1KB
네트워크 시간: ~60ms

# 예제 3: 전체 데이터 (관리자)
query AdminUser {
  user(id: "user123") {
    id
    name
    email
    phone
    address
    allOrders {
      id
      total
      items {
        name
        price
      }
    }
    payments {
      id
      amount
      status
    }
  }
}
응답 크기: 500KB
네트워크 시간: ~300ms

→ 같은 endpoint이지만 응답 크기가 5000배 차이!
```

### BFF 팀 소유권 모델

```
┌──────────────────────────────────────────────────────┐
│ 전통적 모델: 중앙 BFF 팀 (병목)                    │
├──────────────────────────────────────────────────────┤
│                                                      │
│ [Web 팀]     [Mobile 팀]    [Admin 팀]             │
│     ↓             ↓              ↓                  │
│  ┌────────────────────────────────────────┐        │
│  │    중앙 BFF 팀                         │        │
│  │ (모든 BFF 개발)                        │        │
│  │                                        │        │
│  │ 문제: 병목 → 느린 배포 → 중복 개발   │        │
│  └────────────────────────────────────────┘        │
│     ↓                                               │
│  마이크로서비스                                     │
│                                                      │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│ 개선 모델: Frontend 팀이 자신의 BFF 소유 (권장)   │
├──────────────────────────────────────────────────────┤
│                                                      │
│ [Web 팀]     [Mobile 팀]    [Admin 팀]             │
│ ┌────────┐   ┌────────┐    ┌────────┐              │
│ │Web BFF │   │Mobile  │    │Admin   │              │
│ │(Web팀) │   │BFF     │    │BFF     │              │
│ │        │   │(Mobile)│    │(Admin) │              │
│ │GraphQL │   │REST    │    │REST    │              │
│ └────┬───┘   └───┬────┘    └───┬────┘              │
│      │           │             │                   │
│      └───────────┼─────────────┘                   │
│                  │                                 │
│                  ▼                                 │
│           API Gateway                             │
│                  │                                 │
│       ┌──────────┼──────────┐                     │
│       ▼          ▼          ▼                     │
│   User Svc   Order Svc  Payment Svc              │
│                                                   │
│ 장점: 빠른 개발, 독립 배포, 팀별 최적화         │
│                                                   │
└──────────────────────────────────────────────────┘
```

---

## 💻 실제 BFF 구현 코드

```java
// WebBffService.java - 복잡한 데이터 조합
@Service
@RequiredArgsConstructor
public class WebBffService {
    
    private final UserServiceClient userClient;
    private final OrderServiceClient orderClient;
    private final PaymentServiceClient paymentClient;
    private final InventoryServiceClient inventoryClient;
    
    /**
     * 웹에 최적화된 대시보드 데이터
     */
    public Mono<WebDashboardDTO> getDashboard(String userId) {
        // 병렬로 4개 서비스 호출
        Mono<User> userMono = userClient.getUser(userId);
        Mono<List<OrderDTO>> ordersMono = orderClient.getUserOrders(userId, 10);
        Mono<PaymentStats> statsMono = paymentClient.getPaymentStats(userId);
        Mono<List<Recommendation>> recsMono = inventoryClient
                .getRecommendations(userId);
        
        return Mono.zip(userMono, ordersMono, statsMono, recsMono)
                .map(tuple -> {
                    User user = tuple.getT1();
                    List<OrderDTO> orders = tuple.getT2();
                    PaymentStats stats = tuple.getT3();
                    List<Recommendation> recs = tuple.getT4();
                    
                    // 데이터 변환 및 조합
                    return WebDashboardDTO.builder()
                            .greeting(buildGreeting(user))
                            .orderStats(buildOrderStats(orders))
                            .pendingOrders(orders.stream()
                                    .filter(o -> o.getStatus() == OrderStatus.PENDING)
                                    .map(o -> WebOrderSummary.from(o))
                                    .collect(Collectors.toList()))
                            .paymentDue(stats.getDueAmount())
                            .recommendations(recs)
                            .build();
                })
                .cache(Duration.ofSeconds(5));  // 응답 캐싱
    }
    
    private String buildGreeting(User user) {
        LocalDateTime now = LocalDateTime.now();
        int hour = now.getHour();
        
        if (hour < 12) return "좋은 아침, " + user.getName() + "님!";
        else if (hour < 18) return "좋은 오후, " + user.getName() + "님!";
        else return "좋은 저녁, " + user.getName() + "님!";
    }
    
    private OrderStatsDTO buildOrderStats(List<OrderDTO> orders) {
        return OrderStatsDTO.builder()
                .totalOrders(orders.size())
                .totalSpent(orders.stream()
                        .mapToDouble(OrderDTO::getTotal)
                        .sum())
                .averageOrderValue(orders.stream()
                        .mapToDouble(OrderDTO::getTotal)
                        .average()
                        .orElse(0))
                .build();
    }
}

// MobileBffService.java - 간단한 데이터
@Service
@RequiredArgsConstructor
public class MobileBffService {
    
    private final UserServiceClient userClient;
    private final OrderServiceClient orderClient;
    
    /**
     * 모바일에 최적화된 간단한 대시보드
     */
    public Mono<MobileDashboardDTO> getDashboard(String userId) {
        // 2개만 호출 (최소 데이터)
        Mono<User> userMono = userClient.getUser(userId);
        Mono<List<OrderDTO>> ordersMono = orderClient.getUserOrders(userId, 3);
        
        return Mono.zip(userMono, ordersMono)
                .map(tuple -> {
                    User user = tuple.getT1();
                    List<OrderDTO> orders = tuple.getT2();
                    
                    return MobileDashboardDTO.builder()
                            .userName(user.getName())
                            .recentOrders(orders.stream()
                                    .map(o -> MobileOrderDTO.builder()
                                            .id(o.getId())
                                            .status(o.getStatus().name())
                                            .build())
                                    .collect(Collectors.toList()))
                            .build();
                })
                .timeout(Duration.ofSeconds(3))
                .onErrorResume(ex -> 
                    createFallbackResponse(userId)
                );
    }
    
    private Mono<MobileDashboardDTO> createFallbackResponse(String userId) {
        // 네트워크 오류 시 간단한 캐시된 응답 반환
        return userClient.getUserFromCache(userId)
                .map(user -> MobileDashboardDTO.builder()
                        .userName(user.getName())
                        .recentOrders(Collections.emptyList())
                        .build());
    }
}
```

---

## 📊 패턴 비교

| 항목 | 단일 API | 클라이언트 조율 | BFF (REST) | BFF (GraphQL) |
|------|:---:|:---:|:---:|:---:|
| **응답 크기** | 매우 큼 (MB) | 중간 (KB~MB) | 작음 (KB) | 매우 작음 (B~KB) |
| **네트워크 왕복** | 1회 | N회 (대기 누적) | 1회 | 1회 |
| **응답 시간** | 느림 | 매우 느림 | 빠름 | 빠름 |
| **클라이언트 로직** | 복잡함 | 매우 복잡함 | 간단함 | 매우 간단함 |
| **개발 복잡도** | 낮음 | 매우 높음 | 중간 | 높음 (스키마 정의) |
| **캐싱** | 어려움 | 부분적 | 쉬움 | 쉬움 |
| **API 변경 영향** | 모든 클라이언트 | 모든 클라이언트 | 독립적 | 독립적 |
| **모바일 최적화** | ❌ | ❌ | ✅ | ✅ |
| **버전 관리** | 복잡함 | 매우 복잡함 | 간단함 | 간단함 |
| **TypeScript 지원** | 수동 | 수동 | 수동 | 자동 (코드 생성) |

---

## ⚖️ 트레이드오프

### BFF (REST) 장점

✅ **구현 간단**: 일반적인 REST API 패턴
- 대부분의 팀이 이미 알고 있음
- Spring Boot @RestController 활용

✅ **캐싱 용이**: HTTP 캐싱 헤더 활용
```java
return ResponseEntity.ok(dto)
    .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES))
    .eTag("v1")
    .build();
```

✅ **모니터링 쉬움**: 엔드포인트별 지표 추적 가능
```
GET /bff/web/dashboard → 평균 150ms
GET /bff/mobile/dashboard → 평균 80ms
```

### BFF (REST) 단점

❌ **여전히 언더페칭**: 클라이언트가 여러 엔드포인트 호출 필요 가능
- `/bff/users/{id}`
- `/bff/users/{id}/orders`
- `/bff/users/{id}/payments`
→ 3회 왕복

❌ **버전 관리**: 변경될 때마다 `/v1`, `/v2` 새 엔드포인트 필요

### BFF (GraphQL) 장점

✅ **완벽한 필드 선택**: 클라이언트가 정확히 필요한 것만 요청
```graphql
query {
  user(id: "user123") {
    name
    # email, phone 등은 요청하지 않음
  }
}
```

✅ **단일 엔드포인트**: `/graphql` 하나만 필요
- 버전 관리 불필요
- 필드 추가 → 자동으로 사용 가능

✅ **자동 문서화**: GraphQL Schema가 자체 문서
```graphql
{
  __type(name: "User") {
    fields {
      name
      description
      type { ... }
    }
  }
}
```

### BFF (GraphQL) 단점

❌ **높은 학습곡선**: GraphQL 쿼리 언어 학습 필요
- 팀의 역량에 따라 도입 시간 증가
- 레거시 프로젝트에서 도입 어려움

❌ **N+1 쿼리 문제**: 부주의하면 성능 저하
```graphql
query {
  users {
    id
    name
    orders {  # ← 각 user마다 별도 쿼리 발생!
      id
    }
  }
}
```

❌ **복잡한 캐싱**: GraphQL 쿼리는 모두 다를 수 있음
```
GET /api/users/123 → 캐시 가능 (URL 동일)
POST /graphql (query: "{ user { name } }") → 매번 다른 쿼리
→ 캐싱 어려움
```

---

## 📌 핵심 정리

✅ **BFF는 클라이언트별 맞춤 API 레이어**
- 웹: 풍부한 데이터 (10KB+)
- 모바일: 최소 데이터 (1KB)
- 관리자: 모든 데이터 (500KB)

✅ **언더페칭 vs 오버페칭 문제 해결**
- 단일 API: 모든 필드 반환 → 오버페칭
- 다중 호출: 여러 번 요청 → 언더페칭 + 지연
- BFF: 정확한 데이터만 → 최적화

✅ **병렬 요청으로 응답 시간 단축**
```java
// 순차: 300ms + 200ms + 150ms = 650ms
// 병렬: max(300, 200, 150) = 300ms
Mono.zip(userMono, ordersMono, paymentMono)
```

✅ **팀 소유권 모델 중요**
```
중앙 BFF팀 (병목) ❌
→ Frontend 팀이 자신의 BFF 소유 ✅
```

✅ **GraphQL vs REST 선택**
- 프로토타입: REST (빠른 구현)
- 프로덕션: GraphQL (유연성, 성능)
- 하이브리드: REST + GraphQL (점진적 마이그레이션)

---

## 🤔 생각해볼 문제

**Q1.** 웹 BFF가 user-service를 호출할 때 timeout이 발생했어요. 사용자에게는 어떤 응답을 줄까요?

<details><summary>해설 보기</summary>

**상황:**
```
[Web BFF] → [User Service]
  30초 대기 중
  ("Hello?")
  
응답 없음 (장애)
↓
timeout 발생
```

**선택지:**

1. **에러 응답 (나쁨)**
```java
try {
    User user = userClient.getUser(userId);  // timeout!
} catch (TimeoutException e) {
    return ResponseEntity.status(500)
            .body("{\"error\": \"Service unavailable\"}");
}

사용자 경험: "대시보드 로드 실패" → 화면 못 봄
```

2. **부분 응답 (좋음)**
```java
public Mono<DashboardDTO> getDashboard(String userId) {
    Mono<User> userMono = userClient.getUser(userId)
            .timeout(Duration.ofSeconds(2))
            .onErrorResume(ex -> 
                // 캐시된 사용자 정보 반환
                getUserFromCache(userId)
            );
    
    Mono<List<Order>> ordersMono = orderClient.getRecentOrders(userId, 5)
            .timeout(Duration.ofSeconds(2))
            .onErrorResume(ex -> 
                Mono.just(Collections.emptyList())
            );
    
    return Mono.zip(userMono, ordersMono)
            .map(tuple -> {
                User user = tuple.getT1();  // 캐시된 것
                List<Order> orders = tuple.getT2();  // 빈 목록
                
                return DashboardDTO.builder()
                        .user(user)
                        .orders(orders)
                        .warning("주문 정보 로드 실패")  // 사용자에 알림
                        .build();
            });
}

응답:
{
  "user": {
    "name": "홍길동",  // 캐시된 정보
    "avatar": "..."
  },
  "orders": [],
  "warning": "주문 정보를 로드할 수 없습니다"
}

사용자 경험: 사용자 정보는 보이고, 주문 정보는 재시도 표시
```

3. **우아한 저하 (최선)**
```java
public Mono<DashboardDTO> getDashboard(String userId) {
    Mono<User> userMono = userClient.getUser(userId)
            .timeout(Duration.ofSeconds(2))
            .onErrorResume(ex -> Mono.just(
                new User(userId, "사용자", null)  // 최소 정보
            ));
    
    Mono<List<Order>> ordersMono = orderClient.getRecentOrders(userId, 5)
            .timeout(Duration.ofSeconds(2))
            .onErrorResume(ex -> Mono.just(Collections.emptyList()))
            .delaySubscription(Duration.ofMillis(100))  // 재시도 지연
            .retry(1);  // 1회 재시도
    
    return Mono.zip(userMono, ordersMono)
            .map(tuple -> DashboardDTO.from(tuple.getT1(), tuple.getT2()))
            .timeout(Duration.ofSeconds(5))
            .cache(Duration.ofSeconds(10));  // 응답 캐싱
}
```

**권장:**
- 중요한 데이터 (사용자 정보): 캐시 + 재시도
- 보조 데이터 (주문): 타임아웃 후 빈 목록
- 항상 부분 응답이라도 뭔가 보여주기
</details>

**Q2.** Mobile BFF와 Web BFF를 분리하면 코드 중복이 많아질 텐데, 이를 어떻게 해결할까요?

<details><summary>해설 보기</summary>

**문제:**

```java
// WebBff
public DashboardDTO getWebDashboard(String userId) {
    User user = userService.getUser(userId);
    List<Order> orders = orderService.getOrders(userId);
    return DashboardDTO.from(user, orders);  // 공통 로직
}

// MobileBff
public MobileDashboardDTO getMobileDashboard(String userId) {
    User user = userService.getUser(userId);  // 중복!
    List<Order> orders = orderService.getOrders(userId, 3);  // 중복!
    return MobileDashboardDTO.from(user, orders);
}
```

**해결 방법:**

1. **공통 비즈니스 로직 추출 (권장)**
```java
// BffCoreService - 공통 로직
@Service
public class BffCoreService {
    private final UserServiceClient userClient;
    private final OrderServiceClient orderClient;
    
    // 공통: 사용자 데이터 로드
    public Mono<User> loadUser(String userId) {
        return userClient.getUser(userId)
                .timeout(Duration.ofSeconds(2))
                .cache(Duration.ofSeconds(5));
    }
    
    // 공통: 주문 데이터 로드
    public Mono<List<Order>> loadOrders(String userId, int limit) {
        return orderClient.getOrders(userId, limit)
                .timeout(Duration.ofSeconds(2))
                .cache(Duration.ofSeconds(1));
    }
}

// WebBffController
@RestController
@RequestMapping("/bff/web")
public class WebBffController {
    private final BffCoreService coreService;
    
    @GetMapping("/dashboard/{userId}")
    public Mono<WebDashboardDTO> getDashboard(@PathVariable String userId) {
        return Mono.zip(
                coreService.loadUser(userId),
                coreService.loadOrders(userId, 10)
            )
            .map(tuple -> WebDashboardDTO.from(tuple.getT1(), tuple.getT2()));
    }
}

// MobileBffController
@RestController
@RequestMapping("/bff/mobile")
public class MobileBffController {
    private final BffCoreService coreService;
    
    @GetMapping("/dashboard/{userId}")
    public Mono<MobileDashboardDTO> getDashboard(@PathVariable String userId) {
        return Mono.zip(
                coreService.loadUser(userId),
                coreService.loadOrders(userId, 3)
            )
            .map(tuple -> MobileDashboardDTO.from(tuple.getT1(), tuple.getT2()));
    }
}
```

2. **DTO 변환 로직 추출**
```java
// DTOConverter - 변환 로직 통합
@Component
public class DTOConverter {
    
    public WebDashboardDTO toWebDashboard(User user, List<Order> orders) {
        return WebDashboardDTO.builder()
                .userName(user.getName())
                .recentOrders(orders)
                .build();
    }
    
    public MobileDashboardDTO toMobileDashboard(User user, List<Order> orders) {
        return MobileDashboardDTO.builder()
                .name(user.getName())
                .orders(orders.stream()
                        .map(o -> new MobileOrder(o.getId(), o.getStatus()))
                        .collect(Collectors.toList()))
                .build();
    }
}
```

3. **Shared 라이브러리 (마이크로패키지)**
```
project/
├── bff-core/          # 공유 라이브러리
│   ├── pom.xml
│   ├── DashboardDTO
│   └── BffCoreService
├── bff-web/           # Web BFF만
│   ├── WebDashboardDTO
│   └── WebBffController
└── bff-mobile/        # Mobile BFF만
    ├── MobileDashboardDTO
    └── MobileBffController

Maven pom.xml:
<dependency>
    <groupId>com.example</groupId>
    <artifactId>bff-core</artifactId>
    <version>1.0.0</version>
</dependency>
```

**권장: 공통 서비스 + 클라이언트별 DTO**
- 비즈니스 로직: 공유 (BffCoreService)
- 응답 형식: 분리 (WebDTO, MobileDTO)
- 각 팀이 독립적으로 개발 가능
</details>

**Q3.** GraphQL을 사용할 때 성능을 최적화하려면 어떻게 할까요?

<details><summary>해설 보기</summary>

**주요 성능 문제:**

1. **N+1 쿼리 문제**
```graphql
# 나쁜 예: N+1 문제
query {
  users {
    id
    name
    orders {  # 각 user마다 추가 쿼리!
      id
      total
    }
  }
}

실행:
SELECT * FROM users;  # 1 쿼리
SELECT * FROM orders WHERE user_id = 1;  # +1
SELECT * FROM orders WHERE user_id = 2;  # +1
SELECT * FROM orders WHERE user_id = 3;  # +1
...
총 N+1 쿼리 (느림!)
```

**해결책:**

2. **DataLoader로 배치 처리**
```java
// UserGraphQLResolver.java
@Component
public class UserGraphQLResolver implements GraphQLResolver<User> {
    
    private final OrderDataLoader orderDataLoader;
    
    // DataLoader가 자동으로 배치 처리
    public Mono<List<Order>> orders(User user, 
                                     DataFetchingEnvironment env) {
        DataLoader<String, List<Order>> loader = 
                env.getDataLoader("orders");
        
        return loader.load(user.getId());  // 배치됨!
    }
}

// OrderDataLoader.java
@Component
public class OrderDataLoader implements BatchLoaderWithContext<String, List<Order>> {
    
    private final OrderServiceClient orderClient;
    
    @Override
    public Mono<List<List<Order>>> loadManyBatch(
            List<String> userIds, 
            BatchLoaderEnvironment env) {
        
        // 한 번에 모든 사용자의 주문 조회
        return orderClient.getOrdersByUserIds(userIds)
                .map(ordersMap -> 
                    userIds.stream()
                            .map(userId -> ordersMap.getOrDefault(userId, 
                                    Collections.emptyList()))
                            .collect(Collectors.toList())
                );
    }
}

// GraphQLConfig.java
@Configuration
public class GraphQLConfig {
    
    @Bean
    public GraphQLBuilder graphQL(OrderDataLoader orderDataLoader) {
        return GraphQL.newGraphQL()
                .dataLoaders(
                    DataLoaderRegistry.newRegistry()
                            .register("orders", orderDataLoader)
                            .build()
                )
                .build();
    }
}

결과:
SELECT * FROM users;  # 1 쿼리
SELECT * FROM orders WHERE user_id IN (1,2,3,...);  # +1 (배치)
총 2 쿼리 (빠름!)
```

3. **쿼리 복잡도 제한**
```java
// QueryComplexityAnalyzer.java
@Component
public class QueryComplexityAnalyzer {
    
    @Bean
    public GraphQL graphQL() {
        return GraphQL.newGraphQL(schema)
                .queryExecutionStrategy(
                    new QueryComplexityAnalysisQueryExecutionStrategy()
                        .withMax(1000)  // 복잡도 >= 1000 거부
                )
                .build();
    }
}

// 복잡도 계산 규칙
query {
  users {  # 복잡도: 1
    id  # +0 (스칼라)
    orders {  # 1 * 10 (평균 10개 주문) = 10
      id  # +0
      items {  # 10 * 5 (평균 5개 품목) = 50
        name  # +0
      }
    }
  }
}
→ 총 복잡도: 1 + 10 + 50 = 61 (허용)
```

4. **쿼리 깊이 제한**
```java
// 나쁜 예: 깊은 중첩
query {
  user {
    orders {
      items {
        product {
          category {
            parent {
              parent {
                ...  # 무한 중첩 가능!
              }
            }
          }
        }
      }
    }
  }
}

해결:
MaxQueryDepthInstrumentationStrategy.MaxQueryDepth(5)
// 최대 5단계 깊이로 제한
```

5. **캐싱 전략**
```java
// 응답 캐싱
@PostMapping("/graphql")
public Mono<ResponseEntity<Map<String, Object>>> graphql(
        @RequestBody GraphQLRequest request) {
    
    String cacheKey = hashQuery(request.getQuery());
    
    return cache.getIfPresent(cacheKey)
            .switchIfEmpty(
                executeQuery(request)
                        .doOnNext(result -> cache.put(cacheKey, result))
            )
            .map(ResponseEntity::ok);
}

// 필드 레벨 캐싱
@GraphQLField
@Cacheable(value = "userOrders", key = "#userId")
public List<Order> orders(@GraphQLArgument String userId) {
    return orderService.getOrders(userId);
}
```

6. **요청 타임아웃**
```java
@PostMapping("/graphql")
public Mono<ResponseEntity<Map<String, Object>>> graphql(
        @RequestBody GraphQLRequest request) {
    
    return executeQuery(request)
            .timeout(Duration.ofSeconds(5))  // 5초 제한
            .onErrorResume(TimeoutException.class, ex -> 
                Mono.just(Map.of("errors", List.of(
                    Map.of("message", "Query timeout")
                )))
            );
}
```

**Best Practices:**
1. DataLoader로 배치 처리
2. 쿼리 복잡도 제한 (1000 이하)
3. 쿼리 깊이 제한 (5단계)
4. 응답 캐싱 (5분)
5. 타임아웃 설정 (5초)
6. 정기적 성능 모니터링 (APM)
</details>

---

<div align="center">

**[⬅️ 이전: 인증·인가 아키텍처 — JWT 오프로딩과 서비스 간 인증](./04-auth-architecture.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — 분산 추적 — Trace ID로 서비스 흐름 추적 ➡️](../observability/01-distributed-tracing.md)**

</div>

# 07. GraphQL Federation — 분산 스키마 통합

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Apollo Federation이 여러 서비스의 GraphQL 스키마를 하나의 Supergraph로 통합하는 원리는?
- Subgraph (각 서비스)가 자신의 타입을 소유하면서, 다른 서비스의 타입을 참조하는 방법은 (@key 지시자)?
- Federation의 Entity Resolution이 N+1 쿼리 문제를 야기할 수 있고, 어떻게 최적화할까?
- REST vs GraphQL vs gRPC를 어떤 기준으로 선택해야 할까?
- GraphQL Federation의 단점은? 복잡도 증가, 디버깅 어려움, 성능 이슈?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

마이크로서비스에서 **클라이언트가 여러 서비스의 데이터를 조합해야 할 때**, 다음과 같은 문제가 발생합니다:

1. **API Gateway**: 중앙에서 모든 서비스의 응답을 조합하면 병목이 됨
2. **REST + N호출**: 클라이언트가 여러 엔드포인트를 호출해야 함
3. **BFF**: 각 클라이언트별 백엔드를 만들어야 함 (코드 중복)
4. **gRPC**: 브라우저에서 직접 호출 불가능

**GraphQL Federation은 이 문제를 우아하게 해결합니다:**

- **분산 스키마**: 각 서비스가 자신의 GraphQL 스키마만 관리
- **자동 조합**: Apollo Gateway가 여러 Subgraph를 자동으로 연결
- **선언적**: @key, @requires 지시자로 서비스 간 관계 명시
- **타입 안정성**: 클라이언트는 하나의 통합 스키마로 쿼리
- **독립 배포**: 각 서비스가 독립적으로 배포 가능

예를 들어 주문 조회 시:

```graphql
query {
  order(id: "123") {
    id
    totalAmount
    user {              # User Service의 데이터
      name
      email
    }
    items {
      productName       # Product Service의 데이터
      quantity
    }
    shipping {          # Shipping Service의 데이터
      status
      estimatedDate
    }
  }
}
```

클라이언트는 **하나의 쿼리**로 3개 서비스의 데이터를 받습니다. 조합은 Apollo Gateway가 처리합니다.

---

## 😱 흔한 실수 (Before — 강한 결합 또는 N+1 호출)

```graphql
# ❌ 안티패턴 1: 모든 데이터를 한 서비스에서 관리
# Order Service가 User, Product 정보까지 중복 저장

type Query {
  order(id: ID!): Order
}

type Order {
  id: ID!
  totalAmount: Float!
  
  # ❌ Order Service가 User 정보 중복 저장
  userName: String
  userEmail: String
  
  # ❌ Order Service가 Product 정보 중복 저장
  productNames: [String]
  productPrices: [Float]
}

문제점:
1. User 변경 → Order에도 반영 필요
2. Product 가격 변경 → Order 히스토리까지 변경?
3. 데이터 불일치 위험
4. 서비스 간 강한 결합 (Order depends on User, Product)


# ❌ 안티패턴 2: 클라이언트가 여러 엔드포인트 호출
# User 정보 필요 → User Service 호출
# Product 정보 필요 → Product Service 호출

interface IUser {
  id: ID!
  name: String
  email: String
}

interface IProduct {
  id: ID!
  name: String
  price: Float
}

# User Service GraphQL
type User implements IUser {
  id: ID!
  name: String
  email: String
}

# Product Service GraphQL
type Product implements IProduct {
  id: ID!
  name: String
  price: Float
}

# Order Service GraphQL (Federated X)
type Order {
  id: ID!
  totalAmount: Float!
  userId: ID!              # 참조만 가능
  productIds: [ID!]!       # 참조만 가능
}

쿼리 시 문제:
query {
  order(id: "123") {
    id
    totalAmount
    userId              # 받음
    productIds          # 받음
  }
}

클라이언트는 추가로:
1. User Service에 userId 전송 → User 정보 조회
2. Product Service에 productIds 전송 → Product 정보 조회

결과: 3번의 HTTP 요청! (N+1 문제)


# ❌ 안티패턴 3: BFF의 코드 중복
# Mobile BFF
type Query {
  mobileOrder(id: ID!): MobileOrder
}

type MobileOrder {
  id: ID!
  total: Float
  sellerName: String
  productName: String
}

# Web BFF
type Query {
  webOrder(id: ID!): WebOrder
}

type WebOrder {
  id: ID!
  totalAmount: Float
  tax: Float
  shippingCost: Float
  sellerDetails: SellerDetails
  productDetails: ProductDetails
}

문제: 같은 Order 데이터를 위해 2개 BFF 구현 필요
```

---

## ✨ 올바른 접근 (After — GraphQL Federation)

```graphql
# ✅ Order Service Subgraph
extend schema
  @link(url: "https://specs.apollo.dev/federation/v2.0")

type Query {
  order(id: ID!): Order
}

type Order @key(fields: "id") {
  id: ID!
  totalAmount: Float!
  userId: ID!           # User Service 참조
  items: [OrderItem!]!
}

type OrderItem {
  productId: ID!        # Product Service 참조
  quantity: Int!
  unitPrice: Float!
}

---

# ✅ User Service Subgraph
extend schema
  @link(url: "https://specs.apollo.dev/federation/v2.0")

type Query {
  user(id: ID!): User
}

type User @key(fields: "id") {
  id: ID!
  name: String!
  email: String!
  phone: String
}

---

# ✅ Product Service Subgraph
extend schema
  @link(url: "https://specs.apollo.dev/federation/v2.0")

type Query {
  product(id: ID!): Product
}

type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!
  inventory: Int!
}

---

# ✅ Apollo Gateway (Composition)
# @key는 엔티티의 고유 식별자

# Order에서 User 참조:
extend type Order {
  user: User! @requires(fields: "userId")
}

extend type User {
  orders: [Order!]! @requires(fields: "id")
}

# Order에서 Product 참조:
extend type OrderItem {
  product: Product! @requires(fields: "productId")
}

# 결과: 통합 Supergraph
type Query {
  order(id: ID!): Order
  user(id: ID!): User
  product(id: ID!): Product
}

type Order @key(fields: "id") {
  id: ID!
  totalAmount: Float!
  user: User!              # 자동으로 User Service에서 조회
  items: [OrderItem!]!
}

type OrderItem {
  product: Product!        # 자동으로 Product Service에서 조회
  quantity: Int!
  unitPrice: Float!
}

type User @key(fields: "id") {
  id: ID!
  name: String!
  email: String!
  orders: [Order!]!        # 자동으로 Order Service에서 조회
}

type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!
  inventory: Int!
}
```

```java
// ✅ Java 구현 (Spring GraphQL)

// Order Service Subgraph
@Configuration
@EnableGraphQlrepositories
public class OrderServiceGraphQLConfig {
    
    @Bean
    public GraphQlSource graphQlSource(GraphQLSchema schema) {
        return GraphQlSource.builder(schema).build();
    }
}

@GraphQlController
public class OrderController {
    
    @Autowired private OrderService orderService;
    @Autowired private UserServiceClient userClient;
    
    @QueryMapping
    public Order order(@Argument Long id) {
        return orderService.getOrder(id);
    }
    
    // Reference resolver: Order에서 User로 확장
    @SchemaMapping(typeName = "Order", field = "user")
    public User user(Order order) {
        // Federation: order.userId로 User 조회
        return userClient.getUser(order.getUserId());
    }
}

// User Service Subgraph
@GraphQlController
public class UserController {
    
    @Autowired private UserService userService;
    
    @QueryMapping
    public User user(@Argument Long id) {
        return userService.getUser(id);
    }
    
    // Entity resolver: Federation이 호출
    @SchemaMapping(typeName = "User", field = "_entities")
    public List<User> resolveUserEntities(List<Map<String, Object>> references) {
        // @key(fields: "id")로 정의했으므로
        // references = [{"__typename": "User", "id": 1}, ...]
        List<Long> ids = references.stream()
            .map(r -> Long.parseLong((String) r.get("id")))
            .collect(toList());
        
        return userService.getUsersByIds(ids);  // 배치 조회!
    }
}

// Apollo Gateway (TypeScript)
import { ApolloGateway, IntrospectAndCompose } from "@apollo/gateway";
import { ApolloServer } from "apollo-server-express";

const gateway = new ApolloGateway({
  supergraphSdl: new IntrospectAndCompose({
    subgraphs: [
      { name: "order-service", url: "http://order-service:4001/graphql" },
      { name: "user-service", url: "http://user-service:4002/graphql" },
      { name: "product-service", url: "http://product-service:4003/graphql" },
    ],
  }),
});

const server = new ApolloServer({
  gateway,
  context: ({ req }) => ({ userId: req.headers["user-id"] }),
});

await server.start();
app.use("/graphql", expressMiddleware(server));
```

---

## 🔬 내부 동작 원리 — Federation Entity Resolution

### 1. Supergraph 조합 과정

```
┌─────────────────────────────────┐
│ Order Service Subgraph          │
│ type Order @key(fields: "id")   │
│   id: ID!                       │
│   totalAmount: Float!           │
│   userId: ID!                   │
└──────────────┬──────────────────┘
               │ SDL (Schema Definition Language)
               │
┌──────────────▼──────────────────┐
│ User Service Subgraph           │
│ type User @key(fields: "id")    │
│   id: ID!                       │
│   name: String!                 │
│   email: String!                │
└──────────────┬──────────────────┘
               │ SDL
               │
┌──────────────▼──────────────────┐
│ Product Service Subgraph        │
│ type Product @key(fields: "id") │
│   id: ID!                       │
│   name: String!                 │
│   price: Float!                 │
└──────────────┬──────────────────┘
               │
               ▼
    Apollo Gateway Composition
        (각 SDL 수집 → 통합)
        (타입 충돌 감지 → 오류)
        (Reference 분석 → 쿼리 계획)
               │
               ▼
        ┌─────────────────────┐
        │ Supergraph (통합)   │
        │                     │
        │ type Query {        │
        │   order(id: ID!)    │
        │   user(id: ID!)     │
        │   product(id: ID!)  │
        │ }                   │
        │                     │
        │ type Order {        │
        │   id                │
        │   totalAmount       │
        │   user: User! ◀─ 다른 Subgraph
        │   items            │
        │ }                   │
        └─────────────────────┘
```

### 2. 쿼리 실행 흐름 (Entity Resolution)

```
클라이언트 쿼리:

query {
  order(id: "123") {
    id
    totalAmount
    user {
      name
      email
    }
    items {
      product {
        name
        price
      }
    }
  }
}

Apollo Gateway 실행:

┌─ Step 1: Order 조회
│  [Order Service] ← query { order(id: "123") { id totalAmount userId items { productId } } }
│  Response: { id: "123", totalAmount: 50000, userId: "456", items: [{productId: "p1"}] }
│
├─ Step 2: User 조회 (Entity Resolution)
│  [User Service] ← query {
│                    _entities(representations: [
│                      {__typename: "User", id: "456"}
│                    ]) {
│                      ... on User { name email }
│                    }
│                  }
│  Response: { _entities: [{ name: "Alice", email: "alice@..." }] }
│
├─ Step 3: Product 조회 (Entity Resolution)
│  [Product Service] ← query {
│                      _entities(representations: [
│                        {__typename: "Product", id: "p1"}
│                      ]) {
│                        ... on Product { name price }
│                      }
│                    }
│  Response: { _entities: [{ name: "상품1", price: 50000 }] }
│
└─ Step 4: 응답 조합
   {
     order: {
       id: "123",
       totalAmount: 50000,
       user: {
         name: "Alice",
         email: "alice@..."
       },
       items: [{
         product: {
           name: "상품1",
           price: 50000
         }
       }]
     }
   }
```

### 3. N+1 문제와 배치 최적화

```
❌ N+1 문제: 데이터 로더 없을 때

쿼리:
{
  orders {
    id
    user { id name }
    items { product { id name } }
  }
}

실행:
1. orders 조회 (1회)
   → [Order(id:1, userId:1), Order(id:2, userId:2), ...]

2. 각 Order의 User 조회 (N회)
   _entities(id:1)  ← 1회
   _entities(id:2)  ← 1회
   _entities(id:3)  ← 1회
   ...

3. 각 Item의 Product 조회 (M회)
   _entities(p1)  ← 1회
   _entities(p2)  ← 1회
   ...

총 호출: 1 + N + M = 1 + 100 + 500 = 601회! 😱


✅ 배치 최적화: DataLoader 패턴

query {
  orders {
    id
    user { id name }         # DataLoader 자동 배치
    items {
      product { id name }    # DataLoader 자동 배치
    }
  }
}

실행:
1. orders 조회 (1회)
   → [Order(...), Order(...), ...]

2. User 배치 조회 (1회)
   _entities(representations: [
     {__typename: "User", id: 1},
     {__typename: "User", id: 2},
     ...
     {__typename: "User", id: 100}
   ])

3. Product 배치 조회 (1회)
   _entities(representations: [
     {__typename: "Product", id: p1},
     {__typename: "Product", id: p2},
     ...
     {__typename: "Product", id: p500}
   ])

총 호출: 1 + 1 + 1 = 3회! ✅
```

### 4. REST vs GraphQL vs gRPC 선택 기준

```
요청-응답 패턴:

REST:
GET /api/orders/123
GET /api/users/456
GET /api/products/p1
→ 3번 호출, 3번 응답

GraphQL:
query {
  order(id: "123") { ... }
  user(id: "456") { ... }
  product(id: "p1") { ... }
}
→ 1번 호출, 1번 응답

gRPC:
Order rpc GetOrder()
User rpc GetUser()
Product rpc GetProduct()
→ 병렬 호출, 병렬 응답

선택 기준:
┌────────────────┬──────────────┬─────────────┬──────────────┐
│ 특성           │ REST         │ GraphQL     │ gRPC         │
├────────────────┼──────────────┼─────────────┼──────────────┤
│ 요청 수        │ 많음 (N)     │ 적음 (1)    │ 병렬 (1-N)   │
│ 응답 크기      │ 고정         │ 선택        │ 바이너리     │
│ 캐싱           │ ✅ (HTTP)    │ ⚠️         │ ❌          │
│ 브라우저       │ ✅ 직접 호출 │ ✅ 직접호출│ ❌ 웹X      │
│ 타입 안정성    │ 낮음         │ 높음        │ 매우 높음    │
│ 커버 필드 조회 │ over-fetch   │ 정확        │ 고정 스키마  │
│ 실시간         │ 폴링         │ Subscription│ Streaming   │
└────────────────┴──────────────┴─────────────┴──────────────┘

추천 조합:
- 공개 API (브라우저): REST
- 모바일 (최소 요청): GraphQL
- 내부 서비스: gRPC
- 혼합: REST + GraphQL (Gateway)
```

---

## 💻 코드 예제 — GraphQL Federation 실전

### Schema.graphql (Order Service)
```graphql
extend schema
  @link(url: "https://specs.apollo.dev/federation/v2.0")

type Query {
  order(id: ID!): Order
  orders(userId: ID!): [Order!]!
}

# @key로 엔티티 정의 (다른 서비스에서 참조 가능)
type Order @key(fields: "id") {
  id: ID!
  totalAmount: Float!
  userId: ID!
  items: [OrderItem!]!
  createdAt: String!
}

type OrderItem {
  productId: ID!
  quantity: Int!
  unitPrice: Float!
}
```

### OrderController.java
```java
@RestController
@RequestMapping("/graphql")
public class OrderGraphQLController {
    
    @Autowired private OrderRepository orderRepository;
    @Autowired private UserServiceClient userClient;
    @Autowired private ProductServiceClient productClient;
    
    // Query: order 조회
    @GraphQlQueryMapping
    public Order order(@Argument Long id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Order not found"));
    }
    
    // Query: 사용자의 주문 목록
    @GraphQlQueryMapping
    public List<Order> orders(@Argument Long userId) {
        return orderRepository.findByUserId(userId);
    }
    
    // Mutation: 주문 생성
    @GraphQlMutationMapping
    @Transactional
    public Order createOrder(@Argument CreateOrderInput input) {
        Order order = new Order();
        order.setUserId(input.getUserId());
        order.setItems(input.getItems());
        order.setTotalAmount(input.getItems().stream()
            .mapToDouble(item -> item.getUnitPrice() * item.getQuantity())
            .sum());
        
        return orderRepository.save(order);
    }
    
    // Federation: Entity Resolver
    // 다른 서비스에서 Order를 참조할 때 호출됨
    @GraphQlDataFetcher
    public Order _order(Order order) {
        if (order.getId() == null) {
            throw new IllegalArgumentException("Order ID required");
        }
        return orderRepository.findById(order.getId())
            .orElseThrow();
    }
}
```

### UserServiceClient (Remote Subgraph)
```java
@Service
public class UserServiceClient {
    
    private final WebClient webClient;
    private final DataLoader<Long, User> userBatchLoader;
    
    @Autowired
    public UserServiceClient(WebClient.Builder webClientBuilder,
                            DataLoaderRegistry registry) {
        this.webClient = webClientBuilder
            .baseUrl("http://user-service/graphql")
            .build();
        
        // DataLoader: N개를 1번 쿼리로 처리
        this.userBatchLoader = DataLoader.newMappedDataLoader(
            userIds -> {
                String query = """
                    query {
                      _entities(representations: [
                        %s
                      ]) {
                        ... on User { id name email }
                      }
                    }
                    """.formatted(
                    userIds.stream()
                        .map(id -> String.format(
                            "{__typename: \\"User\\", id: \\"%d\\"}", id))
                        .collect(Collectors.joining(", "))
                );
                
                return webClient.post()
                    .body(BodyInserters.fromValue(new GraphQLRequest(query)))
                    .retrieve()
                    .bodyToMono(GraphQLResponse.class)
                    .map(this::mapUsers)
                    .toCompletableFuture();
            }
        );
        
        registry.register("users", userBatchLoader);
    }
    
    public User getUser(Long userId) {
        // DataLoader를 사용하면 자동으로 배치 처리됨
        return userBatchLoader.load(userId).join();
    }
}
```

### Apollo Gateway (Node.js)
```typescript
import { ApolloGateway, IntrospectAndCompose, buildSubgraphSchema } from "@apollo/gateway";
import { ApolloServer } from "apollo-server-express";
import express from "express";

// Subgraph 정의
const subgraphs = [
    {
        name: "order-service",
        url: "http://order-service:8081/graphql"
    },
    {
        name: "user-service",
        url: "http://user-service:8082/graphql"
    },
    {
        name: "product-service",
        url: "http://product-service:8083/graphql"
    }
];

// Federation Gateway 설정
const gateway = new ApolloGateway({
    supergraphSdl: new IntrospectAndCompose({
        subgraphs,
        // 주기적으로 Subgraph 스키마 업데이트
        pollIntervalInMs: 10_000,
    }),
});

// Apollo Server 설정
const server = new ApolloServer({
    gateway,
    // 컨텍스트 (모든 Subgraph에 전달)
    context: ({ req }) => ({
        userId: req.headers["x-user-id"],
        orgId: req.headers["x-org-id"],
    }),
    // 에러 처리
    formatError: (err) => {
        console.error("GraphQL Error:", err);
        return {
            message: err.message,
            extensions: {
                code: err.extensions?.code || "INTERNAL_SERVER_ERROR",
            }
        };
    },
});

const app = express();

server.start().then(() => {
    app.use("/graphql", express.json(), server);
    app.listen(4000, () => {
        console.log("Apollo Gateway listening on http://localhost:4000/graphql");
    });
});
```

### build.gradle (Federation 의존성)
```gradle
dependencies {
    // Spring GraphQL
    implementation 'org.springframework.boot:spring-boot-starter-graphql:3.2.0'
    
    // Apollo Federation
    implementation 'com.apollographql.federation:federation-graphql-java-support:3.0.0'
    
    // GraphQL Java
    implementation 'com.graphql-java:graphql-java:21.0'
    
    // DataLoader (N+1 방지)
    implementation 'org.dataloader:java-dataloader:3.0.0'
    
    // WebClient
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
}
```

### application.yml
```yaml
spring:
  graphql:
    graphiql:
      enabled: true           # GraphiQL IDE 활성화
      path: /graphiql
    
    schema:
      locations: classpath:graphql/
      introspection:
        enabled: true         # Introspection (필수)
    
    cors:
      allowed-origins: "*"
      allowed-methods: GET, POST

# Federation 설정
apollo:
  federation:
    enabled: true
    subgraph: true            # 이 서비스가 Subgraph임을 선언
```

---

## 📊 패턴 비교 — REST vs GraphQL vs gRPC

| 특성 | REST | GraphQL | gRPC |
|------|------|---------|------|
| **클라이언트 요청 수** | N (over-fetching) | 1 (exact) | 1-N (병렬) |
| **응답 크기** | 고정 | 선택적 (정확) | 바이너리 (작음) |
| **캐싱** | ✅ HTTP 캐시 | ⚠️ 복잡 (쿼리 기반) | ❌ (연결 기반) |
| **브라우저 지원** | ✅ | ✅ | ❌ (gRPC-Web) |
| **타입 안정성** | 낮음 | 높음 (스키마) | 매우 높음 (Proto) |
| **실시간** | 폴링 | Subscription | Streaming |
| **다중 버전 지원** | /v1, /v2 | 하나 (필드 추가) | Proto 진화 |
| **개발 용이성** | 높음 | 높음 | 중간 |
| **운영 복잡도** | 낮음 | 중간 (Federation) | 높음 |
| **성능** | 좋음 | 좋음 (적절히) | 매우 좋음 |
| **공개 API** | 권장 | 권장 | 내부만 |

---

## ⚖️ 트레이드오프

```
GraphQL Federation의 이점과 비용:

이점:
✅ 클라이언트 요청 수 감소 (N → 1)
✅ 정확한 응답 크기 (over-fetching 없음)
✅ 강한 타입 안정성 (스키마 기반)
✅ 자동 스키마 조합 (복잡성 감소)
✅ 각 서비스 독립 배포 (Subgraph)

비용:
❌ 구현 복잡도 증가
   - Apollo Gateway, Subgraph, Entity Resolver
   - Schema composition, federated directives
   
❌ N+1 쿼리 문제 가능성
   - Entity Resolution이 N번의 원격 호출 가능
   - DataLoader로 배치화 필요

❌ 디버깅 어려움
   - 여러 Subgraph를 거쳐 실행
   - 느린 Subgraph 찾기 어려움

❌ 캐싱 복잡도
   - HTTP 캐시 사용 불가 (POST, 쿼리 기반)
   - 응용층 캐시 필요

❌ 모니터링 부담
   - 요청이 여러 서비스를 거침
   - 분산 추적(Tracing) 필수

────────────────────────────────────────

언제 도입할까:

Federation 도입 (✅)
- 클라이언트가 여러 서비스의 데이터 필요
- 복잡한 데이터 조합 (JOIN 많음)
- 멀티 클라이언트 (모바일, 웹, 웹훅)
- 운영 팀이 충분함

Federation 불필요 (❌)
- 단순 CRUD (한 서비스만 사용)
- gRPC로 충분함
- REST API로 충분함
- 운영 간단하게 유지하고 싶음

권장:
REST (단순 API) + GraphQL Federation (복잡 조합)
```

---

## 📌 핵심 정리

```
✅ GraphQL Federation의 작동 원리

1. Subgraph (각 서비스)
   - 자신의 타입 정의 (@key로 표시)
   - @key: 엔티티 식별자 (다른 서비스가 참조 가능)
   - Example: type Order @key(fields: "id")

2. Apollo Gateway (중앙)
   - 모든 Subgraph의 스키마 수집
   - 타입 참조 분석 (Order → User 참조)
   - 쿼리 계획 수립 (실행 순서 결정)
   - Supergraph 제공 (클라이언트에게)

3. 클라이언트 (프론트엔드)
   - Supergraph 쿼리
   - GraphQL 모르는 Subgraph → Gateway가 처리

✅ Entity Resolution (핵심)

@key(fields: "id")로 정의한 타입을 다른 서비스에서 사용

Order Service:
type Order @key(fields: "id") {
  id: ID!
  userId: ID!
}

User Service:
type User @key(fields: "id") {
  id: ID!
  name: String!
}

Order에서 User 참조:
extend type Order {
  user: User! @requires(fields: "userId")
}

쿼리 실행:
1. Order Service: order(id: 123) → { id, userId }
2. User Service: _entities(id: ${userId}) → { name, ... }
3. 조합 → 응답

✅ N+1 문제와 DataLoader

❌ 없을 때:
orders { user { name } }
→ Order 100개 조회 + User 100번 개별 조회 = 101회

✅ DataLoader로 배치:
orders { user { name } }
→ Order 100개 조회 + User 100개 배치 조회 = 2회

구현:
DataLoader<Long, User> loader = DataLoader.newMappedDataLoader(
  userIds -> userService.getUsers(userIds)
);

✅ REST vs GraphQL vs gRPC 선택

REST:
- 공개 API, 브라우저 호출
- 캐싱 필요 (HTTP 캐시)
- 다중 엔드포인트 (_v1, _v2 호환)

GraphQL:
- 복잡한 데이터 조합 필요
- 클라이언트별 다양한 응답
- Federation으로 마이크로서비스 지원

gRPC:
- 내부 서비스 간 고성능 통신
- 강한 타입 안정성
- 실시간 양방향 (스트리밍)

조합 권장:
공개 API: REST
클라이언트 앱: GraphQL Federation
마이크로서비스 내부: gRPC

✅ Federation의 장단점 최종 정리

장점:
- 클라이언트: 하나의 쿼리로 여러 서비스 데이터 조회
- 서버: 각 팀이 자신의 스키마만 관리 (독립 배포)
- 게이트웨이: 자동으로 스키마 조합

단점:
- 학습곡선: Federation 개념 복잡
- N+1 위험: Entity Resolution이 여러 호출 가능
- 디버깅: 여러 서비스를 거쳐 복잡
- 모니터링: 분산 추적 필수

사용 기준:
- 프로토타입: 불필요 (REST로 시작)
- 5-10 서비스: 고려 (필수는 아님)
- 20+ 서비스: 강력히 권장 (스케일링 필수)
```

---

## 🤔 생각해볼 문제

**Q1.** Order → User 참조에서 Entity Resolution이 일어날 때, 100개 Order의 User를 조회한다면 User Service에 몇 번 호출할까? (DataLoader 있을 때 vs 없을 때)

<details>
<summary>해설 보기</summary>

**DataLoader 없을 때 (N+1 문제):**

```graphql
query {
  orders {
    id
    user { name }  # User Service 호출
  }
}
```

실행:
1. Order Service: orders 조회 → 100개 Order
2. 각 Order의 user 필드 해결:
   - user1 조회 (1회)
   - user2 조회 (1회)
   - ...
   - user100 조회 (1회)
   
총 호출: 1 + 100 = 101회 💥

```

**DataLoader 있을 때 (배치화):**

```java
@GraphQlDataFetcher
public DataFetcher<User> user(Order order) {
    return env -> {
        DataLoaderRegistry registry = env.getGraphQlContext()
            .get(DataLoaderRegistry.class);
        DataLoader<Long, User> userLoader = 
            registry.getDataLoader("user");
        
        // 모든 Order의 user 필드를 동시에 요청
        // DataLoader가 자동으로 배치 처리
        return userLoader.load(order.getUserId());
    };
}

// DataLoader 설정
dataLoaderRegistry.register("user",
    DataLoader.newMappedDataLoader(userIds -> {
        // userIds = [1, 2, 3, ..., 100]
        return userService.getUsersByIds(userIds);  // 1회 쿼리!
    })
);
```

실행:
1. Order Service: orders 조회 → 100개 Order
2. DataLoader 배치 처리:
   - 모든 user 필드가 userLoader.load() 호출
   - DataLoader가 모아서 1번에 User Service에 요청
   - User Service: getUsersByIds([1,2,...,100]) → SQL IN 쿼리
   
총 호출: 1 + 1 = 2회 ✅

성능 향상: 101회 → 2회 (50배!)
```

**핵심:**

DataLoader는 같은 요청 사이클에서 여러 개의 개별 요청을 자동으로 배치 처리합니다. Java에서는 `DataLoaderRegistry`에 등록하고, GraphQL 필드 resolver에서 사용하면 됩니다.

</details>

**Q2.** GraphQL Federation에서 User Service가 업데이트되어 새로운 필드 `phone: String`을 추가했습니다. Order Service는 어떻게 변경해야 할까?

<details>
<summary>해설 보기</summary>

**GraphQL Federation의 장점: 변경 최소화!**

```graphql
// User Service (변경됨)
type User @key(fields: "id") {
  id: ID!
  name: String!
  email: String!
  phone: String!  // ← 새 필드 추가
}

// Order Service (변경 불필요!)
extend type Order {
  user: User! @requires(fields: "userId")  // 그대로!
}
```

**변경 순서:**

1. **User Service 배포**
   ```graphql
   type User {
     ...
     phone: String!
   }
   ```
   - Subgraph 스키마 업데이트

2. **Apollo Gateway가 자동 감지**
   ```
   Gateway는 주기적으로 Subgraph 스키마를 introspect
   User Service의 새 필드 감지
   Supergraph 자동 업데이트 (수초 내)
   ```

3. **Order Service 변경 불필요!**
   - Order Service는 그대로 유지
   - Federation이 User 타입을 User Service에서 관리하기 때문
   - Order Service는 User 타입의 구조를 모름

4. **클라이언트가 새 필드 사용 가능**
   ```graphql
   query {
     order(id: "123") {
       user {
         name
         email
         phone  // ← 새 필드 사용 가능!
       }
     }
   }
   ```

**비교: REST API라면?**

```java
// REST Order Service
@GetMapping("/orders/{id}")
public OrderDTO getOrder(@PathVariable Long id) {
    Order order = orderRepository.findById(id);
    UserDTO user = userServiceClient.getUser(order.getUserId());
    
    // User에 phone 필드 추가되었으면
    // OrderDTO도 변경 필요
    return new OrderDTO(
        order.getId(),
        order.getTotal(),
        new UserDTO(
            user.getId(),
            user.getName(),
            user.getEmail(),
            user.getPhone()  // ← 추가 코드
        )
    );
}
```

GraphQL Federation은 변경 불필요! (schema composition)

</details>

**Q3.** GraphQL Federation에서 "Order와 관련된 User 정보 + Product 정보를 한 번에 조회"할 때, Apollo Gateway는 어떤 순서로 Subgraph를 호출할까?

<details>
<summary>해설 보기</summary>

**쿼리:**

```graphql
query {
  order(id: "123") {
    id
    totalAmount
    user {
      name
      email
    }
    items {
      product {
        name
        price
      }
    }
  }
}
```

**Apollo Gateway의 쿼리 계획 (Query Planning):**

```
1단계: Root Query (Order Service)
──────────────────────────────
[Order Service] ← query {
  order(id: "123") {
    id
    totalAmount
    userId
    items { productId }
  }
}

응답: {
  id: "123",
  totalAmount: 50000,
  userId: "456",
  items: [
    { productId: "p1", quantity: 2 },
    { productId: "p2", quantity: 1 }
  ]
}

2단계: Entity Resolution (병렬)
──────────────────────────────
[User Service] ← _entities(representations: [
  { __typename: "User", id: "456" }
])

[Product Service] ← _entities(representations: [
  { __typename: "Product", id: "p1" },
  { __typename: "Product", id: "p2" }
])

병렬 처리 (순서 상관없음):
- User Service와 Product Service는 동시에 호출됨!

응답 (User Service):
{ _entities: [{ name: "Alice", email: "alice@..." }] }

응답 (Product Service):
{ _entities: [
  { name: "상품1", price: 50000 },
  { name: "상품2", price: 30000 }
] }

3단계: 응답 조합
──────────────────
{
  order: {
    id: "123",
    totalAmount: 50000,
    user: { name: "Alice", email: "alice@..." },
    items: [
      { product: { name: "상품1", price: 50000 } },
      { product: { name: "상품2", price: 30000 } }
    ]
  }
}
```

**성능 포인트:**

```
호출 수: 3회 (Order 1회 + User 1회 + Product 1회)
  - User와 Product는 병렬 처리 (기다리지 않음)
  
최종 지연: max(
  Order (100ms),
  User (50ms),      ← 병렬
  Product (150ms)   ← 병렬
) + 조합 (5ms)
= ~255ms

만약 순차:
Order (100ms) + User (50ms) + Product (150ms) = 300ms

병렬 이득: 300ms → 255ms (15% 향상)
```

**주의: 의존성이 있으면 순차**

```graphql
query {
  order(id: "123") {
    user {              # Order 조회 후 필요
      recommendedProducts {  # User 조회 후 필요
        name
      }
    }
  }
}
```

이 경우:
1. Order 조회
2. User 조회 (Order userId 필요)
3. Product 조회 (User 필요)

순차 처리됨 (병렬 불가)

</details>

---

<div align="center">

**[⬅️ 이전: Service Mesh — Envoy Sidecar와 mTLS](./06-service-mesh.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — Database per Service 원칙 ➡️](../data-management/01-database-per-service.md)**

</div>

# 03. gRPC 완전 분해 — Protocol Buffers와 HTTP/2

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Protocol Buffers가 JSON보다 직렬화/역직렬화가 빠른 이유는 무엇인가? (바이너리 인코딩, 필드 번호)
- HTTP/2 멀티플렉싱이 하나의 TCP 연결에서 여러 스트림을 처리하는 원리는?
- gRPC의 4가지 통신 패턴(Unary, Server Streaming, Client Streaming, Bidirectional)을 어떻게 구분하고 활용하는가?
- 내부 마이크로서비스 통신은 gRPC, 공개 API는 REST를 쓰는 이유는?
- gRPC 로드 밸런싱이 L4 vs L7에서 어떻게 다르고, 왜 L7 문제가 되는가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

마이크로서비스 아키텍처에서 서비스 간 통신의 성능과 효율성은 **시스템 전체의 응답 시간, 처리량, 리소스 사용량**을 크게 좌우합니다.

REST API는 JSON 직렬화로 인한 오버헤드가 있습니다. 예를 들어 매초 10,000개의 작은 메시지를 주고받는 상황에서 JSON의 문자열 기반 포맷은 불필요한 네트워크 대역폭을 낭비합니다. 반면 gRPC의 Protocol Buffers는 **바이너리 인코딩으로 데이터 크기를 1/3로 줄이면서**, HTTP/2 멀티플렉싱으로 **하나의 연결에서 동시에 여러 요청을 처리**합니다.

또한 REST는 각 요청마다 새로운 HTTP 핸드셰이크가 필요할 수 있지만, gRPC는 지속적인 연결 위에서 스트리밍을 지원해 **양방향 실시간 통신**이 가능합니다. 금융 시스템의 실시간 거래 업데이트, 채팅 서비스, 센서 데이터 수집 등에서 gRPC는 REST보다 훨씬 효율적입니다.

그러나 gRPC는 바이너리 프로토콜이므로 **브라우저 직접 호출이 불가능**하고, 로드 밸런싱이 복잡하며, 디버깅도 어렵습니다. 따라서 **내부 서비스 간 통신은 gRPC, 공개 API는 REST**라는 이원적 아키텍처가 일반적입니다.

---

## 😱 흔한 실수 (Before — REST만 사용한 과도한 트래픽)

```java
// ❌ 안티패턴 1: 모든 서비스 간 통신을 REST로 처리
// Order Service → Product Service 조회 (매초 1000회)

@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProduct(@PathVariable Long id) {
        Product product = productService.getProduct(id);
        return ResponseEntity.ok(new ProductResponse(
            product.getId(),
            product.getName(),
            product.getPrice(),
            product.getDescription(),
            product.getInventoryCount(),
            product.getSupplier(),
            product.getCategory(),
            product.getTags(),
            product.getImages()
        )); // JSON 직렬화
    }
}

// Order Service 클라이언트
public class OrderService {
    @Autowired private RestTemplate restTemplate;
    
    public void placeOrder(OrderRequest request) {
        // 매초 1000회 호출 × 트래픽량 계산
        for (OrderItem item : request.getItems()) {
            // REST 호출: 1개 요청 = ~1KB JSON + 헤더 ~500B
            ProductResponse product = restTemplate.getForObject(
                "http://product-service/api/products/" + item.getProductId(),
                ProductResponse.class);
            // JSON 역직렬화 오버헤드: 문자열 → Java 객체
            
            validateStock(product);
        }
    }
}

// 문제점:
// 1. JSON 직렬화: "name": "상품명" (16B) vs 바이너리 (2B)
// 2. HTTP/1.1: 각 요청마다 새 연결 또는 연결 재사용 최적화 필요
// 3. 네트워크 대역폭 낭비: 매초 ~1MB 트래픽
// 4. 응답 시간 누적: 텍스트 파싱 오버헤드


// ❌ 안티패턴 2: 스트리밍이 필요한데 REST 폴링 사용
// 실시간 주문 상태 업데이트 (Order Service → Notification Service)

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @GetMapping("/{id}/status")
    public ResponseEntity<OrderStatusResponse> getOrderStatus(
        @PathVariable Long id) {
        Order order = orderRepository.findById(id);
        return ResponseEntity.ok(new OrderStatusResponse(
            order.getId(),
            order.getStatus(),
            order.getUpdatedAt()
        ));
    }
}

// Notification Service 클라이언트 (폴링)
public class NotificationService {
    
    public void watchOrderStatus(Long orderId) {
        // ❌ 폴링: 1초마다 상태 확인 (2시간 = 7200 요청!)
        while (true) {
            OrderStatusResponse status = restTemplate.getForObject(
                "http://order-service/api/orders/" + orderId + "/status",
                OrderStatusResponse.class);
            
            if (status.getStatus().equals("SHIPPED")) {
                sendNotification("주문이 배송되었습니다");
                break;
            }
            
            Thread.sleep(1000); // 1초 대기
        }
    }
}

// 문제점:
// 1. 과도한 요청: 변화가 없어도 계속 폴링
// 2. 지연: 최대 1초의 지연 (실시간성 부족)
// 3. 서버 부하: 불필요한 요청 처리
// 4. 네트워크 낭비: 대부분의 응답이 "상태 변화 없음"
```

---

## ✨ 올바른 접근 (After — gRPC + Protocol Buffers)

```protobuf
// ✅ product.proto (Protocol Buffers 정의)
syntax = "proto3";

package com.example.product;

// 필드 번호가 직렬화 키
message Product {
    int64 id = 1;              // 필드 번호 1
    string name = 2;
    double price = 3;
    string description = 4;
    int32 inventory_count = 5;
    
    // 선택적 필드 (proto3에서는 모두 선택적)
    string supplier = 6;
    string category = 7;
    repeated string tags = 8;
    repeated string images = 9;
}

message GetProductRequest {
    int64 id = 1;
}

message GetProductResponse {
    Product product = 1;
    string error_message = 2;
}

// 4가지 RPC 패턴
service ProductService {
    // Unary: 단일 요청 → 단일 응답
    rpc GetProduct(GetProductRequest) returns (GetProductResponse);
    
    // Server Streaming: 단일 요청 → 여러 응답 (스트림)
    rpc GetProductsByCategory(CategoryRequest) 
        returns (stream ProductResponse);
    
    // Client Streaming: 여러 요청 → 단일 응답
    rpc CreateProductBatch(stream CreateProductRequest) 
        returns (BatchResponse);
    
    // Bidirectional Streaming: 쌍방향 스트림
    rpc SyncProducts(stream ProductSyncRequest) 
        returns (stream ProductSyncResponse);
}
```

```java
// ✅ Java gRPC 구현

// build.gradle
plugins {
    id 'com.google.protobuf' version '0.9.4'
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:3.21.0"
    }
    plugins {
        grpc {
            artifact = 'io.grpc:protoc-gen-grpc-java:1.50.0'
        }
    }
    generateProtoTasks {
        all().each { task ->
            task.plugins {
                grpc {}
            }
        }
    }
}

dependencies {
    implementation 'io.grpc:grpc-stub:1.50.0'
    implementation 'io.grpc:grpc-protobuf:1.50.0'
}

// ✅ gRPC 서비스 구현
@Component
public class ProductServiceImpl extends ProductServiceGrpc.ProductServiceImplBase {
    
    @Autowired private ProductRepository productRepository;
    
    // Unary RPC
    @Override
    public void getProduct(GetProductRequest request,
                          StreamObserver<GetProductResponse> responseObserver) {
        try {
            Product product = productRepository.findById(request.getId());
            
            GetProductResponse response = GetProductResponse.newBuilder()
                .setProduct(Product.newBuilder()
                    .setId(product.getId())
                    .setName(product.getName())
                    .setPrice(product.getPrice())
                    .setInventoryCount(product.getInventory())
                    .addAllTags(product.getTags())
                    .addAllImages(product.getImages())
                    .build())
                .build();
            
            responseObserver.onNext(response);
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(Status.INTERNAL
                .withDescription("Product not found")
                .asException());
        }
    }
    
    // Server Streaming RPC
    @Override
    public void getProductsByCategory(CategoryRequest request,
                                     StreamObserver<ProductResponse> responseObserver) {
        try {
            List<Product> products = productRepository.findByCategory(
                request.getCategory());
            
            // 여러 응답을 스트림으로 전송
            for (Product product : products) {
                ProductResponse response = ProductResponse.newBuilder()
                    .setId(product.getId())
                    .setName(product.getName())
                    .setPrice(product.getPrice())
                    .build();
                
                responseObserver.onNext(response); // 하나씩 전송
            }
            
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(e);
        }
    }
    
    // Bidirectional Streaming RPC (실시간 주문 상태)
    @Override
    public StreamObserver<OrderUpdateRequest> watchOrderStatus(
            StreamObserver<OrderUpdateResponse> responseObserver) {
        
        return new StreamObserver<OrderUpdateRequest>() {
            @Override
            public void onNext(OrderUpdateRequest request) {
                // 클라이언트에서 주문 상태 업데이트 요청
                Order order = orderRepository.findById(request.getOrderId());
                order.setStatus(request.getNewStatus());
                orderRepository.save(order);
                
                // 즉시 응답 (양방향)
                OrderUpdateResponse response = OrderUpdateResponse.newBuilder()
                    .setOrderId(order.getId())
                    .setStatus(order.getStatus())
                    .setUpdatedAt(System.currentTimeMillis())
                    .build();
                
                responseObserver.onNext(response);
            }
            
            @Override
            public void onError(Throwable t) {
                logger.error("Error:", t);
            }
            
            @Override
            public void onCompleted() {
                responseObserver.onCompleted();
            }
        };
    }
}

// ✅ gRPC 클라이언트
@Component
public class OrderServiceClient {
    
    private final ProductServiceGrpc.ProductServiceBlockingStub productServiceStub;
    private final ProductServiceGrpc.ProductServiceStub productServiceAsync;
    
    public OrderServiceClient(
            ProductServiceGrpc.ProductServiceBlockingStub productServiceStub,
            ProductServiceGrpc.ProductServiceStub productServiceAsync) {
        this.productServiceStub = productServiceStub;
        this.productServiceAsync = productServiceAsync;
    }
    
    // Unary 호출 (동기)
    public Product getProduct(Long productId) {
        GetProductRequest request = GetProductRequest.newBuilder()
            .setId(productId)
            .build();
        
        GetProductResponse response = productServiceStub.getProduct(request);
        return convertToProduct(response.getProduct());
    }
    
    // Server Streaming (대량 데이터 효율적 처리)
    public List<Product> getProductsByCategory(String category) {
        CategoryRequest request = CategoryRequest.newBuilder()
            .setCategory(category)
            .build();
        
        List<Product> products = new ArrayList<>();
        
        productServiceStub.getProductsByCategory(request)
            .forEachRemaining(response -> 
                products.add(convertToProduct(response)));
        
        return products;
    }
    
    // Bidirectional Streaming (실시간 양방향)
    public void watchOrderStatusBidirectional(Long orderId) {
        StreamObserver<OrderUpdateResponse> responseObserver = 
            new StreamObserver<OrderUpdateResponse>() {
                @Override
                public void onNext(OrderUpdateResponse response) {
                    logger.info("Order {} status: {}",
                        response.getOrderId(),
                        response.getStatus());
                    
                    // 즉시 다음 업데이트 요청
                    sendNextUpdate(response.getOrderId());
                }
                
                @Override
                public void onError(Throwable t) {
                    logger.error("Stream error", t);
                }
                
                @Override
                public void onCompleted() {
                    logger.info("Stream completed");
                }
            };
        
        StreamObserver<OrderUpdateRequest> requestObserver =
            productServiceAsync.watchOrderStatus(responseObserver);
        
        // 클라이언트에서 주문 상태 업데이트 전송
        OrderUpdateRequest request = OrderUpdateRequest.newBuilder()
            .setOrderId(orderId)
            .setNewStatus("PROCESSING")
            .build();
        
        requestObserver.onNext(request);
    }
}

// ✅ gRPC 서버 설정
@Configuration
public class GrpcServerConfig {
    
    @Bean
    public Server grpcServer(ProductServiceImpl productService) throws IOException {
        return ServerBuilder.forPort(9090)
            .addService(productService)
            .build()
            .start();
    }
}

// ✅ gRPC 클라이언트 설정
@Configuration
public class GrpcClientConfig {
    
    @Bean
    public ProductServiceGrpc.ProductServiceBlockingStub 
            productServiceBlockingStub() {
        ManagedChannel channel = ManagedChannelBuilder
            .forAddress("product-service", 9090)
            .usePlaintext() // 개발용, 실운영은 TLS 필수
            .build();
        
        return ProductServiceGrpc.newBlockingStub(channel);
    }
    
    @Bean
    public ProductServiceGrpc.ProductServiceStub productServiceAsync() {
        ManagedChannel channel = ManagedChannelBuilder
            .forAddress("product-service", 9090)
            .usePlaintext()
            .build();
        
        return ProductServiceGrpc.newStub(channel);
    }
}
```

---

## 🔬 내부 동작 원리 — Protocol Buffers와 HTTP/2

### 1. Protocol Buffers의 바이너리 인코딩

```
JSON 직렬화:
{"id":123,"name":"상품명","price":29999.99,"tags":["전자기기","할인"]}
→ 약 78 바이트 (텍스트)

Protocol Buffers (Varint 인코딩):
필드 1 (id=123):       0x01 0x7B             (3 바이트)
필드 2 (name):         0x12 0x06 상품명      (9 바이트)
필드 3 (price):        0x1D [4바이트 double] (5 바이트)
필드 8 (tags):         0x42 0x06 전자기기    (8 바이트)
                       0x42 0x04 할인         (6 바이트)
────────────────────────────────────────────────
합계: 약 31 바이트 (바이너리)

효율성: 78B → 31B (약 60% 감소)

디코딩 속도:
JSON: 문자열 파싱 → 타입 변환 → 검증 (상대적으로 느림)
Protobuf: 필드 번호 기반 직접 매핑 (매우 빠름)

┌─────────────────────────────────────┐
│ Varint 인코딩 원리                   │
├─────────────────────────────────────┤
│ 작은 숫자 (0-127): 1바이트           │
│ 중간 숫자 (128-16383): 2바이트       │
│ 큰 숫자: 가변 길이 (최대 10바이트)   │
│                                     │
│ 예: 123 → 0x7B (1바이트)            │
│     16384 → 0x80 0x80 0x01 (3바이트)│
└─────────────────────────────────────┘
```

### 2. HTTP/2 멀티플렉싱

```
HTTP/1.1 (연결 재사용, 하나의 요청 후 대기)
[요청1] ────────────────────→ [응답1] → 완료
                                        ↓
                            [요청2] ──→ [응답2] → 완료
                                        ↓
                            [요청3] ──→ [응답3] → 완료

문제: 네트워크 왕복 지연 (RTT) 누적
3개 요청 × 100ms RTT = 300ms 소요


HTTP/2 (멀티플렉싱, 동시 처리)
┌────────────────────────────────┐
│ TCP 연결 (1개)                  │
│                                │
│  Stream 1: [요청1] → [응답1]   │
│  Stream 2:   [요청2] → [응답2] │  (동시 진행)
│  Stream 3:     [요청3] → [응답3]│
│                                │
└────────────────────────────────┘

프레임 단위 전송:
[Frame: 요청1] [Frame: 요청2] [Frame: 요청3]
        ↓             ↓             ↓
    [처리중]     [처리중]     [처리중]
        ↓             ↓             ↓
[Frame: 응답1] [Frame: 응답2] [Frame: 응답3]

효과: 3개 요청 × 100ms = 100ms (병렬화)
```

### 3. gRPC 4가지 통신 패턴

```
┌─────────────────────────────────────┐
│ 1. Unary (일반적인 요청/응답)        │
├─────────────────────────────────────┤
│ Client                Server         │
│   │ ──[req]───────→   │             │
│   │                   │ (처리)       │
│   │ ←──[res]──────    │             │
│   │                   │             │
└─────────────────────────────────────┘
사용: 단일 조회, 업데이트
예: getProduct(id) → Product


┌─────────────────────────────────────┐
│ 2. Server Streaming (서버가 계속 전송)│
├─────────────────────────────────────┤
│ Client              Server          │
│   │ ──[req]──────→  │             │
│   │                 │ (처리)       │
│   │ ←──[res1]──     │             │
│   │ ←──[res2]──     │  (계속 스트림)
│   │ ←──[res3]──     │             │
│   │ ←──[end]────    │             │
└─────────────────────────────────────┘
사용: 대량 데이터 전송, 실시간 구독
예: getProductsByCategory(category) → stream Product


┌─────────────────────────────────────┐
│ 3. Client Streaming (클라이언트가 계속 전송)
├─────────────────────────────────────┤
│ Client              Server          │
│   │ ──[req1]──────→ │             │
│   │ ──[req2]──────→ │ (계속 수신)   │
│   │ ──[req3]──────→ │             │
│   │ ──[end]───────→ │             │
│   │                 │ (처리)       │
│   │ ←──[res]──────  │             │
└─────────────────────────────────────┘
사용: 일괄 업로드, 배치 작업
예: createProductBatch(stream requests) → response


┌─────────────────────────────────────┐
│ 4. Bidirectional Streaming (쌍방향) │
├─────────────────────────────────────┤
│ Client              Server          │
│   │ ──[req1]──────→ │             │
│   │                 │ ──→[process]│
│   │ ←──[res1]──     │             │
│   │                 │             │
│   │ ──[req2]──────→ │             │
│   │                 │ ──→[process]│
│   │ ←──[res2]──     │             │
│   │ ──[end]───────→ │             │
└─────────────────────────────────────┘
사용: 실시간 양방향 (채팅, 협업)
예: watchOrderStatus(stream requests) → stream responses
```

### 4. gRPC vs REST 선택 기준

```
내부 서비스 간 통신 (마이크로서비스)
┌────────────────────────────────┐
│         gRPC 선택               │
├────────────────────────────────┤
│ ✅ 높은 성능 필요               │
│ ✅ 실시간 양방향 통신           │
│ ✅ 대량 작은 메시지 처리         │
│ ✅ 내부 서비스만 호출            │
│ ✅ 프로토콜 버전 관리 용이       │
└────────────────────────────────┘

공개 API (브라우저, 모바일)
┌────────────────────────────────┐
│         REST 선택               │
├────────────────────────────────┤
│ ✅ 브라우저 직접 호출 필요       │
│ ✅ 간단한 디버깅 (curl, Postman) │
│ ✅ 캐싱 필요 (HTTP 캐시)        │
│ ✅ 공개 문서화 필요              │
│ ✅ 레거시 클라이언트 지원        │
└────────────────────────────────┘
```

---

## 💻 코드 예제 — gRPC 실전

### Protocol Buffers 정의
```protobuf
// order.proto
syntax = "proto3";

package com.example.order;

option java_package = "com.example.order.pb";
option java_outer_classname = "OrderProto";

message Order {
    int64 order_id = 1;
    string customer_id = 2;
    repeated OrderItem items = 3;
    double total_amount = 4;
    string status = 5;
    int64 created_at = 6;
    int64 updated_at = 7;
}

message OrderItem {
    int64 product_id = 1;
    int32 quantity = 2;
    double unit_price = 3;
    double subtotal = 4;
}

message CreateOrderRequest {
    string customer_id = 1;
    repeated OrderItem items = 2;
    string shipping_address = 3;
}

message CreateOrderResponse {
    int64 order_id = 1;
    string status = 2;
    double total_amount = 3;
}

service OrderService {
    rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
    rpc GetOrder(GetOrderRequest) returns (Order);
    rpc WatchOrderStatus(WatchOrderRequest) returns (stream OrderStatusUpdate);
}

message GetOrderRequest {
    int64 order_id = 1;
}

message WatchOrderRequest {
    int64 order_id = 1;
}

message OrderStatusUpdate {
    int64 order_id = 1;
    string status = 2;
    string message = 3;
    int64 timestamp = 4;
}
```

### gRPC 서버 구현
```java
@Component
public class OrderServiceImpl extends OrderServiceGrpc.OrderServiceImplBase {
    
    @Autowired private OrderRepository orderRepository;
    @Autowired private OrderStatusPublisher statusPublisher;
    
    @Override
    public void createOrder(CreateOrderRequest request,
                           StreamObserver<CreateOrderResponse> responseObserver) {
        try {
            Order order = new Order();
            order.setCustomerId(request.getCustomerId());
            order.setShippingAddress(request.getShippingAddress());
            order.setStatus("PENDING");
            order.setItems(request.getItemsList());
            order.setTotalAmount(
                request.getItemsList().stream()
                    .mapToDouble(OrderItem::getSubtotal)
                    .sum());
            
            Order saved = orderRepository.save(order);
            
            CreateOrderResponse response = CreateOrderResponse.newBuilder()
                .setOrderId(saved.getId())
                .setStatus(saved.getStatus())
                .setTotalAmount(saved.getTotalAmount())
                .build();
            
            responseObserver.onNext(response);
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(Status.INTERNAL
                .withDescription(e.getMessage())
                .asException());
        }
    }
    
    @Override
    public StreamObserver<WatchOrderRequest> watchOrderStatus(
            StreamObserver<OrderStatusUpdate> responseObserver) {
        
        return new StreamObserver<WatchOrderRequest>() {
            private Long orderId;
            private Thread watchThread;
            
            @Override
            public void onNext(WatchOrderRequest request) {
                orderId = request.getOrderId();
                
                // 백그라운드 스레드에서 상태 감시
                watchThread = new Thread(() -> {
                    while (orderId != null) {
                        Order order = orderRepository.findById(orderId)
                            .orElse(null);
                        
                        if (order != null) {
                            OrderStatusUpdate update = OrderStatusUpdate.newBuilder()
                                .setOrderId(order.getId())
                                .setStatus(order.getStatus())
                                .setTimestamp(System.currentTimeMillis())
                                .build();
                            
                            responseObserver.onNext(update);
                        }
                        
                        try {
                            Thread.sleep(1000); // 1초마다 확인
                        } catch (InterruptedException e) {
                            break;
                        }
                    }
                });
                watchThread.start();
            }
            
            @Override
            public void onError(Throwable t) {
                orderId = null;
                if (watchThread != null) {
                    watchThread.interrupt();
                }
            }
            
            @Override
            public void onCompleted() {
                orderId = null;
                if (watchThread != null) {
                    watchThread.interrupt();
                }
                responseObserver.onCompleted();
            }
        };
    }
}

@Configuration
public class GrpcServerConfiguration {
    
    @Bean
    public Server grpcServer(OrderServiceImpl orderService) throws IOException {
        return ServerBuilder.forPort(50051)
            .addService(orderService)
            .maxInboundMessageSize(4 * 1024 * 1024) // 4MB
            .build()
            .start();
    }
}
```

### gRPC 클라이언트
```java
@Component
public class OrderServiceClient {
    
    private final OrderServiceGrpc.OrderServiceBlockingStub blockingStub;
    private final OrderServiceGrpc.OrderServiceStub asyncStub;
    
    public OrderServiceClient() {
        ManagedChannel channel = ManagedChannelBuilder
            .forAddress("order-service", 50051)
            .usePlaintext()
            .build();
        
        this.blockingStub = OrderServiceGrpc.newBlockingStub(channel);
        this.asyncStub = OrderServiceGrpc.newStub(channel);
    }
    
    // Unary 호출
    public CreateOrderResponse createOrder(String customerId,
                                          List<OrderItemRequest> items) {
        CreateOrderRequest request = CreateOrderRequest.newBuilder()
            .setCustomerId(customerId)
            .addAllItems(items)
            .build();
        
        return blockingStub.createOrder(request);
    }
    
    // Server Streaming
    public void watchOrderStatus(Long orderId) {
        WatchOrderRequest request = WatchOrderRequest.newBuilder()
            .setOrderId(orderId)
            .build();
        
        asyncStub.watchOrderStatus(request, new StreamObserver<OrderStatusUpdate>() {
            @Override
            public void onNext(OrderStatusUpdate update) {
                logger.info("Order {} status: {}", update.getOrderId(), update.getStatus());
                
                if ("SHIPPED".equals(update.getStatus()) ||
                    "DELIVERED".equals(update.getStatus())) {
                    // 구독 해제
                }
            }
            
            @Override
            public void onError(Throwable t) {
                logger.error("Stream error", t);
            }
            
            @Override
            public void onCompleted() {
                logger.info("Order status watch completed");
            }
        });
    }
}
```

---

## 📊 패턴 비교 — gRPC vs REST vs GraphQL

| 특성 | gRPC | REST | GraphQL |
|------|------|------|---------|
| **직렬화** | Protobuf (바이너리) | JSON (텍스트) | JSON (텍스트) |
| **전송 크기** | 소 (30-60% 절감) | 중 | 중 (쿼리 따라 다름) |
| **성능** | 매우 빠름 (ms) | 보통 (10s-100ms) | 보통 (네트워크 + 파싱) |
| **실시간** | ✅ 스트리밍 | ❌ 폴링만 가능 | ⚠️ Subscription 필요 |
| **브라우저** | ❌ (gRPC-Web) | ✅ 직접 호출 | ✅ 직접 호출 |
| **캐싱** | ❌ | ✅ HTTP 캐시 | ⚠️ 복잡 |
| **디버깅** | ⚠️ 바이너리 | ✅ 텍스트 | ✅ 텍스트 |
| **로드밸런싱** | ⚠️ L7 필요 | ✅ L7 자동 | ✅ L7 자동 |
| **적합한 사용** | 내부 서비스 | 공개 API | 클라이언트-중심 API |

---

## ⚖️ 트레이드오프

```
gRPC의 성능 이점 vs 운영 복잡도:

성능 이점 (바이너리 + HTTP/2)
✅ 대역폭 60% 절감 (JSON 대비)
✅ 응답 시간 50% 단축 (멀티플렉싱)
✅ 메모리 30% 절감
✅ 실시간 양방향 스트리밍

운영 비용
❌ 로드 밸런싱 복잡 (L7 필요)
❌ 디버깅 어려움 (바이너리)
❌ 모니터링 도구 제한
❌ 프로토콜 버전 관리
❌ gRPC-Web (브라우저 지원) 추가 레이어

선택 기준:
- 매초 1000+ 요청, 저지연 필요 → gRPC
- API 공개, 브라우저 호출 필요 → REST
- 복잡한 쿼리, 유연성 필요 → GraphQL

────────────────────────────────────────

HTTP/2 멀티플렉싱의 비용:
✅ 동시 처리로 응답 시간 개선
✅ 연결 재사용으로 오버헤드 감소

❌ 더 복잡한 상태 관리 (스트림, 프레임)
❌ HOL (Head-of-Line) 블로킹 가능
   (하나의 프레임이 커서 전체 지연 가능)
❌ 메모리 사용 증가 (멀티플렉싱 상태)

실제 성능: 일반적으로 이점 > 비용
```

---

## 📌 핵심 정리

```
✅ Protocol Buffers의 효율성

1. 바이너리 인코딩
   - Varint: 작은 숫자는 1바이트 (JSON은 최소 3바이트)
   - 필드 번호 기반: "fieldName" 불필요 (2 vs 12바이트)
   - 결과: 20-60% 데이터 크기 감소

2. 직렬화 속도
   - 필드 번호 매핑 (O(1))
   - 타입 자동 변환 (JSON 파싱 불필요)
   - 결과: 3-5배 빠른 직렬화/역직렬화

3. 스키마 진화
   - 하위호환 유지 (필드 번호 불변)
   - 선택적 필드 추가 가능
   - 버전 관리 간소화

✅ HTTP/2 멀티플렉싱의 이점

1. 동시 처리
   - 하나의 TCP 연결 → N개 스트림 동시 실행
   - RTT (Round Trip Time) 제거
   - 예: 3개 요청 = 3×100ms (HTTP/1.1) → 100ms (HTTP/2)

2. 우선순위
   - 중요한 요청 먼저 처리
   - 리소스별 가중치 설정 가능

3. 서버 푸시
   - 클라이언트가 요청하지 않은 데이터도 미리 전송
   - 예: HTML 요청 → CSS/JS 자동 푸시

✅ gRPC 4가지 통신 패턴 선택

1. Unary: 일반 CRUD 작업
   - getProduct(id) → product
   - 가장 많이 사용

2. Server Streaming: 대량 데이터 조회
   - getProductsByCategory(cat) → stream Product
   - 클라이언트: 수신 준비, 서버: 여러 응답

3. Client Streaming: 대량 데이터 업로드
   - createProductBatch(stream) → response
   - 클라이언트: 여러 요청, 서버: 최종 응답

4. Bidirectional: 실시간 양방향
   - watchOrderStatus(stream) → stream
   - 채팅, 협업 편집, 실시간 알림

✅ gRPC vs REST 선택 결정 트리

Q1: 내부 서비스 간 통신?
  → YES: gRPC 고려
  → NO: REST 기본

Q2: 실시간 양방향 필요?
  → YES: gRPC (Server/Client/Bidirectional Streaming)
  → NO: 계속

Q3: 대역폭/성능이 중요?
  → YES: gRPC (Protobuf 바이너리)
  → NO: REST (간단성)

Q4: 브라우저 직접 호출?
  → YES: REST (gRPC-Web은 추가 복잡도)
  → NO: gRPC 가능

최종 선택:
- 내부 + 성능 + 실시간 → gRPC
- 공개 API + 브라우저 → REST
- 복잡한 쿼리 + 유연성 → GraphQL
```

---

## 🤔 생각해볼 문제

**Q1.** Order Service가 Product Service를 동시에 10개 상품에 대해 조회합니다. HTTP/1.1 vs HTTP/2에서 응답 시간 차이는? (각 조회 = 100ms)

<details>
<summary>해설 보기</summary>

**HTTP/1.1 (Keep-Alive):**
```
[요청1] → (100ms) → [응답1]
         → [요청2] → (100ms) → [응답2]
                   → [요청3] → (100ms) → [응답3]
                   ...
                   → [요청10] → (100ms) → [응답10]

총 시간: 10 × 100ms = 1000ms (순차 처리)
```

**HTTP/2 (멀티플렉싱):**
```
[요청1] [요청2] [요청3] ... [요청10]
  ↓        ↓       ↓          ↓
(동시 처리 - 병렬)
  ↓        ↓       ↓          ↓
[응답1] [응답2] [응답3] ... [응답10]

총 시간: 100ms (병렬 처리)

성능 향상: 1000ms → 100ms (10배!)
```

**실제 gRPC:**
```
Protobuf 직렬화: 각 조회 10ms (JSON 20ms)
HTTP/2 멀티플렉싱: 병렬 처리

예상 시간:
- 직렬화 (10개): 10ms
- 전송 (병렬): 100ms
- 역직렬화 (10개): 10ms
────────────────
합계: ~120ms (HTTP/1.1 REST는 1200ms+)

약 10배 향상!
```

</details>

**Q2.** Protocol Buffers에서 필드 번호가 왜 중요한가? 번호를 변경하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**필드 번호의 역할:**

Protocol Buffers는 필드 이름이 아니라 **필드 번호로 직렬화/역직렬화**합니다.

```protobuf
// v1.0
message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
}

직렬화 (메시지 → 바이너리):
User(id=1, name="Alice", email="alice@example.com")
→ [1][1바이트][2][5][Alice][3][19][alice@example.com]
  필드1, 길이, 필드2, 길이, 데이터...
```

**필드 번호 변경의 결과:**

```protobuf
// 버그: 필드 번호 변경
message User {
    int64 id = 1;
    string email = 2;      // ❌ 번호 변경 (3 → 2)
    string name = 3;       // ❌ 번호 변경 (2 → 3)
}

Server가 보낸 데이터:
[1][1][2][5][Alice][3][19][alice@example.com]
   ↑ id=1
       ↓ 필드2 (이름은 name이지만)
           ↑ "Alice"라는 데이터

Client가 받아서 역직렬화:
"필드2는 email이다!" → email = "Alice" ❌
"필드3은 name이다!" → name = "alice@example.com" ❌
```

**올바른 진화:**

```protobuf
// v2.0: 하위호환
message User {
    int64 id = 1;          // 번호 유지
    string name = 2;       // 번호 유지
    string email = 3;      // 번호 유지
    string phone = 4;      // 새 필드 추가 (새 번호)
    bool active = 5;       // 새 필드 추가 (새 번호)
}

구버전 클라이언트:
- 필드 4, 5는 무시 (알려지지 않은 필드)
- 필드 1, 2, 3은 정상 처리 ✅

신버전 클라이언트:
- 모든 필드 처리 ✅
```

**결론:**
- 필드 번호는 **영구적 식별자** (절대 변경 금지)
- 필드명 변경은 괜찮음 (번호가 같으면)
- 새 필드는 새 번호로 추가
- 필드 제거는 번호를 "예약" (나중 재사용 금지)

</details>

**Q3.** 로드 밸런서가 gRPC 서비스를 L4 (TCP)로 부하 분산하면 어떤 문제가 발생하는가?

<details>
<summary>해설 보기</summary>

**L4 (TCP) 부하 분산의 문제:**

```
로드 밸런서 (L4 - TCP 레벨)
  ↓
  ├→ Server 1
  ├→ Server 2
  └→ Server 3

gRPC 클라이언트가 요청:
[연결 1] → Server 1
  [요청1]
  [요청2]
  [요청3]

문제:
- L4 LB는 TCP 연결만 본다
- 모든 gRPC 요청 = 1개 TCP 연결
- Server 1만 계속 받음 (Server 2, 3은 유휴)
- 결과: 부하 불균형 ❌

┌─────────────────┐
│   LB (L4)        │
├─────────────────┤
│    [TCP 연결 1]  │→ Server 1 (100% 부하) 🔥
│    모든 요청이   │
│    이 연결 사용  │
└─────────────────┘
    Server 2 (0%)
    Server 3 (0%)
```

**L7 (HTTP/2) 부하 분산:**

```
로드 밸런서 (L7 - HTTP/2 레벨)
  ↓
  ├→ Server 1
  ├→ Server 2
  └→ Server 3

gRPC 클라이언트가 요청:
[연결 1 - 스트림 1, 3, 5]  → Server 1
[연결 1 - 스트림 2, 4, 6]  → Server 2
                            (또는 연결 2)

L7 LB가 HTTP/2 프레임 단위로 분산:
[Stream 1] ──→ Server 1
[Stream 2] ──→ Server 2
[Stream 3] ──→ Server 3
[Stream 4] ──→ Server 1
...

결과: 균등 분산 ✅
┌─────────────────┐
│   LB (L7)        │
├─────────────────┤
│ [스트림 1,4,7]  │→ Server 1 (33%)
│ [스트림 2,5,8]  │→ Server 2 (33%)
│ [스트림 3,6,9]  │→ Server 3 (34%)
└─────────────────┘
```

**해결 방법:**

1. **L7 로드 밸런서 사용**
   - Envoy, nginx (http2 지원)
   - Cloud Load Balancer (gRPC 모드)

2. **클라이언트 측 부하 분산**
   - grpc-go, grpc-java의 내장 LB
   - 클라이언트가 여러 서버로 직접 연결
   ```java
   ManagedChannel channel = ManagedChannelBuilder
       .forTarget("dns:///service:50051")
       .defaultLoadBalancingPolicy("round_robin")
       .usePlaintext()
       .build();
   ```

3. **Kubernetes Service 사용**
   - gRPC 서비스를 Headless Service로 설정
   - 클라이언트가 DNS로 모든 Pod 발견
   - 클라이언트 측 LB로 분산

**권장:** K8s 환경에서는 Headless Service + 클라이언트 측 LB

</details>

---

<div align="center">

**[⬅️ 이전: REST API 설계 — 서비스 간 계약과 버저닝](./02-rest-api-design.md)** | **[홈으로 🏠](../README.md)** | **[다음: 이벤트 기반 통신 — Kafka와 결합도 분리 ➡️](./04-event-driven-communication.md)**

</div>

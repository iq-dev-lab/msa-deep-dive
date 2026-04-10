# 02. REST API 설계 — 서비스 간 계약과 버저닝

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- REST API의 버저닝 방식(URI vs Header vs Content-Type)을 선택하는 기준은?
- Consumer-Driven Contract Testing(Pact)이 Breaking Change를 언제 감지하는가?
- 멱등성(Idempotency)을 설계하지 않으면 어떤 문제가 발생하고, 멱등 키로 어떻게 방지하는가?
- OpenAPI 스펙이 서비스 간 계약으로 작동하는 메커니즘은?
- API 진화 시 하위호환성을 유지하면서 새 필드를 추가하는 방법은?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

MSA 환경에서 서비스 간 REST API는 단순한 통신 프로토콜이 아니라 **서비스 간의 계약(Contract)**입니다. 이 계약이 깨지면 시스템 전체가 부분적으로 장애를 입게 됩니다.

예를 들어 Payment Service가 API를 변경하면서 `amount` 필드를 `paymentAmount`로 이름을 바꾸면, 이를 호출하는 모든 클라이언트(Order Service, Billing Service 등)는 즉시 장애를 입습니다. 동기 호출이 많은 시스템에서는 이런 Breaking Change가 연쇄적 장애로 이어집니다.

또한 네트워크의 불안정성 때문에 같은 요청이 중복 발생할 수 있습니다. 예를 들어 클라이언트가 "₩100 결제" 요청을 보냈는데 응답이 없으면 타임아웃 후 재요청합니다. 멱등성 설계가 없으면 두 번 청구될 수 있습니다.

MSA의 REST API 설계는 이런 문제들을 **배포 전에 감지하고, 런타임에 방지하는 메커니즘**을 포함해야 합니다.

---

## 😱 흔한 실수 (Before — Breaking Change 감지 안 됨)

```java
// ❌ 안티패턴 1: 버전 관리 없이 필드 변경
// Payment Service v1.0 (기존)
@RestController
@RequestMapping("/api/payments")
public class PaymentController {
    @PostMapping
    public ResponseEntity<PaymentResponse> createPayment(
        @RequestBody PaymentRequest request) {
        // amount 필드 사용
        BigDecimal amount = request.getAmount();
        processPayment(amount);
        return ResponseEntity.ok(new PaymentResponse(amount, "SUCCESS"));
    }
}

// Payment Service v2.0 (변경됨) - 혼자 배포
@PostMapping
public ResponseEntity<PaymentResponse> createPayment(
    @RequestBody PaymentRequest request) {
    // ❌ 필드명 변경: amount → paymentAmount
    // 클라이언트에게 알리지 않고 변경
    BigDecimal paymentAmount = request.getPaymentAmount(); // null 에러 발생!
    processPayment(paymentAmount);
    return ResponseEntity.ok(new PaymentResponse(paymentAmount, "SUCCESS"));
}

// Order Service (여전히 구버전 기대)
public void placeOrder(OrderRequest request) {
    PaymentRequest payment = new PaymentRequest(
        request.getTotalPrice()); // amount 필드 사용
    
    // Payment Service v2.0에 요청 → NullPointerException!
    PaymentResponse response = paymentService.createPayment(payment);
}

// 문제점:
// 1. 배포 시간이 엇갈리면 Order Service는 Payment Service 과정 사용 불가
// 2. 런타임 에러 발생 (NullPointerException)
// 3. 배포 전에 감지 불가능


// ❌ 안티패턴 2: 멱등성 없이 재시도 활성화
@PostMapping("/transfer")
public ResponseEntity<TransferResponse> transfer(
    @RequestBody TransferRequest request) {
    // 멱등성 미보장
    account1.debit(request.getAmount());
    account2.credit(request.getAmount());
    return ResponseEntity.ok(new TransferResponse("SUCCESS"));
}

// 클라이언트 재시도 코드
public void transferWithRetry(TransferRequest request) {
    try {
        paymentService.transfer(request);
    } catch (TimeoutException e) {
        // 응답이 없으면 재시도 (하지만 이미 처리됨!)
        paymentService.transfer(request);
        // 결과: 계좌에서 2배 인출, 수신자가 2배 수금 💥
    }
}
```

---

## ✨ 올바른 접근 (After — 계약 기반 설계)

```java
// ✅ 개선 1: 버전 관리 + Consumer-Driven Contract Testing

// Payment Service (OpenAPI 스펙으로 계약 정의)
@Configuration
public class OpenApiConfig {
    @Bean
    public OpenAPI paymentServiceApi() {
        return new OpenAPI()
            .info(new Info()
                .title("Payment Service API")
                .version("2.0.0")
                .description("결제 서비스 API"));
    }
}

// 하위호환 유지하면서 새 필드 추가
@RestController
@RequestMapping("/api/v1/payments")
public class PaymentControllerV1 {
    @PostMapping
    @Operation(summary = "결제 처리 - v1.0 호환")
    public ResponseEntity<PaymentResponse> createPayment(
        @RequestBody @Valid PaymentRequest request) {
        BigDecimal amount = request.getAmount();
        processPayment(amount);
        return ResponseEntity.ok(
            new PaymentResponse(amount, "SUCCESS"));
    }
}

// 새로운 필드를 지원하는 v2.0
@RestController
@RequestMapping("/api/v2/payments")
public class PaymentControllerV2 {
    @PostMapping
    @Operation(summary = "결제 처리 - v2.0 신규")
    public ResponseEntity<PaymentResponseV2> createPayment(
        @RequestBody @Valid PaymentRequestV2 request) {
        // v2.0에서 추가: idempotencyKey, metadata 등
        String idempotencyKey = request.getIdempotencyKey();
        
        // 멱등성 확인: 이미 처리된 요청이면 캐시된 응답 반환
        PaymentResponseV2 cached = idempotencyCache.get(idempotencyKey);
        if (cached != null) {
            return ResponseEntity.ok(cached);
        }
        
        BigDecimal amount = request.getAmount();
        PaymentResult result = processPayment(amount);
        
        PaymentResponseV2 response = new PaymentResponseV2(
            amount, "SUCCESS", result.getTransactionId());
        
        // 응답 캐시 (24시간)
        idempotencyCache.put(idempotencyKey, response, Duration.ofHours(24));
        
        return ResponseEntity.ok(response);
    }
}

// DTO: 하위호환 유지
@Data
class PaymentRequest {
    @JsonProperty("amount")
    private BigDecimal amount;  // v1.0 기존 필드
}

@Data
class PaymentRequestV2 {
    @JsonProperty("amount")
    private BigDecimal amount;
    
    @JsonProperty("idempotencyKey")  // 멱등성 키
    private String idempotencyKey;
    
    @JsonProperty("metadata")  // 새로운 선택 필드
    @JsonInclude(JsonInclude.Include.NON_NULL)
    private Map<String, String> metadata;
}

// ✅ 개선 2: Consumer-Driven Contract Testing
// (Pact 프레임워크 사용)

// Order Service의 테스트 (Consumer)
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "PaymentService", port = "8081")
public class OrderServicePaymentPactTest {
    
    @Pact(consumer = "OrderService", provider = "PaymentService")
    public RequestResponsePact createPaymentPact(PactDslWithProvider builder) {
        return builder
            .given("payment is processed successfully")
            .uponReceiving("a request to create payment")
            .path("/api/v2/payments")
            .method("POST")
            .body(new JsonObject()
                .put("amount", 10000)
                .put("idempotencyKey", "order-123-001"))
            .willRespondWith()
            .status(200)
            .body(new JsonObject()
                .put("status", "SUCCESS")
                .put("transactionId", "txn-456")
                .put("amount", 10000))
            .toPact();
    }
    
    @Test
    @PactVerification
    public void verifyCreatePaymentPact() {
        // Order Service의 실제 코드가 이 계약을 준수하는지 검증
        PaymentResponse response = orderService.createPayment(
            new PaymentRequest(10000, "order-123-001"));
        
        assertThat(response.getStatus()).isEqualTo("SUCCESS");
        assertThat(response.getAmount()).isEqualTo(10000);
    }
}

// Payment Service 입장에서 계약 검증
@Provider("PaymentService")
@PactFolder("pacts")
public class PaymentServiceProviderTest {
    
    @BeforeEach
    public void setUp() {
        // Payment Service 인스턴스 시작
        paymentService.start();
    }
    
    @State("payment is processed successfully")
    public void setupPaymentSuccess() {
        // 테스트 데이터 설정
    }
    
    @Test
    @PactVerification
    public void verifyPaymentServiceContract() {
        // Pact 파일에 정의된 모든 요청/응답을 검증
        // v1.0 → v2.0 마이그레이션 중 이 테스트가
        // Order Service 기대치를 만족하는지 확인
    }
}

// ✅ 개선 3: 멱등성 설계 (Idempotency Key)

@Configuration
public class IdempotencyConfig {
    @Bean
    public IdempotencyKeyValidator idempotencyKeyValidator() {
        return new IdempotencyKeyValidator();
    }
}

@RestController
@RequestMapping("/api/transfers")
public class TransferController {
    
    @Autowired private TransferService transferService;
    @Autowired private IdempotencyKeyCache cache;
    
    @PostMapping
    public ResponseEntity<TransferResponse> transfer(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody @Valid TransferRequest request) {
        
        // 1. 멱등성 키로 중복 요청 감지
        TransferResponse cached = cache.get(idempotencyKey);
        if (cached != null) {
            // 이미 처리된 요청 → 캐시된 응답 반환 (멱등성 보장)
            return ResponseEntity.status(208).body(cached); // 208 Already Reported
        }
        
        // 2. 새로운 요청 → 처리
        TransferResult result = transferService.transfer(request);
        
        TransferResponse response = new TransferResponse(
            result.getTransferId(),
            result.getStatus(),
            result.getAmount());
        
        // 3. 응답 캐시 (24시간 보관)
        cache.put(idempotencyKey, response, Duration.ofHours(24));
        
        return ResponseEntity.ok(response);
    }
}

// 멱등성 캐시 구현 (Redis 권장)
@Component
public class IdempotencyKeyCache {
    @Autowired private RedisTemplate<String, TransferResponse> redisTemplate;
    
    private static final String PREFIX = "idempotency:";
    
    public TransferResponse get(String key) {
        return redisTemplate.opsForValue().get(PREFIX + key);
    }
    
    public void put(String key, TransferResponse value, Duration ttl) {
        redisTemplate.opsForValue().set(
            PREFIX + key, value, ttl);
    }
}
```

---

## 🔬 내부 동작 원리 — REST API 진화와 계약

### 1. API 버저닝 전략 비교

```
전략 1: URI 경로 버저닝
  /api/v1/payments  → v1 사용
  /api/v2/payments  → v2 사용
  장점: 명확함, URL만 봐도 버전 알 수 있음
  단점: 코드 중복, v1 지원 비용 높음

전략 2: Accept Header 버저닝
  GET /api/payments
  Accept: application/vnd.payment.v1+json
  Accept: application/vnd.payment.v2+json
  장점: 하나의 엔드포인트로 버전 관리
  단점: 복잡함, 브라우저 테스트 어려움

전략 3: 쿼리 파라미터 버저닝 (권장하지 않음)
  /api/payments?version=2
  단점: 캐싱 문제, 부정확함

[권장] 하이브리드: 주요 기능은 v1/v2, 수정 사항은 쿼리/헤더
  /api/v1/payments (메인 엔드포인트)
  /api/v1/payments?format=detailed (선택적 확장)
```

### 2. Breaking Change의 정의와 감지

```
Breaking Change의 예:
❌ 필수 필드 추가
   v1.0: POST /api/payments { amount: 10000 }
   v2.0: POST /api/payments { amount: 10000, currency: "USD" }
        currency 필수 → 기존 클라이언트 실패

❌ 필드 제거 또는 이름 변경
   v1.0: amount (사용 중)
   v2.0: paymentAmount (이름 변경)

❌ 응답 구조 변경
   v1.0: { "status": "SUCCESS", "amount": 10000 }
   v2.0: { "result": { "status": "SUCCESS", ... } }

비Breaking Change (하위호환):
✅ 필드 추가 (선택적)
   v2.0: { amount, currency: "USD" } // currency는 선택적
   
✅ 새로운 HTTP 상태 코드 추가
   v1.0: 200 성공, 400 실패
   v2.0: 200 성공, 201 생성, 400 실패, 422 검증 실패
   
✅ 응답에 새로운 필드 추가
   v1.0: { "status": "SUCCESS" }
   v2.0: { "status": "SUCCESS", "transactionId": "..." }
        // 클라이언트는 무시하고 진행 가능

Pact의 감지 메커니즘:
[Consumer 테스트 실행]
  ↓
[기대 계약 정의 (Pact 파일 생성)]
  ↓
[Provider 구현 변경]
  ↓
[Provider 테스트 실행 (Pact 검증)]
  ↓
Breaking Change 감지 → 배포 전 중단! ✋
```

### 3. 멱등성 설계의 필요성

```
시나리오 1: 멱등성 없음
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ [POST /transfer?amount=1000]
       │ (timeout → 요청이 도착했나?)
       │
       ▼
┌──────────────────────┐
│ Server (처리함)       │
│ 계좌1: -1000         │
│ 계좌2: +1000         │
└──────┬───────────────┘
       │ [응답 없음 (네트워크 끊김)]
       │
┌──────▼─────────┐
│   Client       │
│ 재시도 (같은 요청) │
└──────┬─────────┘
       │ [POST /transfer?amount=1000]
       │ (retry)
       ▼
┌──────────────────────┐
│ Server (또 처리함!)  │
│ 계좌1: -2000 💥      │
│ 계좌2: +2000 💥      │
└──────────────────────┘

결과: 같은 금액이 두 번 전송됨 (비멱등적)

────────────────────────────────────────

시나리오 2: 멱등성 있음
┌─────────────────────────────┐
│   Client                    │
│ Idempotency-Key: uuid-001   │
└──────┬──────────────────────┘
       │ [POST /transfer, Key: uuid-001]
       │
       ▼
┌─────────────────────────────────────────┐
│ Server                                  │
│ 1. 캐시 확인: Key "uuid-001" 없음       │
│ 2. 처리 실행                             │
│    계좌1: -1000                         │
│    계좌2: +1000                         │
│ 3. 응답 캐시                             │
│    캐시["uuid-001"] = {status: SUCCESS} │
└──────┬──────────────────────────────────┘
       │ [응답 없음 (네트워크 끊김)]
       │
┌──────▼──────────────────────┐
│   Client                    │
│ 재시도 (같은 Key)           │
└──────┬──────────────────────┘
       │ [POST /transfer, Key: uuid-001]
       │
       ▼
┌─────────────────────────────────────────┐
│ Server                                  │
│ 1. 캐시 확인: Key "uuid-001" 있음! ✅   │
│ 2. 캐시된 응답 반환                     │
│    {status: SUCCESS} (처리 안 함)       │
└──────┬──────────────────────────────────┘
       │ [응답 전송]
       │
┌──────▼──────────────┐
│   Client            │
│ 수신 완료           │
└─────────────────────┘

결과: 같은 금액이 한 번만 전송됨 (멱등적) ✅
```

### 4. OpenAPI 스펙의 역할

```
OpenAPI 스펙 = 서비스 간 계약의 명시적 정의

@GetMapping("/users/{id}")
@Operation(summary = "사용자 조회",
           description = "ID로 사용자 정보를 조회합니다")
@ApiResponse(responseCode = "200", 
             description = "사용자 정보",
             content = @Content(schema = @Schema(implementation = User.class)))
@ApiResponse(responseCode = "404", description = "사용자 없음")
@ApiResponse(responseCode = "500", description = "서버 오류")
public ResponseEntity<User> getUser(
    @PathVariable @Parameter(description = "사용자 ID") Long id) {
    return ResponseEntity.ok(userService.getUser(id));
}

// OpenAPI 3.0 스펙 자동 생성:
openapi: 3.0.0
paths:
  /users/{id}:
    get:
      summary: 사용자 조회
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: 사용자 정보
          content:
            application/json:
              schema:
                type: object
                properties:
                  id: {type: integer}
                  name: {type: string}
                  email: {type: string}
        '404':
          description: 사용자 없음
        '500':
          description: 서버 오류

// 다른 팀이 이 스펙으로 클라이언트 코드 생성 (Swagger Codegen)
// 스펙이 변경되면 → 클라이언트 재생성 → 호환성 자동 검증
```

---

## 💻 코드 예제 — REST API 설계 실전

### OpenAPI/Swagger 설정
```java
@Configuration
public class OpenApiConfiguration {
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("결제 서비스 API")
                .version("2.1.0")
                .description("MSA 환경의 결제 서비스")
                .contact(new Contact()
                    .name("Payment Team")
                    .email("payment@company.com")))
            .externalDocs(new ExternalDocumentation()
                .description("Payment API Docs")
                .url("https://docs.company.com/payment"));
    }
}

@RestController
@RequestMapping("/api/v2/payments")
@Tag(name = "Payments", description = "결제 API")
public class PaymentController {
    
    @PostMapping
    @Operation(
        summary = "결제 처리",
        description = "새로운 결제를 처리합니다")
    @ApiResponse(
        responseCode = "200",
        description = "결제 성공",
        content = @Content(schema = @Schema(implementation = PaymentResponse.class)))
    @ApiResponse(responseCode = "400", description = "잘못된 요청")
    @ApiResponse(responseCode = "409", description = "중복된 멱등성 키")
    public ResponseEntity<PaymentResponse> createPayment(
        @RequestHeader(value = "Idempotency-Key", required = true)
        @Parameter(description = "멱등성 키 (UUID 권장)")
        String idempotencyKey,
        
        @RequestBody
        @Valid
        PaymentRequest request) {
        // ...
    }
}
```

### 멱등성 키 인터셉터
```java
@Component
public class IdempotencyInterceptor implements HandlerInterceptor {
    
    @Autowired private IdempotencyKeyCache cache;
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) {
        // POST, PUT, PATCH만 멱등성 검증
        if (!"GET".equalsIgnoreCase(request.getMethod()) &&
            !"DELETE".equalsIgnoreCase(request.getMethod())) {
            
            String idempotencyKey = request.getHeader("Idempotency-Key");
            if (idempotencyKey == null || idempotencyKey.isEmpty()) {
                response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
                return false;
            }
            
            // 멱등성 키 형식 검증 (UUID)
            if (!isValidUUID(idempotencyKey)) {
                response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
                return false;
            }
        }
        return true;
    }
    
    private boolean isValidUUID(String value) {
        try {
            UUID.fromString(value);
            return true;
        } catch (IllegalArgumentException e) {
            return false;
        }
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired private IdempotencyInterceptor interceptor;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(interceptor);
    }
}
```

### Pact Contract Test
```java
// build.gradle
testImplementation 'au.com.dius.pact.consumer:pact-junit5:4.3.x'

@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "PaymentService", port = "8888")
public class PaymentServicePactTest {
    
    private PaymentServiceClient client;
    
    @BeforeEach
    void setUp(PactTestExecutionContext context) {
        client = new PaymentServiceClient("http://localhost:8888");
    }
    
    @Pact(consumer = "OrderService", provider = "PaymentService")
    public RequestResponsePact createPaymentPact(PactDslWithProvider builder) {
        Map<String, String> headers = new HashMap<>();
        headers.put("Content-Type", "application/json");
        
        return builder
            .given("payment can be processed")
            .uponReceiving("a payment request")
            .path("/api/v2/payments")
            .method("POST")
            .headers(headers)
            .body(new JsonObject()
                .put("amount", 50000)
                .put("currency", "KRW")
                .put("idempotencyKey", "order-abc123-001"))
            .willRespondWith()
            .status(200)
            .body(new JsonObject()
                .put("transactionId", "txn-12345")
                .put("status", "SUCCESS")
                .put("amount", 50000)
                .put("procesedAt", "2024-01-15T10:30:00Z"))
            .toPact();
    }
    
    @Test
    @PactVerification
    void verifyPaymentCreation() {
        PaymentRequest request = new PaymentRequest(
            50000, "KRW", "order-abc123-001");
        PaymentResponse response = client.createPayment(request);
        
        assertThat(response.getStatus()).isEqualTo("SUCCESS");
        assertThat(response.getAmount()).isEqualTo(50000);
        assertThat(response.getTransactionId()).startsWith("txn-");
    }
}
```

---

## 📊 패턴 비교 — API 버저닝 방식

| 기준 | URI 경로 버저닝 | Header 버저닝 | Content-Type 버저닝 |
|------|---------------|-------------|-----------------|
| **명확성** | 매우 명확 | 어려움 | 보통 |
| **URL 가독성** | 좋음 (/v1, /v2) | 나쁨 | 좋음 |
| **캐싱** | 쉬움 | 복잡함 | 보통 |
| **브라우저 테스트** | 쉬움 | 어려움 | 어려움 |
| **코드 중복** | 많음 | 적음 | 적음 |
| **마이그레이션** | 단계별 가능 | 점진적 | 점진적 |
| **권장 시점** | 메이저 버전 | 마이너 버전 | 선택적 |

---

## ⚖️ 트레이드오프

```
REST API 버저닝의 트레이드오프:

URI 경로 버저닝 (/v1, /v2)
✅ 장점
  - 명확성: URL만 봐도 버전 알 수 있음
  - 캐싱: CDN/프록시 캐싱 용이
  - 테스트: curl, 브라우저에서 직접 테스트
  
❌ 비용
  - 코드 중복: v1, v2 동시 유지
  - 메모리: 두 버전 모두 메모리 상주
  - 복잡도: 라우팅, 검증, 비즈니스 로직 중복

────────────────────────────────────────

Header 버저닝
✅ 장점
  - 단일 코드베이스: /api/payments 하나만 유지
  - 자동 협상: Accept 헤더로 콘텐츠 유형 자동 선택
  - 하위호환: 기존 클라이언트 영향 최소화
  
❌ 비용
  - 복잡함: 헤더 기반 라우팅 로직
  - 테스트: curl -H 필요, 브라우저 테스트 어려움
  - 캐싱: 헤더 기반 캐싱 설정 필요

────────────────────────────────────────

멱등성 구현
✅ 장점
  - 재시도 안전성: 네트워크 실패 → 자동 재시도 가능
  - 중복 처리 방지: 같은 요청 2번 = 1번 처리
  
❌ 비용
  - 캐시 관리: 멱등성 키와 응답 캐시 유지
  - 메모리: 24시간 캐시 (보통 Redis)
  - 복잡도: 캐시 키 생성, TTL 관리
  
장기 비용: 재시도 안전성 > 캐시 관리 비용

────────────────────────────────────────

Consumer-Driven Contract Testing (Pact)
✅ 장점
  - 조기 감지: 배포 전 Breaking Change 감지
  - 독립 배포: 서비스 간 배포 순서 자유
  - 명시적 계약: 기대치 문서화
  
❌ 비용
  - 테스트 작성: 각 Consumer마다 Pact 작성
  - 유지보수: 계약 변경 시 테스트 갱신
  - 학습곡선: Pact 프레임워크 학습 필요
  
장기 관점: 초기 비용 > 배포 장애 방지 효과
```

---

## 📌 핵심 정리

```
✅ API 버저닝 선택 기준

1. 메이저 변경 (Breaking Change)
   → URI 경로 버저닝 (/v1, /v2)
   → 병렬 지원 기간: 2-3개월 권장
   
2. 마이너 변경 (선택적 필드 추가)
   → Content-Type 또는 Accept 헤더
   → 기존 클라이언트 자동 호환
   
3. 점진적 진화
   → 하위호환 설계 + 새 필드 추가
   → 기존 클라이언트 영향 최소화

✅ 멱등성 설계 필수 항목

1. 상태 변경 요청 (POST, PUT, PATCH)
   → Idempotency-Key 헤더 필수
   → UUID v4 권장
   
2. 응답 캐싱
   → Redis 사용 (24시간 TTL)
   → 캐시 키: Idempotency-Key
   
3. 재시도 로직
   → 클라이언트 타임아웃 시 자동 재시도
   → 408, 429, 5xx 상태코드에만 재시도

✅ Contract Testing 구현

1. Consumer 입장 (Order Service)
   → 기대하는 Payment API 스펙 정의 (Pact)
   → 로컬 테스트 실행
   
2. Provider 입장 (Payment Service)
   → Consumer의 Pact 파일 가져오기
   → 스펙 준수 여부 검증
   
3. CI/CD 통합
   → 배포 전 Pact 검증 자동 실행
   → Breaking Change 감지 시 배포 중단

✅ API 진화 체크리스트

□ OpenAPI 스펙 정의 (자동 생성)
□ 필드 추가 시 선택적 처리
□ 새 엔드포인트로 메이저 변경 구현
□ Consumer-Driven Contract Test 작성
□ 멱등성 키 설계 (POST/PUT/PATCH)
□ 다양한 HTTP 상태 코드 정의
□ 오류 응답 스키마 표준화
□ API 문서화 (Swagger/OpenAPI)
```

---

## 🤔 생각해볼 문제

**Q1.** Order Service가 Payment Service를 호출합니다. 만약 Payment Service가 API 응답을 변경했는데 (필드 이름 변경), Order Service는 아직 배포되지 않았습니다. 이 상황에서 Consumer-Driven Contract Testing이 어떻게 도움이 되는지?

<details>
<summary>해설 보기</summary>

**Pact의 보호 메커니즘:**

1. **배포 전 검증**
   ```
   Order Service의 Pact 파일:
   - 기대하는 Payment API 응답
     { "status": "SUCCESS", "transactionId": "..." }
   
   Payment Service 배포 전:
   - Pact 검증 실행
   - "transactionId 필드가 Pact에 있나?" 확인
   - ❌ 필드 제거됨! → 배포 중단
   ```

2. **Breaking Change 감지 시점**
   ```
   Payment Service 개발자가 API 변경:
   - { "status": "SUCCESS", "transactionId": "..." }
   - → { "status": "SUCCESS", "txId": "..." }  // 필드명 변경!
   
   CI/CD 파이프라인:
   [코드 푸시] → [테스트 실행] → [Pact 검증]
                                   ↓
                              ❌ Order Service 기대치 위반
                              ❌ 배포 차단
                              ✅ 개발자에게 알림
   ```

3. **올바른 마이그레이션 절차**
   ```
   Step 1: Payment Service v2
   - 새 필드 추가: { transactionId, txId }
   - 기존 필드 유지: transactionId
   - Pact 검증 ✅ 통과
   
   Step 2: Order Service 배포
   - txId 사용으로 변경
   - 기존 transactionId 호환성 유지
   
   Step 3: Payment Service v3 (3개월 후)
   - 레거시 필드 제거: transactionId
   - Order Service 이미 변경됨 ✅
   ```

**결론:** Pact는 배포 순서의 자유도를 보장하면서 Breaking Change를 조기에 감지

</details>

**Q2.** 클라이언트가 같은 멱등성 키로 2시간 간격으로 두 번 요청했습니다. 첫 번째는 성공, 두 번째는 캐시에서 동일 응답을 반환했습니다. 이게 맞는 동작인가?

<details>
<summary>해설 보기</summary>

**멱등성 키의 TTL 트레이드오프:**

```
현재 설정: 24시간 TTL
요청 1: [0시간] Key: "order-123" → 처리 → 캐시 저장
요청 2: [2시간] Key: "order-123" → 캐시에서 응답 반환

결과:
✅ 멱등성 보장: 같은 결과 반환
❓ 의도 확인: 정말 의도한 중복 요청?
```

**멱등성 키 설계 원칙:**

1. **언제까지 중복이라고 봐야 하는가?**
   ```
   - 네트워크 재시도: 초~분 단위 (5분)
   - 사용자 재시도: 분~시간 단위 (1시간)
   - 비정상적 재시도: 시간~일 단위
   
   → 24시간 TTL은 "안전하게 과하게" 설정한 것
   ```

2. **실제 구현:**
   ```
   // 금융거래: 24시간 보관 (법적 요구사항)
   redisTemplate.expire(key, Duration.ofHours(24));
   
   // 일반 API: 1시간 (비용 vs 안전성 균형)
   redisTemplate.expire(key, Duration.ofHours(1));
   
   // 실시간 시스템: 5분 (타임아웃 재시도 커버)
   redisTemplate.expire(key, Duration.ofMinutes(5));
   ```

3. **멱등성 키 생성 지침:**
   ```
   // 클라이언트가 생성 (서버 아님!)
   // 같은 비즈니스 작업 = 같은 키
   
   order-abc123-attempt-1    // 첫 시도
   order-abc123-attempt-2    // 재시도 (같은 키가 아님!)
   
   // 올바른 방식
   order-abc123              // 비즈니스 식별자
   // 재시도해도 같은 키 사용
   ```

**결론:** 2시간 간격 재시도는 비정상이므로, 요청 맥락을 확인 후 멱등성 키 TTL 조정 필요

</details>

**Q3.** API v1과 v2를 동시에 지원하려면, 데이터베이스는 어떻게 관리해야 할까?

<details>
<summary>해설 보기</summary>

**버전별 데이터베이스 관리 전략:**

```
선택지 1: 공유 데이터베이스 (권장)
┌───────────────────────────────────────┐
│         Payment Service DB            │
│  (v1, v2 모두 같은 스키마 사용)        │
├───────────────────────────────────────┤
│ id | amount | status | transactionId  │
│    |        |        |                 │
└───────────────────────────────────────┘
  ↑               ↑
  │               │
/api/v1/payments  /api/v2/payments
(같은 데이터 읽음/씀)

구현:
@RestController
public class PaymentController {
    @PostMapping("/api/v1/payments")
    public ResponseEntity createPaymentV1(...) {
        // v1 요청 → DB 저장
        payment = paymentRepository.save(dto);
        return new PaymentResponseV1(payment);  // v1 형식
    }
    
    @PostMapping("/api/v2/payments")
    public ResponseEntity createPaymentV2(...) {
        // v2 요청 → 같은 DB 저장
        payment = paymentRepository.save(dto);
        return new PaymentResponseV2(payment); // v2 형식
    }
}

장점:
✅ 데이터 동기화 문제 없음
✅ 스키마 관리 단순
✅ 마이그레이션 용이

────────────────────────────────────────

선택지 2: 분리된 스키마 (권장하지 않음)
┌─────────────────────────┬──────────────────────────┐
│   Payment v1 DB         │    Payment v2 DB         │
│ id|amount|status|txId   │ id|amt|status|txnId|meta │
└─────────────────────────┴──────────────────────────┘

문제점:
❌ 데이터 동기화 필요
❌ 트랜잭션 복잡 (2단계 커밋)
❌ 일관성 보장 어려움

────────────────────────────────────────

선택지 3: 하이브리드 (전환기)
1단계: 공유 DB (v1 + v2 운영)
2단계: v1 트래픽 감소 (90% → 10%)
3단계: v1 종료, v2만 운영
```

**스키마 진화 사례:**

```sql
-- v1.0 스키마
CREATE TABLE payment (
    id BIGINT PRIMARY KEY,
    amount DECIMAL(10,2),
    status VARCHAR(20),
    created_at TIMESTAMP
);

-- v2.0 추가 (하위호환)
ALTER TABLE payment ADD COLUMN (
    transaction_id VARCHAR(50),      -- v2에서 필수
    currency VARCHAR(3),              -- v2에서 필수
    metadata JSON,                    -- v2에서 선택
    idempotency_key VARCHAR(100),    -- v2에서 필수
    UNIQUE KEY uk_idempotency_key (idempotency_key)
);

-- v1 쿼리: 기존대로 작동
SELECT id, amount, status FROM payment WHERE id = 123;

-- v2 쿼리: 새 필드 사용
SELECT id, amount, status, transaction_id, currency 
FROM payment WHERE idempotency_key = 'order-abc';
```

**결론:** 공유 DB + 선택적 필드 추가가 가장 실용적

</details>

---

<div align="center">

**[⬅️ 이전: 동기 통신 vs 비동기 통신 — 선택 기준](./01-sync-vs-async.md)** | **[홈으로 🏠](../README.md)** | **[다음: gRPC 완전 분해 — Protocol Buffers와 HTTP/2 ➡️](./03-grpc-deep-dive.md)**

</div>

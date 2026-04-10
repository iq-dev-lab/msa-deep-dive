# 05. Fallback 전략 — 기능 축소 운영 설계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Fallback 전략의 3가지 유형(기본값, 캐시, 기능 축소)은 각각 무엇이며 언제 사용하는가?
- Fallback이 마스킹해서는 안 되는 오류 유형은 무엇인가?
- Fallback Chain이란 무엇이고, 1차 Fallback이 실패했을 때 어떻게 처리하는가?
- Resilience4j의 @CircuitBreaker(fallbackMethod)는 어떻게 동작하는가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

Circuit Breaker, Retry, Timeout 등으로 장애를 인지했다고 해서 시스템이 자동으로 복구되는 것은 아닙니다. 장애 상황에서 사용자 경험을 유지하거나 최소한 우아하게 서비스를 축소하려면 Fallback 전략이 필수입니다.

예를 들어, 추천 서비스가 다운되었을 때 "추천 서비스 이용 불가" 메시지를 보여주기보다는, 인기 상품 목록을 추천으로 보여주거나, 캐시된 지난주 추천 데이터를 보여주는 것이 훨씬 더 좋은 사용자 경험입니다. 이것이 Fallback 전략입니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ Fallback 없이 장애를 그대로 반환
@Service
public class ProductService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    public List<Product> getRecommendedProducts(String userId) {
        try {
            // 추천 서비스 호출 (Circuit Breaker 적용)
            List<Product> recommendations = restTemplate.getForObject(
                "http://recommendation-service/api/recommend?userId=" + userId,
                List.class
            );
            return recommendations;
        } catch (Exception e) {
            // ❌ 문제: 예외를 그대로 던짐
            // "추천 서비스를 이용할 수 없습니다" 에러 페이지 표시
            throw new RuntimeException("추천 서비스 이용 불가", e);
        }
    }
    
    public OrderResult createOrder(String customerId, String productId, int quantity) {
        try {
            // 결제 서비스 호출
            PaymentResult payment = restTemplate.postForObject(
                "http://payment-service/api/pay",
                new PaymentRequest(customerId, productId, quantity),
                PaymentResult.class
            );
            
            if (payment.isSuccess()) {
                return OrderResult.success();
            } else {
                // ❌ 문제: 결제 실패를 그대로 반환 (마스킹 불가)
                return OrderResult.failed(payment.getErrorMessage());
            }
        } catch (Exception e) {
            throw new RuntimeException("주문 생성 실패", e);
        }
    }
}

// 사용자 경험:
// 1. 추천 서비스 장애 → "추천 서비스를 이용할 수 없습니다" 에러 표시
//    (사용자가 상품을 추천받을 수 없음, 나쁜 경험)
// 2. 결제 서비스 장애 → 결제 불가 메시지 표시
//    (사용자가 구매할 수 없음, 매우 나쁜 경험)
```

---

## ✨ 올바른 접근 (After)

```java
// ✅ Fallback 전략으로 기능 축소 운영
@Service
public class ProductService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @CircuitBreaker(
        name = "recommendation-service",
        fallbackMethod = "recommendedProductsFallback"
    )
    public List<Product> getRecommendedProducts(String userId) {
        // 추천 서비스 호출
        List<Product> recommendations = restTemplate.getForObject(
            "http://recommendation-service/api/recommend?userId=" + userId,
            List.class
        );
        
        // 성공 시 캐시에 저장
        redisTemplate.opsForValue().set(
            "recommendations:" + userId,
            serializeToJson(recommendations),
            Duration.ofHours(1)
        );
        
        return recommendations;
    }
    
    // Fallback 1: 캐시된 추천 데이터 반환
    public List<Product> recommendedProductsFallback(String userId, Exception ex) {
        String cached = redisTemplate.opsForValue().get("recommendations:" + userId);
        
        if (cached != null) {
            return deserializeFromJson(cached);
        }
        
        // Fallback 2: 캐시도 없으면 인기 상품 반환
        return getPopularProducts();
    }
    
    // 보조 Fallback: 인기 상품 목록 (캐시 불가능할 때)
    private List<Product> getPopularProducts() {
        return List.of(
            new Product("001", "베스트셀러 1", 10000),
            new Product("002", "베스트셀러 2", 20000),
            new Product("003", "베스트셀러 3", 15000)
        );
    }
    
    // 결제는 마스킹하면 안 됨
    public OrderResult createOrder(String customerId, String productId, int quantity) {
        try {
            PaymentResult payment = restTemplate.postForObject(
                "http://payment-service/api/pay",
                new PaymentRequest(customerId, productId, quantity),
                PaymentResult.class
            );
            
            if (payment.isSuccess()) {
                return OrderResult.success(payment.getTransactionId());
            } else {
                // ✅ 결제 실패는 사용자에게 알림 (마스킹 불가)
                return OrderResult.failed(payment.getErrorMessage());
            }
        } catch (Exception e) {
            // 결제 서비스 장애도 마스킹하면 안 됨
            // 사용자에게 명확하게 전달
            return OrderResult.failed("결제 서비스 일시 점검 중입니다. 나중에 재시도해주세요.");
        }
    }
}

// 개선된 사용자 경험:
// 1. 추천 서비스 장애 → 캐시된 지난주 추천 표시 (또는 인기 상품)
//    (사용자가 여전히 상품을 탐색할 수 있음, 좋은 경험)
// 2. 결제 서비스 장애 → "일시 점검 중이므로 잠시 후 재시도해주세요" 안내
//    (명확한 상황 설명, 신뢰도 유지)
```

---

## 🔬 내부 동작 원리 — Fallback 전략 분류 및 적용

### 1. 기본값 반환 (Default Response)

```
특징: 의미 있는 기본값을 반환하여 기능을 유지

정상 상황:
서비스 호출 성공 → 실제 데이터 반환
[추천 서비스] → [A상품, B상품, C상품]

장애 상황:
서비스 호출 실패 → 기본값 반환
[추천 서비스 다운] → [인기 상품 기본 추천]
```

**적용 예시:**

```java
// 조회 서비스에 적합
@CircuitBreaker(
    name = "recommendation-service",
    fallbackMethod = "emptyRecommendations"
)
public List<Product> getRecommendations(String userId) {
    return restTemplate.getForObject(
        "http://recommendation-service/api/recommend?userId=" + userId,
        List.class
    );
}

// 기본값: 빈 리스트 또는 기본 추천
public List<Product> emptyRecommendations(String userId, Exception ex) {
    // 방법 1: 빈 리스트
    return Collections.emptyList();
    
    // 방법 2: 기본 추천 (더 나음)
    return getDefaultRecommendations();
}

private List<Product> getDefaultRecommendations() {
    return List.of(
        new Product("NEW", "신규 상품"),
        new Product("BEST", "베스트셀러"),
        new Product("DISCOUNT", "할인 상품")
    );
}
```

**장점:** 간단, 빠름
**단점:** 개인화 없음, 사용자 경험 약간 저하

### 2. 캐시 데이터 반환 (Stale Cache)

```
특징: 예전 데이터를 사용하여 일부 정확성 희생, 서비스 유지

정상 상황:
[서비스 호출] → [최신 데이터] → [캐시 저장] → [응답]
시간 T: 추천 = [A, B, C]

5분 후:

장애 상황 (T+5min):
[서비스 다운] → [캐시에서 읽음] → [응답]
반환값: 5분 전 캐시 = [A, B, C] (약간 오래된 데이터)

사용자 관점:
"추천이 약간 오래된 것 같지만, 서비스는 여전히 작동한다"
```

**구현:**

```java
@Service
public class CacheAwareFallbackService {
    
    private static final String CACHE_PREFIX = "fallback:";
    private static final int CACHE_TTL_MINUTES = 60;
    
    @Autowired
    private RedisTemplate<String, String> redis;
    
    @CircuitBreaker(
        name = "user-profile-service",
        fallbackMethod = "getProfileFromCache"
    )
    public UserProfile getUserProfile(String userId) {
        UserProfile profile = restTemplate.getForObject(
            "http://user-service/api/users/" + userId,
            UserProfile.class
        );
        
        // 성공 시 캐시 업데이트
        cacheProfile(userId, profile);
        return profile;
    }
    
    // Fallback: 캐시에서 읽음
    public UserProfile getProfileFromCache(String userId, Exception ex) {
        String cachedData = redis.opsForValue().get(
            CACHE_PREFIX + userId
        );
        
        if (cachedData != null) {
            UserProfile profile = objectMapper.readValue(cachedData, UserProfile.class);
            profile.setStatus("CACHED");  // 캐시 데이터임을 표시
            return profile;
        }
        
        // 캐시도 없으면 최소한의 정보 반환
        return UserProfile.minimal(userId);
    }
    
    private void cacheProfile(String userId, UserProfile profile) {
        try {
            String json = objectMapper.writeValueAsString(profile);
            redis.opsForValue().set(
                CACHE_PREFIX + userId,
                json,
                Duration.ofMinutes(CACHE_TTL_MINUTES)
            );
        } catch (Exception e) {
            // 캐시 실패는 무시
            logger.warn("프로필 캐싱 실패: {}", userId);
        }
    }
}
```

**장점:** 사용자 경험 유지, 최신성과 가용성의 트레이드오프
**단점:** 캐시 인프라 필요, 오래된 데이터 제공

### 3. 기능 축소 운영 (Graceful Degradation)

```
특징: 핵심 기능은 유지하면서 부가 기능만 제한

정상 상황:
┌─────────────────────────┐
│ 핵심: 상품 조회, 장바구니 │
│ 부가: 추천, 리뷰, 배송예상 │
└─────────────────────────┘
모든 기능 정상 작동

장애 상황 (결제 서비스 장애):
┌─────────────────────────┐
│ 핵심: 상품 조회, 장바구니 │
│ 부가: [비활성화]          │
│ 결제: [조회만 가능]       │
└─────────────────────────┘
핵심 기능만 유지, 결제 불가
```

**구현:**

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private RecommendationService recommendationService;
    
    @GetMapping("/{orderId}")
    public OrderResponse getOrder(@PathVariable String orderId) {
        // 핵심 기능: 주문 조회 (필수)
        Order order = orderService.getOrder(orderId);
        
        OrderResponse response = new OrderResponse()
            .setOrder(order)
            .setDeliverable(true);
        
        // 부가 기능 1: 추천 상품 (선택)
        try {
            List<Product> recommendations = recommendationService
                .getRelatedProducts(order.getProductId());
            response.setRecommendations(recommendations);
        } catch (Exception e) {
            // 추천 실패는 무시 (기능 축소)
            logger.debug("추천 조회 실패, 계속 진행: {}", e.getMessage());
            response.setRecommendations(Collections.emptyList());
        }
        
        // 부가 기능 2: 배송 추적 (선택)
        try {
            ShippingInfo shipping = orderService.getShippingInfo(orderId);
            response.setShipping(shipping);
        } catch (Exception e) {
            logger.debug("배송 정보 조회 실패, 계속 진행");
            response.setShipping(null);
        }
        
        return response;
    }
    
    @PostMapping
    public OrderResponse createOrder(@RequestBody OrderRequest request) {
        // 핵심 기능: 주문 생성, 결제 처리 (필수)
        // ❌ 이 부분에서 장애나면 전체 실패 (마스킹 불가)
        PaymentResult payment = orderService.processPayment(request);
        
        if (!payment.isSuccess()) {
            throw new PaymentFailedException("결제에 실패했습니다: " + 
                payment.getErrorMessage());
        }
        
        Order order = orderService.createOrder(request, payment);
        
        OrderResponse response = new OrderResponse()
            .setOrder(order)
            .setPaymentStatus("SUCCESS");
        
        // 부가 기능: 마일리지 적립 (선택)
        try {
            orderService.awardPoints(order.getCustomerId(), order.getTotalPrice());
            response.setPointsAwarded(calculatePoints(order.getTotalPrice()));
        } catch (Exception e) {
            // 마일리지 적립 실패는 무시
            logger.warn("마일리지 적립 실패: {}", order.getCustomerId());
            response.setPointsAwarded(0);
        }
        
        return response;
    }
}
```

**장점:** 사용자가 시스템 장애를 느끼지 않음, 핵심 기능 유지
**단점:** 구현 복잡도 높음, 어떤 기능을 축소할지 판단 필요

---

## 💻 Fallback Chain과 고급 구현

### Fallback Chain: 1차 실패 시 2차로 넘어가기

```
호출 흐름:

[실제 서비스 호출]
    │
    ├─ 성공? → [응답 반환]
    │
    └─ 실패? → [Fallback 1 시도]
         │
         ├─ 성공? → [응답 반환]
         │
         └─ 실패? → [Fallback 2 시도]
              │
              ├─ 성공? → [응답 반환]
              │
              └─ 실패? → [최종 기본값]
```

**구현:**

```java
@Service
public class FallbackChainService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Autowired
    private RedisTemplate<String, String> redis;
    
    @CircuitBreaker(
        name = "recommendation-service",
        fallbackMethod = "recommendationChainFallback1"
    )
    public List<Product> getRecommendations(String userId) {
        List<Product> recommendations = restTemplate.getForObject(
            "http://recommendation-service/api/recommend?userId=" + userId,
            List.class
        );
        
        // 성공 시 캐시 저장
        redis.opsForValue().set(
            "recommendations:" + userId,
            serializeToJson(recommendations),
            Duration.ofHours(1)
        );
        
        return recommendations;
    }
    
    // Fallback 1: 캐시된 데이터
    public List<Product> recommendationChainFallback1(
        String userId,
        Exception ex
    ) {
        logger.warn("추천 서비스 실패, 캐시 확인: {}", userId);
        
        String cached = redis.opsForValue().get("recommendations:" + userId);
        if (cached != null) {
            logger.info("캐시된 추천 데이터 사용: {}", userId);
            return deserializeFromJson(cached);
        }
        
        // Fallback 1이 실패하면 Fallback 2 실행 (수동)
        logger.warn("캐시도 없음, Fallback 2 시도: {}", userId);
        return recommendationChainFallback2(userId, ex);
    }
    
    // Fallback 2: 인기 상품
    public List<Product> recommendationChainFallback2(
        String userId,
        Exception ex
    ) {
        logger.warn("Fallback 2 실행: 인기 상품 반환");
        return getPopularProducts();
    }
    
    private List<Product> getPopularProducts() {
        return List.of(
            new Product("P001", "베스트셀러 1", 10000),
            new Product("P002", "베스트셀러 2", 20000)
        );
    }
}
```

### Fallback을 마스킹하면 안 되는 에러

```java
@Service
public class ErrorMaskingPrevention {
    
    // ✅ 마스킹 가능: 조회/추천 서비스 장애
    @CircuitBreaker(
        name = "recommendation-service",
        fallbackMethod = "fallbackRecommendations"
    )
    public List<Product> getRecommendations(String userId) {
        return restTemplate.getForObject(
            "http://recommendation-service/api/recommend",
            List.class
        );
    }
    
    public List<Product> fallbackRecommendations(String userId, Exception ex) {
        return Collections.emptyList();  // 또는 기본값
    }
    
    // ✅ 마스킹 가능: 리뷰 서비스 장애 (부가 기능)
    @CircuitBreaker(
        name = "review-service",
        fallbackMethod = "fallbackReviews"
    )
    public List<Review> getProductReviews(String productId) {
        return restTemplate.getForObject(
            "http://review-service/api/products/" + productId + "/reviews",
            List.class
        );
    }
    
    public List<Review> fallbackReviews(String productId, Exception ex) {
        return Collections.emptyList();
    }
    
    // ❌ 마스킹 불가: 결제 서비스 장애 (핵심 기능)
    public OrderResult processPayment(
        String orderId,
        double amount
    ) {
        try {
            PaymentResult result = restTemplate.postForObject(
                "http://payment-service/api/pay",
                new PaymentRequest(orderId, amount),
                PaymentResult.class
            );
            
            if (!result.isSuccess()) {
                // 결제 실패는 사용자에게 알려야 함
                return OrderResult.failed(
                    "결제에 실패했습니다. 신용카드를 확인해주세요."
                );
            }
            
            return OrderResult.success(result.getTransactionId());
        } catch (Exception e) {
            // 결제 서비스 장애도 숨기면 안 됨
            // 사용자에게 상황을 명확히 설명
            return OrderResult.failed(
                "결제 서비스 점검 중입니다. 잠시 후 다시 시도해주세요."
            );
        }
    }
    
    // ❌ 마스킹 불가: 재고 확인 장애 (비즈니스 로직)
    public boolean checkInventory(String productId, int quantity) {
        try {
            InventoryCheckResult result = restTemplate.getForObject(
                "http://inventory-service/api/check",
                InventoryCheckResult.class
            );
            
            return result.hasEnoughStock();
        } catch (Exception e) {
            // 재고 확인 실패를 true로 마스킹하면 안 됨
            // (오버셀링 위험!)
            throw new InventoryCheckFailedException(
                "재고 확인 중 오류가 발생했습니다. 잠시 후 다시 시도해주세요.", e
            );
        }
    }
    
    // ❌ 마스킹 불가: 주소 검증 장애 (배송 관련)
    public void validateDeliveryAddress(String address) {
        try {
            AddressValidationResult result = restTemplate.postForObject(
                "http://address-service/validate",
                address,
                AddressValidationResult.class
            );
            
            if (!result.isValid()) {
                throw new InvalidAddressException("유효하지 않은 주소입니다.");
            }
        } catch (Exception e) {
            // 주소 검증 실패를 무시하고 진행하면 안 됨
            // (잘못된 배송 위험!)
            throw new AddressValidationException(
                "주소 검증 서비스 점검 중입니다. 잠시 후 다시 시도해주세요.", e
            );
        }
    }
}
```

**마스킹 가능 vs 불가능 판단 기준:**

```
마스킹 가능:
- 부가 기능 (추천, 리뷰, 최근 본 상품)
- 성능 최적화 (캐싱, 프리페칭)
- 개인화 기능 (사용자 맞춤, AI 추천)
- 통지/알림

마스킹 불가능:
- 핵심 비즈니스 로직 (결제, 주문)
- 비즈니스 제약 (재고, 한도)
- 배송/배송비 계산
- 사용자 신원 확인
- 금융 거래
- 법적 요구사항 (나이 확인 등)
```

---

## 📊 패턴 비교

| 항목 | 기본값 반환 | 캐시 데이터 | 기능 축소 |
|------|---|---|---|
| **구현 복잡도** | 매우 낮음 | 중간 | 높음 |
| **사용자 경험** | 나쁨 | 좋음 | 최고 |
| **데이터 정확성** | 낮음 | 중간 (오래됨) | 높음 |
| **인프라 요구** | 없음 | Redis 등 필요 | 복잡한 로직 |
| **적용 범위** | 부가 기능 | 조회 기능 | 모든 기능 |

---

## ⚖️ 트레이드오프

### 장점
- **사용자 경험 유지**: 장애 상황에서도 서비스 제공
- **데이터 손실 방지**: 부분 기능으로도 트랜잭션 완료 가능
- **신뢰성 향상**: 단일 서비스 장애가 전체 시스템에 미치는 영향 최소화
- **우아한 성능 저하 (Graceful Degradation)**: 모 아니면 도 방식 피함

### 단점
- **구현 복잡도 증가**: 각 서비스별로 fallbackMethod 구현 필요
- **테스트 어려움**: 정상/장애 상황 모두 테스트해야 함
- **일관성 문제**: 캐시 데이터가 최신이 아닐 수 있음
- **마스킹 판단 어려움**: 어떤 에러를 마스킹할지 판단 필요

---

## 📌 핵심 정리

✅ **3가지 Fallback 전략: 기본값 반환 (간단), 캐시 데이터 (권장), 기능 축소 (최고)**

✅ **Fallback Chain으로 1차 실패 시 2차로 넘어가기: 캐시 → 기본값**

✅ **Fallback을 마스킹하면 안 되는 에러: 결제, 재고, 주소 등 비즈니스 로직**

✅ **부가 기능(추천, 리뷰)은 언제든 마스킹 가능, 핵심 기능은 불가능**

✅ **Fallback도 실패할 수 있으므로, 최악의 경우를 대비하여 최소값(기본값) 반환**

---

## 🤔 생각해볼 문제

**Q1.** 추천 서비스 장애 시 캐시가 있으면 캐시 반환, 캐시가 없으면 인기 상품 반환하는 Fallback Chain을 구현했습니다. 하지만 이 두 Fallback도 모두 실패할 수 있을까요? 어떤 경우에?

<details><summary>해설 보기</summary>
네, 실패할 수 있습니다. 예를 들어:
1. Redis 캐시 서버가 다운된 경우: 캐시에서 데이터를 읽지 못함
2. 인기 상품 정보가 저장된 DB가 다운된 경우: 인기 상품도 조회 불가

이를 대비하려면:
```java
public List<Product> recommendationChainFallback2(String userId, Exception ex) {
    try {
        return getPopularProductsFromDB();
    } catch (Exception e) {
        // Fallback 2도 실패 → Fallback 3 (최악의 경우)
        return getHardcodedDefaultProducts();
    }
}

private List<Product> getHardcodedDefaultProducts() {
    // 메모리에 하드코딩된 기본 상품
    return List.of(
        new Product("DEFAULT1", "상품1", 10000),
        new Product("DEFAULT2", "상품2", 20000)
    );
}
```
</details>

**Q2.** 캐시 기반 Fallback을 사용하면, 서비스가 복구된 후에도 캐시된 오래된 데이터를 받을 수 있습니다. 이를 어떻게 처리할까요?

<details><summary>해설 보기</summary>
이를 "스테일 캐시(Stale Cache)" 문제라고 합니다. 해결 방법:

1. TTL(Time To Live) 설정: 캐시 유효 시간 제한 (1시간 등)
```java
redis.opsForValue().set(
    key,
    value,
    Duration.ofHours(1)  // 1시간 후 자동 삭제
);
```

2. 캐시 버전 관리: 서비스 복구 시 캐시 버전 변경
```java
String cacheKey = "recommendations:v2:userId";  // v2로 강제 갱신
```

3. 사용자에게 상황 공지: 응답에 "CACHED" 표시
```java
response.setDataAge("5분 전 캐시")
        .setDataStatus("최신 정보가 아닙니다")
```
</details>

**Q3.** 결제 서비스 장애 시 "일시 점검 중" 메시지를 반환하지 않고, 정의된 금액(예: 1000원)으로 자동 결제 처리하는 것은 어떨까요?

<details><summary>해설 보기</summary>
절대 안 됩니다! 이는 여러 심각한 문제를 야기합니다:
1. 사용자 동의 없는 결제 (법적 문제)
2. 금액 오류로 인한 분쟁 (1000원은 임의)
3. 중복 결제 위험 (재시도 시)
4. 감사 추적(Audit Trail) 불가
5. PCI-DSS 규정 위반 (금융 보안)

올바른 처리:
- 결제 실패를 사용자에게 명확히 전달
- 재시도 옵션 제공
- 결제 상태를 "PENDING" 또는 "FAILED"로 저장
- 나중에 결제 재시도 기능 제공

마스킹 가능한 것: 부가 기능, 성능 최적화
마스킹 불가능한 것: 금융, 법적, 비즈니스 제약사항
</details>

---

<div align="center">

**[⬅️ 이전: Timeout 전략 — 자원 고갈을 막는 타임아웃 설정](./04-timeout-strategies.md)** | **[홈으로 🏠](../README.md)** | **[다음: 헬스체크와 자가 치유 — Actuator와 Kubernetes Probe ➡️](./06-health-check-and-self-healing.md)**

</div>

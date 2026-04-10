# 03. Rate Limiting 구현 — Token Bucket과 분산 카운터

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Rate Limiting이 필요한 이유는 무엇이고, API 게이트웨이 단계에서 구현하는 것이 서비스 레벨에서 구현하는 것과 어떻게 다른가?
- Token Bucket 알고리즘은 어떻게 작동하며, Sliding Window 알고리즘과의 차이점은?
- Redis를 사용한 분산 Rate Limiting에서 Lua 스크립트가 필수인 이유는?
- 사용자별, API별, 클라이언트별로 서로 다른 Rate Limit을 적용하려면 어떻게 구성해야 하는가?
- 분산 환경에서 로컬 카운터를 사용할 때 어떤 문제가 발생하고, 왜 중앙 저장소(Redis)가 필수인가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

API 게이트웨이 없이 각 마이크로서비스가 독립적으로 요청을 처리한다면, 한 개의 악의적 클라이언트가 모든 서비스를 마비시킬 수 있습니다. Rate Limiting은 **공유 자원(인프라, 데이터베이스)을 보호하고 비용을 제어하는 핵심 메커니즘**입니다.

특히 MSA에서는 각 서비스가 **독립적인 속도로 처리**되므로, Gateway에서의 Rate Limiting이 없으면:
- DDoS 공격이 전체 시스템을 마비시킴
- 배치 작업이 실시간 서비스를 배제 (리소스 독점)
- 써드파티 API 호출 비용이 통제 불가능하게 증가 (외부 API 요금제에 따라)
- 데이터베이스 연결 풀이 고갈되어 합법적 요청도 거부됨

따라서 **단일 진입점(API Gateway)**에서 Rate Limiting을 구현하여 모든 후단 서비스를 보호합니다.

---

## 😱 흔한 실수 (Before)

### 오류 1: 로컬 메모리 카운터 사용 (분산 환경에서)

```java
// RateLimitFilter.java - ❌ 잘못된 방법
@Component
public class RateLimitFilter implements GlobalFilter, Ordered {
    
    // 로컬 메모리 카운터 (인스턴스마다 독립적)
    private final ConcurrentHashMap<String, AtomicInteger> requestCount = 
            new ConcurrentHashMap<>();
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String userId = extractUserId(exchange);
        
        // 이 인스턴스의 카운트만 증가 (다른 Gateway 인스턴스는 모름)
        AtomicInteger count = requestCount.computeIfAbsent(userId, k -> new AtomicInteger(0));
        
        if (count.incrementAndGet() > 100) {  // 100 req/min
            return sendRateLimitExceeded(exchange);
        }
        
        return chain.filter(exchange);
    }
}
```

**문제점:**
```
요청이 1,000 req/min으로 들어오는 상황:

┌────────────────┐          ┌────────────────┐          ┌────────────────┐
│ Gateway1       │          │ Gateway2       │          │ Gateway3       │
│ Count: 350     │          │ Count: 350     │          │ Count: 350     │
│ Limit: 100 ✗   │          │ Limit: 100 ✗   │          │ Limit: 100 ✗   │
└────────────────┘          └────────────────┘          └────────────────┘

각 Gateway는 100개만 카운트하고 있지만,
실제로는 1,000개의 요청이 백단 서비스로 전달됨!

Rate Limiting이 전혀 작동하지 않음.
```

### 오류 2: Token Bucket 구현 누락 (Sliding Window만 사용)

```java
// ❌ Sliding Window만 사용 (버스트 트래픽 대응 불가)
@Component
public class SlidingWindowRateLimiter {
    
    private final ConcurrentHashMap<String, Queue<Long>> timestamps = 
            new ConcurrentHashMap<>();
    
    public boolean isAllowed(String userId) {
        Queue<Long> userTimestamps = timestamps.computeIfAbsent(userId, k -> new ConcurrentLinkedQueue<>());
        
        long now = System.currentTimeMillis();
        long oneMinuteAgo = now - 60000;
        
        // 1분 이내 요청만 유지
        userTimestamps.removeIf(ts -> ts < oneMinuteAgo);
        
        if (userTimestamps.size() >= 100) {  // 100 req/min
            return false;  // 제한
        }
        
        userTimestamps.offer(now);
        return true;
    }
}
```

**문제점:**
```
구간 예시:
[T=0s]    ◄── 1분 시작
[T=1s]  ✓ ✓ ✓ (3개 요청)
[T=2s]  ✓ ✓ ✓ (3개)
...
[T=30s] ✓ ✓ ✓ (3개 = 총 99개)  ← 한계에 거의 도달
[T=59s] ◄── 거의 끝
[T=60s] ◄── 1분 정확히 경과

이 시점에서:
- 첫 번째 요청(T=0s)이 삭제됨 (1분 초과)
- 새로운 1분 구간 시작

[T=60s~61s] ✗ ✗ ✗ ✗ ... (100개 버스트 가능)

즉, 1분의 경계에서 갑자기 200개 요청 허용 가능!
(T=30~60초 동안 100개 + T=60~61초 동안 100개)

"Token Bucket"처럼 토큰이 쌓이지 않으므로,
버스트 트래픽에서 경계 근처 스파이크 발생.
```

---

## ✨ 올바른 접근 (After)

### Token Bucket 알고리즘 구현 (Redis 사용)

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: api-service
          uri: http://api-service:8080
          predicates:
            - Path=/api/**
          filters:
            - name: RequestRateLimiter
              args:
                # Redis 기반 Token Bucket
                redis-rate-limiter.replenishRate: 100  # 초당 100개 토큰 추가
                redis-rate-limiter.burstCapacity: 200  # 최대 200개 토큰 보유 가능
                key-resolver: "#{@userKeyResolver}"    # 사용자별 제한
```

```java
// RateLimiterConfig.java
@Configuration
public class RateLimiterConfig {
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.just(
            // JWT에서 사용자 ID 추출
            extractUserIdFromJwt(exchange)
                    .orElse("anonymous")  // 인증 안 된 사용자
        );
    }
    
    @Bean
    public KeyResolver clientIpResolver() {
        return exchange -> Mono.just(
            exchange.getRequest()
                    .getRemoteAddress()
                    .map(addr -> addr.getAddress().getHostAddress())
                    .orElse("unknown")
        );
    }
    
    @Bean
    public KeyResolver apiEndpointResolver() {
        return exchange -> Mono.just(
            exchange.getRequest()
                    .getPath()
                    .value()  // /api/users → 별도 한도
        );
    }
    
    private Optional<String> extractUserIdFromJwt(ServerWebExchange exchange) {
        String authHeader = exchange.getRequest()
                .getHeaders()
                .getFirst("Authorization");
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return Optional.empty();
        }
        
        String token = authHeader.substring(7);
        // JWT 파싱 로직 (jjwt 라이브러리 사용)
        try {
            Claims claims = Jwts.parserBuilder()
                    .setSigningKey(SignatureAlgorithm.HS256.getJavaSigningKey("secret"))
                    .build()
                    .parseClaimsJws(token)
                    .getBody();
            return Optional.ofNullable((String) claims.get("userId"));
        } catch (JwtException e) {
            return Optional.empty();
        }
    }
}
```

### 다중 KeyResolver 사용 (계층별 제한)

```yaml
# application.yml - 여러 레벨에서 Rate Limiting
spring:
  cloud:
    gateway:
      routes:
        # 1. 글로벌 레이트 제한 (모든 API)
        - id: global-rate-limit
          uri: http://backend:8080
          predicates:
            - Path=/api/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10000   # 초당 10,000 req
                redis-rate-limiter.burstCapacity: 20000
                key-resolver: "#{@clientIpResolver}"       # IP 기반
        
        # 2. 사용자별 제한 (개별 사용자)
        - id: user-rate-limit
          uri: http://backend:8080
          predicates:
            - Path=/api/users/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100     # 초당 100 req
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@userKeyResolver}"        # 사용자 ID 기반
        
        # 3. API 엔드포인트별 제한
        - id: payment-rate-limit
          uri: http://payment-service:8080
          predicates:
            - Path=/api/payments/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10      # 초당 10 req (엄격)
                redis-rate-limiter.burstCapacity: 20
                key-resolver: "#{@userKeyResolver}"
```

---

## 🔬 내부 동작 원리 — Token Bucket과 Lua 스크립트 심층 분석

### Token Bucket 알고리즘 흐름

```
┌────────────────────────────────────────────────────────────────┐
│ Token Bucket 상태 변화 (replenishRate=100/s, burstCapacity=200) │
└────────────────────────────────────────────────────────────────┘

시간    | 버킷 상태     | 요청 | 결과        | 설명
--------|--------------|-----|-----------|-------------------------------------------
T=0s    | 200토큰 (꽉) | 100 | ✓ 허용    | 토큰 100개 소비, 남은 토큰: 100
        |              |     |           |
T=0.1s  | 110토큰      | 150 | ✗ 거부    | 토큰 110개 < 요청 150개
        |              |     |           |
T=0.2s  | 120토큰      | 50  | ✓ 허용    | 토큰 120개 > 요청 50개
        |              |     |           | 토큰 남음: 70
        |              |     |           |
T=1s    | 170토큰      | 100 | ✓ 허용    | 1초에 100토큰 충전
        |              |     |           | 남은 토큰: 70
        |              |     |           |
T=2s    | 200토큰 (꽉) | 300 | ✗ 거부    | 버킷 용량 초과, 수용 불가
        |              |     |           | (200토큰만 가능)
        |              |     |           |
T=2.5s  | 150토큰      | 150 | ✓ 허용    | 정확히 일치
        |              |     |           |
```

### Redis Lua 스크립트를 사용하는 이유

```lua
-- Spring Cloud Gateway가 Redis에서 실행하는 Lua 스크립트
-- (org/springframework/cloud/gateway/filter/ratelimit/RedisRateLimiter.lua)

local key = KEYS[1]  -- Rate Limiter 키 (예: "rate_limit:user123")
local limit = tonumber(ARGV[1])           -- 버킷 용량 (200)
local window = tonumber(ARGV[2])          -- 타임윈도우 (초)
local current = tonumber(ARGV[3])         -- 현재 타임스탬프
local requested = tonumber(ARGV[4])       -- 요청 토큰 개수
local rate = tonumber(ARGV[5])            -- 충전 비율 (토큰/초, 100)

-- 문제: Redis는 단일 명령만 원자적 보장
-- 다음 작업들을 한 번에 해야 함:
-- 1. 마지막 요청 시간 조회
-- 2. 경과 시간 계산
-- 3. 토큰 충전
-- 4. 토큰 소비 가능 여부 확인
-- 5. 토큰 감소
-- 6. TTL 설정

-- Lua 스크립트의 원자성 보장:
-- Redis는 Lua 스크립트를 완전히 실행한 후에만
-- 다른 클라이언트의 명령을 처리함

local fill_time = math.floor((limit - current) / rate)
local ttl = math.floor(fill_time) + window

local current_bucket = redis.call('GETRANGE', key, 0, -1)
if current_bucket == false or current_bucket == '' then
    current_bucket = limit
else
    current_bucket = tonumber(current_bucket)
end

-- 경과 시간에 따른 토큰 충전
local elapsed = math.floor((redis.call('TIME')[1] - initial_time) / 1000)
current_bucket = math.min(limit, current_bucket + elapsed * rate)

-- 토큰 소비 가능 여부
if current_bucket >= requested then
    current_bucket = current_bucket - requested
    redis.call('SETEX', key, ttl, current_bucket)
    return {1, current_bucket}  -- {1: 허용, 남은 토큰}
else
    return {0, current_bucket}  -- {0: 거부, 남은 토큰}
end
```

**Lua 스크립트가 필수인 이유:**

```java
// ❌ Lua 없이 Java에서 구현하면 Race Condition 발생

public boolean isAllowed(String userId) {
    // Step 1: 현재 토큰 조회
    long tokens = redis.get(userId);  // 예: 150
    
    // ← 이 순간에 다른 스레드가 같은 키에 접근 가능!
    
    // Step 2: 토큰 충전 계산
    long elapsed = System.currentTimeMillis() - lastRefill;
    tokens = Math.min(200, tokens + elapsed * 100);  // 50 토큰 충전?
    
    // ← 또 다른 스레드가 같은 계산을 할 수 있음!
    
    // Step 3: 토큰 소비 확인
    if (tokens >= 100) {  // ← 동시성 문제!
        // Step 4: 토큰 저장
        redis.set(userId, tokens - 100);  // 네트워크 지연...
        return true;
    }
    return false;
}
```

**실제 동작 시나리오:**
```
Thread 1                          Thread 2
┌─────────────────────┐          ┌─────────────────────┐
│ tokens = 100        │          │                     │
│                     │          │                     │
│                     ├─────────► │ tokens = 100        │
│ if (tokens >= 100)  │          │                     │
│   ✓ 허용            │          │ if (tokens >= 100)  │
│   tokens = 0        │          │   ✓ 허용            │
│   저장!             │          │   tokens = 0        │
│                     │          │   저장!             │
└─────────────────────┘          └─────────────────────┘

결과: 토큰 100개가 2번 소비됨!
원래 의도: 토큰 100개만 1번 소비

이 문제를 해결하려면 Redis 단일 명령으로 모든 작업을 원자적으로 수행해야 함.
→ Lua 스크립트 사용!
```

### 분산 환경에서의 동기화

```
┌─────────────────────────────────────────────────────────────────┐
│ 3개 Gateway 인스턴스에서 동일한 사용자 요청이 동시에 들어옴    │
└─────────────────────────────────────────────────────────────────┘

[Local Counter 방식 - 문제]

Gateway1          Gateway2          Gateway3
┌──────────┐      ┌──────────┐      ┌──────────┐
│ Count: 0 │      │ Count: 0 │      │ Count: 0 │
│ Limit:100│      │ Limit:100│      │ Limit:100│
└──────────┘      └──────────┘      └──────────┘
    ↓                 ↓                 ↓
100 req           100 req            100 req
    ↓                 ↓                 ↓
Count: 100        Count: 100        Count: 100
    ↓                 ↓                 ↓
    ✓ OK              ✓ OK              ✓ OK

문제: 각 Gateway가 독립적으로 관리 → 총 300개 요청이 통과!


[Redis 중앙 저장소 방식 - 해결]

Gateway1          Gateway2          Gateway3
┌──────────┐      ┌──────────┐      ┌──────────┐
│ Local:0  │      │ Local:0  │      │ Local:0  │
└──────────┘      └──────────┘      └──────────┘
    │                 │                 │
    └─────────────────┼─────────────────┘
                      ↓
            ┌──────────────────────┐
            │ Redis Counter       │
            │ (단일 진실 공급원)   │
            │ Count: 0 → 100 → 200│ → 300 거부
            └──────────────────────┘
            
각 Gateway가 Redis로 자신의 요청을 카운트
→ 정확하게 총 300개 요청 중 100개만 허용
```

---

## 💻 Rate Limiting 상세 구현 (다양한 전략)

```java
// RateLimiterStrategyConfig.java
@Configuration
public class RateLimiterStrategyConfig {
    
    // 전략 1: 사용자별 Rate Limiting
    @Bean
    public KeyResolver userIdKeyResolver() {
        return exchange -> Mono.just(
            extractUserIdFromToken(exchange)
                    .orElse("anonymous")
        );
    }
    
    // 전략 2: IP 기반 Rate Limiting
    @Bean
    public KeyResolver clientIpKeyResolver() {
        return exchange -> Mono.just(
            getClientIP(exchange)
        );
    }
    
    // 전략 3: 조합형 (사용자 ID + API 엔드포인트)
    @Bean
    public KeyResolver compositeKeyResolver() {
        return exchange -> Mono.just(
            extractUserIdFromToken(exchange)
                    .orElse("anonymous")
                    + ":"
                    + exchange.getRequest().getPath().value()
        );
    }
    
    // 전략 4: VIP 사용자는 더 높은 한도
    @Bean
    public KeyResolver tieredKeyResolver() {
        return exchange -> Mono.just(
            isVipUser(exchange) 
                    ? "vip:" + extractUserIdFromToken(exchange).orElse("unknown")
                    : "standard:" + extractUserIdFromToken(exchange).orElse("unknown")
        );
    }
    
    private Optional<String> extractUserIdFromToken(ServerWebExchange exchange) {
        try {
            String authHeader = exchange.getRequest()
                    .getHeaders()
                    .getFirst("Authorization");
            
            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                return Optional.empty();
            }
            
            String token = authHeader.substring(7);
            // JWT 파싱 (실제로는 더 robust한 방식 필요)
            Claims claims = parseJwt(token);
            return Optional.ofNullable((String) claims.get("sub"));
        } catch (Exception e) {
            return Optional.empty();
        }
    }
    
    private String getClientIP(ServerWebExchange exchange) {
        // X-Forwarded-For 헤더 확인 (프록시 뒤에 있을 때)
        String xForwardedFor = exchange.getRequest()
                .getHeaders()
                .getFirst("X-Forwarded-For");
        
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0];  // 첫 번째 IP
        }
        
        return exchange.getRequest()
                .getRemoteAddress()
                .map(addr -> addr.getAddress().getHostAddress())
                .orElse("unknown");
    }
    
    private boolean isVipUser(ServerWebExchange exchange) {
        return extractUserIdFromToken(exchange)
                .map(userId -> userId.equals("vip-customer-123"))  // 하드코딩 예시
                .orElse(false);
    }
    
    private Claims parseJwt(String token) {
        // 실제 구현: JWT 라이브러리 사용
        return null;  // 생략
    }
}
```

### 커스텀 Rate Limiter 필터

```java
// CustomRateLimitFilter.java - 고급 기능 포함
@Component
public class CustomRateLimitFilter implements GlobalFilter, Ordered {
    
    private static final String RATE_LIMIT_HEADER_LIMIT = "X-RateLimit-Limit";
    private static final String RATE_LIMIT_HEADER_REMAINING = "X-RateLimit-Remaining";
    private static final String RATE_LIMIT_HEADER_RESET = "X-RateLimit-Reset";
    
    private final ReactiveRedisTemplate<String, String> redisTemplate;
    
    public CustomRateLimitFilter(ReactiveRedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String key = resolveKey(exchange);
        int limit = resolveLimit(exchange);
        int window = 60;  // 1분 단위
        
        return checkRateLimit(key, limit, window)
                .flatMap(rateLimitInfo -> {
                    // Rate Limit 정보를 헤더에 추가
                    exchange.getResponse()
                            .getHeaders()
                            .add(RATE_LIMIT_HEADER_LIMIT, String.valueOf(limit));
                    exchange.getResponse()
                            .getHeaders()
                            .add(RATE_LIMIT_HEADER_REMAINING, 
                                    String.valueOf(rateLimitInfo.getRemaining()));
                    exchange.getResponse()
                            .getHeaders()
                            .add(RATE_LIMIT_HEADER_RESET,
                                    String.valueOf(rateLimitInfo.getResetTime()));
                    
                    if (rateLimitInfo.isAllowed()) {
                        return chain.filter(exchange);
                    } else {
                        // 429 Too Many Requests
                        exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
                        return exchange.getResponse()
                                .writeWith(
                                    Mono.fromCallable(() ->
                                        exchange.getResponse()
                                                .bufferFactory()
                                                .wrap("{\"error\":\"Rate limit exceeded\"}"
                                                        .getBytes())
                                    )
                                );
                    }
                });
    }
    
    private Mono<RateLimitInfo> checkRateLimit(String key, int limit, int window) {
        long now = System.currentTimeMillis() / 1000;  // 초 단위
        long windowStart = now - window;
        
        return redisTemplate.opsForZSet()
                .removeRangeByScore(key, 0, windowStart)  // 만료된 요청 제거
                .flatMap(removedCount ->
                    redisTemplate.opsForZSet()
                            .zCard(key)  // 현재 요청 개수
                            .map(currentCount -> currentCount == null ? 0 : currentCount.intValue())
                )
                .flatMap(currentCount -> {
                    if (currentCount < limit) {
                        // 새 요청 추가
                        return redisTemplate.opsForZSet()
                                .add(key, String.valueOf(now), (double) now)
                                .flatMap(added ->
                                    redisTemplate.expire(key, Duration.ofSeconds(window + 1))
                                )
                                .map(expireResult ->
                                    new RateLimitInfo(true, limit - currentCount - 1, now + window)
                                );
                    } else {
                        // 제한 도달
                        return redisTemplate.opsForZSet()
                                .range(key, 0, 0)  // 가장 오래된 요청
                                .next()
                                .map(oldestScore -> {
                                    Long resetTime = (Long) oldestScore;
                                    return new RateLimitInfo(false, 0, resetTime + window);
                                })
                                .defaultIfEmpty(
                                    new RateLimitInfo(false, 0, now + window)
                                );
                    }
                });
    }
    
    private String resolveKey(ServerWebExchange exchange) {
        String userId = extractUserId(exchange)
                .orElse("anonymous");
        return "ratelimit:" + userId;
    }
    
    private int resolveLimit(ServerWebExchange exchange) {
        // 엔드포인트별 다른 한도
        String path = exchange.getRequest().getPath().value();
        
        if (path.startsWith("/api/payments")) {
            return 10;  // 결제: 엄격
        } else if (path.startsWith("/api/search")) {
            return 1000;  // 검색: 관대
        }
        
        return 100;  // 기본값
    }
    
    private Optional<String> extractUserId(ServerWebExchange exchange) {
        String authHeader = exchange.getRequest()
                .getHeaders()
                .getFirst("Authorization");
        
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            // JWT 파싱 로직
            return Optional.of("user123");  // 생략
        }
        
        return Optional.empty();
    }
    
    @Override
    public int getOrder() {
        // 다른 필터보다 먼저 실행 (인증 전에 Rate Limit 체크)
        return Ordered.HIGHEST_PRECEDENCE + 1;
    }
    
    // 내부 클래스
    @Data
    @AllArgsConstructor
    private static class RateLimitInfo {
        private boolean allowed;
        private int remaining;
        private long resetTime;
    }
}
```

---

## 📊 패턴 비교

| 항목 | Token Bucket | Sliding Window | Fixed Window | Leaky Bucket |
|------|:---:|:---:|:---:|:---:|
| **토큰 충전** | 주기적 자동 충전 | 자동 충전 없음 | 시간 경계 초기화 | 일정 속도 유출 |
| **버스트 처리** | ✓ 우수 (버킷 용량) | ✗ 불공정 (경계 근처 스파이크) | ✗ 매우 불공정 (경계 시 리셋) | ✓ 공정 (항상 일정) |
| **구현 복잡도** | 중간 (Lua 필요) | 낮음 (메모리 집약) | 낮음 (간단) | 높음 (큐 관리) |
| **메모리 사용** | 적음 (숫자만 저장) | 많음 (타임스탐프 저장) | 적음 | 중간 (큐) |
| **정확도** | 높음 | 높음 | 낮음 | 높음 |
| **분산 환경** | ✓ Redis Lua | ✓ Redis Sorted Set | ✓ Redis String | ✓ Redis List |
| **클라이언트 친화성** | ✓ 예측 가능 | ✓ 공정함 | ✗ 예측 불가능 | ✓ 일정함 |
| **리셋 타이밍** | 매끄러움 | 매끄러움 | 급격함 | 매끄러움 |

---

## ⚖️ 트레이드오프

### Token Bucket 장점

✅ **버스트 트래픽 수용**: 평상시 토큰이 쌓였다가 갑자기 많은 요청을 처리 가능
```
정상 트래픽 (50 req/s) → 버킷에 토큰 축적 (50초 동안 2,500 토큰)
갑자기 대량 요청 (100 req/s) → 축적된 토큰으로 즉시 처리 가능
```

✅ **예측 가능**: 클라이언트가 정확히 언제 다시 요청할 수 있는지 알 수 있음
- "X-RateLimit-Reset: 2026-04-10T10:00:00Z"를 보고 기다릴 수 있음

✅ **공정한 분배**: 모든 클라이언트가 동일한 토큰 충전 속도

### Token Bucket 단점

❌ **높은 메모리/계산 오버헤드**: Redis Lua 스크립트 실행 필요
- 매 요청마다 Lua 스크립트 실행 (약간의 지연)

❌ **복잡한 구현**: 토큰 충전 시간 계산, 버킷 크기 관리
- 실수하기 쉬움 (토큰 누적 계산 오류 등)

### Sliding Window 장점

✅ **완벽한 정확성**: 항상 정확한 시간 윈도우 내 요청 개수 추적
- 토큰 계산 오류 없음

✅ **구현 간단**: 타임스탐프 저장 + 범위 삭제만으로 가능

### Sliding Window 단점

❌ **많은 메모리**: 모든 요청의 타임스탐프 저장
- 높은 트래픽 (1,000 req/s): 매초 1,000개의 데이터 포인트 저장
- 1시간 유지 시: 3,600,000개 데이터

❌ **시간 윈도우 경계 문제**:
```
[T=0s~60s]: 최대 100개 요청
[T=30s]: 이미 99개 요청 소비
[T=59.999s]: 마지막 1개 남음
[T=60s]: 타임스탐프 삭제 시작 (첫 요청 삭제)
[T=60.001s]: 갑자기 100개 새로운 요청 가능!

→ 순간적으로 200개 요청 처리 가능 (경계 근처)
```

---

## 📌 핵심 정리

✅ **Rate Limiting은 API Gateway의 필수 기능**
- 개별 서비스 보호 (과부하 방지)
- 비용 제어 (외부 API 호출 제한)
- DDoS 방어

✅ **Token Bucket이 MSA 환경의 표준**
- 버스트 트래픽 처리 능력
- 예측 가능한 동작
- Redis + Lua 스크립트로 원자성 보장

✅ **분산 환경에서는 중앙 저장소(Redis) 필수**
- 로컬 메모리 카운터 사용 불가
- 여러 Gateway 인스턴스가 공유 카운터 필요
- Lua 스크립트로 Race Condition 방지

✅ **사용자/클라이언트/엔드포인트별 다른 한도 설정**
```java
KeyResolver를 통해:
- 사용자별: userKeyResolver() → 사용자 ID 기반
- IP별: clientIpKeyResolver() → IP 기반
- 엔드포인트별: compositeKeyResolver() → URL + 사용자 조합
- VIP 구분: tieredKeyResolver() → 회원 등급별 차등 한도
```

✅ **429 응답과 Rate Limit 헤더 제공**
```
X-RateLimit-Limit: 100          (한도)
X-RateLimit-Remaining: 45       (남은 요청)
X-RateLimit-Reset: 1681122000   (리셋 시간)
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 설정에서 사용자 A가 초당 100개 요청을 보냈을 때 정확히 몇 개가 통과할까요?

```yaml
replenishRate: 100      # 초당 100 토큰 충전
burstCapacity: 200      # 최대 200 토큰 보유
```

시나리오:
```
T=0.0s: 사용자 A가 200개 요청 (버킷이 가득 참)
T=0.1s: 사용자 A가 100개 요청
T=0.2s: 사용자 A가 100개 요청
T=1.0s: 다른 사용자 B가 50개 요청
```

<details><summary>해설 보기</summary>

**분석:**

```
T=0.0s, 버킷 상태: 200 토큰 (만찼음)
▶ 200개 요청
  - 토큰 200개 소비
  - 남은 토큰: 0
  - ✓ 200개 모두 통과

T=0.1s (0.1초 경과)
▶ 버킷 충전: 0 + (100 token/s * 0.1s) = 10 토큰
▶ 100개 요청
  - 토큰 10개만 있음
  - ✗ 100개 요청 중 10개만 통과
  - 남은 토큰: 0

T=0.2s (0.1초 추가 경과)
▶ 버킷 충전: 0 + (100 * 0.1) = 10 토큰
▶ 100개 요청
  - 토큰 10개만 있음
  - ✗ 100개 요청 중 10개만 통과
  - 남은 토큰: 0

T=1.0s (0.8초 추가 경과)
▶ 버킷 충전: 0 + (100 * 0.8) = 80 토큰
▶ 사용자 B가 50개 요청 (다른 사용자, 다른 버킷)
  - 사용자 B의 버킷: 200 토큰 (새로운 사용자)
  - ✓ 50개 모두 통과
  - 남은 토큰: 150
```

**결과:**
- 사용자 A: T=0에서 200개, T=0.1에서 10개, T=0.2에서 10개 = **220개 통과**
- 사용자 B: T=1.0에서 50개 = **50개 통과**
- 차단된 요청: T=0.1과 T=0.2에서 각 90개 = **총 180개 차단**

**Key Point:** KeyResolver가 사용자별로 다른 버킷을 관리하므로, 각 사용자는 독립적인 한도를 가집니다.
</details>

**Q2.** Redis 단일 인스턴스가 장애난다면 Rate Limiting은 어떻게 될까요? 이를 해결하기 위한 전략을 제시하세요.

<details><summary>해설 보기</summary>

**현재 상황:**

```
┌──────────────┐
│ Gateway      │
│ (3개)        │
└──────────────┘
     ↓
┌──────────────┐
│ Redis        │ ← 단일 지점 장애!
│ (1개)        │
└──────────────┘
```

**Redis 장애 시 문제:**
1. **전체 Rate Limiting 중단** → 모든 요청이 제한 없이 통과
2. **과부하 발생** → 백단 서비스가 처리할 수 없을 정도의 요청 수신
3. **캐스케이딩 장애** → API Gateway 뒤의 모든 마이크로서비스 마비

**해결 전략:**

1. **Redis Sentinel (자동 페일오버)**
   ```yaml
   spring:
     redis:
       sentinel:
         master: mymaster
         nodes:
           - 192.168.1.10:26379
           - 192.168.1.11:26379
           - 192.168.1.12:26379
   ```
   - Redis 마스터 장애 → 슬레이브 자동 승격
   - Spring이 자동으로 새 마스터에 연결

2. **Redis Cluster (분산 저장)**
   ```yaml
   spring:
     redis:
       cluster:
         nodes:
           - 192.168.1.10:6379
           - 192.168.1.11:6379
           - 192.168.1.12:6379
   ```
   - Rate Limit 데이터가 여러 Redis 노드에 분산 저장
   - 노드 하나 장애 → 다른 노드에서 계속 서비스

3. **Fallback 전략 (응급 처치)**
   ```java
   @Component
   public class ResilientRateLimiter {
       
       @Override
       public Mono<Boolean> isAllowed(String userId) {
           return checkRedisRateLimit(userId)
                   .onErrorResume(ex -> {
                       // Redis 장애 시
                       log.warn("Redis 장애, Fallback 로직 적용: {}", ex.getMessage());
                       
                       // 선택지:
                       // 1. 제한 없이 모든 요청 허용 (위험)
                       // 2. 모든 요청 거부 (안전)
                       // 3. 로컬 카운터로 간단히 제한 (절충)
                       
                       return Mono.just(true);  // 일단 허용
                   });
       }
   }
   ```

4. **Circuit Breaker 패턴 (Resilience4j)**
   ```java
   @Component
   @CircuitBreaker(name = "redis-rate-limiter")
   public Mono<Boolean> isAllowed(String userId) {
       return redisTemplate.opsForValue()
               .get("ratelimit:" + userId)
               .timeout(Duration.ofMillis(100))  // 빠른 실패
               .onErrorResume(ex -> {
                   // Circuit 열림 상태: Fallback
                   return Mono.just(true);
               });
   }
   ```

5. **다중 레이어 Rate Limiting (권장)**
   ```
   [Layer 1] Local Gateway: IP별 간단한 카운트 (동기, 빠름)
                 ↓ (Redis 정상)
   [Layer 2] Redis: 사용자별 정확한 Token Bucket (분산)
                 ↓ (Redis 장애)
   [Layer 3] Circuit Breaker: 모든 요청 거부 (안전)
   ```

**최종 권장:**
- **Redis Sentinel** + **Circuit Breaker** 조합
- Redis 장애 시 Circuit이 열려서 모든 요청 거부 (안전)
- 관리자가 직접 Circuit을 닫을 때까지 유지
- 수동 개입 필요하지만 여러 서비스가 동시에 폭주하는 것을 방지
</details>

**Q3.** 사용자별 한도(100 req/min)와 IP별 한도(1,000 req/min)를 동시에 적용하려면 어떻게 구성할까요?

<details><summary>해설 보기</summary>

**문제 정의:**

```
요청 1 (사용자: user123, IP: 10.0.1.100)
┌─────────────────────────────────┐
│ 사용자 한도: 100 req/min        │ (user123 사용)
│ IP 한도: 1,000 req/min          │ (10.0.1.100 사용)
│                                 │
│ 둘 다 체크해야 함!              │
└─────────────────────────────────┘
```

**해결 방법:**

1. **순차 체크 (가장 단순)**
   ```java
   @Component
   public class DualLayerRateLimiter implements GlobalFilter, Ordered {
       
       @Override
       public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
           String userId = extractUserId(exchange);
           String clientIP = getClientIP(exchange);
           
           return checkUserRateLimit(userId)  // Layer 1
                   .flatMap(userAllowed -> {
                       if (!userAllowed) {
                           return sendRateLimitExceeded(exchange);
                       }
                       
                       return checkIPRateLimit(clientIP)  // Layer 2
                               .flatMap(ipAllowed -> {
                                   if (!ipAllowed) {
                                       return sendRateLimitExceeded(exchange);
                                   }
                                   
                                   return chain.filter(exchange);
                               });
                   });
       }
   }
   ```

2. **병렬 체크 (성능 최적화)**
   ```java
   @Override
   public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
       String userId = extractUserId(exchange);
       String clientIP = getClientIP(exchange);
       
       // 두 체크를 동시에 수행
       return Mono.zip(
               checkUserRateLimit(userId),
               checkIPRateLimit(clientIP)
           )
           .flatMap(tuple -> {
               boolean userAllowed = tuple.getT1();
               boolean ipAllowed = tuple.getT2();
               
               if (!userAllowed || !ipAllowed) {
                   return sendRateLimitExceeded(exchange);
               }
               
               return chain.filter(exchange);
           });
   }
   ```

3. **Spring Cloud Gateway Route 설정 (간단하지만 제한적)**
   ```yaml
   spring:
     cloud:
       gateway:
         routes:
           - id: user-rate-limit
             uri: http://backend
             predicates:
               - Path=/api/**
             filters:
               # 첫 번째 Rate Limiter: 사용자별
               - name: RequestRateLimiter
                 args:
                   redis-rate-limiter.replenishRate: 100
                   redis-rate-limiter.burstCapacity: 200
                   key-resolver: "#{@userIdKeyResolver}"
               
               # 두 번째 Rate Limiter: IP별
               - name: RequestRateLimiter
                 args:
                   redis-rate-limiter.replenishRate: 1000
                   redis-rate-limiter.burstCapacity: 2000
                   key-resolver: "#{@ipKeyResolver}"
   ```

4. **Redis의 집합 연산 활용 (고급)**
   ```java
   // user:user123 한도에 도달했는지 확인
   // AND ip:10.0.1.100 한도에 도달했는지 확인
   
   Mono<Boolean> checkBoth = Mono.zip(
       redisTemplate.getExpire("ratelimit:user:" + userId),
       redisTemplate.getExpire("ratelimit:ip:" + clientIP)
   )
   .map(tuple -> {
       // 둘 다 통과했는지 확인
       return tuple.getT1() < 100 && tuple.getT2() < 1000;
   });
   ```

**권장 구현:**

```java
@Component
public class HierarchicalRateLimiter implements GlobalFilter, Ordered {
    
    private final ReactiveRedisTemplate<String, String> redis;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String userId = extractUserId(exchange);
        String clientIP = getClientIP(exchange);
        
        // 둘 다 동시에 체크 (성능 우수)
        return Mono.zip(
                checkLimit(redis, "user:" + userId, 100),
                checkLimit(redis, "ip:" + clientIP, 1000)
            )
            .flatMap(tuple -> {
                if (!tuple.getT1() || !tuple.getT2()) {
                    // 둘 중 하나라도 위반
                    exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
                    return exchange.getResponse().setComplete();
                }
                
                return chain.filter(exchange);
            });
    }
    
    private Mono<Boolean> checkLimit(
            ReactiveRedisTemplate<String, String> redis,
            String key,
            int limit) {
        
        long now = System.currentTimeMillis();
        
        return redis.opsForZSet()
                .add(key, String.valueOf(now), (double) now)
                .flatMap(added ->
                    redis.opsForZSet()
                            .zCard(key)
                            .map(count -> count != null && count < limit)
                );
    }
}
```

**결과:**
- 사용자 user123이 한도 도달 → 즉시 거부
- IP 10.0.1.100이 한도 도달 → 즉시 거부
- 둘 다 한도 내 → 통과
</details>

---

<div align="center">

**[⬅️ 이전: 서비스 디스커버리 — Eureka vs Kubernetes Service](./02-service-discovery.md)** | **[홈으로 🏠](../README.md)** | **[다음: 인증·인가 아키텍처 — JWT 오프로딩과 서비스 간 인증 ➡️](./04-auth-architecture.md)**

</div>

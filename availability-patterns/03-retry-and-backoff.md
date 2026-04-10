# 03. Retry와 Exponential Backoff — 재시도가 장애를 키우는 경우

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 단순 재시도(Naive Retry)가 문제가 되는 이유는 무엇인가? (Thundering Herd 현상)
- Exponential Backoff는 어떻게 계산되며, 수식은 무엇인가?
- Jitter의 두 가지 방식 (Full Jitter vs Equal Jitter)의 차이는 무엇인가?
- 멱등성(Idempotency)이 없을 때 재시도 시 발생할 수 있는 문제는 무엇인가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

마이크로서비스 아키텍처에서는 네트워크 지연, 일시적 서비스 장애, 타임아웃 등으로 인해 호출이 실패합니다. 이런 상황에서 즉시 에러를 반환하는 것보다, 일시적 오류는 재시도하는 것이 합리적입니다. 하지만 재시도 전략이 잘못되면, 오히려 장애를 악화시킬 수 있습니다.

예를 들어, 서버가 장애 상태에서 모든 클라이언트가 동시에 재시도를 하면, 서버로 유입되는 요청 수가 급증하여 복구 시간이 더 길어집니다. 이를 Thundering Herd(천둥벌레떼) 현상이라고 합니다. Exponential Backoff와 Jitter는 이 문제를 해결하기 위한 필수 패턴입니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ 단순 재시도 (즉시 + 동시)
@Service
public class PaymentService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    public PaymentResult processPayment(String orderId, double amount) {
        Exception lastException = null;
        
        // 3번 재시도 (대기 없음)
        for (int attempt = 0; attempt < 3; attempt++) {
            try {
                return restTemplate.postForObject(
                    "http://payment-service/pay",
                    new PaymentRequest(orderId, amount),
                    PaymentResult.class
                );
            } catch (Exception e) {
                lastException = e;
                // ❌ 문제: 기다리지 않고 즉시 재시도
                // 서버가 복구할 시간 없음
            }
        }
        
        throw new RuntimeException("결제 실패", lastException);
    }
}

// 장애 시나리오:
// 시간: 13:00:00
// - 결제 서버 장애 (메모리 누수 → OOM)
// - 클라이언트 A: 요청 1, 2, 3 즉시 재시도 (0ms, 1ms, 2ms)
// - 클라이언트 B: 요청 1, 2, 3 즉시 재시도 (3ms, 4ms, 5ms)
// - 클라이언트 C: 요청 1, 2, 3 즉시 재시도 (6ms, 7ms, 8ms)
// - ... (1000개 클라이언트가 동일 동작)
//
// 결과: 정상적인 요청 = 1000개
//      재시도된 요청 = 3000개
//      총 요청 = 4000개 (3배 증가!)
//      서버 부하 폭증 → 복구 불가능 상태
```

**Thundering Herd 시각화:**

```
시간 →

13:00:00   서버 장애 발생
           │
           ├─→ 클라이언트 A: 즉시 재시도 (요청 3개)
           ├─→ 클라이언트 B: 즉시 재시도 (요청 3개)
           ├─→ 클라이언트 C: 즉시 재시도 (요청 3개)
           ├─→ ... (모든 클라이언트가 동시 재시도)
           │
           └─→ 서버 부하 3배 증가 → 복구 시간 3배 증가

정상 상황:   ░░░░░░░░░░ (처리량 정상)
재시도 없음:  △△△△△△△△△ (부분 실패)

재시도 있음:  ███████████████████ (부하 폭증)
             (서버가 더 못 버티고 완전 다운)
```

---

## ✨ 올바른 접근 (After)

```java
// ✅ Exponential Backoff + Jitter를 통한 재시도
@Service
public class PaymentService {
    
    private final Retry retry;
    private static final Logger logger = LoggerFactory.getLogger(PaymentService.class);
    
    @Autowired
    public PaymentService(Retry retry) {
        this.retry = retry;
    }
    
    // Resilience4j 사용
    public PaymentResult processPayment(String orderId, double amount) {
        try {
            return retry.executeSupplier(() ->
                callPaymentService(orderId, amount)
            );
        } catch (Exception e) {
            logger.error("결제 최종 실패: {}", orderId, e);
            throw new RuntimeException("결제 실패", e);
        }
    }
    
    private PaymentResult callPaymentService(String orderId, double amount) {
        return new RestTemplate().postForObject(
            "http://payment-service/pay",
            new PaymentRequest(orderId, amount),
            PaymentResult.class
        );
    }
}

// 개선 효과:
// 13:00:00   서버 장애 발생
// 13:00:01   클라이언트 A: 1차 실패 → 100ms + jitter(0~100ms) 대기
// 13:00:01.2 클라이언트 B: 1차 실패 → 100ms + jitter(0~100ms) 대기 (다른 시점!)
// 13:00:02   클라이언트 A: 2차 재시도 (200ms + jitter 대기)
// 13:00:02.5 클라이언트 B: 2차 재시도 (200ms + jitter 대기)
// 13:00:04   클라이언트 A: 3차 재시도 (400ms + jitter 대기)
//
// 결과: 요청이 분산됨 → 서버 부하 증가 제한 → 복구 가능
```

---

## 🔬 내부 동작 원리 — Exponential Backoff와 Jitter 심층 분석

### 1. Exponential Backoff 계산

#### 기본 공식

```
delay = initialDelay * (multiplier ^ attempt) + jitter

예시 (initialDelay=100ms, multiplier=2):
- 1차 재시도: 100ms * 2^0 = 100ms
- 2차 재시도: 100ms * 2^1 = 200ms
- 3차 재시도: 100ms * 2^2 = 400ms
- 4차 재시도: 100ms * 2^3 = 800ms
- 5차 재시도: 100ms * 2^4 = 1600ms
```

**시간 축에서 보면:**

```
시간 →
0ms   100ms  300ms  700ms  1500ms  3100ms
│      │      │      │       │       │
├──────┤      ├──────┤      ├───────┤
  1차        2차         3차

지수적으로 대기 시간이 증가!

1차: 100ms 대기
2차: 200ms 대기 (1차의 2배)
3차: 400ms 대기 (2차의 2배)
4차: 800ms 대기 (3차의 2배)
```

**장점:**
- 초기에는 빠르게 재시도하여 일시적 오류 빠른 복구
- 시간이 지남에 따라 대기 시간이 증가하여 서버 복구 시간 확보
- 최대 대기 시간(maxDelay) 설정으로 무한 대기 방지

#### 최대 대기 시간 제한

```
delay = min(initialDelay * (multiplier ^ attempt), maxDelay) + jitter

예시 (maxDelay=5000ms):
- 1차: min(100, 5000) = 100ms
- 2차: min(200, 5000) = 200ms
- 3차: min(400, 5000) = 400ms
- 4차: min(800, 5000) = 800ms
- 5차: min(1600, 5000) = 1600ms
- 6차: min(3200, 5000) = 3200ms
- 7차: min(6400, 5000) = 5000ms ← 최대값으로 제한
- 8차: min(12800, 5000) = 5000ms (이후 계속 5초)
```

### 2. Jitter의 두 가지 방식

#### Full Jitter

```
delay = random(0, initialDelay * 2^attempt)

예시 (initialDelay=100ms):
- 1차: random(0, 100ms) → 65ms, 32ms, 89ms, ...
- 2차: random(0, 200ms) → 142ms, 81ms, 198ms, ...
- 3차: random(0, 400ms) → 287ms, 123ms, 398ms, ...

특징:
- 최대 대기 시간이 예측 불가능함
- 클라이언트들의 재시도 시점이 완전히 분산됨 (최고의 효율)
- 최악의 경우 이전 대기 시간보다 짧아질 수 있음
```

**시각화:**

```
1차 재시도 (0~100ms):
┌─────────────────────────────────────┐
│■                                    │  A클라이언트 (15ms)
│    ■                                │  B클라이언트 (42ms)
│  ■                                  │  C클라이언트 (38ms)
│                 ■                   │  D클라이언트 (75ms)
└─────────────────────────────────────┘
0ms               50ms               100ms

2차 재시도 (0~200ms):
┌──────────────────────────────────────────────────┐
│        ■                                         │  A클라이언트 (58ms)
│            ■                                     │  B클라이언트 (89ms)
│                    ■                             │  C클라이언트 (132ms)
│                            ■                     │  D클라이언트 (167ms)
└──────────────────────────────────────────────────┘
0ms              100ms              200ms

→ 클라이언트들의 재시도 시점이 완전히 분산됨!
```

#### Equal Jitter (권장)

```
delay = (initialDelay * 2^attempt / 2) + random(0, initialDelay * 2^attempt / 2)

정리하면:
delay = (base / 2) + random(0, base / 2)
where base = initialDelay * 2^attempt

예시 (initialDelay=100ms):
- 1차: 50 + random(0, 50) = 50~100ms
- 2차: 100 + random(0, 100) = 100~200ms
- 3차: 200 + random(0, 200) = 200~400ms

특징:
- 최소 대기 시간이 예측 가능함 (base / 2)
- 최대 대기 시간도 예측 가능함 (base)
- Full Jitter보다 더 빠른 평균 복구 시간
- Full Jitter보다 더 안정적
```

**비교 그래프:**

```
Full Jitter: random(0, 2^attempt * 100)
└─────────────────────────────────┘
0ms        50ms       100ms       200ms
 ●  ●    ●    ●   ●        ●
산발적이고 예측 불가능, 평균 50ms (절반)

Equal Jitter: 2^(attempt-1)*100 + random(0, 2^(attempt-1)*100)
└──────┬───────────────────────────────┐
       ├─────────────────────────────────┘
50ms   100ms      150ms      200ms
 ─────────●──────────●──────────●
더 예측 가능하고 안정적, 평균 75ms (3/4)
```

### 3. 멱등성(Idempotency)과 재시도의 관계

```
멱등성이 있는 경우:
- GET /api/user/123 → 10번 호출해도 같은 결과
- 안전하게 재시도 가능 (중복 제거 걱정 없음)

멱등성이 없는 경우:
- POST /api/payment → 1번 호출: 100원 차감
                     2번 호출: 200원 차감
                     3번 호출: 300원 차감
- 재시도 시 중복 처리 위험!
```

**Idempotency Key를 이용한 해결:**

```
1차 요청:
POST /api/payment
Header: Idempotency-Key: "order-12345-v1"
Body: { orderId: "12345", amount: 100 }
→ 서버: 차감 100원, "order-12345-v1" → Transaction ID 저장

2차 재시도:
POST /api/payment
Header: Idempotency-Key: "order-12345-v1"  ← 동일한 키
Body: { orderId: "12345", amount: 100 }
→ 서버: "order-12345-v1"을 이미 봤음 → 저장된 결과 반환
        (실제로 차감하지 않음, 결과만 반환)

3차 재시도:
POST /api/payment
Header: Idempotency-Key: "order-12345-v1"  ← 동일한 키
Body: { orderId: "12345", amount: 100 }
→ 서버: "order-12345-v1"을 이미 봤음 → 저장된 결과 반환
        (실제로 차감하지 않음, 결과만 반환)

최종 결과: 100원만 차감됨 (멱등성 보장!)
```

---

## 💻 Resilience4j Retry 설정 및 구현

### 1. application.yml 상세 설정

```yaml
resilience4j:
  retry:
    configs:
      default:
        maxAttempts: 3                    # 최대 재시도 횟수 (1차 + 2차 재시도)
        waitDuration: 100                 # 초기 대기 시간 (ms)
        intervalFunction: exponential     # exponential 또는 random
        exponentialRandomizationMultiplier: 2.0  # 지수의 밑
        # waitDuration * exponentialRandomizationMultiplier ^ (attempt - 1)
        
        # Jitter 설정 (Resilience4j 1.7.0+)
        retryOnException: true
        recordFailureForRetry: true
        recordSuccessForRetry: false
    
    instances:
      # 결제 서비스 - 재시도 중요
      payment-service:
        baseConfig: default
        maxAttempts: 3
        waitDuration: 200                 # 더 긴 초기 대기
        intervalFunction: exponential
        exponentialRandomizationMultiplier: 2.0
        exponentialBackoffMultiplier: 2   # 2배씩 증가
        retryOnException: true
        ignoreExceptions:
          - java.lang.IllegalArgumentException  # 비즈니스 에러는 재시도 안 함
      
      # 재고 서비스 - 빠른 재시도
      inventory-service:
        baseConfig: default
        maxAttempts: 5                    # 더 많은 재시도
        waitDuration: 50                  # 짧은 초기 대기
        exponentialBackoffMultiplier: 1.5  # 천천히 증가

  timelimiter:
    instances:
      payment-service:
        timeoutDuration: 2s
      inventory-service:
        timeoutDuration: 1s
```

### 2. Java 프로그래매틱 설정

```java
package com.msa.resilience;

import io.github.resilience4j.core.IntervalFunction;
import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryConfig;
import io.github.resilience4j.retry.RetryRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeoutException;

@Configuration
public class RetryConfiguration {
    
    // 결제 서비스용 Retry 설정
    @Bean
    public Retry paymentRetry(RetryRegistry registry) {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(java.time.Duration.ofMillis(200))
            // Exponential Backoff: 200ms, 400ms, 800ms
            .intervalFunction(
                IntervalFunction.ofExponentialBackoff(200, 2)
            )
            // Retry할 예외 지정
            .retryOnException(throwable ->
                throwable instanceof java.net.SocketException ||
                throwable instanceof java.net.SocketTimeoutException ||
                throwable instanceof java.net.ConnectException
            )
            // Retry하지 않을 예외
            .ignoreExceptions(
                IllegalArgumentException.class,  // 비즈니스 로직 에러
                java.lang.NullPointerException.class
            )
            // 정상 결과도 재시도할 조건 (선택사항)
            .retryOnResult(result ->
                result != null && result.getStatusCode() == 503  // Service Unavailable
            )
            .build();
        
        return registry.retry("payment-service", config);
    }
    
    // Equal Jitter를 이용한 고급 설정
    @Bean
    public Retry inventoryRetry(RetryRegistry registry) {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(5)
            // Equal Jitter 구현
            // delay = base/2 + random(0, base/2)
            .intervalFunction(
                IntervalFunction.ofExponentialRandomBackoff(
                    50,          // initialInterval (ms)
                    1.5,         // multiplier
                    0.5,         // randomizationMultiplier (0~1)
                    java.time.Duration.ofSeconds(10)  // maxInterval
                )
            )
            .retryOnException(throwable ->
                !(throwable instanceof IllegalArgumentException)
            )
            .build();
        
        return registry.retry("inventory-service", config);
    }
    
    // Random Interval (Jitter만 있고 지수적 증가 없음)
    @Bean
    public Retry lightRetry(RetryRegistry registry) {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(2)
            // Random 대기: 100~200ms 사이의 무작위 값
            .intervalFunction(
                IntervalFunction.ofRandomDelay(100, 200)
            )
            .build();
        
        return registry.retry("light-service", config);
    }
}
```

### 3. @Retry 애노테이션 사용

```java
package com.msa.payment;

import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class PaymentServiceImpl implements PaymentService {
    
    private static final Logger logger = LoggerFactory.getLogger(PaymentServiceImpl.class);
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Retry(
        name = "payment-service",
        fallbackMethod = "paymentFallback"
    )
    public PaymentResult processPayment(String orderId, double amount) {
        logger.info("결제 처리 시작: orderId={}, amount={}", orderId, amount);
        
        try {
            PaymentResult result = restTemplate.postForObject(
                "http://payment-service:8080/api/payment",
                new PaymentRequest(
                    orderId,
                    amount,
                    generateIdempotencyKey(orderId)  // ← 멱등성 키
                ),
                PaymentResult.class
            );
            
            logger.info("결제 성공: transactionId={}", result.getTransactionId());
            return result;
        } catch (Exception e) {
            logger.warn("결제 호출 실패, Retry 예정: {}", e.getMessage());
            throw e;  // Retry 트리거
        }
    }
    
    // Fallback: 모든 재시도 실패 시 호출
    public PaymentResult paymentFallback(String orderId, double amount, Exception ex) {
        logger.error("결제 최종 실패: orderId={}", orderId, ex);
        return PaymentResult.failed(
            "결제 서비스 일시 점검 중입니다. 나중에 재시도해주세요."
        );
    }
    
    // Idempotency Key 생성 (클라이언트와 주문ID 기반)
    private String generateIdempotencyKey(String orderId) {
        return "payment-" + orderId + "-" + System.currentTimeMillis();
    }
}
```

### 4. Idempotency Key 구현

```java
package com.msa.idempotency;

import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpRequest;
import org.springframework.http.client.ClientHttpRequestExecution;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.http.client.ClientHttpResponse;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.UUID;

/**
 * REST 클라이언트에 Idempotency-Key 헤더를 자동 추가하는 인터셉터
 */
@Component
public class IdempotencyKeyInterceptor implements ClientHttpRequestInterceptor {
    
    private static final String IDEMPOTENCY_KEY_HEADER = "Idempotency-Key";
    
    @Override
    public ClientHttpResponse intercept(
        HttpRequest request,
        byte[] body,
        ClientHttpRequestExecution execution
    ) throws IOException {
        // POST, PUT 요청에만 Idempotency-Key 추가
        if (request.getMethod().name().equalsIgnoreCase("POST") ||
            request.getMethod().name().equalsIgnoreCase("PUT")) {
            
            // 이미 헤더가 있으면 사용, 없으면 생성
            if (request.getHeaders().get(IDEMPOTENCY_KEY_HEADER) == null) {
                request.getHeaders().set(
                    IDEMPOTENCY_KEY_HEADER,
                    UUID.randomUUID().toString()
                );
            }
        }
        
        return execution.execute(request, body);
    }
}

// RestTemplate 설정에 인터셉터 추가
@Configuration
public class RestTemplateConfiguration {
    
    @Bean
    public RestTemplate restTemplate(IdempotencyKeyInterceptor interceptor) {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.setInterceptors(java.util.Collections.singletonList(interceptor));
        return restTemplate;
    }
}
```

### 5. Retry 모니터링

```java
package com.msa.monitoring;

import io.github.resilience4j.core.registry.EntryAddedEvent;
import io.github.resilience4j.core.registry.RegistryEventConsumer;
import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.event.RetryOnErrorEvent;
import io.github.resilience4j.retry.event.RetryOnIgnoredExceptionEvent;
import io.github.resilience4j.retry.event.RetryOnSuccessEvent;
import org.springframework.stereotype.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Component
public class RetryEventLogger implements RegistryEventConsumer<Retry> {
    
    private static final Logger logger = LoggerFactory.getLogger(RetryEventLogger.class);
    
    @Override
    public void onEntryAdded(EntryAddedEvent<Retry> event) {
        Retry retry = event.getAddedEntry();
        
        retry.getEventPublisher()
            .onRetry(this::logRetry)
            .onSuccess(this::logSuccess)
            .onIgnoredError(this::logIgnoredError);
    }
    
    private void logRetry(RetryOnErrorEvent event) {
        logger.warn(
            "[{}] 재시도 예정: 시도 {}/{}, 대기: {}ms, 사유: {}",
            event.getRetryName(),
            event.getNumberOfRetryAttempts(),
            event.getRetry().getRetryConfig().getMaxAttempts(),
            event.getLastThrowable().getClass().getSimpleName()
        );
    }
    
    private void logSuccess(RetryOnSuccessEvent event) {
        if (event.getNumberOfRetryAttempts() > 0) {
            logger.info(
                "[{}] 재시도 후 성공: 총 {} 번 시도",
                event.getRetryName(),
                event.getNumberOfRetryAttempts() + 1
            );
        }
    }
    
    private void logIgnoredError(RetryOnIgnoredExceptionEvent event) {
        logger.debug(
            "[{}] 무시된 예외 (재시도 안 함): {}",
            event.getRetryName(),
            event.getLastThrowable().getClass().getSimpleName()
        );
    }
    
    @Override
    public void onEntryRemoved(EntryAddedEvent<Retry> event) {
        // 제거 처리
    }
}
```

---

## 📊 패턴 비교

| 항목 | No Retry | Naive Retry | Exponential Backoff | Exponential + Jitter |
|------|---|---|---|---|
| **초기 재시도** | - | 즉시 | 100ms | 100ms ± 50ms |
| **2차 재시도** | - | 즉시 | 200ms | 200ms ± 100ms |
| **3차 재시도** | - | 즉시 | 400ms | 400ms ± 200ms |
| **서버 복구 시간** | 최악 | 최악 | 좋음 | 최고 |
| **클라이언트 복구 시간** | 최악 | 중간 | 중간 | 좋음 |
| **동시 요청 분산** | N/A | 없음 | 적음 | 높음 |

---

## ⚖️ 트레이드오프

### 장점
- **일시적 오류 극복**: 네트워크 지연, 타임아웃 등 일시적 장애 자동 복구
- **서버 복구 시간 확보**: Exponential Backoff로 서버가 복구될 시간 제공
- **Thundering Herd 방지**: Jitter로 동시 재시도 분산
- **최종 성공률 향상**: 재시도로 인해 최종 성공 확률 증가

### 단점
- **멱등성 보장 필수**: 재시도 가능한 작업은 멱등성 보장 필요
- **응답 시간 증가**: Backoff 대기로 인해 평균 응답 시간 증가
- **설정 복잡도**: maxAttempts, waitDuration, multiplier 등을 정확히 결정해야 함
- **과도한 재시도 위험**: 너무 많은 재시도는 오히려 복구를 방해
- **Circuit Breaker와의 상호작용**: Circuit Breaker가 열려 있을 때도 재시도하면 의미 없음

---

## 📌 핵심 정리

✅ **단순 재시도는 Thundering Herd를 일으켜 장애를 악화시킬 수 있음**

✅ **Exponential Backoff: delay = initialDelay × 2^(attempt-1), 지수적으로 증가하는 대기**

✅ **Jitter는 클라이언트들의 재시도 시점을 분산: Full Jitter vs Equal Jitter**

✅ **멱등성이 없는 작업(결제 등)은 Idempotency Key를 사용하여 중복 실행 방지**

✅ **maxAttempts, waitDuration, exponentialRandomizationMultiplier를 서비스 특성에 맞게 설정**

---

## 🤔 생각해볼 문제

**Q1.** 결제 서비스에 대해 maxAttempts=3, waitDuration=100ms, exponentialRandomizationMultiplier=2를 설정했습니다. 실제 재시도 타이밍은 언제일까요?

<details><summary>해설 보기</summary>
1차 시도: T=0ms (초기 요청)
1차 실패 → 대기: 100ms * 2^0 = 100ms
2차 시도: T=100ms (1차 재시도)
2차 실패 → 대기: 100ms * 2^1 = 200ms
3차 시도: T=300ms (2차 재시도)
3차 실패 → 대기: 100ms * 2^2 = 400ms
4차 시도: T=700ms (3차 재시도)

maxAttempts=3은 "최대 3번의 재시도"를 의미하므로, 총 1번의 초기 요청 + 3번의 재시도 = 4번 시도합니다. (또는 설정에 따라 "총 3번 시도"를 의미할 수 있으므로 문서를 확인해야 합니다.)
</details>

**Q2.** Jitter가 없다면 어떤 문제가 발생할까요?

<details><summary>해설 보기</summary>
Jitter가 없으면 모든 클라이언트가 동일한 대기 시간을 가집니다. 예를 들어, 서버 장애 발생 시 모든 클라이언트가 정확히 100ms, 200ms, 400ms 후에 동시에 재시도를 합니다. 이는 재시도 요청이 동시에 몰려 서버 부하를 크게 증가시키는 Thundering Herd 현상을 일으킵니다. Jitter는 각 클라이언트의 재시도 시점을 약간씩 다르게 하여 동시 부하를 분산시킵니다.
</details>

**Q3.** 결제 작업은 멱등성이 없습니다 (재시도하면 중복 차감). 이 경우 어떻게 해야 할까요?

<details><summary>해설 보기</summary>
Idempotency Key를 사용합니다. 클라이언트가 "payment-order-12345-uuid" 같은 고유한 key를 생성하여 매번 같은 요청에 포함시킵니다. 서버는 이 key를 추적하여, 동일한 key의 요청이 여러 번 들어와도 처음 한 번만 처리하고 나머지는 저장된 결과를 반환합니다. 이렇게 하면 클라이언트 측에서는 안전하게 재시도할 수 있고, 서버는 비즈니스 로직의 중복 실행을 방지할 수 있습니다.
</details>

---

<div align="center">

**[⬅️ 이전: Bulkhead 패턴 — 스레드 풀 격리로 장애 전파 차단](./02-bulkhead.md)** | **[홈으로 🏠](../README.md)** | **[다음: Timeout 전략 — 자원 고갈을 막는 타임아웃 설정 ➡️](./04-timeout-strategies.md)**

</div>

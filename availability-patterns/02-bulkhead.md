# 02. Bulkhead 패턴 — 스레드 풀 격리로 장애 전파 차단

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Bulkhead 패턴의 이름이 "격벽(bulkhead)"인 이유는 무엇인가?
- Semaphore 기반 Bulkhead와 Thread Pool 기반 Bulkhead의 차이는 무엇인가?
- 왜 하나의 느린 서비스가 다른 서비스의 스레드 풀을 고갈시킬 수 있는가?
- Little's Law를 이용해 스레드 풀 크기를 어떻게 결정해야 하는가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

마이크로서비스 환경에서는 하나의 API 서버가 여러 외부 서비스를 동시에 호출합니다. 예를 들어, 주문 서비스가 결제 서비스, 재고 서비스, 배송 서비스를 모두 호출한다고 가정해봅시다. 만약 결제 서비스가 갑자기 느려져서 응답 시간이 30초로 늘어난다면, 결제 호출을 기다리는 스레드들이 계속 쌓여나갑니다. 결국 서버의 모든 스레드가 결제 서비스 응답을 기다리느라 다른 요청들(재고 조회, 배송 예약)은 처리하지 못하게 됩니다.

Bulkhead 패턴은 마치 선박의 격벽처럼, 각 서비스별로 별도의 스레드 풀을 할당하여 한 서비스의 장애가 다른 서비스에 영향을 주지 않도록 격리합니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ 단일 스레드 풀로 모든 외부 서비스 호출
@Service
public class OrderService {
    
    // 스레드 풀 크기 100 (공유)
    private final ExecutorService executor = Executors.newFixedThreadPool(100);
    
    @Autowired
    private RestTemplate restTemplate;
    
    public Order createOrder(String customerId, List<Item> items) {
        // 결제 서비스 호출 (느린 상태)
        Future<PaymentResult> paymentFuture = executor.submit(() ->
            restTemplate.postForObject(
                "http://payment-service:8080/api/payment",
                new PaymentRequest(customerId, getTotalPrice(items)),
                PaymentResult.class
            )
        );
        
        // 재고 서비스 호출
        Future<InventoryResult> inventoryFuture = executor.submit(() ->
            restTemplate.getForObject(
                "http://inventory-service:8080/api/check",
                InventoryResult.class
            )
        );
        
        // 배송 서비스 호출
        Future<ShippingResult> shippingFuture = executor.submit(() ->
            restTemplate.postForObject(
                "http://shipping-service:8080/api/booking",
                new ShippingRequest(customerId, items),
                ShippingResult.class
            )
        );
        
        try {
            PaymentResult payment = paymentFuture.get(30, TimeUnit.SECONDS);    // 30초 대기
            InventoryResult inventory = inventoryFuture.get(30, TimeUnit.SECONDS);
            ShippingResult shipping = shippingFuture.get(30, TimeUnit.SECONDS);
            
            return Order.success(payment, inventory, shipping);
        } catch (TimeoutException e) {
            throw new RuntimeException("주문 처리 타임아웃", e);
        }
    }
}

// 문제점:
// 1. 결제 서비스가 느려지면 스레드 100개가 모두 결제 대기 중
// 2. 재고 조회, 배송 예약도 스레드를 기다려야 함
// 3. 전체 API 서버의 응답성 악화 → Cascading Failure
```

**상황 시나리오:**

```
시간: 13:00
- 결제 서비스 응답 시간 30초로 급증
- OrderService 스레드 풀 100개 모두 결제 대기 중
- 새로운 주문 요청 들어옴 → 스레드 없음 → 요청 큐에 쌓임
- 큐 크기 증가 → 메모리 오버플로우 → 서버 다운

13:01: 결제 서비스가 복구되어도 OrderService는 이미 다운 상태
```

---

## ✨ 올바른 접근 (After)

```java
// ✅ Bulkhead로 서비스별 스레드 풀 격리
@Service
public class OrderService {
    
    // 결제 서비스용 전용 스레드 풀 (크기 20)
    private final ExecutorService paymentExecutor = 
        Executors.newFixedThreadPool(20);
    
    // 재고 서비스용 전용 스레드 풀 (크기 30)
    private final ExecutorService inventoryExecutor = 
        Executors.newFixedThreadPool(30);
    
    // 배송 서비스용 전용 스레드 풀 (크기 25)
    private final ExecutorService shippingExecutor = 
        Executors.newFixedThreadPool(25);
    
    @Autowired
    private RestTemplate restTemplate;
    
    public Order createOrder(String customerId, List<Item> items) {
        // 각 서비스별 격리된 스레드 풀에서 호출
        Future<PaymentResult> paymentFuture = paymentExecutor.submit(() ->
            restTemplate.postForObject(
                "http://payment-service:8080/api/payment",
                new PaymentRequest(customerId, getTotalPrice(items)),
                PaymentResult.class
            )
        );
        
        Future<InventoryResult> inventoryFuture = inventoryExecutor.submit(() ->
            restTemplate.getForObject(
                "http://inventory-service:8080/api/check",
                InventoryResult.class
            )
        );
        
        Future<ShippingResult> shippingFuture = shippingExecutor.submit(() ->
            restTemplate.postForObject(
                "http://shipping-service:8080/api/booking",
                new ShippingRequest(customerId, items),
                ShippingResult.class
            )
        );
        
        try {
            PaymentResult payment = paymentFuture.get(30, TimeUnit.SECONDS);
            InventoryResult inventory = inventoryFuture.get(30, TimeUnit.SECONDS);
            ShippingResult shipping = shippingFuture.get(30, TimeUnit.SECONDS);
            
            return Order.success(payment, inventory, shipping);
        } catch (TimeoutException e) {
            throw new RuntimeException("주문 처리 타임아웃", e);
        }
    }
}

// 개선점:
// 1. 결제 서비스가 느려도 결제용 스레드 풀 20개만 영향
// 2. 재고/배송은 독립적인 스레드 풀에서 계속 정상 작동
// 3. 전체 API 서버의 응답성 유지
```

---

## 🔬 내부 동작 원리 — Thread Pool Isolation 심층 분석

### 공유 스레드 풀 vs 격리된 스레드 풀

```
┌─────────────────────────────────────────────────────────────┐
│            문제: 공유 스레드 풀 (Shared Pool)                │
│                                                             │
│  메인 스레드 풀 (크기 100)                                   │
│  ┌──────────────────────────────────────────────┐          │
│  │ [결제][결제][결제][결제][결제]  ← 모두 대기  │          │
│  │ [재고][재고][결제][결제][결제]  ← 결제 때문에 대기  │      │
│  │ [배송][배송][결제][결제][결제]  ← 결제 때문에 대기  │      │
│  │ ... (90개 더) [결제][결제] ...                │          │
│  └──────────────────────────────────────────────┘          │
│                                                             │
│  결제 서비스 느림 → 전체 스레드 고갈                         │
│  재고, 배송 요청도 처리 불가 → Cascading Failure            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│         해결: 격리된 스레드 풀 (Thread Pool Isolation)       │
│                                                             │
│  결제 스레드 풀 (크기 20)                                    │
│  ┌────────────────────────┐                               │
│  │ [대기][대기][대기]      │  ← 결제만 처리                  │
│  │ [대기][대기][대기]      │  ← 결제 느려도 20개만 영향      │
│  │ [대기][20개 모두 사용]  │                                │
│  └────────────────────────┘                               │
│                                                             │
│  재고 스레드 풀 (크기 30)                                    │
│  ┌──────────────────────────────┐                         │
│  │ [작동][작동][작동]            │  ← 재고 서비스 정상      │
│  │ [작동][작동][작동] ...        │  ← 독립적으로 작동       │
│  │ [작동][작동][작동] (30개)     │                         │
│  └──────────────────────────────┘                         │
│                                                             │
│  배송 스레드 풀 (크기 25)                                    │
│  ┌────────────────────────────┐                           │
│  │ [작동][작동][작동]          │  ← 배송 서비스 정상        │
│  │ [작동][작동][작동] ...      │  ← 독립적으로 작동         │
│  │ [작동][작동][작동] (25개)   │                           │
│  └────────────────────────────┘                           │
│                                                             │
│  결제 서비스 느림 → 결제만 영향, 재고/배송은 정상 작동!     │
└─────────────────────────────────────────────────────────────┘
```

### Bulkhead 두 가지 구현 방식

#### 1. Semaphore 기반 Bulkhead (가벼움, 격리 약함)

```
Semaphore: 동시 호출 수 제한 (최대 5개)

시간 →
T1: [호출 1 시작] ─────────────────────────── [호출 1 끝]
T2: [호출 2 시작] ─────────────────────────── [호출 2 끝]
T3: [호출 3 시작] ─────────────────────────── [호출 3 끝]
T4: [호출 4 시작] ─────────────────────────── [호출 4 끝]
T5: [호출 5 시작] ─────────────────────────── [호출 5 끝]
T6: [대기] 
T7: [대기]
T8: [대기]

특징:
- 최대 5개의 동시 호출만 허용
- 하지만 같은 스레드에서 처리되거나 메인 스레드 풀 사용
- 메모리 오버헤드 낮음
- 하지만 한 서비스의 느린 호출이 다른 호출까지 영향 미칠 수 있음
```

장점:
- 구현이 간단하고 메모리 사용량 적음
- Permit 기반이라 가볍고 빠름

단점:
- 각 서비스별 진정한 격리가 아님
- 느린 호출이 빠른 호출을 블로킹할 수 있음
- 스레드 우선순위 제어 불가

#### 2. Thread Pool 기반 Bulkhead (무거움, 격리 강함)

```
각 서비스별 별도의 스레드 풀

결제 서비스 스레드 풀 (크기 20):
T1: [호출 1]────────────────────── [끝]
T2: [호출 2]────────────────────── [끝]
...
T20: [호출 20]────────────────────── [끝]  ← 최대 20개 동시 실행

재고 서비스 스레드 풀 (크기 30):
T1: [호출 1]──── [끝]
T2: [호출 2]──── [끝]
...
T30: [호출 30]──── [끝]  ← 최대 30개 동시 실행

배송 서비스 스레드 풀 (크기 25):
T1: [호출 1]───────── [끝]
T2: [호출 2]───────── [끝]
...
T25: [호출 25]───────── [끝]  ← 최대 25개 동시 실행

특징:
- 각 서비스마다 완전히 독립된 스레드 풀
- 스레드 스케줄링과 관리가 완전히 격리됨
- 한 서비스가 느려도 다른 서비스는 전혀 영향 받지 않음
- 메모리와 CPU 오버헤드 있음 (스레드 풀마다 별도 관리)
```

장점:
- 진정한 장애 격리 (한 서비스의 문제가 다른 서비스에 영향 없음)
- 각 서비스별로 독립적인 성능 튜닝 가능
- 스레드 관리가 예측 가능

단점:
- 각 스레드 풀마다 메모리 오버헤드 (Stack: ~1MB/스레드)
- 스레드 수가 많아지면 Context Switching 오버헤드
- 스레드 풀 크기 결정이 어려움

---

## 💻 Resilience4j Bulkhead와 Spring Cloud 통합

### 1. application.yml 상세 설정

```yaml
resilience4j:
  bulkhead:
    configs:
      default:
        maxConcurrentCalls: 100        # 동시 호출 최대 수
        maxWaitDuration: 10            # 호출 대기 최대 시간 (초)
        bulkheadType: SEMAPHORE        # SEMAPHORE 또는 THREADPOOL
    
    instances:
      # 결제 서비스 - 중요도 높음, 격리 강함
      payment-service:
        baseConfig: default
        maxConcurrentCalls: 20         # 동시 호출 최대 20개
        maxWaitDuration: 5
        bulkheadType: THREADPOOL
        threadPoolBulkheadConfig:
          coreThreadPoolSize: 20       # 최소 스레드 수
          maxThreadPoolSize: 20        # 최대 스레드 수
          queueCapacity: 10            # 대기 큐 크기
          keepAliveDuration: 60        # 유휴 스레드 유지 시간 (초)
      
      # 재고 서비스 - 일반 서비스
      inventory-service:
        baseConfig: default
        maxConcurrentCalls: 30
        maxWaitDuration: 10
        bulkheadType: SEMAPHORE
      
      # 배송 서비스 - 지연 가능한 서비스
      shipping-service:
        baseConfig: default
        maxConcurrentCalls: 25
        maxWaitDuration: 15
        bulkheadType: THREADPOOL
        threadPoolBulkheadConfig:
          coreThreadPoolSize: 15
          maxThreadPoolSize: 25
          queueCapacity: 20

  # Thread Pool Bulkhead 모니터링
  thread-pool-bulkhead:
    instances:
      payment-service:
        coreThreadPoolSize: 20
        maxThreadPoolSize: 20
        queueCapacity: 10
        keepAliveDuration: 60s

  # Semaphore Bulkhead 모니터링
  semaphore:
    instances:
      inventory-service:
        maxConcurrentCalls: 30
        maxWaitDuration: 10s
```

### 2. Java 설정 클래스

```java
package com.msa.resilience;

import io.github.resilience4j.bulkhead.Bulkhead;
import io.github.resilience4j.bulkhead.BulkheadConfig;
import io.github.resilience4j.bulkhead.BulkheadRegistry;
import io.github.resilience4j.core.registry.EntryAddedEvent;
import io.github.resilience4j.core.registry.RegistryEventConsumer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class BulkheadConfiguration {
    
    // Semaphore 기반 Bulkhead (재고 서비스)
    @Bean
    public Bulkhead inventoryBulkhead(BulkheadRegistry registry) {
        BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(30)
            .maxWaitDuration(Duration.ofSeconds(10))
            .build();
        
        return registry.bulkhead("inventory-service", config);
    }
    
    // Thread Pool 기반 Bulkhead (결제 서비스)
    @Bean
    public ThreadPoolBulkhead paymentThreadPoolBulkhead(
        ThreadPoolBulkheadRegistry registry
    ) {
        ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
            .coreThreadPoolSize(20)
            .maxThreadPoolSize(20)
            .queueCapacity(10)
            .keepAliveDuration(Duration.ofSeconds(60))
            .build();
        
        return registry.threadPoolBulkhead("payment-service", config);
    }
}
```

### 3. @Bulkhead 애노테이션 사용

```java
package com.msa.order;

import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import io.github.resilience4j.core.annotation.Async;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.concurrent.CompletableFuture;

@Service
public class OrderService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Bulkhead(
        name = "payment-service",
        type = Bulkhead.Type.THREADPOOL,
        fallbackMethod = "paymentFallback"
    )
    @Async
    public CompletableFuture<PaymentResult> processPayment(String orderId, double amount) {
        try {
            PaymentResult result = restTemplate.postForObject(
                "http://payment-service:8080/api/payment",
                new PaymentRequest(orderId, amount),
                PaymentResult.class
            );
            return CompletableFuture.completedFuture(result);
        } catch (Exception e) {
            return CompletableFuture.failedFuture(
                new RuntimeException("결제 처리 실패", e)
            );
        }
    }
    
    public CompletableFuture<PaymentResult> paymentFallback(
        String orderId,
        double amount,
        BulkheadFullException ex
    ) {
        // 스레드 풀이 가득 찼을 때의 폴백
        return CompletableFuture.completedFuture(
            PaymentResult.pending("결제 처리 대기 중입니다.")
        );
    }
    
    @Bulkhead(
        name = "inventory-service",
        type = Bulkhead.Type.SEMAPHORE,
        fallbackMethod = "inventoryFallback"
    )
    public InventoryResult checkInventory(List<String> itemIds) {
        try {
            return restTemplate.postForObject(
                "http://inventory-service:8080/api/check",
                itemIds,
                InventoryResult.class
            );
        } catch (Exception e) {
            throw new RuntimeException("재고 조회 실패", e);
        }
    }
    
    public InventoryResult inventoryFallback(
        List<String> itemIds,
        BulkheadFullException ex
    ) {
        // 동시 호출이 초과되었을 때의 폴백
        return InventoryResult.unknown("일시적 혼잡으로 재고 조회 불가");
    }
}
```

### 4. Little's Law를 이용한 스레드 풀 크기 결정

```java
package com.msa.capacity;

/**
 * Little's Law: L = λ * W
 * 
 * L: 시스템 내 평균 요청 수 (필요한 스레드 수)
 * λ: 도착률 (초당 도착하는 요청 수)
 * W: 평균 대기 시간 (응답 시간 포함)
 */
public class ThreadPoolSizingCalculator {
    
    /**
     * 결제 서비스 스레드 풀 크기 계산
     * 
     * 요구사항:
     * - QPS: 초당 100개 결제 요청
     * - 평균 응답 시간: 2초
     * - 안전 계수: 1.2 (여유)
     */
    public static int calculatePaymentThreadPoolSize() {
        double qps = 100;                      // 초당 요청 수
        double avgResponseTime = 2.0;          // 초 단위 응답 시간
        double safetyFactor = 1.2;             // 안전 계수
        
        // Little's Law: 필요한 스레드 수 = QPS * 평균 응답 시간
        double requiredThreads = qps * avgResponseTime;
        
        // 안전 계수 적용
        int finalThreadPoolSize = (int) Math.ceil(requiredThreads * safetyFactor);
        
        // 결과: 240개 스레드 필요
        // 하지만 너무 많으므로 QPS 기반 조정
        return Math.min(finalThreadPoolSize, 50);  // 최대 50개로 제한
    }
    
    /**
     * 재고 서비스 스레드 풀 크기 계산
     * 
     * 요구사항:
     * - QPS: 초당 200개 재고 조회
     * - 평균 응답 시간: 0.5초 (매우 빠름)
     * - 안전 계수: 1.5
     */
    public static int calculateInventoryThreadPoolSize() {
        double qps = 200;
        double avgResponseTime = 0.5;
        double safetyFactor = 1.5;
        
        double requiredThreads = qps * avgResponseTime;
        int finalThreadPoolSize = (int) Math.ceil(requiredThreads * safetyFactor);
        
        // 결과: 150개 필요 → 80개로 제한 (Semaphore 사용하므로 스레드 풀 불필요)
        return Math.min(finalThreadPoolSize, 80);
    }
    
    /**
     * 배송 서비스 스레드 풀 크기 계산
     * 
     * 요구사항:
     * - QPS: 초당 50개 배송 예약
     * - 평균 응답 시간: 5초 (외부 API 호출)
     * - 안전 계수: 1.3
     */
    public static int calculateShippingThreadPoolSize() {
        double qps = 50;
        double avgResponseTime = 5.0;
        double safetyFactor = 1.3;
        
        double requiredThreads = qps * avgResponseTime;
        int finalThreadPoolSize = (int) Math.ceil(requiredThreads * safetyFactor);
        
        // 결과: 325개 필요 → 100개로 제한 (메모리 고려)
        return Math.min(finalThreadPoolSize, 100);
    }
}

// 실제 설정
/*
계산 결과:
- 결제: 20-50개 스레드 추천 → 설정값: 20 (동시 요청 많지만 빠른 응답)
- 재고: 50-100개 스레드 → 하지만 Semaphore로 동시 호출 제한 (비용 절감)
- 배송: 100-300개 스레드 필요 → 설정값: 25 (부족한 경우 Queue에 쌓임)
*/
```

### 5. Bulkhead 모니터링

```java
package com.msa.monitoring;

import io.github.resilience4j.bulkhead.Bulkhead;
import io.github.resilience4j.bulkhead.event.BulkheadOnCallFinishedEvent;
import io.github.resilience4j.bulkhead.event.BulkheadOnCallRejectedEvent;
import io.github.resilience4j.core.registry.EntryAddedEvent;
import io.github.resilience4j.core.registry.RegistryEventConsumer;
import org.springframework.stereotype.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Component
public class BulkheadEventLogger implements RegistryEventConsumer<Bulkhead> {
    
    private static final Logger logger = LoggerFactory.getLogger(BulkheadEventLogger.class);
    
    @Override
    public void onEntryAdded(EntryAddedEvent<Bulkhead> event) {
        Bulkhead bulkhead = event.getAddedEntry();
        
        bulkhead.getEventPublisher()
            .onCallRejected(this::logRejected)
            .onCallFinished(this::logFinished);
    }
    
    private void logRejected(BulkheadOnCallRejectedEvent event) {
        logger.warn(
            "[{}] 호출 거부됨. 동시 호출 수: {}/{}",
            event.getBulkheadName(),
            event.getBulkheadMetrics().getAvailableConcurrentCalls(),
            event.getBulkheadMetrics().getMaxConcurrentCalls()
        );
    }
    
    private void logFinished(BulkheadOnCallFinishedEvent event) {
        logger.debug(
            "[{}] 호출 완료. 가용 슬롯: {}",
            event.getBulkheadName(),
            event.getBulkheadMetrics().getAvailableConcurrentCalls()
        );
    }
    
    @Override
    public void onEntryRemoved(EntryRemovedEvent<Bulkhead> event) {
        // 제거 처리
    }
}
```

---

## 📊 패턴 비교

| 항목 | Semaphore | Thread Pool |
|------|---|---|
| **구현** | 동시 호출 수 제한 | 별도 스레드 풀 |
| **격리 수준** | 약함 (같은 스레드 사용) | 강함 (독립적 스레드) |
| **메모리 사용** | 낮음 | 높음 (스레드당 1MB) |
| **응답성** | 좋음 | 보통 (Context Switching) |
| **장애 격리** | 부분적 | 완전 |
| **권장 시나리오** | 빠르고 가벼운 호출 | 느리고 무거운 호출 |

---

## ⚖️ 트레이드오프

### 장점
- **장애 격리 (Fault Isolation)**: 한 서비스의 문제가 다른 서비스에 영향을 주지 않음
- **자원 관리**: 각 서비스별로 독립적인 자원(스레드) 할당으로 예측 가능한 성능
- **스레드 부족 방지**: 한 서비스가 모든 스레드를 독점하는 것을 방지
- **Cascading Failure 방지**: 느린 서비스가 전체 시스템을 마비시키지 못함

### 단점
- **메모리 오버헤드 (Thread Pool 기반)**: 각 스레드 풀마다 메모리 사용 (스레드당 ~1MB)
- **복잡한 설정**: maxConcurrentCalls, coreThreadPoolSize 등을 정확히 결정해야 함
- **Context Switching 오버헤드**: 스레드 풀이 많을수록 CPU 컨텍스트 스위칭 증가
- **스레드 풀 크기 결정 어려움**: Little's Law를 사용해도 정확한 크기 결정 어려움

---

## 📌 핵심 정리

✅ **Bulkhead 패턴은 선박의 격벽처럼, 각 서비스별로 별도의 자원을 할당하여 장애 격리**

✅ **Semaphore 기반 Bulkhead는 가볍지만 진정한 격리가 아니므로 빠른 서비스에 적합**

✅ **Thread Pool 기반 Bulkhead는 무겁지만 완전한 격리를 제공하여 느린 서비스에 적합**

✅ **Little's Law (L = λ × W)를 이용해 스레드 풀 크기를 계산: 스레드 수 = QPS × 평균 응답시간**

✅ **스레드 풀이 가득 차면 새 요청은 큐에 쌓이므로, queueCapacity도 함께 설정해야 함**

---

## 🤔 생각해볼 문제

**Q1.** 결제 서비스에 Thread Pool Bulkhead (크기 20)를 적용했는데, 초당 100개의 결제 요청이 들어오고 평균 응답 시간이 2초라고 합니다. 이 상황에서 어떤 문제가 발생할까요?

<details><summary>해설 보기</summary>
Little's Law에 의해 필요한 스레드 수는 100 × 2 = 200개입니다. 하지만 스레드 풀 크기가 20개이므로, 대부분의 요청이 큐에 쌓이게 되고 결과적으로 응답 시간이 매우 길어집니다. 예를 들어, 처음 20개 요청이 2초에 처리되면 (2초 후 20개 완료), 그 다음 20개가 처리되는 데 또 2초가 걸리므로 (4초), 마지막 요청은 최대 10초 이상 기다려야 합니다. 따라서 스레드 풀 크기를 50개 이상으로 늘리거나, queueCapacity를 설정하여 대기 큐 크기를 제한해야 합니다.
</details>

**Q2.** Semaphore 기반 Bulkhead와 Thread Pool 기반 Bulkhead 중 어떤 것을 선택해야 할까요?

<details><summary>해설 보기</summary>
응답 시간과 호출 특성을 고려합니다. (1) 응답 시간이 빠른 서비스 (< 100ms): Semaphore 사용 (메모리 절감). (2) 응답 시간이 느린 서비스 (> 500ms): Thread Pool 사용 (완전 격리). (3) 중요도가 높은 서비스 (결제, 인증): Thread Pool 사용 (진정한 격리). (4) 부가 기능 (추천, 광고): Semaphore 사용 (비용 절감). 일반적으로 느리고 중요한 서비스에는 Thread Pool, 빠르고 덜 중요한 서비스에는 Semaphore를 추천합니다.
</details>

**Q3.** 스레드 풀 크기를 100으로 설정했는데, 실제 필요한 것이 20개라고 판단되었습니다. 스레드 풀 크기를 줄일 때 주의할 점은 무엇일까요?

<details><summary>해설 보기</summary>
스레드 풀 크기를 줄이면 메모리와 Context Switching 오버헤드는 감소하지만, 갑작스러운 트래픽 증가에 대응할 수 없습니다. 예를 들어, 정상적으로는 초당 10개 요청이지만 이벤트 시에는 50개가 들어올 수 있습니다. 이 경우 (1) 안전 계수 적용 (필요한 크기 × 1.3~1.5), (2) queueCapacity 설정으로 급증 요청 수용, (3) 동적 모니터링으로 실제 사용량 확인 후 점진적으로 축소하는 것이 권장됩니다.
</details>

---

<div align="center">

**[⬅️ 이전: Circuit Breaker 완전 분해 — Resilience4j 내부](./01-circuit-breaker.md)** | **[홈으로 🏠](../README.md)** | **[다음: Retry와 Exponential Backoff — 재시도가 장애를 키우는 경우 ➡️](./03-retry-and-backoff.md)**

</div>

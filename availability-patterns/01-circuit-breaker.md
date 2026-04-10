# 01. Circuit Breaker 완전 분해 — Resilience4j 내부

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Circuit Breaker의 3가지 상태가 무엇이며, 언제 상태 전이가 일어나는가?
- COUNT_BASED 슬라이딩 윈도우와 TIME_BASED 슬라이딩 윈도우의 차이는 무엇인가?
- Half-Open 상태에서 Circuit Breaker가 어떻게 복구 가능성을 판단하는가?
- Resilience4j의 CircuitBreakerConfig에서 어떤 파라미터들이 상태 전이를 제어하는가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

마이크로서비스 아키텍처에서는 수십 개의 서비스가 서로를 호출합니다. 만약 한 서비스가 갑자기 느려지거나 다운되면, 그것을 호출하는 모든 클라이언트들이 무한정 대기하게 됩니다. 이는 연쇄적으로 다른 서비스까지 영향을 미쳐 전체 시스템이 마비되는 Cascading Failure를 초래합니다.

Circuit Breaker는 이런 상황에서 문제 있는 서비스로의 요청을 미리 차단함으로써, 불필요한 자원 낭비를 방지하고 시스템을 보호합니다. 마치 전기 회로에서 과전류 차단기가 위험한 상황에서 전원을 차단하듯이, 소프트웨어 Circuit Breaker도 문제가 있는 서비스로의 호출을 차단합니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ Circuit Breaker 없이 느린 서비스 호출
@Service
public class PaymentService {
    @Autowired
    private RestTemplate restTemplate;
    
    public Order processOrder(String orderId) {
        // 외부 결제 서비스 호출 — 응답 안 올 때까지 무한 대기
        try {
            String response = restTemplate.getForObject(
                "http://payment-service/pay?orderId=" + orderId,
                String.class
            );
            return parseResponse(response);
        } catch (Exception e) {
            // 예외 처리 없음 — 응답만 기다림
            throw new RuntimeException("결제 실패", e);
        }
    }
}

// 결과: 결제 서비스 장애 → OrderService 스레드 고갈 → 주문 API 응답 불가
// Cascading Failure가 발생하여 전체 시스템 다운
```

---

## ✨ 올바른 접근 (After)

```java
// ✅ Circuit Breaker로 장애 서비스 차단
@Service
public class PaymentService {
    
    private final RestTemplate restTemplate;
    private final CircuitBreakerFactory circuitBreakerFactory;
    
    @Autowired
    public PaymentService(
        RestTemplate restTemplate,
        CircuitBreakerFactory circuitBreakerFactory
    ) {
        this.restTemplate = restTemplate;
        this.circuitBreakerFactory = circuitBreakerFactory;
    }
    
    public Order processOrder(String orderId) {
        // Circuit Breaker를 통한 호출
        return circuitBreakerFactory.create("payment-service")
            .run(() -> callPaymentService(orderId),
                 throwable -> fallbackPaymentResponse(orderId));
    }
    
    private Order callPaymentService(String orderId) {
        String response = restTemplate.getForObject(
            "http://payment-service/pay?orderId=" + orderId,
            String.class
        );
        return parseResponse(response);
    }
    
    private Order fallbackPaymentResponse(String orderId) {
        // Circuit Breaker가 열려 있거나 호출 실패 시 폴백
        return Order.builder()
            .orderId(orderId)
            .status("PENDING_PAYMENT")
            .message("결제 서비스 일시 점검 중입니다.")
            .build();
    }
}
```

---

## 🔬 내부 동작 원리 — Circuit Breaker 상태 전이 심층 분석

### Circuit Breaker의 3가지 상태

```
┌──────────────────────────────────────────────────────────────┐
│                      CIRCUIT BREAKER                          │
│                                                              │
│    ┌────────────┐       실패율 초과           ┌────────┐    │
│    │  CLOSED    │─────────────────────────→  │  OPEN   │   │
│    │(정상 작동)  │                            │(차단)   │   │
│    └────────────┘                            └────────┘    │
│         ↑                                          │         │
│         │                          waitDuration   │         │
│         │                          경과           │         │
│         │                                         ↓         │
│         └─────────────────────┬──────────────┐   │         │
│                               │ HALF-OPEN   │   │         │
│                               │(탐색)       │───┘         │
│                               └──────────────┘             │
│                                                            │
│  CLOSED: 정상적으로 요청 통과                              │
│  OPEN: 모든 요청 즉시 실패 반환                            │
│  HALF-OPEN: 제한된 요청만 통과하여 서비스 복구 확인       │
└──────────────────────────────────────────────────────────────┘
```

### 상태별 상세 설명

**1. CLOSED (정상 작동)**
- Circuit Breaker의 기본 상태입니다.
- 모든 요청이 정상적으로 실제 서비스로 전달됩니다.
- 호출 결과 (성공/실패)를 슬라이딩 윈도우에 기록합니다.
- 실패율이 설정된 임계값을 초과하면 OPEN으로 전이합니다.

**2. OPEN (차단)**
- Circuit Breaker가 "열려" 있는 상태로, 모든 요청을 즉시 거부합니다.
- CallNotPermittedException 예외를 발생시키므로 실제 서비스 호출이 발생하지 않습니다.
- 이로 인해 불필요한 네트워크 요청과 타임아웃 대기를 방지합니다.
- waitDurationInOpenState(기본 60초)만큼 경과하면 HALF-OPEN으로 전이합니다.

**3. HALF-OPEN (탐색)**
- 서비스가 복구되었는지 확인하는 탐색 상태입니다.
- permittedNumberOfCallsInHalfOpenState(기본 3개)만큼의 제한된 요청만 통과시킵니다.
- 이 제한된 요청들의 결과를 기반으로:
  - 성공률이 임계값 이상 → CLOSED로 복구
  - 실패율이 임계값 초과 → 다시 OPEN으로 전환

### 슬라이딩 윈도우 두 가지 방식

**COUNT_BASED 윈도우 (기본값)**

```
최근 100개 호출의 성공/실패 기록

호출 101: Success ─┐
호출 102: Success ─┤
호출 103: Failure ─┤ 
호출 104: Failure ─┤ → 실패율 계산
호출 105: Success ─┤   (2 failures / 5 calls = 40%)
호출 106: Success ─┤
...              ─┤
호출 199: Success ─┤
호출 200: Success ─┘

실패율: 40 / 100 = 40% (임계값 50% 이하 → CLOSED 유지)
```

장점: 호출 수 기반이므로 이해하기 쉽습니다.
단점: QPS(분당 호출 수)에 따라 윈도우 크기가 시간적으로 달라집니다.

**TIME_BASED 윈도우**

```
최근 60초 동안의 호출 기록 (슬라이딩)

시간 →
[0초 ─────────────────────── 60초]
  ↑                           ↑
  윈도우 시작                 현재 시간
  
  SUCCESS SUCCESS FAILURE SUCCESS FAILURE ...
  
최근 60초 동안의 실패율을 계산
```

장점: 시간 기반이므로 예측 가능한 관찰 윈도우를 제공합니다.
단점: 네트워크 지연으로 인한 시간 기반 실패를 정확히 추적하기 어렵습니다.

---

## 💻 Resilience4j CircuitBreakerConfig 및 Spring Cloud 통합

### 1. application.yml 상세 설정

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        # 슬라이딩 윈도우 설정
        slidingWindowType: COUNT_BASED  # COUNT_BASED 또는 TIME_BASED
        slidingWindowSize: 100          # 최근 100개 호출 기반
        
        # 실패 판정 조건
        failureRateThreshold: 50        # 실패율 50% 이상 시 OPEN
        slowCallRateThreshold: 100      # 느린 호출(slowCallDuration 초과)이 100% 시 OPEN
        slowCallDurationThreshold: 2000 # 2000ms 이상 걸리면 느린 호출로 판정
        
        # 상태 전이
        waitDurationInOpenState: 60000  # 60초 대기 후 HALF-OPEN으로 전이
        permittedNumberOfCallsInHalfOpenState: 3  # HALF-OPEN에서 최대 3개 호출만 허용
        
        # 자동 전이 (HALF-OPEN → CLOSED)
        automaticTransitionFromOpenToHalfOpenEnabled: true
        
        # 최소 호출 수
        minimumNumberOfCalls: 10        # 최소 10개 호출이 기록되어야 실패율 계산
        
        # 예외 처리
        recordExceptions:               # 이 예외들을 실패로 기록
          - java.net.ConnectException
          - java.net.SocketException
          - java.net.SocketTimeoutException
          - org.springframework.web.client.ResourceAccessException
        ignoreExceptions:               # 이 예외들은 무시 (실패로 기록 안 함)
          - java.lang.IllegalArgumentException
    
    instances:
      # 외부 결제 서비스용 Circuit Breaker
      payment-service:
        baseConfig: default
        slidingWindowSize: 50           # 결제는 호출이 적으므로 50개
        failureRateThreshold: 40        # 더 엄격하게 40%
        waitDurationInOpenState: 30000  # 빠른 복구 시도 (30초)
      
      # 외부 추천 서비스용 Circuit Breaker
      recommendation-service:
        baseConfig: default
        failureRateThreshold: 60        # 추천은 덜 중요하므로 60%
        waitDurationInOpenState: 120000 # 더 긴 대기 (2분)

  timelimiter:
    configs:
      default:
        cancelRunningFuture: false
        timeoutDuration: 2s
    instances:
      payment-service:
        timeoutDuration: 1s
      recommendation-service:
        timeoutDuration: 3s
```

### 2. Java 설정 클래스

```java
package com.msa.resilience;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class CircuitBreakerConfiguration {
    
    // 프로그래매틱 방식의 Circuit Breaker 설정
    @Bean
    public CircuitBreaker paymentCircuitBreaker(CircuitBreakerRegistry registry) {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(50)
            .failureRateThreshold(40.0f)
            .slowCallRateThreshold(100.0f)
            .slowCallDurationThreshold(Duration.ofSeconds(2))
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(3)
            .automaticTransitionFromOpenToHalfOpenEnabled(true)
            .recordExceptions(
                java.net.ConnectException.class,
                java.net.SocketException.class,
                org.springframework.web.client.ResourceAccessException.class
            )
            .build();
        
        return registry.circuitBreaker("payment-service", config);
    }
}
```

### 3. @CircuitBreaker 애노테이션 사용

```java
package com.msa.payment;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class PaymentServiceImpl implements PaymentService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @CircuitBreaker(
        name = "payment-service",
        fallbackMethod = "paymentFallback"
    )
    public PaymentResult processPayment(String orderId, double amount) {
        // 실제 외부 결제 서비스 호출
        String url = "http://payment-service:8080/api/payment";
        PaymentRequest request = new PaymentRequest(orderId, amount);
        
        try {
            PaymentResponse response = restTemplate.postForObject(
                url,
                request,
                PaymentResponse.class
            );
            
            return PaymentResult.success(response.getTransactionId());
        } catch (Exception e) {
            // 이 예외는 자동으로 Circuit Breaker에 의해 기록됨
            throw new RuntimeException("결제 처리 실패", e);
        }
    }
    
    // fallbackMethod: @CircuitBreaker가 실패할 때 호출되는 메서드
    // 원본 메서드와 동일한 시그니처 필요
    public PaymentResult paymentFallback(
        String orderId,
        double amount,
        Exception ex
    ) {
        return PaymentResult.pending(
            "결제 서비스 일시점검 중입니다. 나중에 재시도해주세요.",
            orderId
        );
    }
}
```

### 4. Circuit Breaker 모니터링

```java
package com.msa.monitoring;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.core.registry.EntryAddedEvent;
import io.github.resilience4j.core.registry.EntryRemovedEvent;
import io.github.resilience4j.core.registry.RegistryEventConsumer;
import io.github.resilience4j.circuitbreaker.event.CircuitBreakerOnStateTransitionEvent;
import org.springframework.stereotype.Component;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Component
public class CircuitBreakerEventLogger implements RegistryEventConsumer<CircuitBreaker> {
    
    private static final Logger logger = LoggerFactory.getLogger(CircuitBreakerEventLogger.class);
    
    @Override
    public void onEntryAdded(EntryAddedEvent<CircuitBreaker> event) {
        CircuitBreaker circuitBreaker = event.getAddedEntry();
        circuitBreaker.getEventPublisher()
            .onStateTransition(this::logStateTransition)
            .onError(this::logError)
            .onSuccess(this::logSuccess);
    }
    
    private void logStateTransition(CircuitBreakerOnStateTransitionEvent event) {
        logger.warn(
            "[{}] 상태 전이: {} → {}",
            event.getCircuitBreakerName(),
            event.getStateTransition().getFromState(),
            event.getStateTransition().getToState()
        );
    }
    
    private void logError(CircuitBreakerOnErrorEvent event) {
        logger.error(
            "[{}] 호출 실패. 실패율: {}%",
            event.getCircuitBreakerName(),
            event.getCircuitBreakerMetrics().getFailureRate()
        );
    }
    
    private void logSuccess(CircuitBreakerOnSuccessEvent event) {
        logger.debug(
            "[{}] 호출 성공. 실패율: {}%",
            event.getCircuitBreakerName(),
            event.getCircuitBreakerMetrics().getFailureRate()
        );
    }
    
    @Override
    public void onEntryRemoved(EntryRemovedEvent<CircuitBreaker> event) {
        // 제거 처리
    }
}
```

### 5. Actuator를 통한 모니터링 (Spring Boot)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,circuitbreakerevents
  health:
    circuitbreakers:
      enabled: true
  metrics:
    distribution:
      percentiles-histogram:
        resilience4j.circuitbreaker.calls: true
```

**Actuator 엔드포인트 활용:**

```bash
# 모든 Circuit Breaker 상태 조회
curl http://localhost:8080/actuator/health/circuitbreakers

# 특정 Circuit Breaker 이벤트 확인
curl http://localhost:8080/actuator/circuitbreakerevents

# 메트릭 확인
curl http://localhost:8080/actuator/metrics/resilience4j.circuitbreaker.calls
```

---

## 📊 패턴 비교

| 항목 | COUNT_BASED | TIME_BASED |
|------|---|---|
| **계산 기준** | 최근 N개 호출 | 최근 N초 동안의 호출 |
| **QPS 변동 영향** | 높음 (QPS가 낮으면 윈도우 시간 길어짐) | 낮음 (항상 일정 시간 윈도우) |
| **실패율 계산 시점** | 새 호출이 추가될 때마다 | 시간 경과에 따라 자동 갱신 |
| **메모리 사용** | 낮음 | 높음 (타임스탬프 추적) |
| **권장 시나리오** | QPS가 일정한 서비스 | QPS가 급격히 변하는 서비스 |
| **구현 복잡도** | 낮음 | 높음 |

---

## ⚖️ 트레이드오프

### 장점
- **Cascading Failure 방지**: 문제 서비스로의 요청을 차단하여 연쇄 장애 방지
- **자원 절약**: Open 상태에서는 불필요한 네트워크 호출과 타임아웃 대기를 하지 않음
- **빠른 장애 감지**: 실시간으로 실패율을 모니터링하여 문제를 조기에 발견
- **자동 복구**: HALF-OPEN 상태에서 자동으로 서비스 복구 가능성을 재검증

### 단점
- **복잡한 설정**: failureRateThreshold, waitDurationInOpenState 등 여러 파라미터를 정확히 설정해야 함
- **거짓 양성(False Positive)**: 일시적 네트워크 지연으로 인해 정상 서비스도 차단될 수 있음
- **폴백 구현 필수**: Circuit Breaker가 열려 있을 때 어떻게 처리할지 정해야 함
- **모니터링 오버헤드**: 상태 전이와 실패율을 지속적으로 추적하기 위한 메트릭 수집 필요

---

## 📌 핵심 정리

✅ **Circuit Breaker는 3가지 상태(CLOSED, OPEN, HALF-OPEN)를 가지며, 실패율 임계값에 따라 자동으로 상태 전이**

✅ **슬라이딩 윈도우 (COUNT_BASED vs TIME_BASED)의 선택은 서비스의 QPS 패턴에 따라 결정**

✅ **HALF-OPEN 상태에서 제한된 수의 요청(예: 3개)만 허용하여 서비스 복구 여부를 판단**

✅ **failureRateThreshold, waitDurationInOpenState, permittedNumberOfCallsInHalfOpenState가 상태 전이의 핵심 파라미터**

✅ **@CircuitBreaker 애노테이션과 fallbackMethod를 함께 사용하여 선언적으로 구현 가능**

---

## 🤔 생각해볼 문제

**Q1.** 결제 서비스(payment-service)의 Circuit Breaker 설정에서 failureRateThreshold를 40%로 설정하고, 추천 서비스(recommendation-service)는 60%로 설정했습니다. 왜 이렇게 다르게 설정해야 할까요?

<details><summary>해설 보기</summary>
결제 서비스는 사용자의 금전이 관여된 중요한 서비스이므로, 약간의 오류만 나타나도 즉시 차단하여 잘못된 결제를 방지해야 합니다. 반면 추천 서비스는 사용자 경험을 향상시키는 부가 기능이므로, 일정 수준의 오류는 허용하면서도 계속 서비스를 제공할 수 있습니다. 이처럼 서비스의 중요도에 따라 Circuit Breaker의 민감도를 조정합니다.
</details>

**Q2.** HALF-OPEN 상태에서 3개의 호출을 시도했는데, 모두 실패했습니다. 이 경우 Circuit Breaker는 어떻게 동작할까요?

<details><summary>해설 보기</summary>
HALF-OPEN 상태에서 3개 중 모두 실패하면, 실패율이 100%가 되어 failureRateThreshold(보통 50%)를 초과합니다. 따라서 Circuit Breaker는 다시 OPEN 상태로 전환되고, waitDurationInOpenState(예: 60초)만큼 더 기다린 후 다시 HALF-OPEN으로 전환하여 복구를 재시도합니다.
</details>

**Q3.** COUNT_BASED 슬라이딩 윈도우에서 slidingWindowSize를 100으로 설정했을 때, minimumNumberOfCalls를 10으로 설정한 이유는 무엇일까요?

<details><summary>해설 보기</summary>
Circuit Breaker가 최근 100개 호출 중 몇 개의 실패만으로 즉시 OPEN으로 전환되면 안 됩니다. 예를 들어 호출이 1개만 있었는데 1개가 실패하면 실패율 100%로 OPEN 상태가 되어 거짓 양성(false positive)이 발생합니다. minimumNumberOfCalls는 실패율을 계산하기 전에 최소한의 호출 데이터가 필요함을 의미하므로, 최소 10개 호출이 기록된 후에야 실패율 계산을 시작합니다. 이는 우연의 실패로 인한 과도한 차단을 방지합니다.
</details>

---

<div align="center">

**[⬅️ 이전: Chapter 4 — Saga 실패 처리와 모니터링](../saga/06-saga-failure-and-monitoring.md)** | **[홈으로 🏠](../README.md)** | **[다음: Bulkhead 패턴 — 스레드 풀 격리로 장애 전파 차단 ➡️](./02-bulkhead.md)**

</div>

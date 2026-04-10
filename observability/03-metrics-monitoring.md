# 03. 메트릭 모니터링 — RED 방법론과 Grafana 대시보드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- RED 방법론(Rate, Errors, Duration)이란 무엇이고, 왜 MSA 모니터링에 최적화되었는가?
- Spring Boot Actuator와 Micrometer의 관계는 무엇이며, Prometheus 포맷으로 메트릭을 노출하는 방법은?
- Prometheus의 Counter, Gauge, Histogram, Summary 중언제 어떤 메트릭 타입을 사용해야 하는가?
- Grafana 대시보드에서 RED 메트릭을 시각화하고 임계값을 설정하는 방법은?
- AlertManager로 에러율 급증 시 Slack 또는 PagerDuty에 자동으로 알림을 전송하려면?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

마이크로서비스 아키텍처는 수십 개의 서비스가 동시에 실행됩니다. 모놀리식 애플리케이션이라면 "서버 CPU 사용률이 95%"라는 한 줄의 알림으로도 충분하지만, MSA에서는 "OrderService의 결제 API 응답시간이 2초에서 10초로 증가했다"는 구체적인 정보가 필요합니다. 어느 서비스가 느린지, 왜 느린지, 비정상 상황인지 정상 범위인지를 판단해야 합니다.

RED 방법론은 이 세 가지를 명확하게 정의합니다. Rate는 "얼마나 많은 요청이 들어왔는가", Errors는 "그중 몇 개가 실패했는가", Duration은 "평균적으로 얼마나 걸렸는가"입니다. 이 세 가지만 추적해도 대부분의 성능 문제를 찾을 수 있습니다. 예를 들어 Rate가 정상이지만 Errors가 5%로 증가했다면 새로운 배포에서 버그가 도입되었을 것이고, Duration이 2배로 증가했다면 데이터베이스 성능 저하를 의심할 수 있습니다.

또한 메트릭은 로그와 달리 저장공간이 적고 수집이 간단합니다. 로그는 매 요청마다 텍스트를 생성하므로 저장소가 폭발하지만, 메트릭은 "초당 처리 요청 수"처럼 요약된 숫자만 저장합니다. 프로덕션 환경에서 로그만 의존하면 저장소 비용이 급증하는데, 메트릭이 이를 보완합니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ 메트릭 없이 모니터링하려는 시도
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody OrderRequest request) {
        
        long startTime = System.currentTimeMillis();
        
        try {
            OrderResponse response = orderService.processOrder(request);
            
            // 응답시간을 로그에만 기록 (쿼리 불가능)
            long duration = System.currentTimeMillis() - startTime;
            System.out.println("Order creation took " + duration + "ms");
            
            return ResponseEntity.ok(response);
        } catch (Exception e) {
            // 에러도 로그에만 기록
            System.err.println("Error: " + e.getMessage());
            return ResponseEntity.status(500).build();
        }
    }
}

// 문제점:
// 1. 응답시간이 로그에만 기록됨 → 시간에 따른 그래프 불가능
// 2. 에러율을 계산하려면 로그를 수동으로 파싱해야 함
// 3. 평균, P95, P99 응답시간을 계산할 수 없음
// 4. Grafana 같은 모니터링 도구와 연동 불가능
```

```bash
# 문제 상황: 성능 저하 보고
# "결제가 느리다"는 불평이 들어옴

# 로그 분석 시도
$ grep "Order creation took" /var/log/app.log | tail -100
Order creation took 150ms
Order creation took 180ms
Order creation took 2500ms  # 이것만 느림?
Order creation took 160ms
...

# 문제:
# - 100개 로그 중에서 느린 요청이 숨어있음
# - 패턴을 찾기 위해 수동으로 계산해야 함
# - 시간대별 분석 불가능
# - 정확한 에러율 계산 불가능
```

```yaml
# 모니터링 도구 없이 수동 모니터링
# 15:30 OrderService 응답 느림 (로그에서 눈으로 확인)
# 15:31 여전히 느림
# 15:32 어, 이제 정상인 건가?
# 15:33 또 느려짐?

# 문제:
# - 수동 모니터링은 비효율적이고 오류가 많음
# - 야간에 발생한 문제는 아침까지 몰랐을 수도 있음
# - 자동 알림이 없어서 장애 대응이 늦음
```

---

## ✨ 올바른 접근 (After)

```java
// ✅ Micrometer를 사용한 메트릭 수집
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    // Micrometer의 Timer와 Counter 자동 주입
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Timer orderCreationTimer;
    private final Counter orderCreationSuccess;
    private final Counter orderCreationFailure;
    
    public OrderController(MeterRegistry meterRegistry) {
        // 메트릭 등록
        this.orderCreationTimer = Timer.builder("order.creation.duration")
                .description("주문 생성 소요 시간")
                .publishPercentiles(0.5, 0.95, 0.99)  // P50, P95, P99 자동 계산
                .tag("service", "order-service")
                .register(meterRegistry);
        
        this.orderCreationSuccess = Counter.builder("order.creation.success")
                .description("성공한 주문 생성")
                .tag("service", "order-service")
                .register(meterRegistry);
        
        this.orderCreationFailure = Counter.builder("order.creation.failure")
                .description("실패한 주문 생성")
                .tag("service", "order-service")
                .register(meterRegistry);
    }
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody OrderRequest request) {
        
        // Timer로 실행시간 자동 측정
        return orderCreationTimer.recordCallable(() -> {
            try {
                OrderResponse response = orderService.processOrder(request);
                
                // 성공 카운터 증가
                orderCreationSuccess.increment();
                
                // 상태 메트릭 기록
                meterRegistry.gauge("order.amount",
                        request.getAmount().doubleValue());
                
                return ResponseEntity.ok(response);
            } catch (Exception e) {
                // 실패 카운터 증가
                orderCreationFailure.increment();
                
                throw new RuntimeException(e);
            }
        });
    }
}

// 결과: Prometheus 형식으로 메트릭 노출
// # HELP order_creation_duration_seconds 주문 생성 소요 시간
// # TYPE order_creation_duration_seconds summary
// order_creation_duration_seconds_count{service="order-service"} 1000
// order_creation_duration_seconds_sum{service="order-service"} 156234.5
// order_creation_duration_seconds{quantile="0.5",service="order-service"} 0.156   (P50)
// order_creation_duration_seconds{quantile="0.95",service="order-service"} 0.512  (P95)
// order_creation_duration_seconds{quantile="0.99",service="order-service"} 1.234  (P99)
// 
// order_creation_success_total{service="order-service"} 950
// order_creation_failure_total{service="order-service"} 50
```

```yaml
# Prometheus에서 실시간 쿼리 가능
# 결과: Grafana 대시보드에서 시계열 그래프로 시각화

# Rate (요청 처리율)
rate(order_creation_duration_seconds_count[1m])
# 결과: 초당 약 16.67개 요청

# Errors (에러율)
100 * (rate(order_creation_failure_total[1m]) / rate(order_creation_duration_seconds_count[1m]))
# 결과: 에러율 5%

# Duration (응답시간)
order_creation_duration_seconds{quantile="0.95"}
# 결과: P95 응답시간 512ms
```

---

## 🔬 내부 동작 원리 — RED 방법론 심층 분석

### RED 방법론의 세 기둥

```
┌────────────────────────────────────────────────────────────────────┐
│                        RED 방법론                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Rate (요청 처리율)        Errors (에러율)      Duration (응답시간)  │
│  ─────────────────        ──────────────────    ──────────────────  │
│                                                                      │
│  초당 몇 개의 요청?        전체 요청 중         평균 응답시간은?      │
│  → 50 req/sec             몇 %가 실패?        → P50: 150ms         │
│  → 변화 추세               → 5% 에러율          → P95: 500ms         │
│  → 피크 시간대             → 임계값 초과        → P99: 1.2sec        │
│                                                                      │
│  의미:                     의미:                의미:                │
│  시스템 부하 수준          비정상 상황 감지     성능 최적화 기준      │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘

시간대별 메트릭 변화:

시간: 08:00 ~ 18:00 (업무 시간)

Rate:  [====] [======] [========] [======] [====]
         일정   점진    최고점     감소    정상화
        
Errors:[=] [===] [=====] [===] [=]
         저  상승  경고!  하강  정상
        
Duration:[==] [===] [=====] [===] [==]
          정상  상승  병목!  정상  정상
```

### 메트릭 타입별 사용 시기

```
┌────────────────────────────────────────────────────────────────┐
│                    메트릭 타입 비교                            │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│ 1. Counter (카운터)                                            │
│    ─────────────                                              │
│    - 증가만 가능, 감소 불가능                                 │
│    - 누적 값 (리셋 안 됨)                                     │
│    - 예: 처리한 요청 수, 발생한 에러 수                       │
│    - 그래프: ↗ (항상 우상향)                                  │
│                                                                 │
│    값:  [10] [12] [15] [20] [25] [31]  ← 항상 증가          │
│                                                                 │
│                                                                 │
│ 2. Gauge (게이지)                                              │
│    ──────────────                                             │
│    - 증가/감소 모두 가능                                      │
│    - 순간의 값 (메모리 사용량처럼)                            │
│    - 예: Heap 메모리, CPU 사용률, DB 연결 수                 │
│    - 그래프: 파동형                                           │
│                                                                 │
│    값:  [45%] [62%] [38%] [71%] [42%] [55%]  ← 변동        │
│                                                                 │
│                                                                 │
│ 3. Histogram (히스토그램)                                      │
│    ──────────────────────                                     │
│    - 값의 분포를 버킷으로 관리                                │
│    - P50, P95, P99 등 백분위수 계산 가능                     │
│    - 예: 응답시간 분포 (100ms 이상 몇개, 500ms 이상 몇개)   │
│    - 그래프: 분포도                                           │
│                                                                 │
│    값: [0-100ms: 500] [100-500ms: 300] [500-1000ms: 150]    │
│         [1000ms+: 30]                                         │
│                                                                 │
│                                                                 │
│ 4. Summary (서머리)                                            │
│    ────────────────                                           │
│    - Histogram과 유사하지만 서버 측에서 백분위수 계산        │
│    - Histogram: 클라이언트 측(Prometheus)에서 계산           │
│    - Summary: 서버 측에서 계산 후 결과만 전송               │
│    - 권장: Histogram 사용 (더 정확함)                        │
│                                                                 │
│    값: [quantile="0.5": 150ms] [quantile="0.95": 500ms]     │
│         [quantile="0.99": 1200ms]                            │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Prometheus 메트릭 형식

```
# TYPE order_creation_duration_seconds histogram
# HELP order_creation_duration_seconds 주문 생성 소요 시간 (초)

# Counter
order_creation_success_total{service="order-service"} 950
order_creation_failure_total{service="order-service"} 50

# Gauge
jvm_memory_used_bytes{area="heap"} 524288000
db_connection_pool_active_connections 23

# Histogram (Prometheus가 버킷으로 분류)
order_creation_duration_seconds_bucket{le="0.005",service="order-service"} 0
order_creation_duration_seconds_bucket{le="0.01",service="order-service"} 10
order_creation_duration_seconds_bucket{le="0.025",service="order-service"} 100
order_creation_duration_seconds_bucket{le="0.05",service="order-service"} 300
order_creation_duration_seconds_bucket{le="0.075",service="order-service"} 450
order_creation_duration_seconds_bucket{le="0.1",service="order-service"} 600
order_creation_duration_seconds_bucket{le="0.25",service="order-service"} 850
order_creation_duration_seconds_bucket{le="0.5",service="order-service"} 950
order_creation_duration_seconds_bucket{le="0.75",service="order-service"} 975
order_creation_duration_seconds_bucket{le="1.0",service="order-service"} 990
order_creation_duration_seconds_bucket{le="+Inf",service="order-service"} 1000

order_creation_duration_seconds_sum{service="order-service"} 156.234
order_creation_duration_seconds_count{service="order-service"} 1000

# Summary (이미 계산된 백분위수)
order_creation_duration_seconds{quantile="0.5",service="order-service"} 0.156
order_creation_duration_seconds{quantile="0.95",service="order-service"} 0.512
order_creation_duration_seconds{quantile="0.99",service="order-service"} 1.234
```

---

## 💻 Spring Boot + Micrometer + Prometheus 설정

```yaml
# application-prod.yml
spring:
  application:
    name: order-service

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  
  # 메트릭 수집 설정
  metrics:
    enable:
      jvm: true      # JVM 메트릭
      process: true  # 프로세스 메트릭
      system: true   # 시스템 메트릭
    distribution:
      percentiles-histogram:
        order.creation.duration: true  # Histogram 활성화
      slo:
        order.creation.duration: 50ms,100ms,500ms,1000ms  # SLO 버킷
    
    # 태그 설정 (모든 메트릭에 자동 추가)
    tags:
      application: ${spring.application.name}
      environment: prod
      region: us-east-1
  
  # Prometheus 설정
  prometheus:
    export:
      enabled: true
      step: 1m  # 메트릭 수집 간격
```

```java
// pom.xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```java
// MetricsConfiguration.java - 커스텀 메트릭
@Configuration
public class MetricsConfiguration {
    
    @Bean
    public MeterBinder orderServiceMetrics() {
        return (registry) -> {
            // 주문 처리 시간 (Timer)
            Timer.builder("order.processing.duration")
                    .description("주문 처리 소요 시간")
                    .publishPercentiles(0.5, 0.75, 0.95, 0.99)
                    .tag("service", "order")
                    .tag("operation", "process")
                    .register(registry);
            
            // 주문 생성 성공 (Counter)
            Counter.builder("order.created.total")
                    .description("생성된 주문 수")
                    .tag("service", "order")
                    .register(registry);
            
            // 결제 실패 (Counter)
            Counter.builder("payment.failed.total")
                    .description("실패한 결제 수")
                    .tag("service", "payment")
                    .register(registry);
            
            // DB 연결 풀 활용률 (Gauge)
            Gauge.builder("db.connection.pool.active", () -> {
                // 데이터베이스 연결 풀의 활성 연결 수 반환
                return DataSourceUtil.getActiveConnections();
            })
                    .description("활성 데이터베이스 연결 수")
                    .register(registry);
            
            // 캐시 히트율 (Gauge)
            Gauge.builder("cache.hit.ratio", () -> {
                long hits = CacheMetrics.getHits();
                long total = CacheMetrics.getTotal();
                return total > 0 ? (double) hits / total : 0;
            })
                    .description("캐시 히트율 (0.0 ~ 1.0)")
                    .register(registry);
        };
    }
}

// OrderService.java
@Service
public class OrderService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    public OrderResponse processOrder(Order order) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            // 비즈니스 로직
            OrderResponse response = executeOrder(order);
            
            // 성공 카운터
            meterRegistry.counter("order.processed.success",
                    "type", "standard",
                    "amount.range", getAmountRange(order.getAmount())
            ).increment();
            
            return response;
            
        } catch (PaymentException e) {
            meterRegistry.counter("order.failed.payment").increment();
            throw e;
            
        } catch (InventoryException e) {
            meterRegistry.counter("order.failed.inventory").increment();
            throw e;
            
        } finally {
            // 응답시간 기록 (밀리초)
            sample.stop(meterRegistry.timer("order.processing.duration"));
        }
    }
    
    private String getAmountRange(BigDecimal amount) {
        if (amount.compareTo(new BigDecimal("100")) < 0) return "small";
        if (amount.compareTo(new BigDecimal("1000")) < 0) return "medium";
        return "large";
    }
}
```

```yaml
# Prometheus 설정 (prometheus.yml)
global:
  scrape_interval: 15s  # 15초마다 메트릭 수집
  evaluation_interval: 15s
  external_labels:
    monitor: 'msa-monitoring'

# 알림 규칙
rule_files:
  - 'alert_rules.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - localhost:9093

scrape_configs:
  # Spring Boot Actuator에서 메트릭 수집
  - job_name: 'order-service'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s
  
  - job_name: 'payment-service'
    static_configs:
      - targets: ['localhost:8081']
    metrics_path: '/actuator/prometheus'
  
  - job_name: 'inventory-service'
    static_configs:
      - targets: ['localhost:8082']
    metrics_path: '/actuator/prometheus'
```

```yaml
# alert_rules.yml - AlertManager 규칙
groups:
  - name: msa_alerts
    interval: 10s
    rules:
      # 에러율이 5% 이상이면 알림
      - alert: HighErrorRate
        expr: |
          (rate(order_creation_failure_total[5m]) /
           rate(order_creation_duration_seconds_count[5m])) > 0.05
        for: 2m
        annotations:
          summary: "주문 서비스 에러율 높음: {{ $value | humanizePercentage }}"
          description: "최근 5분간 에러율이 5%를 초과했습니다"
      
      # 응답시간이 P95 > 1000ms이면 알림
      - alert: SlowResponseTime
        expr: order_creation_duration_seconds{quantile="0.95"} > 1
        for: 5m
        annotations:
          summary: "주문 서비스 응답 느림: {{ $value }}초"
          description: "P95 응답시간이 1초를 초과했습니다"
      
      # Heap 메모리 사용률이 90% 이상이면 알림
      - alert: HighHeapUsage
        expr: |
          (jvm_memory_used_bytes{area="heap"} /
           jvm_memory_max_bytes{area="heap"}) > 0.9
        for: 1m
        annotations:
          summary: "Heap 메모리 부족: {{ $value | humanizePercentage }}"
          description: "즉시 스케일링이 필요합니다"
```

```yaml
# AlertManager 설정 (alertmanager.yml)
global:
  resolve_timeout: 5m

route:
  receiver: 'msa-team'
  group_by: ['alertname', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h

receivers:
  - name: 'msa-team'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts'
        title: 'MSA 장애 알림'
        text: '{{ .GroupLabels.alertname }} - {{ .CommonAnnotations.summary }}'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_KEY'
        description: '{{ .GroupLabels.alertname }}'
```

---

## 📊 패턴 비교

| 항목 | RED 방법론 | Google의 USE | 전통적 모니터링 |
|------|----------|-------------|---------|
| **초점** | API/서비스 성능 | 자원 활용도 | 시스템 상태 |
| **Rate** | 요청 처리율 | Utilization | CPU/Disk 사용률 |
| **Errors** | 에러 비율 | Saturation | 에러 로그 |
| **Duration** | 응답시간 | Errors | 응답시간 |
| **모니터링 대상** | 마이크로서비스 | 인프라 | 서버 |
| **권장 환경** | MSA | 컨테이너/K8s | 모놀리식 |
| **활용도** | 매우 높음 | 높음 | 낮음 |

---

## ⚖️ 트레이드오프

### 장점 ✅
- **간단한 핵심 지표**: Rate, Errors, Duration 3가지만 추적
- **낮은 오버헤드**: 로그와 달리 메트릭은 저장공간 매우 적음
- **실시간 분석**: Prometheus는 초 단위로 메트릭 수집
- **자동 임계값 검증**: AlertManager로 SLO 위반 자동 감지
- **통합 대시보드**: Grafana로 모든 서비스를 한 화면에서 모니터링
- **근본 원인 분석**: Rate ↑ + Duration ↑ = 새로운 배포의 버그

### 단점 ❌
- **상세 정보 부족**: 특정 요청의 상세 로그 필요 시 분산 추적과 병행 필수
- **커스텀 메트릭 정의 필요**: 비즈니스 특화 메트릭은 수동으로 추가
- **Prometheus 저장소 관리**: 시계열 데이터베이스로 장기 보관 시 비용
- **쿼리 언어 학습**: PromQL을 배워야 정확한 쿼리 작성 가능
- **카디널리티 폭발**: 태그가 많을 경우 메트릭 수가 기하급수적 증가
- **샘플링 불가능**: 메트릭은 모두 수집 (로그는 샘플링 가능)

### 메트릭 저장소 비용 분석

```
메트릭 저장소 크기 추정:
- 서비스 개수: 10개
- 서비스당 메트릭: 50개
- 총 메트릭 개수: 500개

1년 저장소 계산 (15초 간격 수집):
- 초당 메트릭: 500개
- 1일: 500 × 86,400 / 15 = 2,880,000개
- 1년: 2,880,000 × 365 = 1,051,200,000개
- 저장소 (1메트릭 = 100 바이트): 약 100GB
- 비용 ($0.10/GB): $10/월

프로메테우스는 로그(월 $300)에 비해 매우 저렴!
```

---

## 📌 핵심 정리

✅ **RED 방법론의 세 기둥**
- Rate: 초당 처리 요청 수 (정상 범위 확인)
- Errors: 에러율 (장애 감지)
- Duration: 응답시간 P50/P95/P99 (성능 저하 감지)

✅ **메트릭 타입 선택 기준**
- Counter: 누적 값 (요청 수, 에러 수)
- Gauge: 순간 값 (메모리 사용률, 연결 수)
- Histogram: 분포 (응답시간 버킷)
- Summary: 서버 계산 백분위수 (구식, Histogram 권장)

✅ **Prometheus + Grafana 스택**
- Prometheus: 메트릭 수집 + PromQL 쿼리
- Grafana: 시각화 + 대시보드 + 알림
- AlertManager: 알림 라우팅 (Slack, PagerDuty)

✅ **SLO 설정의 중요성**
- P99 응답시간 < 1초
- 에러율 < 0.1%
- 가용성 > 99.99%
- 이를 초과하면 자동으로 알림

✅ **비용 효율성**
- 메트릭: 월 $10-50 (매우 저렴)
- 로그: 월 $300-1,000
- 메트릭으로 먼저 대부분의 문제 감지, 필요시 로그 조회

---

## 🤔 생각해볼 문제

**Q1.** 다음 PromQL 쿼리가 의미하는 바를 설명하시오.

```promql
histogram_quantile(0.95,
  rate(order_creation_duration_seconds_bucket[5m])
)
```

<details><summary>해설 보기</summary>
이 쿼리는 "최근 5분간 주문 생성 응답시간의 P95(95 백분위수)"를 계산합니다.

의미:
- 주문 생성 요청 100개 중 95개는 이 시간 이내에 완료됨
- 예: 결과가 512ms면, 95%의 요청은 512ms 이내, 5%는 512ms 이상 소요

사용 예:
- SLO: P95 < 500ms 로 설정하면, 512ms > 500ms → 알림 발생
- 성능 최적화: P95가 계속 증가하면 → DB 쿼리 최적화 필요
</details>

**Q2.** 서비스 에러율이 갑자기 5%에서 8%로 증가했다. RED 메트릭으로는 뭘 확인할 수 있고, 뭘 못 할까?

<details><summary>해설 보기</summary>
**확인 가능한 것:**
- Rate 확인: 요청 수가 많아져서인지 (요청 폭증 → 자원 부족) 확인
- Duration 확인: 응답시간도 같이 증가했는지 (과부하 신호)
- 어느 API에서 에러가 발생했는지 (태그 필터링)

**확인 불가능한 것:**
- 구체적인 에러 메시지 (어떤 에러인지)
- 스택 트레이스
- 특정 고객의 요청이 실패했는지
- 어느 라인에서 에러 발생했는지

**해결책:**
1. Prometheus 메트릭: 어디서 에러가 발생했는지 파악 (어느 서비스, 어느 API)
2. 분산 추적(Jaeger): 해당 요청의 상세 로그와 스택 트레이스 확인
3. 중앙화 로깅(ELK): 에러 원인 분석
</details>

**Q3.** 초당 1,000,000개 요청을 처리하는 시스템에서 모든 요청에 대해 응답시간을 Histogram으로 기록하면 어떤 문제가 발생할까?

<details><summary>해설 보기</summary>
**문제점:**

1. **카디널리티 폭발** (가장 심각)
```
가정:
- 100개의 다양한 엔드포인트
- 각 엔드포인트마다 50개의 버킷 (Histogram)
- 태그: service, endpoint, method, status 등

메트릭 개수 계산:
100 엔드포인트 × 50 버킷 × 4 태그 조합 = 20,000개 시계열

실제는 이보다 훨씬 더 많을 수 있음
```

2. **Prometheus 메모리 부족**
- Prometheus는 메모리에 모든 시계열을 로드
- 20,000개 시계열 × 100KB 이상 = 2GB 이상 메모리 필요

3. **저장소 폭발**
- 초당 1M 요청 × 초당 메트릭 1M개 저장
- 시간당 36억 개 데이터 포인트
- 월 저장소 수백TB

**해결 방법:**

1. **태그 제한**: 카디널리티 낮추기
```yaml
# ❌ 나쁜 예: customer_id를 태그로 사용
histogram_quantile(0.95,
  http_request_duration_seconds_bucket{customer_id="12345"}
)
# 이러면 고객 수만큼 메트릭 증가 (카디널리티 폭발)

# ✅ 좋은 예: 필수 태그만 사용
histogram_quantile(0.95,
  http_request_duration_seconds_bucket{endpoint="/api/orders", method="POST"}
)
```

2. **샘플링**: 모든 요청 대신 1%만 수집
3. **버킷 수 줄이기**: 50개 → 10개로 축소
4. **Prometheus 대신 다른 솔루션**: VictoriaMetrics 같은 대용량 시계열 DB
</details>

---

<div align="center">

**[⬅️ 이전: 중앙화 로깅 — Trace ID로 분산 로그 조합](./02-centralized-logging.md)** | **[홈으로 🏠](../README.md)** | **[다음: 배포 전략 — Blue/Green, Canary, Feature Flag ➡️](./04-deployment-strategies.md)**

</div>

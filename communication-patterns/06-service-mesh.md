# 06. Service Mesh — Envoy Sidecar와 mTLS

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Envoy Sidecar가 iptables를 통해 애플리케이션 코드 수정 없이 트래픽을 가로채는 원리는?
- Control Plane(Istiod)과 Data Plane(Envoy)의 역할 구분과 통신 흐름은?
- mTLS가 SPIFFE/SPIRE 인증서로 서비스 간 상호 인증하는 과정은?
- Istio Circuit Breaker가 Hystrix/Resilience4j 같은 코드 레벨 구현과 다른 점은?
- 언제 Service Mesh가 필요하고, 언제는 오버엔지니어링일까?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

마이크로서비스의 수가 늘어나면서 서비스 간 통신은 **기하급수적으로 복잡**해집니다. 100개 서비스가 있으면 최대 9,900개의 서비스 간 연결이 가능합니다(100 × 99 / 2). 각 연결에서 재시도, 타임아웃, 회로 차단, 속도 제한, 암호화 등을 처리해야 합니다.

**Service Mesh(예: Istio)는 이런 복잡성을 인프라 레벨로 끌어올립니다:**

1. **투명한 프록시**: 애플리케이션 코드 변경 없이 모든 트래픽을 가로챔 (Envoy Sidecar)
2. **중앙 정책 관리**: Control Plane에서 모든 서비스의 통신 규칙을 관리
3. **자동 mTLS**: 서비스 간 암호화 통신 자동화 (인증서 관리 불필요)
4. **관찰성**: 모든 트래픽의 메트릭, 로그, 트레이싱 자동 수집
5. **복원력**: Circuit Breaker, Retry, Timeout을 인프라에서 처리

예를 들어 Resilience4j로 Circuit Breaker를 구현하면, 각 서비스가 몇 줄의 코드를 추가해야 합니다. 반면 Service Mesh는 한 번의 설정으로 모든 서비스에 적용됩니다.

그러나 Service Mesh는 **복잡도, 성능 오버헤드, 학습 곡선**이 높습니다. 따라서 **언제 필요한지 판단**하는 것이 중요합니다.

---

## 😱 흔한 실수 (Before — 코드 레벨 복잡도)

```java
// ❌ 안티패턴: 각 서비스가 Circuit Breaker를 관리
// Payment Service 클라이언트 (Order Service에서 사용)

@Configuration
public class PaymentClientConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        // Resilience4j 설정
        RestTemplate template = new RestTemplate();
        
        // 각 서비스마다 설정 필요!
        CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults(
            "paymentService");
        
        Retry retry = Retry.ofDefaults("paymentService");
        
        TimeLimiter timeLimiter = TimeLimiter.ofDefaults(
            "paymentService");
        
        Bulkhead bulkhead = Bulkhead.ofDefaults("paymentService");
        
        return template;
    }
}

@Service
public class OrderService {
    
    @Autowired private RestTemplate restTemplate;
    @Autowired private CircuitBreakerRegistry registry;
    
    public PaymentResult processPayment(Order order) {
        // 복잡한 resilience 로직
        CircuitBreaker circuitBreaker = 
            registry.circuitBreaker("paymentService");
        
        Supplier<PaymentResult> supplier = () -> 
            restTemplate.postForObject(
                "http://payment-service/payments",
                new PaymentRequest(order),
                PaymentResult.class);
        
        // Decorators: Circuit Breaker → Retry → TimeLimiter
        Supplier<PaymentResult> decorated = 
            Decorators.ofSupplier(supplier)
                .withCircuitBreaker(circuitBreaker)
                .withRetry(Retry.ofDefaults("paymentService"))
                .withTimeLimiter(TimeLimiter.ofDefaults("paymentService"))
                .get();
        
        return Try.ofCallable(decorated::get)
            .recover(e -> new PaymentResult("FAILED", "Service unavailable"))
            .get();
    }
}

// 문제점:
// 1. 모든 서비스가 같은 설정 반복
// 2. Circuit Breaker 임계값 조정 시 모든 서비스 배포 필요
// 3. 새 서비스 추가 시 같은 코드 작성
// 4. Resilience4j 의존성이 모든 서비스에 필요
// 5. 개별 서비스에서 장애를 처리하다 보니 불일치 가능


// ❌ 안티패턴 2: mTLS를 코드에서 관리
@Configuration
public class MutualTlsConfig {
    
    @Bean
    public RestTemplate restTemplate(SSLContext sslContext) {
        // 각 서비스마다 인증서 관리 필수
        
        // 1. 클라이언트 인증서 로드
        KeyStore clientKeyStore = KeyStore.getInstance("PKCS12");
        try (InputStream input = 
            Files.newInputStream(Paths.get(
                "/etc/ssl/certs/client-cert.p12"))) {
            clientKeyStore.load(input, "password".toCharArray());
        }
        
        // 2. CA 인증서 로드
        KeyStore trustStore = KeyStore.getInstance("JKS");
        try (InputStream input = 
            Files.newInputStream(Paths.get(
                "/etc/ssl/certs/ca-trust.jks"))) {
            trustStore.load(input, "password".toCharArray());
        }
        
        // 3. SSLContext 설정
        KeyManagerFactory kmf = 
            KeyManagerFactory.getInstance("SunX509");
        kmf.init(clientKeyStore, "password".toCharArray());
        
        TrustManagerFactory tmf = 
            TrustManagerFactory.getInstance("SunX509");
        tmf.init(trustStore);
        
        SSLContext sslContext = SSLContext.getInstance("TLSv1.2");
        sslContext.init(kmf.getKeyManagers(), 
            tmf.getTrustManagers(), 
            new SecureRandom());
        
        // 4. RestTemplate에 적용
        HttpClientBuilder httpClientBuilder = 
            HttpClients.custom()
                .setSSLContext(sslContext)
                .setSSLHostnameVerifier(
                    new DefaultHostnameVerifier());
        
        return new RestTemplate(
            new HttpComponentsClientHttpRequestFactory(
                httpClientBuilder.build()));
    }
}

// 문제점:
// 1. 모든 서비스가 인증서 관리
// 2. 인증서 갱신 시 모든 서비스 재배포
// 3. 암호 평문 저장 (보안 문제)
// 4. 운영 복잡도 극도로 높음
// 5. 서비스가 100개면 100배 관리 부담

```

---

## ✨ 올바른 접근 (After — Service Mesh)

```yaml
# ✅ 개선 1: Service Mesh (Istio)로 통일된 정책 관리

# namespace에 sidecar 자동 주입 활성화
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled  # 모든 Pod에 Envoy sidecar 자동 주입

---

# Circuit Breaker 정책 (Istio)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-circuit-breaker
  namespace: production
spec:
  host: payment-service.production.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 1000
        maxRequestsPerConnection: 2
    outlierDetection:  # Circuit Breaker
      consecutive5xxErrors: 5         # 5개 연속 5xx → 제거
      interval: 30s                   # 30초마다 확인
      baseEjectionTime: 30s           # 30초 격리
      maxEjectionPercent: 50          # 최대 50% 격리
      minRequestVolume: 10            # 최소 10개 요청 필요

---

# Retry 정책
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
  namespace: production
spec:
  hosts:
  - payment-service
  http:
  - match:
    - uri:
        prefix: "/api/v1/payments"
    route:
    - destination:
        host: payment-service
        port:
          number: 8080
    retries:
      attempts: 3               # 최대 3회 재시도
      perTryTimeout: 5s         # 요청당 5초 타임아웃
    timeout: 15s                # 전체 타임아웃 15초

---

# 자동 mTLS (SPIFFE/SPIRE)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # 모든 서비스 간 mTLS 필수

---

# Authorization Policy (접근 제어)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-access
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/order-service"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["/api/v1/payments"]
```

```java
// ✅ 개선된 Order Service (코드 단순화)
@Configuration
@EnableEurekaClient
public class OrderServiceConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        // Istio가 처리하므로 단순한 설정만
        return new RestTemplate();
    }
}

@Service
public class OrderService {
    
    @Autowired private RestTemplate restTemplate;
    
    public Order createOrder(OrderRequest request) {
        Order order = new Order(request);
        
        // Circuit Breaker, Retry, Timeout, mTLS?
        // 모두 Istio (Service Mesh)가 처리!
        // 애플리케이션 코드는 깨끗함
        
        PaymentResult payment = restTemplate.postForObject(
            "http://payment-service/api/v1/payments",
            new PaymentRequest(order.getTotalAmount()),
            PaymentResult.class);
        
        if (payment.isSuccess()) {
            orderRepository.save(order);
        }
        
        return order;
    }
}
```

```bash
# ✅ Istio 설치 및 활성화
kubectl apply -f https://istio.io/latest/yaml/istio-operator.yaml

# Sidecar 자동 주입 확인
kubectl get pods -n production -o jsonpath='{.items[*].spec.containers[*].name}'
# 출력: order-service envoy payment-service envoy ...
# (각 Pod 옆에 envoy sidecar가 자동으로 주입됨)

# Traffic 관찰 (Kiali 대시보드)
kubectl port-forward -n istio-system svc/kiali 20000:20000
# http://localhost:20000 에서 Service Mesh 시각화
```

---

## 🔬 내부 동작 원리 — Envoy Sidecar와 mTLS

### 1. Envoy Sidecar가 트래픽을 가로채는 방식 (iptables)

```
Pod 구조 (Service Mesh 적용 후):

┌─────────────────────────────────────────┐
│ Pod: order-service                       │
├─────────────────────────────────────────┤
│                                         │
│  ┌────────────────────────┐             │
│  │ Application Container   │             │
│  │ (Order Service)         │             │
│  │                         │             │
│  │ 요청 → localhost:8080   │             │
│  └──────────────┬──────────┘             │
│                 │                        │
│     iptables 규칙이 가로챔                │
│     (투명한 프록시)                      │
│                 ↓                        │
│  ┌────────────────────────┐             │
│  │ Envoy Container        │             │
│  │ (Sidecar Proxy)        │             │
│  │                         │             │
│  │ :15001 (inbound)       │             │
│  │ :15006 (outbound)      │             │
│  │ :15000 (admin)         │             │
│  │ :15006 (local traffic)  │             │
│  │                         │             │
│  │ [Circuit Breaker]       │             │
│  │ [Retry Logic]           │             │
│  │ [mTLS Encryption]       │             │
│  │ [Rate Limiting]         │             │
│  │ [Logging/Metrics]       │             │
│  └──────────────┬──────────┘             │
│                 │                        │
└─────────────────┼──────────────────────┘
                  │
                  ↓ 외부 서비스로 전송
            Payment Service Pod
```

**iptables 규칙 (자동으로 설정됨):**

```bash
# 아웃바운드 트래픽 가로채기
iptables -t nat -A ISTIO_REDIRECT \
  -p tcp -j REDIRECT --to-port 15006

# 인바운드 트래픽 가로채기
iptables -t nat -A ISTIO_INBOUND \
  -p tcp --dport 8080 -j REDIRECT --to-port 15006

# 결과:
# localhost:8080 → :15006 (Envoy) → 실제 8080 (앱)
# 또는
# :8080 (외부) → :15006 (Envoy) → localhost:8080 (앱)
```

### 2. Control Plane (Istiod) vs Data Plane (Envoy)

```
┌─────────────────────────────────────────┐
│ Control Plane (Istiod)                  │
│ 역할: 정책 정의, 설정 관리              │
├─────────────────────────────────────────┤
│                                         │
│  VirtualService, DestinationRule,      │
│  AuthorizationPolicy 해석               │
│         ↓                               │
│  Envoy 설정 파일 생성 (.envoy.json)     │
│         ↓                               │
│  각 Pod의 Envoy에 푸시 (xDS protocol)  │
│                                         │
└────────────────────────┬────────────────┘
                         │ gRPC (xDS)
                         │ 설정 푸시
                         ↓
    ┌────────────────────────────────┐
    │ Data Plane (Envoy Sidecar)     │
    │ 역할: 실제 트래픽 처리          │
    ├────────────────────────────────┤
    │                                │
    │ Listener (15001, 15006, ...)  │
    │ Route (경로 기반 라우팅)        │
    │ Cluster (끝점 관리)            │
    │ Endpoint (서비스 인스턴스)      │
    │                                │
    │ Circuit Breaker, Retry,        │
    │ mTLS, Rate Limit 적용          │
    │                                │
    └────────────────────────────────┘

통신 흐름:
1. User가 VirtualService 정의 (kubectl apply)
2. Istiod가 감지 (etcd watch)
3. Istiod가 Envoy 설정 생성
4. gRPC xDS로 모든 Envoy에 푸시
5. Envoy가 설정 적용 (다운타임 없이)
```

### 3. mTLS (Mutual TLS)와 SPIFFE/SPIRE

```
mTLS 핸드셰이크 (Service Mesh):

클라이언트 (Order Service)              서버 (Payment Service)
         │                                    │
         │ ← Self-signed 인증서 (Istiod)     │
         │   SPIFFE ID 포함                   │
         │   spiffe://cluster.local/ns/prod/  │
         │   sa/order-service                 │
         │                                    │
         ├──── ClientHello + Cert ────→       │
         │                              ← ServerHello + Cert
         │                                    │
         ├──── ClientKeyExchange ────→        │
         │                              ← Finished
         │                                    │
         └────────── 암호화된 통신 ──────────→│
         │                              │
         │ ← Self-signed 인증서 (Istiod)     │
         │   SPIFFE ID 포함                   │
         │   spiffe://cluster.local/ns/prod/  │
         │   sa/payment-service               │
         │                                    │

SPIFFE (Secure Production Identity Framework For Everyone):
- 서비스 간 신뢰 기반 구축
- SPIFFE ID = URI로 서비스 식별
- 예: spiffe://cluster.local/ns/production/sa/payment-service

SPIRE (SPIFFE Runtime Environment):
- SPIFFE ID와 인증서 발급
- 자동 인증서 갱신 (만료 3시간 전)
- Kubernetes Service Account와 매핑

인증서 생명주기:
[Istiod/CA] → 인증서 생성 → Envoy에 주입 (시크릿으로)
                ↓
           2.5시간 전 (만료 3시간)
                ↓
           새 인증서 생성 → Envoy에 업데이트
                ↓
           기존: 계속 사용, 새 요청: 새 인증서 사용
```

### 4. Circuit Breaker 정책 실행

```
Outlier Detection (Circuit Breaker):

[Payment Service Cluster]
├─ Instance 1 (10.0.0.1:8080) ← Healthy
├─ Instance 2 (10.0.0.2:8080) ← Healthy
└─ Instance 3 (10.0.0.3:8080) ← Unhealthy

요청 흐름:

[Order Service]
    ↓
[Envoy (Circuit Breaker)]
    ├─ 요청 1 → Instance 1 (200 OK)
    ├─ 요청 2 → Instance 2 (200 OK)
    ├─ 요청 3 → Instance 3 (500 ERROR) ← 카운트 증가
    ├─ 요청 4 → Instance 3 (500 ERROR) ← 카운트: 2
    ├─ 요청 5 → Instance 3 (500 ERROR) ← 카운트: 3
    │...
    ├─ 요청 8 → Instance 3 (500 ERROR) ← 카운트: 5
    │           Circuit Open! ⚡
    │
    │ 다음 30초간:
    ├─ 요청 9 → Instance 1 (200 OK)
    ├─ 요청 10 → Instance 2 (200 OK)
    ├─ 요청 11 → Instance 1 (200 OK)
    │ (Instance 3 완전 우회)
    │
    │ 30초 후 (baseEjectionTime):
    ├─ 요청 12 → Instance 1 (200 OK)
    ├─ 요청 13 → Instance 2 (200 OK)
    ├─ 요청 14 → Instance 3 (성공!) ← 복구 감지
    │ Instance 3 다시 추가
    │
    └─ 정상 상태로 복구 ✅

DestinationRule 설정:
outlierDetection:
  consecutive5xxErrors: 5      # 5회 연속 5xx
  interval: 30s                # 30초마다 확인
  baseEjectionTime: 30s        # 30초 격리
  maxEjectionPercent: 50       # 최대 50% 제거
```

---

## 💻 코드 예제 — Istio 설정

### Istio VirtualService (라우팅)
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
  namespace: production
spec:
  # 이 VirtualService가 적용되는 호스트
  hosts:
  - payment-service
  - payment-service.production
  - payment-service.production.svc.cluster.local
  
  # HTTP 라우팅 규칙
  http:
  # 규칙 1: /api/v1  경로
  - match:
    - uri:
        prefix: "/api/v1"
    route:
    - destination:
        host: payment-service
        subset: v1        # DestinationRule에서 정의한 subset
        port:
          number: 8080
      weight: 80          # 80% 트래픽
    - destination:
        host: payment-service
        subset: v2
        port:
          number: 8080
      weight: 20          # 20% 트래픽 (카나리 배포)
    timeout: 10s          # 요청 타임아웃
    retries:
      attempts: 3         # 최대 3회 재시도
      perTryTimeout: 5s   # 요청당 5초
  
  # 규칙 2: 기본 경로
  - route:
    - destination:
        host: payment-service
        port:
          number: 8080
```

### Istio DestinationRule (트래픽 정책)
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
  namespace: production
spec:
  host: payment-service
  
  # 트래픽 정책 (모든 subset에 적용)
  trafficPolicy:
    connectionPool:
      # TCP 연결 풀
      tcp:
        maxConnections: 100
      # HTTP 연결 풀
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 1000
        maxRequestsPerConnection: 2
    
    # Circuit Breaker (Outlier Detection)
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minRequestVolume: 10
    
    # 로드 밸런싱
    loadBalancer:
      simple: ROUND_ROBIN
  
  # 서브셋 정의 (버전 기반)
  subsets:
  # v1 버전
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      connectionPool:
        http:
          http1MaxPendingRequests: 100
  
  # v2 버전 (카나리)
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      connectionPool:
        http:
          http1MaxPendingRequests: 10
```

### Istio AuthorizationPolicy (접근 제어)
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-policy
  namespace: production
spec:
  # 이 정책이 적용되는 워크로드
  selector:
    matchLabels:
      app: payment-service
  
  # STRICT: 정책이 없으면 거부
  # CUSTOM: 혼합 (정책 + 논리)
  # AUDIT: 감시만 (거부하지 않음)
  action: ALLOW
  
  # 접근 허용 규칙
  rules:
  
  # 규칙 1: Order Service에서 오는 요청 허용
  - from:
    - source:
        # SPIFFE ID로 식별
        principals:
        - "cluster.local/ns/production/sa/order-service"
    to:
    - operation:
        methods: ["POST"]
        paths: ["/api/v1/payments"]
    
    # 사용자 정의 조건
    when:
    - key: request.auth.claims[sub]
      notValues: ["system:serviceaccount"]
  
  # 규칙 2: Istio Ingress Gateway에서 오는 요청 허용
  - from:
    - source:
        namespaces: ["istio-system"]
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["*"]
  
  # 규칙 3: 같은 namespace 내 모든 요청 허용
  - from:
    - source:
        principals:
        - "cluster.local/ns/production/*"
```

### mTLS 설정 (PeerAuthentication)
```yaml
# Namespace 전체에 mTLS 적용
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  # UNSET: 상속
  # DISABLE: mTLS 비활성화
  # PERMISSIVE: mTLS 선택적 (plaintext + mTLS 동시 지원)
  # STRICT: mTLS 필수
  mtls:
    mode: STRICT

---

# 특정 서비스에 다른 정책 적용
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: payment-mtls
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  
  mtls:
    mode: STRICT
  
  # 포트별 다른 정책
  portLevelMtls:
    8090:
      mode: DISABLE  # 헬스체크는 plaintext 허용
```

### Java 애플리케이션 (설정 불필요)
```java
// Istio가 모든 처리를 하므로
// 애플리케이션 코드는 평상시와 동일

@Configuration
public class PaymentServiceConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        // 단순한 RestTemplate (Istio가 모든 처리)
        return new RestTemplate();
    }
}

@Service
public class PaymentService {
    
    @Autowired private RestTemplate restTemplate;
    
    public Payment processPayment(PaymentRequest request) {
        // mTLS? Circuit Breaker? Retry?
        // 모두 Istio (Envoy Sidecar)가 처리!
        
        Payment payment = restTemplate.postForObject(
            "http://invoice-service/api/v1/invoices",
            request,
            Payment.class);
        
        return payment;
    }
}
```

---

## 📊 패턴 비교 — Service Mesh vs 코드 레벨

| 구분 | Service Mesh (Istio) | 코드 레벨 (Resilience4j) |
|------|-------------------|----------------------|
| **Circuit Breaker** | 설정 (YAML) | 코드 (Annotation) |
| **Retry** | 설정 | 코드 |
| **Timeout** | 설정 | 코드 |
| **mTLS** | 자동 (SPIFFE) | 수동 (인증서 관리) |
| **Rate Limiting** | 설정 | 코드 |
| **메트릭 수집** | 자동 (모든 트래픽) | 선택적 |
| **로깅** | 자동 (모든 트래픽) | 선택적 |
| **개발 변경** | 없음 | 필수 (의존성, 코드) |
| **배포 시간** | 즉시 (설정 적용) | 배포 필요 |
| **언어 독립** | ✅ (인프라 레벨) | ❌ (언어별) |
| **운영 복잡도** | 중간 (Service Mesh 관리) | 높음 (각 서비스) |
| **성능 오버헤드** | 2-5% (Sidecar) | <1% (라이브러리) |
| **학습 곡선** | 가파름 | 낮음 |

---

## ⚖️ 트레이드오프

```
Service Mesh의 이점과 비용:

이점:
✅ 중앙 정책 관리 (모든 서비스에 일관되게 적용)
✅ 자동 mTLS (인증서 관리 자동화)
✅ 언어 독립적 (Go, Java, Python, Node.js 통일)
✅ 배포 중단 없이 정책 변경 (YAML 업데이트)
✅ 완전한 관찰성 (모든 트래픽 메트릭/로그)
✅ 트래픽 조정 쉬움 (Canary, A/B 테스트)

비용:
❌ 인프라 복잡도 증가
   - Istiod, Envoy, CSR 관리
   - 문제 추적 어려움 (Sidecar에서 뭔가 잘못됨)
❌ 성능 오버헤드 (2-5%)
   - 각 Pod에 Envoy Sidecar (메모리 추가)
   - 네트워크 홉 증가 (app → sidecar → network)
❌ 학습 곡선 가파름
   - Istio 개념 학습 (VirtualService, DestinationRule)
   - gRPC, xDS, SPIFFE 이해 필요
❌ 문제 해결 어려움
   - 트래픽이 Envoy를 거쳐서 흐름
   - 네트워크 정책과 Istio 정책 충돌 가능

────────────────────────────────────────

언제 필요한가:

Service Mesh 필요 (✅)
- 20+ 마이크로서비스
- 다양한 언어/프레임워크 혼재
- 높은 보안 요구사항 (mTLS)
- 복잡한 트래픽 관리 (카나리, A/B)
- 운영 팀이 충분함

Service Mesh 불필요 (❌)
- < 10 서비스
- 단일 언어/프레임워크
- 보안 덜 중요함
- 간단한 트래픽 패턴
- 운영 인력 부족

중간: 코드 레벨 라이브러리 사용
- Resilience4j, Hystrix
- 개발 팀이 각 서비스에서 관리
```

---

## 📌 핵심 정리

```
✅ Envoy Sidecar의 투명한 프록시

1. iptables 규칙으로 트래픽 가로채기
   - 애플리케이션 코드 수정 없음
   - 모든 트래픽 처리 (TCP, HTTP, gRPC)

2. 인바운드 (수신 요청)
   :15006 → 로컬 앱
   (요청 헤더 검증, RBAC 확인)

3. 아웃바운드 (발신 요청)
   로컬 앱 → :15006 → 원격 서비스
   (Circuit Breaker, Retry, mTLS)

✅ Control Plane (Istiod) vs Data Plane (Envoy)

Control Plane:
- VirtualService, DestinationRule 해석
- Envoy 설정 파일 생성
- gRPC xDS로 Envoy에 푸시 (다운타임 없이)

Data Plane:
- 실제 트래픽 처리
- Circuit Breaker, Retry, mTLS 적용
- 메트릭, 로그 수집

✅ mTLS (Mutual TLS) + SPIFFE

1. 인증서 자동 발급 (Istiod/CA)
   - Kubernetes Service Account 기반
   - SPIFFE ID: spiffe://cluster/ns/prod/sa/service

2. 인증서 자동 갱신
   - 만료 3시간 전부터 새 인증서 발급
   - Envoy가 자동으로 적용 (zero-downtime)

3. 서비스 간 상호 인증
   - 클라이언트: 서버의 인증서 검증
   - 서버: 클라이언트의 인증서 검증
   - 결과: 신뢰할 수 있는 서비스 간 통신

✅ Istio Circuit Breaker vs Resilience4j

Istio Circuit Breaker (Outlier Detection):
```yaml
outlierDetection:
  consecutive5xxErrors: 5  # 5회 연속 실패
  baseEjectionTime: 30s    # 30초 격리
  maxEjectionPercent: 50   # 최대 50% 제거
```

Resilience4j Circuit Breaker:
```java
CircuitBreaker circuitBreaker = CircuitBreaker.of(
    "paymentService",
    CircuitBreakerConfig.custom()
        .failureThreshold(50)
        .waitDurationInOpenState(30)
        .build());
```

차이점:
- Istio: 인프라 레벨, 모든 서비스에 동일 정책
- Resilience4j: 코드 레벨, 서비스별 커스터마이징

✅ Service Mesh 도입 판단

도입 필요:
- 20+ 마이크로서비스
- 복잡한 트래픽 관리 (카나리, A/B, 트래픽 분할)
- 높은 보안 요구사항 (mTLS, RBAC)
- 운영 팀이 있음

도입 미연기:
- 10개 이하 서비스
- 단일 언어/프레임워크
- 단순한 배포 전략
- 운영 인력 부족

단계별 도입:
1. Namespace 격리 + Sidecar 자동 주입
2. mTLS 활성화 (STRICT)
3. VirtualService + DestinationRule 적용
4. AuthorizationPolicy 추가 (RBAC)
5. 트래픽 관리 (Canary, A/B) 추가
```

---

## 🤔 생각해볼 문제

**Q1.** Payment Service에 5개 인스턴스가 있는데, 1개가 계속 500 에러를 냅니다. Istio Circuit Breaker가 자동으로 격리하고 나머지 4개로만 요청을 라우팅할까?

<details>
<summary>해설 보기</summary>

**Istio Outlier Detection (Circuit Breaker):**

```
상황:
- Payment Service Cluster: instance-1~5
- instance-3가 계속 500 에러 발생

DestinationRule 설정:
outlierDetection:
  consecutive5xxErrors: 5      # 5회 연속 실패
  interval: 30s                # 30초마다 확인
  baseEjectionTime: 30s        # 30초 격리

실행 흐름:
1. 요청 1 → instance-3 (500)  [count: 1]
2. 요청 2 → instance-3 (500)  [count: 2]
3. 요청 3 → instance-3 (500)  [count: 3]
4. 요청 4 → instance-3 (500)  [count: 4]
5. 요청 5 → instance-3 (500)  [count: 5]
                              → 격리! ⚡

6-36초 동안:
요청 6 → instance-1 (200)
요청 7 → instance-2 (200)
요청 8 → instance-4 (200)
요청 9 → instance-5 (200)
(instance-3은 완전히 무시)

36초 후 (baseEjectionTime 만료):
요청 10 → instance-3 시도
         (200) ← 복구됨!
         → 다시 추가 ✅

결과: 자동 격리 및 복구! ✅
사용자 경험: 요청 6부터는 지연 없음
```

**주의사항:**

```java
// ❌ 주의: 인스턴스 5개가 모두 실패하면?
outlierDetection:
  maxEjectionPercent: 50  # 최대 50%까지만 제거

// 5개 중 5개가 실패해도 최대 50%만 격리 가능
// → 최소 2개는 트래픽 받음 (보호)
// → 모든 인스턴스가 죽어도 최소 일부는 시도

// 권장: maxEjectionPercent < 50 (보수적)
maxEjectionPercent: 30  # 최대 30% (안전)
```

**Self-Healing:**

```yaml
# Circuit Breaker도 일부 트래픽을 격리된 인스턴스에 보냄
# (주기적으로 복구 확인)

minRequestVolume: 10    # 최소 10개 요청 후 판단
                        # → 우발적 오류로 격리 방지

# 예: Instance-3이 1개 요청에서 500 오류
# minRequestVolume: 10이므로
# 10개 요청까지는 계속 시도 후 판단
```

</details>

**Q2.** Service Mesh (Istio)를 도입하면 애플리케이션 성능이 얼마나 떨어질까? 2-5% 오버헤드가 실제로 감당할 수 있는 수준?

<details>
<summary>해설 보기</summary>

**Performance Overhead 분석:**

```
요청 경로:

Without Service Mesh:
[Client] → [App] → [Network] → [App] → [Response]
          (100ms)  (50ms)     (100ms)

With Service Mesh:
[Client] → [Envoy] → [App] → [Network] → [App] → [Envoy] → [Response]
          (5ms)    (100ms)  (50ms)     (100ms)   (5ms)

추가 지연: 10ms (2-5%)

실제 측정값:
- Kubernetes 클러스터 (e2e): 1-3% 지연
- 네트워크 요청 (gRPC): 2-5% 지연
- 로컬 프로세스 간: 5-10% 지연

```

**비용/효과 분석:**

```
시나리오 1: 낮은 지연 요구사항 (P99 < 100ms)
기존: 평균 80ms, P99 100ms
Istio: 평균 82ms, P99 105ms (추가 5ms)

영향: 거의 없음 (5%)
선택: Istio 도입 가능 ✅

────────────────────────────────────

시나리오 2: 초저지연 요구사항 (P99 < 10ms)
기존: 평균 5ms, P99 10ms
Istio: 평균 5.5ms, P99 15ms (추가 50%)

영향: 심각 (50% 증가)
선택: Istio 미도입, 코드 레벨 관리 ❌

────────────────────────────────────

시나리오 3: 일반적인 비즈니스 로직 (P99 < 500ms)
기존: 평균 400ms, P99 500ms
Istio: 평균 410ms, P99 510ms (추가 10ms)

영향: 무시할 수준 (2%)
선택: Istio 도입 권장 ✅
```

**오버헤드 최소화:**

```yaml
# 1. Envoy 리소스 설정
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: 500m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 512Mi
  
  # Sidecar (Envoy)
  - name: istio-proxy
    resources:
      requests:
        cpu: 100m          # 100m CPU 할당
        memory: 128Mi      # 128Mi 메모리
      limits:
        cpu: 200m
        memory: 256Mi      # 적절한 리소스 제한

# 2. Envoy 설정 최적화
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-config
data:
  mesh.yaml: |
    enableAutoMtls: true
    inboundTrafficPolicy: ALLOW_ANY
    
    # 커넥션 풀 최적화
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 1000
        maxRequestsPerConnection: 2
```

**권장:**

```
지연이 < 500ms인 대부분의 비즈니스 애플리케이션:
→ Istio 도입 (2-5% 오버헤드 감수)
→ 이득 (중앙 정책, mTLS 자동화, 관찰성) > 비용

지연이 < 50ms인 초저지연 시스템:
→ Istio 미도입
→ 코드 레벨 라이브러리 (Resilience4j) 사용
→ 성능 우선

트레이드오프:
오버헤드 5% < 운영 간소화 30% → Istio 도입 ✅
```

</details>

**Q3.** DestinationRule의 `maxEjectionPercent: 50`이 무엇일까? 5개 인스턴스 중 몇 개까지 격리할 수 있나?

<details>
<summary>해설 보기</summary>

**maxEjectionPercent 의미:**

```
maxEjectionPercent: 격리할 수 있는 최대 호스트 비율

설정: maxEjectionPercent: 50

인스턴스 5개 (instance-1~5):
- 5 × 50% = 2.5 → 최대 2개 격리 가능
- 최소 3개는 항상 트래픽 받음 (보호)

인스턴스 10개:
- 10 × 50% = 5개 격리 가능
- 최소 5개는 항상 트래픽 받음

인스턴스 100개:
- 100 × 50% = 50개 격리 가능
- 최소 50개는 항상 트래픽 받음
```

**실제 시나리오:**

```
상황: 5개 인스턴스 모두 장애 (예: 배포 오류)

maxEjectionPercent: 50
- instance-1: 500 (count: 5) → 격리
- instance-2: 500 (count: 5) → 격리
- instance-3: 500 (하지만 maxEjectionPercent 50% 도달)
             → 더 이상 격리 안 함

결과:
- instance-1, 2: 격리 (우회)
- instance-3, 4, 5: 계속 트래픽 받음
  (모두 500 에러)

사용자 영향:
- 1/3 요청: 에러 (instance-3,4,5가 500 반환)
- 2/3 요청: 계속 재시도 (circuit breaker)

좋은 점:
- 전체 장애보다는 부분 장애
- 자동 복구 모니터링 계속
```

**권장 설정:**

```yaml
# 작은 클러스터 (인스턴스 < 5)
maxEjectionPercent: 30  # 더 보수적

# 중간 클러스터 (5-20)
maxEjectionPercent: 50

# 대규모 클러스터 (> 50)
maxEjectionPercent: 50-80  # 가능

예:
outlierDetection:
  consecutive5xxErrors: 5
  interval: 30s
  baseEjectionTime: 30s
  maxEjectionPercent: 30    # 보수적
  minRequestVolume: 10
```

**vs Circuit Breaker Open 상태:**

```
Outlier Detection (Envoy):
- 특정 인스턴스 격리
- 다른 인스턴스는 계속 사용
- Partial failure 처리

Circuit Breaker Open:
- 전체 서비스 거부
- 모든 요청 fast-fail
- 전체 장애

차이점:
Outlier Detection = 세밀한 제어
Circuit Breaker = 극단적인 보호
```

</details>

---

<div align="center">

**[⬅️ 이전: API Composition 패턴 — 여러 서비스 데이터 조합](./05-api-composition.md)** | **[홈으로 🏠](../README.md)** | **[다음: GraphQL Federation — 분산 스키마 통합 ➡️](./07-graphql-federation.md)**

</div>

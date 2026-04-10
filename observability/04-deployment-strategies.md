# 04. 배포 전략 — Blue/Green, Canary, Feature Flag

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Blue/Green 배포와 Canary 배포의 차이점은 무엇이고, 각각의 장단점은?
- Kubernetes의 RollingUpdate 전략이 내부적으로 어떻게 동작하는가?
- Canary 배포에서 새 버전으로 점진적으로 트래픽을 이전하는 방법은?
- Feature Flag를 사용하여 배포와 기능 활성화를 분리하는 이유는?
- A/B 테스트를 구현하기 위해 필요한 것들은 무엇인가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

모놀리식 애플리케이션에서 배포는 전체 애플리케이션을 새 버전으로 교체하는 작업입니다. 전체 사용자가 새 버전으로 이동하므로, 버그가 있으면 모든 사용자가 영향을 받습니다. 반면 MSA에서는 수십 개의 서비스 중 하나만 배포합니다. 하지만 이것이 동시에 문제가 됩니다. OrderService를 배포했는데 버그가 있어서 주문 생성이 실패하기 시작하면, 고객들은 5초 안에 불평합니다.

배포 전략의 목표는 "장애를 최소화하면서 빠르게 새 기능을 출시"하는 것입니다. Blue/Green 배포는 이전 버전(Blue)을 유지하면서 새 버전(Green)을 준비한 후, 한 번에 전환합니다. 문제 발생 시 즉시 Blue로 되돌릴 수 있습니다. Canary 배포는 더 보수적으로, 새 버전에 1%의 트래픽을 먼저 보낸 후, 안정성을 확인하고 점진적으로 늘립니다. Feature Flag는 한 단계 더 나아가, 코드는 배포했지만 기능은 끌 수 있으므로, 배포와 활성화를 완전히 분리합니다.

특히 MSA 환경에서는 여러 서비스의 배포를 조율해야 합니다. OrderService 1.2와 PaymentService 1.1의 호환성을 보장해야 하고, 한 서비스가 다운되어도 다른 서비스는 계속 운영되어야 합니다. 이러한 복잡성을 관리하기 위해 체계적인 배포 전략이 필수입니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ 모든 사용자에게 한 번에 배포 (Big Bang Deploy)
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody OrderRequest request) {
        
        // v1.0: 정상 동작
        // v2.0: 새로운 로직 추가 (버그 있음)
        
        // 문제: 새 버전으로 모든 서버 동시에 교체
        // 만약 버그가 있으면 모든 사용자 영향!
        
        return orderService.processOrder(request);
    }
}

// 배포 과정:
// 시간 10:00 - 배포 시작
// 시간 10:05 - 모든 서버(10개)를 v2.0으로 교체
// 시간 10:06 - 오류 발생! 주문 생성 실패율 100%
// 시간 10:07 - 고객 불평 폭증
// 시간 10:15 - 문제 원인 파악
// 시간 10:20 - v1.0으로 롤백 시작
// 시간 10:25 - 롤백 완료, 15분간 장애

// 피해:
// - 15분 동안 1,000개 주문 손실
// - 고객 신뢰도 급락
// - 긴급 디버깅 비용
```

```bash
# 배포 스크립트 (위험함)
#!/bin/bash
set -e

# 모든 주문 서비스 인스턴스 중지
for instance in order-service-{1..10}; do
    ssh $instance "systemctl stop order-service"
    echo "Stopped $instance"
done

# 새 버전 배포
for instance in order-service-{1..10}; do
    scp order-service-2.0.jar $instance:/app/
    ssh $instance "cd /app && java -jar order-service-2.0.jar &"
    echo "Started v2.0 on $instance"
done

# 문제:
# - 모든 서버가 동시에 다운됨 (가용성 0%)
# - v2.0에 버그가 있어도 즉시 모든 사용자 영향
# - 롤백 시간도 오래 걸림
# - Blue 버전을 유지하지 않음 (복구 불가능)
```

---

## ✨ 올바른 접근 (After)

```yaml
# Blue/Green 배포 전략 (모든 서버를 한 번에 전환)
# 특징: 빠른 배포, 빠른 롤백, 높은 복구 가능성

# 배포 전:
# Blue 클러스터 (v1.0)        Green 클러스터 (준비 중)
# ├─ Pod-1 (v1.0)            ├─ Pod-1 (준비 중)
# ├─ Pod-2 (v1.0)            ├─ Pod-2 (준비 중)
# └─ Pod-3 (v1.0)            └─ Pod-3 (준비 중)
# 
# 로드밸런서 → Blue 100%

# Green 준비 (v2.0 배포, 테스트)
# Green 클러스터가 준비되고 헬스 체크 통과

# 배포 후 (전환):
# 로드밸런서 → Green 100%
# 
# Blue 클러스터 (v1.0)        Green 클러스터 (v2.0)
# ├─ Pod-1 (v1.0)            ├─ Pod-1 (v2.0)  ← 이제 서빙
# ├─ Pod-2 (v1.0)            ├─ Pod-2 (v2.0)
# └─ Pod-3 (v1.0)            └─ Pod-3 (v2.0)

# 문제 발생 시 (5분 내):
# 로드밸런서 → Blue 100% (즉시 롤백!)

# Kubernetes 구현
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
    version: blue  # blue 또는 green으로 전환
  ports:
    - port: 8080
      targetPort: 8080

# Blue 배포
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      version: blue
  template:
    metadata:
      labels:
        app: order-service
        version: blue
    spec:
      containers:
      - name: order-service
        image: order-service:1.0
        ports:
        - containerPort: 8080
        readinessProbe:  # 트래픽 수신 가능 확인
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5

# Green 배포 (새 버전)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      version: green
  template:
    metadata:
      labels:
        app: order-service
        version: green
    spec:
      containers:
      - name: order-service
        image: order-service:2.0  # 새 버전
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5

# 전환 스크립트
# kubectl patch service order-service -p '{"spec":{"selector":{"version":"green"}}}'
```

```yaml
# Canary 배포 (점진적 트래픽 이전)
# 특징: 위험도 낮음, 느린 배포, 세밀한 모니터링

# 배포 초기 (1% 트래픽)
# ┌─────────────────────────────────────────┐
# │ 전체 트래픽: 100개 요청/초              │
# └─────────────────────────────────────────┘
#   ├─ 기존 v1.0: 99개 (99%)
#   └─ 새로운 v2.0: 1개 (1%) ← 모니터링
#
# 에러율 확인: 1% 트래픽에서 에러 없음 → OK!

# 5분 후 (5% 트래픽)
# 기존 v1.0: 95개 (95%)
# 새로운 v2.0: 5개 (5%)
# 응답시간 확인: P95 50ms → 1000ms (증가!)
# 롤백 결정!

# 배포 최종 (100% 트래픽)
# 기존 v1.0: 0개 (0%)
# 새로운 v2.0: 100개 (100%)
# 배포 완료!

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - match:
    - uri:
        regex: '.*'
    route:
    - destination:
        host: order-service-v1
        port:
          number: 8080
      weight: 90  # 기존 버전 90%
    - destination:
        host: order-service-v2
        port:
          number: 8080
      weight: 10  # 새 버전 10% (Canary)
    timeout: 10s
    retries:
      attempts: 3
      perTryTimeout: 2s
```

```java
// Feature Flag를 사용한 배포 분리
// 코드는 배포했지만, 기능은 끌 수 있음

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private FeatureFlagClient featureFlagClient;
    
    @Autowired
    private OrderService orderService;
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody OrderRequest request) {
        
        // Feature Flag로 새 로직 활성화 여부 확인
        if (featureFlagClient.isEnabled("NEW_ORDER_VALIDATION", request.getCustomerId())) {
            // v2.0 로직: 새로운 검증 규칙
            return ResponseEntity.ok(orderService.processOrderV2(request));
        } else {
            // v1.0 로직: 기존 검증 규칙
            return ResponseEntity.ok(orderService.processOrderV1(request));
        }
    }
}

// Feature Flag 설정 (LaunchDarkly, Unleash 등)
// {
//   "name": "NEW_ORDER_VALIDATION",
//   "status": "on",
//   "rolloutPercentage": 10,  // 10% 사용자에게만 활성화
//   "targetUsers": [
//     "customer_123",  // 특정 고객에게 먼저 활성화
//     "customer_456"
//   ],
//   "attributes": {
//     "country": "US",  // 특정 국가에만 활성화
//     "plan": "premium"  // premium 사용자에게만
//   }
// }

// 장점:
// 1. 배포와 활성화 분리: 코드는 배포했지만 플래그 off → 사용자는 변화 감지 못 함
// 2. A/B 테스트: 동일 코드에서 10%는 v2.0, 90%는 v1.0
// 3. 즉시 롤백: 플래그 하나만 off하면 모든 사용자가 v1.0으로 복구
// 4. 점진적 활성화: 1% → 5% → 50% → 100% 단계적 확대
```

---

## 🔬 내부 동작 원리 — Kubernetes Rolling Update 상세 분석

### Rolling Update 메커니즘

```
┌────────────────────────────────────────────────────────────┐
│         Kubernetes RollingUpdate 상세 프로세스              │
├────────────────────────────────────────────────────────────┤
│                                                              │
│ 1단계: 초기 상태 (v1.0, 3개 Pod)                           │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ Pod-1 (v1.0) ✓  Ready=1/1                             │ │
│ │ Pod-2 (v1.0) ✓  Ready=1/1                             │ │
│ │ Pod-3 (v1.0) ✓  Ready=1/1                             │ │
│ │ 서비스 중인 Pod: 3개 (100%)                           │ │
│ └───────────────────────────────────────────────────────┘ │
│                                                              │
│ 2단계: 새 Pod 생성 (maxSurge=1)                            │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ Pod-1 (v1.0) ✓  Ready=1/1                             │ │
│ │ Pod-2 (v1.0) ✓  Ready=1/1                             │ │
│ │ Pod-3 (v1.0) ✓  Ready=1/1                             │ │
│ │ Pod-4 (v2.0) ⚙  Ready=0/1  ← 새 Pod 시작            │ │
│ │ 서비스 중인 Pod: 3개, 시작 중인 Pod: 1개            │ │
│ └───────────────────────────────────────────────────────┘ │
│                                                              │
│ 3단계: Readiness Probe 통과 (Pod-4 준비 완료)             │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ Pod-1 (v1.0) ✓  Ready=1/1                             │ │
│ │ Pod-2 (v1.0) ✓  Ready=1/1                             │ │
│ │ Pod-3 (v1.0) ✓  Ready=1/1                             │ │
│ │ Pod-4 (v2.0) ✓  Ready=1/1  ← 준비 완료!             │ │
│ │ 서비스 중인 Pod: 4개 (잠시 4개까지 증가)             │ │
│ └───────────────────────────────────────────────────────┘ │
│                                                              │
│ 4단계: 구 Pod 제거 (maxUnavailable=1)                     │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ Pod-1 (v1.0) ⏹  Terminating                           │ │
│ │ Pod-2 (v1.0) ✓  Ready=1/1                             │ │
│ │ Pod-3 (v1.0) ✓  Ready=1/1                             │ │
│ │ Pod-4 (v2.0) ✓  Ready=1/1                             │ │
│ │ 서비스 중인 Pod: 3개 (다시 3개로 축소)                │ │
│ └───────────────────────────────────────────────────────┘ │
│                                                              │
│ 5단계: 반복 (Pod-5 생성)                                   │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ Pod-2 (v1.0) ✓  Ready=1/1                             │ │
│ │ Pod-3 (v1.0) ✓  Ready=1/1                             │ │
│ │ Pod-4 (v2.0) ✓  Ready=1/1                             │ │
│ │ Pod-5 (v2.0) ⚙  Ready=0/1  ← 새 Pod 시작            │ │
│ └───────────────────────────────────────────────────────┘ │
│                                                              │
│ ... 반복 ...                                                │
│                                                              │
│ 최종 상태: 모든 Pod이 v2.0으로 교체됨                     │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ Pod-4 (v2.0) ✓  Ready=1/1                             │ │
│ │ Pod-5 (v2.0) ✓  Ready=1/1                             │ │
│ │ Pod-6 (v2.0) ✓  Ready=1/1                             │ │
│ │ 서비스 중인 Pod: 3개 (100%, 모두 v2.0)                │ │
│ └───────────────────────────────────────────────────────┘ │
│                                                              │
└────────────────────────────────────────────────────────────┘

핵심 파라미터:
- maxSurge: 원하는 Pod 수보다 몇 개 더 만들 수 있나?
  - 3개 원함 + maxSurge=1 → 최대 4개까지 (메모리 비용)
  
- maxUnavailable: 동시에 최대 몇 개 Pod이 없어도 되나?
  - 3개 원함 + maxUnavailable=1 → 최소 2개는 항상 서빙
  - 가용성과 속도의 트레이드오프
```

### Canary 배포 모니터링

```
메트릭 추적:

시간: 10:00 ~ 10:30

에러율:
├─ v1.0: [0.1%] [0.1%] [0.1%] [0.2%] [0.1%] ...
├─ v2.0: [0.2%] [0.2%] [5.0%] [12%] [18%] ← 급증!
│        [1%]   [5%]   [10%]  [20%] [50%] ← 롤백 필요!

응답시간 (P99):
├─ v1.0: [800ms] [850ms] [800ms] [900ms] [850ms] ...
├─ v2.0: [900ms] [950ms] [2000ms] [3000ms] ← 느려짐!
│        [1%]    [5%]    [10%]    [20%]    ← 롤백!

Canary 배포 결정:
├─ 0~5분: 1% 트래픽, 에러율 정상 → 다음 단계로
├─ 5~10분: 5% 트래픽, 에러율 정상 → 다음 단계로
├─ 10~15분: 10% 트래픽, 에러율 5% 초과 → ⚠️ 알림!
├─ 15분: 에러율 18% → 자동 롤백!
└─ 롤백 완료: 모든 트래픽을 v1.0으로 복구

총 영향: 1,000개 요청 중 180개 실패 (v2.0의 18%)
         = 약 180개 사용자만 영향 (전체 대비 0.018%)
```

### Feature Flag의 세 가지 방식

```
┌────────────────────────────────────────────────────────────┐
│        Feature Flag 제어 방식 비교                          │
├────────────────────────────────────────────────────────────┤
│                                                              │
│ 1. Kill Switch (기능 On/Off)                               │
│    ┌──────────────────────────────────────────────────┐   │
│    │ if (flags.isEnabled("new_payment")) {             │   │
│    │   processPaymentV2();  // 새 로직                 │   │
│    │ } else {                                          │   │
│    │   processPaymentV1();  // 기존 로직               │   │
│    │ }                                                 │   │
│    │                                                   │   │
│    │ 설정: "new_payment" = ON/OFF                     │   │
│    │ 적용: 모든 사용자 또는 특정 사용자                │   │
│    └──────────────────────────────────────────────────┘   │
│    장점: 배포와 활성화 분리, 빠른 롤백                     │
│    단점: 코드에 분기 로직 필요                              │
│                                                              │
│ 2. Percentage Rollout (비율 기반)                          │
│    ┌──────────────────────────────────────────────────┐   │
│    │ 설정: "new_payment" = ON, rolloutPercentage=30%  │   │
│    │                                                   │   │
│    │ 사용자 10명이 요청:                               │   │
│    │ ├─ user1: hash("user1") % 100 = 45 > 30 → v1.0  │   │
│    │ ├─ user2: hash("user2") % 100 = 12 < 30 → v2.0  │   │
│    │ ├─ user3: hash("user3") % 100 = 78 > 30 → v1.0  │   │
│    │ ├─ user4: hash("user4") % 100 = 25 < 30 → v2.0  │   │
│    │ └─ ...                                            │   │
│    │                                                   │   │
│    │ 자동으로 30%의 사용자가 v2.0을 사용              │   │
│    └──────────────────────────────────────────────────┘   │
│    장점: 결정적 (같은 사용자는 항상 같은 버전)             │
│    단점: 정확히 30%를 보장하지는 않음 (확률적)             │
│                                                              │
│ 3. Targeting (특정 사용자 지정)                            │
│    ┌──────────────────────────────────────────────────┐   │
│    │ 설정: "new_payment" = ON                          │   │
│    │   targetUsers: [                                  │   │
│    │     "admin@company.com",                          │   │
│    │     "qa@company.com",                             │   │
│    │     "beta_tester_1"                               │   │
│    │   ]                                               │   │
│    │   attributes: {                                   │   │
│    │     "country": "US",  // 미국 사용자              │   │
│    │     "plan": "premium" // 프리미엄 사용자          │   │
│    │   }                                               │   │
│    │                                                   │   │
│    │ 이 조건에 맞는 사용자만 v2.0을 사용              │   │
│    └──────────────────────────────────────────────────┘   │
│    장점: 정확한 타겟팅, QA/관리자 먼저 테스트             │
│    단점: 수동으로 사용자를 지정해야 함                    │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

---

## 💻 배포 전략 구현 코드

```yaml
# Kubernetes Rolling Update 설정
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 원하는 Pod 수보다 최대 1개 더 생성
      maxUnavailable: 1  # 동시에 최대 1개 Pod 다운 허용
  
  selector:
    matchLabels:
      app: order-service
  
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: order-service:2.0  # 새 버전으로 자동 업데이트
        ports:
        - containerPort: 8080
        
        # 트래픽 수신 가능 확인 (새 Pod이 트래픽을 받기 전 대기)
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 2
          successThreshold: 1
          failureThreshold: 3
        
        # Pod이 살아있는지 확인 (실패 시 재시작)
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        
        # graceful shutdown (기존 요청 처리 후 종료)
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]  # 15초 대기

---
# Canary 배포 (Istio VirtualService)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  # 초기: 1% 트래픽을 v2.0으로
  - match:
    - uri:
        regex: '.*'
    route:
    - destination:
        host: order-service-v1
      weight: 99
    - destination:
        host: order-service-v2
      weight: 1
    timeout: 10s
    retries:
      attempts: 3

---
# Feature Flag 설정 (Unleash)
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  flags.json: |
    {
      "features": [
        {
          "name": "NEW_PAYMENT_VALIDATION",
          "enabled": true,
          "strategies": [
            {
              "name": "flexibleRollout",
              "parameters": {
                "rollout": "30",  # 30% 사용자에게 활성화
                "stickiness": "userId"  # 같은 사용자는 항상 같은 버전
              }
            }
          ]
        },
        {
          "name": "PREMIUM_ONLY_FEATURE",
          "enabled": true,
          "strategies": [
            {
              "name": "userWithId",
              "parameters": {
                "userIds": "admin@company.com,qa@company.com"
              }
            }
          ]
        }
      ]
    }
```

```java
// Feature Flag 클라이언트 구현
@Component
public class UnleashFeatureFlagClient {
    
    @Autowired
    private Unleash unleash;
    
    public boolean isEnabled(String flagName, String userId) {
        // Unleash 서버에서 플래그 상태 확인
        // 동시에 사용자 ID, 세션, 커스텀 컨텍스트 전달
        UnleashContext context = UnleashContext.builder()
                .userId(userId)
                .sessionId(UUID.randomUUID().toString())
                .addProperty("plan", getPlanType(userId))
                .addProperty("country", getUserCountry(userId))
                .build();
        
        return unleash.isEnabled(flagName, context);
    }
}

// Canary 배포 모니터링
@Component
public class CanaryMonitor {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    public void recordCanaryMetrics(String version, boolean success, long durationMs) {
        String tag = "version:" + version;
        
        if (success) {
            meterRegistry.counter("canary.request.success", tag).increment();
        } else {
            meterRegistry.counter("canary.request.failure", tag).increment();
        }
        
        meterRegistry.timer("canary.request.duration", tag)
                .record(Duration.ofMillis(durationMs));
    }
}

// Canary 배포 자동 롤백 로직
@Service
public class CanaryRollbackService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @Autowired
    private KubernetesClient k8sClient;
    
    // 주기적으로 에러율 확인
    @Scheduled(fixedRate = 10000)  // 10초마다
    public void checkCanaryHealth() {
        double errorRate = calculateCanaryErrorRate();
        double p99Duration = calculateCanaryP99Duration();
        
        // 임계값 초과 시 자동 롤백
        if (errorRate > 0.05) {  // 5% 이상
            logger.error("High error rate detected: {}%", errorRate * 100);
            rollbackCanary();
        }
        
        if (p99Duration > 2000) {  // P99 > 2초
            logger.error("Slow response time detected: {}ms", p99Duration);
            rollbackCanary();
        }
    }
    
    private void rollbackCanary() {
        // VirtualService를 이전 배포로 복구
        // Canary 버전의 트래픽을 0%로 설정
        k8sClient.updateVirtualService("order-service", 
                "v1-weight", 100,
                "v2-weight", 0);
        
        logger.info("Canary rolled back successfully");
    }
    
    private double calculateCanaryErrorRate() {
        // Prometheus에서 v2.0 버전의 에러율 계산
        // rate(canary_request_failure[5m]) / rate(canary_request_success[5m] + canary_request_failure[5m])
        return 0;  // 실제 구현: Prometheus 쿼리
    }
    
    private double calculateCanaryP99Duration() {
        // Prometheus에서 v2.0 버전의 P99 응답시간 조회
        return 0;  // 실제 구현: Prometheus 쿼리
    }
}
```

---

## 📊 패턴 비교

| 항목 | Blue/Green | Canary | Rolling Update | Feature Flag |
|------|-----------|--------|----------|------------|
| **배포 시간** | 수초 | 수십분 | 수분 | 즉시 |
| **롤백 시간** | 수초 | 수초 | 수분 | 수초 |
| **인프라 비용** | 2배 | 정상 | 정상 | 정상 |
| **위험도** | 중간 | 낮음 | 중간 | 매우 낮음 |
| **테스트 커버리지** | 필수 | 권장 | 권장 | 필수 |
| **고객 영향 (문제 발생)** | 전체 | 1-10% | 일부 | 0% (플래그 off) |
| **A/B 테스트** | 불가능 | 어려움 | 불가능 | 가능 |

---

## ⚖️ 트레이드오프

### 장점 ✅
- **배포 안전성**: Blue/Green은 즉시 롤백, Canary는 피해 최소화
- **무중단 배포**: 기존 버전을 유지하면서 새 버전 준비
- **모니터링**: Canary는 자동으로 메트릭 기반 롤백 가능
- **유연성**: Feature Flag로 배포와 활성화 분리
- **A/B 테스트**: Feature Flag로 기능 영향도 측정

### 단점 ❌
- **복잡성**: 여러 버전 동시 운영으로 관리 어려움
- **인프라 비용**: Blue/Green은 2배 리소스 필요
- **긴 배포 시간**: Canary는 수십분 소요
- **코드 복잡도**: Feature Flag는 분기 로직 증가
- **테스트 부담**: 여러 버전 동시 테스트 필요

### 배포 전략 선택 기준

```
┌─────────────────────────────────────────────────────────┐
│           언제 어떤 전략을 사용할 것인가?                │
├─────────────────────────────────────────────────────────┤
│                                                           │
│ 1. Blue/Green 배포                                      │
│    ├─ 사용 시기: 광범위한 테스트가 완료된 경우         │
│    ├─ 예시: 월간 메이저 배포                          │
│    ├─ 리스크: 높음 (모든 사용자 동시 영향)             │
│    └─ 권장: 금융, 의료 (즉시 롤백이 중요한 경우)       │
│                                                           │
│ 2. Canary 배포                                          │
│    ├─ 사용 시기: 새 기능이지만 안정성 미확실한 경우     │
│    ├─ 예시: 알고리즘 변경, 성능 최적화                 │
│    ├─ 리스크: 낮음 (1-10% 사용자만 영향)               │
│    └─ 권장: 대부분의 경우 최고의 선택                  │
│                                                           │
│ 3. Rolling Update                                       │
│    ├─ 사용 시기: Kubernetes에서 기본 배포 방식         │
│    ├─ 예시: 버그 픽스, 패치 배포                       │
│    ├─ 리스크: 중간 (일부 Pod이 다운되지만 자동 재시작) │
│    └─ 권장: 마이크로서비스 일상 배포                   │
│                                                           │
│ 4. Feature Flag                                         │
│    ├─ 사용 시기: 배포와 활성화를 완전히 분리하고 싶을 때
│    ├─ 예시: A/B 테스트, gradual rollout                │
│    ├─ 리스크: 매우 낮음 (플래그 off → 즉시 복구)       │
│    └─ 권장: 신규 기능, 실험적 기능                      │
│                                                           │
│ ★ 조합 전략:                                             │
│   Canary 배포 + Feature Flag                            │
│   ├─ 10% 트래픽을 새 버전으로 라우팅 (Canary)          │
│   ├─ 그 중 30%만 새 기능 활성화 (Feature Flag)          │
│   ├─ = 전체 사용자의 3%만 새 기능 사용                 │
│   └─ 매우 낮은 위험도로 배포 가능                      │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

---

## 📌 핵심 정리

✅ **Blue/Green 배포**
- 모든 사용자를 한 번에 전환 (수초 내)
- 즉시 롤백 가능 (5초 내)
- 인프라 2배 비용, 위험도 높음

✅ **Canary 배포**
- 1% → 5% → 50% → 100%로 점진적 확대
- 각 단계에서 메트릭 모니터링
- 문제 발생 시 자동 롤백
- 가장 안전한 방식 (권장)

✅ **Rolling Update**
- Kubernetes 기본 배포 전략
- 구 Pod 제거 → 신 Pod 생성 반복
- maxSurge, maxUnavailable으로 속도/비용 조절

✅ **Feature Flag**
- 배포와 기능 활성화 완전 분리
- 플래그 off → 즉시 모든 사용자 복구
- A/B 테스트 가능

✅ **모니터링이 모든 배포의 핵심**
- Rate, Errors, Duration 메트릭 필수
- P95 응답시간 > 2배 → 자동 롤백
- 에러율 > 임계값 → 자동 알림

---

## 🤔 생각해볼 문제

**Q1.** Canary 배포에서 1% → 5% → 20% → 100%의 비율로 확대한다면, 각 단계에서 최소 몇 분씩 기다려야 할까? (통계적으로 유의미한 데이터 수집 필요)

<details><summary>해설 보기</summary>
초당 100개 요청 기준:

1단계 (1% = 1 req/sec):
- 에러율을 정확히 측정하려면 최소 100개 요청 필요
- 100 req / 1 req/sec = 100초 ≈ 2분

5단계 (5% = 5 req/sec):
- 5개 req/sec × 300초 = 1,500개 요청
- 통계적으로 유의미 ≈ 5분

20단계 (20% = 20 req/sec):
- 에러 1개가 발생할 때까지 기다리기
- 에러율 0.1% 기준: 1,000개 요청 ≈ 50초
- 안전하게 보기: 10분

총 배포 시간: 2 + 5 + 10 + 5(100%) = 22분

아니면 자동 상승: 기설정된 메트릭이 정상이면 자동으로 다음 단계로 (각 5분)
- 총 20분으로 단축 가능
</details>

**Q2.** Blue/Green 배포에서 Blue를 5분 후 삭제하는 것이 위험한 이유를 설명하시오.

<details><summary>해설 보기</summary>
**문제점:**

1. **장시간 연결이 있는 WebSocket**
```
- 사용자가 Blue 버전에 접속한 WebSocket 연결 (실시간 알림)
- Green으로 즉시 전환되면 WebSocket 연결이 끊김
- 사용자가 "연결 끊김" 경험
```

2. **장기 배치 작업**
```
- Blue에서 시작된 주문 처리 (30분 소요)
- Blue가 5분 후 삭제되면 처리 중단
- 데이터 일관성 문제 발생 가능
```

3. **캐시 및 상태 동기화**
```
- Blue의 In-Memory 캐시
- Green에는 동일한 캐시가 없음
- 성능 저하 가능
```

**해결책:**
1. Blue를 최소 30분 이상 유지
2. "연결 드레이닝" 활성화: 기존 연결은 Blue에서 처리, 신규 연결은 Green으로
3. 상태 동기화: Redis 같은 외부 캐시 사용
</details>

**Q3.** Feature Flag로 점진적 롤아웃 중에 데이터베이스 스키마 변경이 필요하다면 어떻게 처리할까?

<details><summary>해설 보기</summary>
**문제:**
- 새 기능 (v2.0)이 새로운 DB 컬럼을 필요로 함
- 플래그 on인 사용자: 새 컬럼 읽음
- 플래그 off인 사용자: 기존 로직 사용
- 30% 롤아웃 중: 일부 요청은 새 컬럼, 일부는 기존 컬럼 사용

**해결 방법:**

1. **Two-Phase Migration (권장)**
```
Phase 1: DB 스키마 추가 (호환성 유지)
├─ 새 컬럼 추가 (NOT NULL 아님, DEFAULT 값 설정)
├─ 기존 코드는 여전히 구 컬럼 사용
├─ Feature Flag off (아직 신기능 비활성)

Phase 2: 데이터 마이그레이션
├─ 백그라운드 작업: 구 데이터 → 새 컬럼 복사
├─ 검증: 새 컬럼 데이터 정합성 확인

Phase 3: Feature Flag 점진적 활성화
├─ 1% → 5% → 50% → 100%
├─ 이때 신기능은 새 컬럼, 기존 기능은 구 컬럼 사용 (하이브리드)

Phase 4: 구 컬럼 삭제
├─ Feature Flag 100% 확인 후 1주일 대기
├─ 구 컬럼 삭제 (롤백 불가능)
```

2. **Feature Flag 그룹 관리**
```java
// Phase 3 구현 예시
@Service
public class OrderService {
    
    public Order getOrder(String orderId) {
        if (featureFlagClient.isEnabled("NEW_ORDER_SCHEMA", userId)) {
            // 새 스키마: 새 컬럼 사용
            return orderRepository.findOrderUsingNewSchema(orderId);
        } else {
            // 구 스키마: 구 컬럼 사용
            return orderRepository.findOrderUsingOldSchema(orderId);
        }
    }
}
```

총 소요 시간: 2~3주 (안전성 최우선)
</details>

---

<div align="center">

**[⬅️ 이전: 메트릭 모니터링 — RED 방법론과 Grafana 대시보드](./03-metrics-monitoring.md)** | **[홈으로 🏠](../README.md)** | **[다음: 운영 중 발생하는 장애 패턴 — 순환 의존성과 카스케이드 장애 ➡️](./05-operational-failure-patterns.md)**

</div>

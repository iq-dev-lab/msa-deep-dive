# 06. 헬스체크와 자가 치유 — Actuator와 Kubernetes Probe

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Spring Boot Actuator의 /actuator/health는 실제로 무엇을 확인하는가?
- Kubernetes의 Liveness Probe와 Readiness Probe의 차이는 무엇인가?
- 왜 Liveness와 Readiness를 같은 엔드포인트로 설정하면 안 되는가?
- 서비스가 자동으로 복구되는 메커니즘은 어떻게 되는가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

마이크로서비스 환경에서는 수십 개의 서비스가 컨테이너로 실행됩니다. 문제는 각 서비스의 정상 여부를 어떻게 알 수 있냐는 것입니다. 서비스가 응답하지 않거나 내부적으로 데드락 상태에 있을 수 있기 때문입니다.

Spring Boot Actuator와 Kubernetes Probe는 이런 상황을 자동으로 감지하여, 문제 있는 서비스를 재시작하거나 트래픽에서 제외시킵니다. 이것이 "자가 치유(Self-Healing)"입니다. 관리자가 수동으로 개입하지 않아도 시스템이 자동으로 복구됩니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ 헬스체크 없이 서비스 배포
@SpringBootApplication
public class OrderServiceApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

// application.yml
spring:
  application:
    name: order-service

# management 설정 없음 (기본값: Actuator 비활성화)

---

// Kubernetes 배포 설정 (문제!)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  # ❌ 문제: livenessProbe, readinessProbe 없음
  # ❌ 결과: 
  #    1. 서비스가 죽어도 Kubernetes가 모름
  #    2. 데드락 상태도 감지 불가
  #    3. 수동으로 재시작해야 함
  
  template:
    spec:
      containers:
      - name: order-service
        image: order-service:1.0.0

# 장애 시나리오:
# 13:00:00 Order Service Pod 1에 메모리 누수 발생
# 13:00:30 Pod 1 메모리 사용량 90%
# 13:01:00 Pod 1 메모리 사용량 100% → OOM Kill
# 13:01:05 Pod 1 프로세스 중단, 서비스 응답 불가
# 13:01:06 트래픽이 여전히 Pod 1로 전달됨 (Kubernetes가 모르고 있음)
# 13:05:00 관리자가 "뭔가 느려졌다"며 문제 인지
# 13:10:00 관리자가 kubectl delete pod order-service-xxx로 수동 재시작
# 
# 결과: 약 10분의 서비스 장애
```

---

## ✨ 올바른 접근 (After)

```java
// ✅ 헬스체크 구현 및 Kubernetes 프로브 설정
@SpringBootApplication
public class OrderServiceApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

// application.yml
spring:
  application:
    name: order-service
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQL10Dialect

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics  # Actuator 활성화
      base-path: /actuator
  
  health:
    db:
      enabled: true
    diskspace:
      enabled: true
    livenessState:
      enabled: true
    readinessState:
      enabled: true

---

// Kubernetes 배포 설정 (개선!)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
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
        image: order-service:1.0.0
        ports:
        - containerPort: 8080
        
        # ✅ Readiness Probe: "준비됐나?" 확인
        # 실패 → 트래픽 차단 (Pod는 살아있지만 요청 받지 않음)
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10    # 시작 후 10초 후 첫 확인
          periodSeconds: 5           # 5초마다 확인
          timeoutSeconds: 2          # 2초 내 응답 없으면 실패
          failureThreshold: 3        # 3회 연속 실패 시 UNREADY
        
        # ✅ Liveness Probe: "살아있나?" 확인
        # 실패 → 컨테이너 재시작 (Pod 자체를 재시작)
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30    # 시작 후 30초 후 첫 확인 (앱 구동 시간)
          periodSeconds: 10          # 10초마다 확인
          timeoutSeconds: 2          # 2초 내 응답 없으면 실패
          failureThreshold: 3        # 3회 연속 실패 시 KILL & RESTART
        
        # Startup Probe: 앱 시작 시간이 긴 경우 (Spring Boot 부팅 20초 등)
        startupProbe:
          httpGet:
            path: /actuator/health/startup
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 5
          failureThreshold: 60       # 최대 300초(60회 × 5초) 대기
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"

# 개선된 장애 시나리오:
# 13:00:00 Pod 1에 메모리 누수 발생
# 13:00:30 Pod 1 메모리 사용량 90%
# 13:01:00 Pod 1 메모리 사용량 100% → 헬스체크 실패 시작
# 13:01:05 readinessProbe 실패 → Kubernetes가 "Pod 준비 안 됨" 인지
#          → 트래픽이 Pod 2, Pod 3로만 전달됨 (자동!)
# 13:01:15 livenessProbe 실패 (3회) → 컨테이너 자동 재시작
# 13:01:30 Pod 1 재시작 완료 → readiness/liveness 다시 PASS
#          → 트래픽 다시 받기 시작
#
# 결과: 약 30초의 무중단 자동 복구! (사용자 영향 최소화)
```

---

## 🔬 내부 동작 원리 — Actuator와 Kubernetes Probe

### 1. Spring Boot Actuator Health Endpoints

```
/actuator/health (상위 레벨)
    │
    ├─ /actuator/health/liveness (Kubernetes Liveness Probe용)
    │  ├─ 앱이 "살아있나?"
    │  ├─ 실패 예: 데드락, 무한 루프, JVM 크래시 예정
    │  └─ 반응: 컨테이너 재시작
    │
    ├─ /actuator/health/readiness (Kubernetes Readiness Probe용)
    │  ├─ 앱이 "준비됐나?"
    │  ├─ 실패 예: DB 미연결, 캐시 초기화 중
    │  └─ 반응: 트래픽 차단
    │
    └─ /actuator/health/startup (앱 초기화 완료?)
       ├─ 시작 구간 동안만 사용
       └─ 완료 후 무시됨

Spring Boot 자동 헬스 체크:
┌─────────────────────────────────────────┐
│ Database                                 │
│ └─ JDBC 드라이버 로드 & 쿼리 실행    │
│    SELECT 1 (기본 쿼리)                 │
│                                         │
│ Redis                                   │
│ └─ Redis 서버에 PING 명령 전송      │
│                                         │
│ Disk Space                              │
│ └─ 디스크 여유 공간 확인            │
│    (기본: 10MB 미만이면 DOWN)         │
│                                         │
│ RabbitMQ / Kafka                        │
│ └─ 브로커 연결 확인                   │
│                                         │
│ ElasticSearch                           │
│ └─ 클러스터 상태 확인                 │
└─────────────────────────────────────────┘

응답 예시:

정상 상황:
GET /actuator/health/readiness
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "redis": {
      "status": "UP"
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 209715200,
        "free": 102400000,
        "threshold": 10485760,
        "exists": true
      }
    }
  }
}

DB 다운 상황:
GET /actuator/health/readiness
{
  "status": "DOWN",
  "components": {
    "db": {
      "status": "DOWN",
      "details": {
        "error": "org.springframework.jdbc.CannotGetJdbcConnectionException: 
                  Failed to obtain JDBC Connection; nested exception is 
                  java.sql.SQLException: Connection refused"
      }
    }
  }
}
```

### 2. Liveness Probe vs Readiness Probe 상세 비교

```
┌─────────────────────────────────────────────────────────────┐
│                      정상 상황                               │
│                                                             │
│  Liveness:  ✅ UP     (앱이 살아있음)                        │
│  Readiness: ✅ UP     (요청 처리 준비됨)                     │
│  결과:      트래픽 수신 O                                  │
│             [●●●] ← 정상 요청 처리                         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│             시나리오 1: 앱 초기화 중 (시작 직후)              │
│                                                             │
│  Liveness:  ✅ UP     (JVM은 돌아가고 있음)                 │
│  Readiness: ❌ DOWN   (DB 초기화, 캐시 로딩 중)              │
│  결과:      트래픽 수신 X (초기화 완료 대기)                │
│             [⏳⏳⏳] ← 초기화 중, 요청 받지 않음              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│       시나리오 2: DB 연결 끊김 (일시적, 자가 복구 예상)      │
│                                                             │
│  Liveness:  ✅ UP     (앱 프로세스는 정상 실행)              │
│  Readiness: ❌ DOWN   (DB 연결 불가)                        │
│  결과:      트래픽 수신 X (재연결 대기, 자가 복구)          │
│             [⏳⏳⏳] ← 재연결 시도 중                        │
│                                                             │
│  → 5분 후 DB 복구 시 자동으로 Readiness UP                 │
│    (Pod 재시작 필요 없음!)                                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│           시나리오 3: 데드락 / 무한 루프                     │
│                                                             │
│  Liveness:  ❌ DOWN   (응답 없음, 타임아웃)                 │
│  Readiness: N/A       (Liveness 실패로 확인 중단)           │
│  결과:      Pod 재시작 (강제 KILL & 재시작)                 │
│             [💥] ← 재시작                                   │
│             [🔄] ← 새 Pod 시작                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│     시나리오 4: 메모리 누수 / 리소스 고갈 단계                │
│                                                             │
│  단계 1:     Readiness ✅, Liveness ✅ (정상)               │
│  단계 2 (10초): 메모리 80% 사용                              │
│  단계 3 (20초): 메모리 95% 사용 → Readiness ❌              │
│             → 트래픽 차단 (Pod 살려놓음)                   │
│  단계 4 (30초): 메모리 100% → Liveness ❌                   │
│             → Pod 재시작                                   │
│                                                             │
│  결과: Readiness가 문제를 먼저 감지 → 트래픽 차단           │
│        Liveness가 완전 실패 감지 → 재시작                   │
│        (두 단계의 자동 복구!)                               │
└─────────────────────────────────────────────────────────────┘
```

### 3. 같은 엔드포인트를 사용할 때의 문제

```
❌ 잘못된 구성:
readinessProbe:
  httpGet:
    path: /actuator/health  ← 일반 health endpoint
livenessProbe:
  httpGet:
    path: /actuator/health  ← 같은 endpoint

발생하는 문제:

상황: DB 연결 끊김 (일시적, 자가 복구 예상)
  /actuator/health 응답: DOWN (전체 상태)
  
  → Readiness 실패 (맞음) → 트래픽 차단
  → Liveness 실패 (틀림!) → Pod 재시작
  
결과: DB가 자가 복구될 기회를 주지 않음!
     불필요한 Pod 재시작 → 트래픽 끊김
     재시작 → 또 DB 재연결 → 반복

vs

✅ 올바른 구성:
readinessProbe:
  httpGet:
    path: /actuator/health/readiness  ← Readiness 전용
livenessProbe:
  httpGet:
    path: /actuator/health/liveness   ← Liveness 전용

의도:
  Readiness: "요청 처리 준비됐나?" → DB 상태 포함
  Liveness: "앱이 살아있나?" → JVM 상태만

DB 연결 끊김 상황:
  /actuator/health/readiness: DOWN (DB 불가)
  /actuator/health/liveness: UP (JVM 정상)
  
  → Readiness 실패 → 트래픽 차단 ✅
  → Liveness 성공 → Pod 유지 ✅
  
결과: DB가 자가 복구될 때까지 기다림 (재시작 안 함)
```

---

## 💻 커스텀 HealthIndicator 구현

### 1. 기본 커스텀 헬스 체크

```java
package com.msa.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.client.RestTemplate;

// 외부 서비스 상태 확인 (Readiness용)
@Component
public class ExternalPaymentServiceHealthIndicator implements HealthIndicator {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Override
    public Health health() {
        try {
            // 결제 서비스 헬스 엔드포인트 호출
            String response = restTemplate.getForObject(
                "http://payment-service:8080/actuator/health",
                String.class
            );
            
            if (response != null && response.contains("UP")) {
                return Health.up()
                    .withDetail("service", "payment-service")
                    .withDetail("endpoint", "http://payment-service:8080/actuator/health")
                    .build();
            } else {
                return Health.down()
                    .withDetail("service", "payment-service")
                    .withDetail("reason", "Service responded but status is not UP")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("service", "payment-service")
                .withException(e)
                .build();
        }
    }
}

// 비즈니스 로직 상태 확인 (Liveness용)
@Component
public class BusinessLogicHealthIndicator implements HealthIndicator {
    
    @Autowired
    private OrderRepository orderRepository;
    
    private volatile long lastSuccessfulCheck = System.currentTimeMillis();
    private static final long DEADLOCK_TIMEOUT_MS = 30000;  // 30초
    
    @Override
    public Health health() {
        try {
            // 간단한 DB 쿼리로 데드락 감지
            long count = orderRepository.count();
            lastSuccessfulCheck = System.currentTimeMillis();
            
            return Health.up()
                .withDetail("totalOrders", count)
                .build();
        } catch (Exception e) {
            long timeSinceLastSuccess = System.currentTimeMillis() - lastSuccessfulCheck;
            
            if (timeSinceLastSuccess > DEADLOCK_TIMEOUT_MS) {
                // 30초 이상 DB 접근 불가 → 데드락 의심
                return Health.down()
                    .withDetail("reason", "Possible deadlock - no DB access for " + 
                               (timeSinceLastSuccess / 1000) + "s")
                    .withException(e)
                    .build();
            } else {
                // 일시적 오류 (DB 재연결 중 등)
                return Health.degraded()
                    .withDetail("reason", "Temporary database issue")
                    .withDetail("secondsSinceLastSuccess", timeSinceLastSuccess / 1000)
                    .withException(e)
                    .build();
            }
        }
    }
}

// 캐시 서비스 상태 확인
@Component
public class CacheHealthIndicator implements HealthIndicator {
    
    @Autowired
    private RedisTemplate<String, String> redis;
    
    @Override
    public Health health() {
        try {
            // Redis PING 명령
            Boolean result = redis.getConnectionFactory()
                .getConnection()
                .ping();
            
            if (Boolean.TRUE.equals(result)) {
                return Health.up()
                    .withDetail("type", "Redis")
                    .build();
            } else {
                return Health.down()
                    .withDetail("type", "Redis")
                    .withDetail("reason", "PING failed")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("type", "Redis")
                .withException(e)
                .build();
        }
    }
}
```

### 2. Liveness와 Readiness 분리 설정

```java
package com.msa.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthComponent;
import org.springframework.boot.actuate.health.HealthContributor;
import org.springframework.boot.actuate.health.HealthContributorRegistry;
import org.springframework.context.annotation.Configuration;
import org.springframework.boot.actuate.autoconfigure.health.HealthIndicatorProperties;

@Configuration
public class HealthConfiguration {
    
    // application.yml에서 management.health 설정
    /*
    management:
      health:
        # Liveness: 핵심 요소만
        livenessState:
          enabled: true
          # 이 그룹에는 중요한 것들만 (JVM, 데드락 감지 등)
        
        # Readiness: 모든 요소 포함
        readinessState:
          enabled: true
          # 이 그룹에는 모든 것 포함 (DB, 캐시, 외부 서비스 등)
        
        # 그룹 정의
        livenessState:
          include: jvm, businessLogic
        
        readinessState:
          include: db, redis, cache, externalPaymentService, businessLogic
    */
}

// 선택적 Health Indicator (조건부 활성화)
@Component
@ConditionalOnProperty(
    name = "app.health.external-services.enabled",
    havingValue = "true"
)
public class ConditionalExternalServiceHealth implements HealthIndicator {
    
    @Override
    public Health health() {
        // 설정에 따라 외부 서비스 확인 (개발 환경에선 스킵 가능)
        return Health.up().build();
    }
}
```

### 3. Kubernetes 배포 설정 상세

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
    version: v1

spec:
  replicas: 3
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 최대 1개 추가 Pod 허용
      maxUnavailable: 0  # 0개 Pod만 동시에 다운 (무중단 배포)
  
  selector:
    matchLabels:
      app: order-service
  
  template:
    metadata:
      labels:
        app: order-service
        version: v1
    
    spec:
      # Pod 시작 시간이 오래 걸리면 startupProbe 추가
      # Startup Probe: "앱이 완전히 시작됐나?" (한 번만 확인)
      startupProbe:
        httpGet:
          path: /actuator/health/startup
          port: http
        initialDelaySeconds: 0
        periodSeconds: 2         # 자주 확인 (시작만 빠르게)
        timeoutSeconds: 1
        failureThreshold: 150    # 최대 300초(150 * 2초) 대기
      
      containers:
      - name: order-service
        image: order-service:1.2.0
        imagePullPolicy: IfNotPresent
        
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        
        env:
        - name: JAVA_OPTS
          value: "-Xmx256m -Xms128m"
        - name: LOG_LEVEL
          value: "INFO"
        
        # Readiness Probe
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: http
            httpHeaders:
            - name: User-Agent
              value: Kubernetes-Readiness
          
          # 포드가 트래픽을 받기까지의 대기
          initialDelaySeconds: 15    # 앱 부팅 후 15초부터 확인 시작
          periodSeconds: 5           # 5초마다 확인
          timeoutSeconds: 3          # 3초 이내 응답 필요
          successThreshold: 1        # 1회 성공하면 READY
          failureThreshold: 3        # 3회 연속 실패하면 NOT READY
        
        # Liveness Probe
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: http
            httpHeaders:
            - name: User-Agent
              value: Kubernetes-Liveness
          
          # Pod 재시작 판단 기준
          initialDelaySeconds: 60    # 앱 완전 구동 후 60초부터 확인
          periodSeconds: 10          # 10초마다 확인
          timeoutSeconds: 5          # 5초 이내 응답 필요
          successThreshold: 1
          failureThreshold: 3        # 3회 연속 실패하면 RESTART
        
        resources:
          requests:                 # Pod 스케줄링 시 필요 리소스
            memory: "256Mi"
            cpu: "100m"
          limits:                   # Pod이 사용할 수 있는 최대 리소스
            memory: "512Mi"
            cpu: "500m"
        
        # 올바른 종료 (Graceful Shutdown)
        lifecycle:
          preStop:
            exec:
              # Pod 삭제 신호 받을 때 15초 대기 (진행 중인 요청 완료 시간)
              command: ["/bin/sh", "-c", "sleep 15"]
        
        terminationGracePeriodSeconds: 30  # 최대 30초 대기 후 강제 종료
      
      # Pod가 준비되지 않았으면 트래픽 보내지 않기
      terminationGracePeriodSeconds: 30
      restartPolicy: Always  # 자동 재시작
      
      # DNS 설정
      dnsPolicy: ClusterFirst

---

# Service 설정
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production

spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: http
    protocol: TCP
    name: http
  
  selector:
    app: order-service

---

# Horizontal Pod Autoscaler (자동 스케일링)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: production

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  
  minReplicas: 3        # 최소 3개 Pod
  maxReplicas: 10       # 최대 10개 Pod
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # CPU 70% 초과 시 스케일 업
  
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80    # 메모리 80% 초과 시 스케일 업
```

### 4. 모니터링 및 알림

```java
package com.msa.monitoring;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Counter;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthEndpoint;
import org.springframework.stereotype.Service;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class HealthMonitoringService {
    
    private static final Logger logger = LoggerFactory.getLogger(HealthMonitoringService.class);
    private final MeterRegistry meterRegistry;
    private final HealthEndpoint healthEndpoint;
    
    private Counter readinessFailures;
    private Counter livenessFailures;
    
    public HealthMonitoringService(
        MeterRegistry meterRegistry,
        HealthEndpoint healthEndpoint
    ) {
        this.meterRegistry = meterRegistry;
        this.healthEndpoint = healthEndpoint;
        
        initializeMetrics();
    }
    
    private void initializeMetrics() {
        readinessFailures = Counter.builder("health.readiness.failures")
            .description("Readiness probe 실패 횟수")
            .register(meterRegistry);
        
        livenessFailures = Counter.builder("health.liveness.failures")
            .description("Liveness probe 실패 횟수")
            .register(meterRegistry);
    }
    
    // 주기적으로 헬스 상태를 모니터링하고 알림
    @Scheduled(fixedDelay = 60000)  // 1분마다
    public void monitorHealth() {
        Health readinessHealth = healthEndpoint.health()
            .getComponentDetail("readiness");
        Health livenessHealth = healthEndpoint.health()
            .getComponentDetail("liveness");
        
        if (!"UP".equals(readinessHealth.getStatus().toString())) {
            readinessFailures.increment();
            logger.error("Readiness 실패: {}", readinessHealth.getDetails());
            // 알림 전송 (Slack, PagerDuty 등)
            sendAlert("Readiness Probe Failed", readinessHealth.getDetails().toString());
        }
        
        if (!"UP".equals(livenessHealth.getStatus().toString())) {
            livenessFailures.increment();
            logger.error("Liveness 실패: {}", livenessHealth.getDetails());
            sendAlert("Liveness Probe Failed", livenessHealth.getDetails().toString());
        }
    }
    
    private void sendAlert(String title, String details) {
        // Slack Webhook 호출 또는 다른 알림 시스템 사용
        logger.warn("ALERT: {} - {}", title, details);
    }
}
```

---

## 📊 패턴 비교

| 항목 | Liveness | Readiness | Startup |
|------|---|---|---|
| **확인 대상** | JVM, 데드락 | DB, 캐시, 준비 상태 | 앱 초기화 완료 |
| **실패 시 동작** | Pod 재시작 | 트래픽 차단 | 재시작 대기 |
| **초기 지연** | 30~60초 | 10~15초 | 0초 |
| **확인 주기** | 10~30초 | 5초 | 2~5초 |
| **최대 대기** | 없음 | 없음 | 300초 |

---

## ⚖️ 트레이드오프

### 장점
- **자동 복구 (Self-Healing)**: 관리자 개입 없이 자동 재시작
- **무중단 배포**: Readiness로 준비되지 않은 Pod에 트래픽 전달 안 함
- **빠른 장애 감지**: Probe로 문제를 즉시 감지
- **리소스 절약**: 불필요한 Pod을 자동으로 제거하고 재시작

### 단점
- **헬스 체크 오버헤드**: 자주 확인하면 서버 부하 증가
- **거짓 양성**: 일시적 네트워크 지연도 실패로 판정 가능
- **설정 복잡도**: 각 Probe마다 타이밍 최적화 필요
- **디버깅 어려움**: 자동 재시작으로 인해 문제 추적 어려움

---

## 📌 핵심 정리

✅ **Liveness는 "앱이 살아있나?", Readiness는 "요청 처리 준비됐나?"**

✅ **Liveness 실패 → Pod 재시작, Readiness 실패 → 트래픽 차단 (재시작 아님)**

✅ **다른 엔드포인트 사용 필수: /health/liveness vs /health/readiness**

✅ **DB 재연결 같은 자가 복구 상황에서는 Liveness는 UP, Readiness는 DOWN으로 설정**

✅ **Startup Probe는 앱 초기화 시간이 긴 경우(Spring Boot 부팅 등)에 추가**

---

## 🤔 생각해볼 문제

**Q1.** Liveness와 Readiness를 모두 같은 endpoint(/actuator/health)로 설정하면 어떤 문제가 생길까요?

<details><summary>해설 보기</summary>
DB 연결이 끊긴 경우를 예로 들면:
1. /actuator/health는 DOWN을 반환 (전체 상태)
2. Readiness 실패 → 트래픽 차단 (맞음)
3. Liveness 실패 → Pod 재시작 (틀림!)

문제: DB가 자가 복구될 기회를 주지 않음
결과: 불필요한 Pod 재시작 → 다시 DB 재연결 시도 → 반복

올바른 방법:
- /health/readiness: DB, 캐시, 외부 서비스 포함 (모든 준비 사항)
- /health/liveness: JVM, 데드락 감지만 포함 (프로세스 자체 상태)

DB 연결 끊김:
- Readiness: DOWN (트래픽 차단) ✅
- Liveness: UP (재시작 안 함) ✅
→ DB 복구 대기, 자가 복구 가능
</details>

**Q2.** readinessProbe의 failureThreshold를 1로 설정하면 어떻게 될까요? (현재는 3)

<details><summary>해설 보기</summary>
readinessProbe:
  failureThreshold: 1  # 1회 실패 즉시 NOT READY

문제점:
1. 일시적 네트워크 지연도 트래픽 차단
2. DB 연결 풀 재연결 중에도 차단
3. 짧은 시간에 여러 번 READY/NOT READY 전환 반복
4. "flapping"이라 불리는 상태 불안정 발생

결과: 과도한 트래픽 중단, 사용자 경험 악화

권장값:
- failureThreshold: 3 (3회 연속 실패 후 차단)
- periodSeconds: 5 (5초마다 확인)
= 최대 15초 후 차단 (합리적)
</details>

**Q3.** Startup Probe의 failureThreshold를 60으로 설정했는데, 앱이 300초 이상 걸립니다. 이 경우?

<details><summary>해설 보기</summary>
현재 설정:
- periodSeconds: 5
- failureThreshold: 60
- 최대 대기 시간: 5초 × 60회 = 300초

앱이 350초 걸리면:
- 300초 경과: Startup Probe 실패 판정
- Pod 재시작
- 무한 반복

해결 방법:
```yaml
startupProbe:
  httpGet:
    path: /actuator/health/startup
    port: http
  periodSeconds: 5
  failureThreshold: 90  # 5초 × 90 = 450초 대기
```

또는 앱 시작 시간 최적화:
1. 불필요한 Bean 초기화 지연 (Lazy Initialization)
2. DB 마이그레이션 비동기 처리
3. 캐시 사전 로드 비동기 처리
```java
@Bean
@Lazy
public ExpensiveComponent expensiveComponent() {
    // 필요할 때만 초기화
    return new ExpensiveComponent();
}
```
</details>

---

<div align="center">

**[⬅️ 이전: Fallback 전략 — 기능 축소 운영 설계](./05-fallback-strategies.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — API Gateway 완전 분해 ➡️](../api-gateway/01-api-gateway-deep-dive.md)**

</div>

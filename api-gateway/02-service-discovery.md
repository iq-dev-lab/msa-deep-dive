# 02. 서비스 디스커버리 — Eureka vs Kubernetes Service

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- MSA 환경에서 마이크로서비스의 IP/포트를 하드코딩할 수 없는 이유는 무엇인가?
- Client-Side Discovery (Eureka)와 Server-Side Discovery (Kubernetes Service)의 근본적인 차이는 무엇인가?
- Eureka의 Heartbeat, Eviction, Self-Preservation Mode는 어떻게 동작하는가?
- Kubernetes Service의 EndpointSlices와 kube-proxy의 역할은 무엇인가?
- 새 서비스 인스턴스가 등록되거나 제거될 때 라우팅 테이블이 업데이트되는 시간 지연이 발생하는 이유는?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

모노리식 애플리케이션에서는 모든 코드가 하나의 프로세스에서 실행되므로 서비스 위치 발견이 필요 없습니다. 하지만 MSA에서는 수십 개의 독립 서비스가 **동적으로 생성되고 제거**됩니다:

- **수평 확장**: CPU 사용률 증가 → 자동으로 새 인스턴스 시작 (IP: 10.0.1.50, 10.0.1.51, ...)
- **배포**: 무중단 배포 중 일부 인스턴스가 셧다운 되었다가 새 버전으로 재시작
- **장애**: 인스턴스 충돌 → 자동으로 다른 노드에서 재생성 (IP가 변경될 수 있음)
- **개발/테스트**: 로컬 개발 환경과 클라우드 프로덕션의 IP가 완전히 다름

이러한 **동적 환경**에서 클라이언트가 서비스를 찾기 위해서는 **자동 발견(Service Discovery)** 메커니즘이 필수입니다.

---

## 😱 흔한 실수 (Before)

```java
// UserServiceClient.java - 하드코딩된 URL
@RestTemplate
public class UserServiceClient {
    
    private final RestTemplate restTemplate = new RestTemplate();
    
    public User getUser(String userId) {
        // ❌ 문제: user-service가 3개 인스턴스로 확장되면?
        // ❌ 인스턴스가 10.0.1.50에서 10.0.1.51로 이동하면?
        String url = "http://10.0.1.50:8080/users/" + userId;
        return restTemplate.getForObject(url, User.class);
    }
}
```

```yaml
# application.yml - 고정된 서비스 URL 목록
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: http://user-service-1:8080  # ❌ 인스턴스 1개 고정
          predicates:
            - Path=/users/**
        
        - id: order-service
          uri: http://order-service-1:8080  # ❌ 인스턴스 1개 고정
          predicates:
            - Path=/orders/**
```

**문제점:**
- 새 인스턴스 추가 시 설정 파일 수동 수정 필요
- 장애 인스턴스 자동 감지 불가능
- 배포 중 다운타임 발생 (모든 인스턴스가 한 번에만 교체 가능)

---

## ✨ 올바른 접근 (After)

### Client-Side Discovery: Eureka 사용

```java
// UserServiceClient.java - Eureka 기반 동적 발견
@RestController
@RequiredArgsConstructor
public class UserServiceClient {
    
    private final DiscoveryClient discoveryClient;
    private final RestTemplate restTemplate;
    
    public User getUser(String userId) {
        // Eureka에서 user-service의 모든 건강한 인스턴스 조회
        List<ServiceInstance> instances = discoveryClient
                .getInstances("user-service");
        
        if (instances.isEmpty()) {
            throw new ServiceUnavailableException("user-service");
        }
        
        // 라운드로빈 또는 커스텀 로드밸런싱
        ServiceInstance instance = instances.get(
            ThreadLocalRandom.current().nextInt(instances.size())
        );
        
        String url = instance.getUri() + "/users/" + userId;
        return restTemplate.getForObject(url, User.class);
    }
}
```

```java
// Eureka Client 자동 등록 설정
@SpringBootApplication
@EnableDiscoveryClient  // Eureka에 자동 등록
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

```yaml
# application.yml - Eureka 클라이언트 설정
spring:
  application:
    name: user-service
  
  cloud:
    discovery:
      enabled: true

eureka:
  client:
    serviceUrl:
      defaultZone: http://eureka-server-1:8761/eureka/,
                   http://eureka-server-2:8761/eureka/
    # 주기적으로 Eureka에 heartbeat 전송
    heartbeatExecutorThreadPoolSize: 2
    registryFetchIntervalSeconds: 30  # 30초마다 다른 서비스 목록 갱신
  
  instance:
    hostname: user-service-1
    instance-id: ${spring.cloud.client.hostname}:${spring.application.name}:${server.port}
    # Heartbeat 설정
    leaseRenewalIntervalInSeconds: 30  # 30초마다 Eureka에 "나는 살아있다"는 신호
    leaseExpirationDurationInSeconds: 90  # 90초 동안 heartbeat 없으면 제거
```

### Server-Side Discovery: Kubernetes Service 사용

```yaml
# k8s-deployment.yaml - user-service 배포
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: default
spec:
  replicas: 3  # ✅ 3개 인스턴스로 자동 확장 설정
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: myregistry/user-service:v1.2.0
        ports:
        - containerPort: 8080
          name: http
        # 헬스체크: Kubernetes가 인스턴스 상태 모니터링
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
---
# k8s-service.yaml - user-service 서비스 추상화
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: default
spec:
  type: ClusterIP  # 내부 통신 (ClusterIP), 외부 통신 (LoadBalancer/NodePort)
  selector:
    app: user-service
  ports:
  - port: 80         # Service 포트
    targetPort: 8080  # Pod 포트
    protocol: TCP
    name: http
  sessionAffinity: None  # 각 요청을 독립적으로 로드밸런싱
```

```java
// Java에서 Kubernetes Service 활용
@RestController
@RequiredArgsConstructor
public class UserServiceClient {
    
    private final RestTemplate restTemplate;
    
    public User getUser(String userId) {
        // ✅ Kubernetes Service DNS: http://user-service:80
        // kube-proxy가 자동으로 Pod 인스턴스들 중 하나로 라우팅
        String url = "http://user-service/users/" + userId;
        return restTemplate.getForObject(url, User.class);
    }
}
```

---

## 🔬 내부 동작 원리 — 서비스 발견 메커니즘 심층 분석

### Client-Side Discovery: Eureka 상세 흐름

```
┌────────────────────────────────────────────────────────────────┐
│ 타임라인: Eureka 인스턴스 등록 → 발견 → 제거                   │
└────────────────────────────────────────────────────────────────┘

[T=0초] ► user-service-1이 시작
        ┌──────────────────────────────────────┐
        │ 1. ServiceRegistry에 등록 신청       │
        │    - App: user-service               │
        │    - Instance ID: host:port          │
        │    - IP: 10.0.1.50                   │
        │    - Port: 8080                      │
        │    - Status: UP                      │
        └──────────────────────────────────────┘
                          ↓
        ┌──────────────────────────────────────────────┐
        │ Eureka Server (registry 메모리)            │
        │                                              │
        │ user-service (3개 인스턴스)                 │
        │ ├─ user-service-1: UP (10.0.1.50:8080)     │
        │ ├─ user-service-2: UP (10.0.1.51:8080)     │
        │ └─ user-service-3: DOWN (10.0.1.52:8080)   │
        │                                              │
        │ order-service (2개 인스턴스)                │
        │ ├─ order-service-1: UP (10.0.2.50:8080)   │
        │ └─ order-service-2: UP (10.0.2.51:8080)   │
        └──────────────────────────────────────────────┘

[T=30초] ► user-service-1이 주기적으로 Heartbeat 전송
        ┌──────────────────────────────────────┐
        │ "나는 여전히 살아있다!"              │
        │ Last Heartbeat: 2026-04-10 09:00:30 │
        └──────────────────────────────────────┘
                          ↓
        Eureka Server의 타임스탬프 업데이트

[T=35초] ► API Gateway가 user-service 목록 조회
        ┌──────────────────────────────────────┐
        │ 1. 캐시된 레지스트리 확인            │
        │ 2. 갱신 주기(30초) 지났으면          │
        │    Eureka Server에 요청              │
        │ 3. 건강한 인스턴스만 반환            │
        │    (Status=UP)                       │
        └──────────────────────────────────────┘
                          ↓
        반환: [10.0.1.50:8080, 10.0.1.51:8080]
        (10.0.1.52는 DOWN이므로 제외)

[T=65초] ► user-service-3이 충돌 (장애)
        ┌──────────────────────────────────────┐
        │ user-service-3이 Heartbeat를        │
        │ 30초 이상 보내지 않음                │
        │                                      │
        │ Last Heartbeat: 2026-04-10 09:00:35 │
        │ Current Time:   2026-04-10 09:01:05 │
        │ Difference:     30초 > threshold      │
        └──────────────────────────────────────┘

[T=90초] ► Eureka Server에서 user-service-3 제거
        ┌──────────────────────────────────────────────┐
        │ Eviction 발생: Heartbeat 없는 인스턴스 제거  │
        │                                              │
        │ user-service (2개 인스턴스)                 │
        │ ├─ user-service-1: UP (10.0.1.50:8080)     │
        │ └─ user-service-2: UP (10.0.1.51:8080)     │
        │                                              │
        │ (user-service-3 제거됨)                     │
        └──────────────────────────────────────────────┘

[T=120초] ► API Gateway의 다음 갱신 시점
        반환: [10.0.1.50:8080, 10.0.1.51:8080]
        (자동으로 user-service-3 제외)
```

### Server-Side Discovery: Kubernetes Service 상세 흐름

```
┌────────────────────────────────────────────────────────────────┐
│ Kubernetes Service 내부 동작 (iptables/ipvs 기반)             │
└────────────────────────────────────────────────────────────────┘

[설정]
┌─────────────────────────────────┐
│ Service: user-service           │
│ - ClusterIP: 10.96.0.10         │
│ - Port: 80                      │
│ - Selector: app=user-service    │
└─────────────────────────────────┘

[Pods (동적으로 생성/제거)]
┌─────────────────────────────────────────────────────┐
│ user-service-pod-1 (Pod IP: 10.244.1.10:8080)      │
│ user-service-pod-2 (Pod IP: 10.244.1.11:8080)      │
│ user-service-pod-3 (Pod IP: 10.244.1.12:8080)      │
└─────────────────────────────────────────────────────┘

[로드밸런싱 메커니즘: kube-proxy]
┌──────────────────────────────────────────────────────────┐
│ Node의 kube-proxy (Daemonset 실행)                       │
│                                                          │
│ iptables 또는 ipvs 규칙 설정:                           │
│                                                          │
│ $ iptables -L -n -t nat | grep user-service            │
│ DNAT 10.96.0.10:80 → (라운드로빈)                       │
│    ├─ 10.244.1.10:8080 (33%)                           │
│    ├─ 10.244.1.11:8080 (33%)                           │
│    └─ 10.244.1.12:8080 (34%)                           │
└──────────────────────────────────────────────────────────┘

[클라이언트 요청 흐름]
클라이언트 (API Gateway) 
    ↓ DNS 조회: user-service.default.svc.cluster.local
    ↓ 응답: ClusterIP = 10.96.0.10
    ↓ TCP 연결: 10.96.0.10:80
    ↓ kube-proxy의 iptables 규칙 적용
    ↓ DNAT: 10.96.0.10:80 → 10.244.1.10:8080 (예)
    ↓
실제 Pod (user-service-pod-1)

[Pod가 추가되면 (Deployment scale-up)]
1. Kubernetes Scheduler가 새 Pod 생성
2. kubelet이 Pod 시작 (readinessProbe 확인)
3. EndpointController가 Pod IP 감지
4. Endpoints 리소스 업데이트
5. kube-proxy가 iptables 규칙 재생성
6. 클라이언트가 새 Pod로 트래픽 수신 시작

시간: ~1-5초 (Pod 시작 지연 포함)
```

### Self-Preservation Mode (Eureka 특유 기능)

```
상황: Eureka Server와 클라이언트 간의 네트워크 분할
     (client들의 heartbeat가 일시적으로 도달 불가)

[정상 상황: Heartbeat 도달]
Client 1 → heartbeat ✓ → Eureka Server
Client 2 → heartbeat ✓ → Eureka Server
Client 3 → heartbeat ✓ → Eureka Server

[네트워크 분할 (Network Partition)]
Client 1 → heartbeat ✗ (손실)
Client 2 → heartbeat ✗ (손실)
Client 3 → heartbeat ✗ (손실)

Eureka Server 입장:
- 예상 heartbeat율: 90% (설정값)
- 실제 heartbeat율: 0% (모두 손실)
- 차이: > 임계값 (15%)

[자가 보존 모드 활성화]
┌─────────────────────────────────────────────────────────┐
│ Self-Preservation Mode 활성화!                          │
│                                                         │
│ 이유: 클라이언트들이 모두 죽지 않았을 가능성 높음      │
│      (네트워크 문제일 수 있음)                         │
│                                                         │
│ 행동: Eviction 중지                                    │
│     - 기존 등록 정보 유지                              │
│     - 새 인스턴스 등록은 받음                          │
└─────────────────────────────────────────────────────────┘

[네트워크 회복]
Client 1 → heartbeat ✓
Client 2 → heartbeat ✓
Client 3 → heartbeat ✓

Eureka Server:
- heartbeat율 정상화
- 자가 보존 모드 해제
- 정상 운영 재개
```

---

## 💻 Eureka 서버 및 클라이언트 구현

```java
// EurekaServerApplication.java
@SpringBootApplication
@EnableEurekaServer  // Eureka 서버 활성화
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# eureka-server application.yml
spring:
  application:
    name: eureka-server
  
  cloud:
    config:
      enabled: true

server:
  port: 8761

eureka:
  instance:
    hostname: eureka-server-1
  
  server:
    # 서버 간 레플리케이션 설정 (고가용성)
    peerEurekaNodesUpdateIntervalMs: 30000
    # Eviction 스케줄 (주기적으로 실행)
    evictionIntervalTimerInMs: 60000  # 60초마다 Eviction 체크
    # 자가 보존 모드 임계값
    renewalPercentThreshold: 0.85     # heartbeat rate < 85% → 자가 보존 모드
    # 계층화된 레지스트리 읽기 (성능 최적화)
    useReadOnlyResponseCache: true
    readOnlyResponseCacheUpdateIntervalMs: 30000
    # 레지스트리 사본 공유 (클라이언트가 캐시 활용)
    enableSelfPreservation: true  # 자가 보존 모드 활성화

  client:
    # 자기 자신(Eureka Server)도 Eureka 클라이언트로 동작
    serviceUrl:
      defaultZone: http://eureka-server-2:8761/eureka/,
                   http://eureka-server-3:8761/eureka/
```

```java
// UserServiceApplication.java - Eureka 클라이언트
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
    
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

```yaml
# user-service application.yml
spring:
  application:
    name: user-service
  jpa:
    hibernate:
      ddl-auto: validate

server:
  port: 8080
  servlet:
    context-path: /

eureka:
  client:
    serviceUrl:
      # Eureka 서버 주소 (3개 인스턴스로 고가용성)
      defaultZone: http://eureka-server-1:8761/eureka/,
                   http://eureka-server-2:8761/eureka/,
                   http://eureka-server-3:8761/eureka/
    
    # 레지스트리 갱신 주기
    registryFetchIntervalSeconds: 30  # 30초마다 레지스트리 갱신
    
    # 초기 레지스트리 지연 로딩 (시작 속도 개선)
    shouldFetchRegistry: true
    shouldRegisterWithEureka: true
  
  instance:
    # 호스트명 (DNS 해석 가능해야 함)
    hostname: ${spring.cloud.client.hostname:localhost}
    
    # 인스턴스 ID (고유성 중요)
    instance-id: ${spring.application.name}:${spring.cloud.client.ipaddress}:${server.port}
    
    # Heartbeat 설정
    leaseRenewalIntervalInSeconds: 30  # 30초마다 갱신
    leaseExpirationDurationInSeconds: 90  # 90초 동안 heartbeat 없으면 제거
    
    # 상태 페이지 (Eureka UI에 표시)
    statusPageUrlPath: /actuator/info
    healthCheckUrlPath: /actuator/health
    
    # 예선 제거 보호 (잘못된 제거 방지)
    evictionTimerIntervalInMs: 30000

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
```

---

## 📊 패턴 비교

| 항목 | Client-Side (Eureka) | Server-Side (Kubernetes) | DNS 기반 로드밸런싱 |
|------|:---:|:---:|:---:|
| **발견 주체** | 클라이언트 (직접 레지스트리 조회) | 서버 (Service DNS) | DNS 라운드로빈 |
| **로드밸런싱** | 클라이언트가 선택 (RestTemplate) | kube-proxy (iptables/ipvs) | DNS 클라이언트 |
| **등록 메커니즘** | 클라이언트가 heartbeat 전송 | kubelet + EndpointController | DNS 서버 수동 업데이트 |
| **인스턴스 제거** | Eviction (heartbeat 없음) | Pod 삭제 시 자동 제거 | DNS TTL 후 제거 |
| **실패 감지 속도** | 30-90초 (heartbeat + Eviction) | 1-5초 (readinessProbe) | 15-300초 (DNS TTL) |
| **클라이언트 로직** | RestTemplate + DiscoveryClient | 단순 HTTP 호출 (DNS) | 단순 HTTP 호출 (DNS) |
| **장애 격리** | Self-Preservation Mode 지원 | Pod 재시작으로 자동 복구 | 수동 개입 필요 |
| **분산 환경 복잡도** | 높음 (heartbeat, eviction 타이밍) | 낮음 (Kubernetes 추상화) | 매우 낮음 (표준 DNS) |
| **성능 오버헤드** | 중간 (heartbeat 주기적 전송) | 매우 낮음 (iptables 커널 레벨) | 낮음 (DNS 캐시) |
| **다중 클라우드 지원** | 우수 (클라우드 독립적) | 클라우드 종속 (Kubernetes 필요) | 우수 (표준 DNS) |

---

## ⚖️ 트레이드오프

### Eureka (Client-Side Discovery)

#### 장점

✅ **클라우드 독립적**: AWS, Azure, GCP, 온프레미스 모두 동일한 방식
- Kubernetes 없이도 마이크로서비스 아키텍처 구현 가능
- 유연한 배포 옵션

✅ **세밀한 제어**: 클라이언트가 로드밸런싱 정책 결정
- 커스텀 로드밸런싱 알고리즘 구현 가능
- 클라이언트별로 다른 전략 적용 가능

✅ **빠른 적응**: 클라이언트가 로컬 캐시 유지
- 레지스트리 서버 장애 시에도 캐시된 정보로 계속 작동
- 네트워크 분할(Network Partition) 환경에서 Strong Consistency 대신 Availability 선택

#### 단점

❌ **높은 클라이언트 복잡도**: 모든 마이크로서비스가 발견 로직 구현 필요
- 각 언어별 클라이언트 라이브러리 필요 (Java, Python, Node.js, ...)
- 클라이언트 버전 불일치 문제

❌ **느린 장애 감지**: 30-90초 지연
- Heartbeat 전송 → Eviction 체크까지 최소 60초
- 중요한 서비스 장애 시 빠른 복구 불가

❌ **분산 환경 복잡성**: Heartbeat 타이밍, Self-Preservation Mode 등 이해 필요
- 네트워크 지연이 높으면 자가 보존 모드 오작동 가능
- 운영 복잡도 증가

### Kubernetes Service (Server-Side Discovery)

#### 장점

✅ **빠른 장애 감지**: 1-5초 (readinessProbe)
- 장애 인스턴스를 빠르게 트래픽 흐름에서 제외
- 자동 재시작으로 즉각 복구

✅ **간단한 클라이언트 로직**: 단순 DNS 호출
```java
// Kubernetes Service 사용
String url = "http://user-service/users/" + userId;
restTemplate.getForObject(url, User.class);  // 그 끝!
```

✅ **내장 자동 복구**: 장애 인스턴스 자동 재시작
- Pod 충돌 → kubelet이 자동 재시작
- 노드 장애 → 다른 노드에서 Pod 재생성

#### 단점

❌ **Kubernetes 종속성**: 클라우드 제약
- Kubernetes 없으면 사용 불가능
- Kubernetes 버전 업그레이드 시 호환성 이슈

❌ **디버깅 어려움**: DNS 및 iptables 규칙이 블랙박스
- Service IP로 실제 Pod IP 추적 어려움
- 네트워크 정책(NetworkPolicy) 위반 시 디버깅 복잡

❌ **DNS 캐싱 문제**: Pod IP 변경 후 DNS TTL 기간 동안 구 IP로 접근
```
Pod 2 재시작 (IP 변경: 10.244.1.11 → 10.244.1.20)
↓
클라이언트는 여전히 10.244.1.11로 연결 시도 (DNS 캐시)
↓
DNS TTL(기본 30초) 후 새 IP 인식
↓
그동안 Connection 실패 가능
```

---

## 📌 핵심 정리

✅ **MSA에서 서비스 발견은 필수 인프라**
- 동적 환경에서 하드코딩된 URL은 작동 불가능
- 자동으로 인스턴스를 발견하고 장애를 감지해야 함

✅ **Eureka는 마이크로서비스 레벨의 발견 (Application Layer)**
- 클라이언트가 주기적으로 레지스트리 조회
- Heartbeat 기반 건강 확인
- 높은 가용성, 분산된 아키텍처

✅ **Kubernetes Service는 인프라 레벨의 발견 (Network Layer)**
- kube-proxy가 iptables/ipvs로 자동 라우팅
- readinessProbe로 빠른 장애 감지
- 간단한 인터페이스 (Service DNS)

✅ **실제 배포 시 혼합 사용**
- Kubernetes 환경: Service + Istio/Network Policies로 고급 라우팅
- 하이브리드 환경: Eureka + Kubernetes (Istio 없이 기본 Service 사용)
- 멀티클라우드: 전적으로 Eureka에 의존

✅ **장애 감지와 복구 시간이 중요**
- Eureka: 30-90초 (Heartbeat 주기 + Eviction 지연)
- Kubernetes: 1-5초 (readinessProbe 확인 + kubelet 재시작)
- 금융/의료 같은 중요 서비스는 빠른 감지 필수

---

## 🤔 생각해볼 문제

**Q1.** 다음 상황에서 어떤 문제가 발생하고, 어떻게 해결할 수 있을까요?

```
Eureka 클라이언트가 정상이지만, 매 30초마다 heartbeat를 보낼 때
네트워크 지연이 임의로 발생하고 있습니다.

▶ leaseRenewalIntervalInSeconds: 30
▶ leaseExpirationDurationInSeconds: 90
▶ 네트워크 지연: 10-50초 (불규칙)

시뮬레이션:
T=0s: Heartbeat 전송 (도착: T=5s) ✓
T=30s: Heartbeat 전송 (도착: T=70s) ✗ 이미 제거됨
T=60s: Heartbeat 전송 (도착: T=110s)

→ T=90s에 클라이언트가 Eureka에서 제거됨
```

<details><summary>해설 보기</summary>

**문제점:**
- `leaseExpirationDurationInSeconds: 90`은 마지막 heartbeat으로부터 90초를 의미
- T=30s에서 heartbeat를 보냈지만, 네트워크 지연으로 T=70s에 도착
- T=90s (= T=0 + 90)에 첫 heartbeat가 "만료"됨
- T=30s의 heartbeat는 아직 Eureka에 도착하지 않았으므로 인정되지 않음

**해결 방법:**

1. **Heartbeat 간격 단축 (권장하지 않음)**
   ```yaml
   leaseRenewalIntervalInSeconds: 10  # 더 자주 전송
   leaseExpirationDurationInSeconds: 30
   ```
   - 네트워크 오버헤드 증가
   - 여전히 근본 해결 아님

2. **만료 시간 연장 (권장)**
   ```yaml
   leaseRenewalIntervalInSeconds: 30
   leaseExpirationDurationInSeconds: 120  # 연장
   ```
   - Heartbeat 손실에 더 관대
   - 장애 감지 속도는 느려짐 (120초 → 2분)

3. **네트워크 안정화 (근본 해결)**
   - 네트워크 경로 최적화
   - DNS 서버 캐싱 설정 조정
   - Eureka 클라이언트 쪽의 타임아웃 설정 증가

4. **Kubernetes로 마이그레이션**
   - Eureka의 Heartbeat 메커니즘 제거
   - readinessProbe로 더 정확한 상태 확인
   ```yaml
   readinessProbe:
     httpGet:
       path: /actuator/health/readiness
       port: 8080
     initialDelaySeconds: 10
     periodSeconds: 5
     timeoutSeconds: 2
     failureThreshold: 3  # 3번 실패 → 제거
   ```
</details>

**Q2.** Kubernetes Service를 사용하는데, DNS TTL이 30초인 경우 어떤 상황에서 문제가 발생할 수 있고, 어떻게 대처할 수 있을까요?

<details><summary>해설 보기</summary>

**문제 상황:**

1. **Pod 재시작 시 클라이언트의 구 IP로 연결**
   ```
   T=0s: 클라이언트가 user-service DNS 조회
         응답: user-service → 10.244.1.10
   
   T=0s: 클라이언트가 10.244.1.10에 연결 시작
         (DNS 응답 로컬 캐시: 30초)
   
   T=5s: Pod 장애, Kubernetes가 새 Pod 생성
         새 IP: 10.244.1.20
         Endpoints 즉시 업데이트
   
   T=5s~30s: 클라이언트는 여전히 캐시된 10.244.1.10 사용
             → Connection Refused (이미 제거된 Pod)
   
   T=30s: 클라이언트의 DNS 캐시 만료
          새로운 DNS 조회
          응답: 10.244.1.20
          → 성공
   ```

2. **대량 Pod 교체 시 점진적 트래픽 이동 실패**
   - 무중단 배포 중 구 Pod를 종료하는데
   - DNS 캐시 때문에 구 Pod로의 트래픽이 계속 이어짐

**대처 방법:**

1. **DNS TTL 단축 (권장하지 않음)**
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: user-service
   spec:
     # Kubernetes는 기본 30초 TTL로 고정
     # (직접 설정 불가능)
   ```
   - TTL 단축은 DNS 서버 부하 증가

2. **Connection Pooling 최소화 (실용적)**
   ```java
   RestTemplate restTemplate = new RestTemplate(
       new HttpComponentsClientHttpRequestFactory() {{
           HttpClientBuilder builder = HttpClientBuilder.create();
           // Keep-Alive 비활성화
           builder.disableConnectionState();
           // 연결 재사용 최소화
           setConnectTimeout(2000);  // 2초
           setReadTimeout(2000);      // 2초
       }}
   );
   ```
   - 기존 연결 대신 새 연결 자주 생성
   - DNS 캐시 갱신 빈도 증가

3. **마이크로서비스가 아닌 Pod 직접 접근 (권장)**
   ```yaml
   # StatefulSet 사용 (각 Pod가 고유 DNS 이름)
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: user-service
   spec:
     serviceName: user-service
     # → Pod 이름: user-service-0, user-service-1, ...
     # → DNS: user-service-0.user-service.default.svc.cluster.local
   ```
   - Pod별 안정적인 DNS 이름
   - 무중단 배포 시 정확한 제어

4. **Istio/Linkerd 같은 Service Mesh 사용 (권장)**
   - kube-proxy 대신 사이드카 프록시가 라우팅
   - DNS 캐싱 우회
   - 실시간 Pod 상태 반영

5. **Readiness Gates 사용 (고급)**
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: user-service-1
   spec:
     readinessGates:
     - conditionType: custom.example.com/ready
     containers:
     - name: user-service
   ```
   - Pod가 준비되어도 트래픽 수신 지연 가능
   - 무중단 배포 제어
</details>

**Q3.** Eureka와 Kubernetes를 동시에 사용하는 하이브리드 환경에서 어떤 문제가 발생할 수 있을까요?

<details><summary>해설 보기</summary>

**문제점:**

1. **이중 발견 메커니즘 (Double Discovery)**
   ```java
   // 문제: 어느 것을 신뢰할 것인가?
   
   // 방법 1: Eureka 사용
   List<ServiceInstance> instances = discoveryClient.getInstances("user-service");
   
   // 방법 2: Kubernetes Service 사용
   String url = "http://user-service:80";
   ```
   - 클라이언트가 두 시스템을 동시에 쿼리하면 성능 저하
   - 한쪽만 사용하면 다른 쪽의 정보가 쓸모없음

2. **상태 정보 불일치**
   ```
   Eureka에서는 user-service-pod-1이 UP으로 표시
   Kubernetes에서는 같은 Pod가 NotReady (readinessProbe 실패)
   
   → 어느 것을 따를 것인가?
   → Eureka는 30초마다 갱신, Kubernetes는 실시간
   ```

3. **장애 감지 속도 불일치**
   ```
   Kubernetes: Pod 장애 → 1-5초 감지 및 재시작
   Eureka:     장애 → 30-90초 후 Eviction
   
   → Eureka 정보에만 의존하면 장애에 느리게 대응
   ```

4. **운영 복잡도 증가**
   ```
   Eureka 클라이언트 설정 + Kubernetes 리소스 관리
   + 두 시스템의 로그, 모니터링, 알림 분산
   ```

**권장 해결 방법:**

1. **Kubernetes만 사용 (최선)**
   ```yaml
   # Eureka 제거
   # Kubernetes Service로 모든 발견 처리
   # Istio/Linkerd로 고급 라우팅
   ```

2. **마이그레이션 전략 (현실적)**
   ```
   Phase 1: Eureka 유지, 새 서비스는 Kubernetes Service 사용
   Phase 2: 기존 서비스를 Kubernetes로 점진 마이그레이션
   Phase 3: Eureka 제거
   ```

3. **Eureka 제거, Kubernetes Service 전환 (단기 전략)**
   ```java
   // 기존 Eureka 클라이언트 코드
   List<ServiceInstance> instances = discoveryClient.getInstances("user-service");
   
   // → Kubernetes Service로 변경
   String url = "http://user-service/api/v1/users";
   restTemplate.getForObject(url, User.class);
   ```

4. **Eureka → Consul/etcd로 마이그레이션**
   - Kubernetes와 호환성 더 좋음
   - 하지만 여전히 이중 발견 문제 존재
</details>

---

<div align="center">

**[⬅️ 이전: API Gateway 완전 분해 — 필터 체인 내부](./01-api-gateway-deep-dive.md)** | **[홈으로 🏠](../README.md)** | **[다음: Rate Limiting 구현 — Token Bucket과 분산 카운터 ➡️](./03-rate-limiting.md)**

</div>

# 01. API Gateway 완전 분해 — 필터 체인 내부

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- API Gateway가 단순 리버스 프록시(Nginx)와 근본적으로 다른 점은 무엇인가?
- Spring Cloud Gateway의 요청 처리 흐름은 어떻게 동작하며, 필터 체인의 순서는 어떻게 결정되는가?
- GlobalFilter와 GatewayFilter의 차이점은 무엇이고, 언제 어떤 것을 사용해야 하는가?
- 커스텀 필터를 작성할 때 요청 및 응답 바디를 안전하게 처리하는 방법은?
- 분산 환경에서 필터 체인 실행 순서가 보장되지 않으면 어떤 문제가 발생하는가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

MSA 환경에서 API Gateway는 단순한 네트워크 계층의 로드밸런서가 아닙니다. 클라이언트와 수십 개의 마이크로서비스 사이에서 **통합된 관심사를 처리하는 정책 엔진** 역할을 합니다. Nginx 같은 리버스 프록시는 기본적인 트래픽 분산만 하지만, API Gateway는 비즈니스 로직에 영향을 미치지 않으면서 **인증, 인가, 요청 변환, 응답 집계, Rate Limiting, 로깅, 추적** 등을 처리합니다.

Spring Cloud Gateway의 필터 체인은 이러한 모든 기능을 **플러그인 가능한 아키텍처**로 구현합니다. 필터 순서와 실행 타이밍을 정확히 이해하지 못하면, JWT 검증 후 요청이 변조되거나, 응답 인코딩이 깨지거나, Rate Limit이 바이패스될 수 있습니다.

---

## 😱 흔한 실수 (Before)

```yaml
# application.yml - 필터 순서를 고려하지 않은 설정
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: http://user-service:8080
          predicates:
            - Path=/users/**
          filters:
            - RewritePath=/users/(?<segment>.*), /api/$\{segment}
            - AddRequestHeader=Authorization, Bearer token123  # 위험!

      # GlobalFilter들이 어떤 순서로 실행될까? 예측 불가능
      default-filters:
        - name: RequestSize
          args:
            maxSize: 5242880
```

**문제점:**
- `AddRequestHeader` 필터가 로드밸런싱 **전에** 실행될 수도 있고 **후에** 실행될 수도 있음
- 토큰이 경로 변환 후에 추가되면 URL이 다시 쓰여 검증 로직을 우회할 수 있음
- 응답 스트림이 이미 커밋된 후 헤더를 추가하려고 시도하면 에러 발생

---

## ✨ 올바른 접근 (After)

```yaml
# application.yml - 명시적 필터 순서 지정
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service  # 로드밸런서 활용
          predicates:
            - Path=/users/**
          filters:
            # Pre-filter: 요청 전처리 단계
            - name: RequestSize
              args:
                maxSize: 5242880
            
            # 1. 경로 변환 (Pre-phase 시작)
            - RewritePath=/users/(?<segment>.*), /api/$\{segment}
            
            # 2. 인증 헤더 추가 (경로 변환 후, 라우팅 전)
            - AddRequestHeader=X-User-ID, ${header.x-user-id}
            
            # 3. 요청 헤더 로깅 (Custom filter)
            - name: LogRequest
              args:
                logLevel: DEBUG
            
            # Post-filter: 응답 처리 단계 (라우팅 후)
            - name: AddResponseHeader
              args:
                name: X-Response-Time
                value: ${header.x-response-time:0}ms
      
      # GlobalFilter 순서 명시
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: "*"
            allowedMethods: GET,POST,PUT,DELETE
            allowedHeaders: "*"
            maxAge: 3600
```

**개선 사항:**
- `lb://` 스키마로 로드밸런싱을 명시적으로 선언
- 필터 순서를 주석으로 명확히 표시
- Pre-phase와 Post-phase를 분리하여 이해 용이

---

## 🔬 내부 동작 원리 — Spring Cloud Gateway 필터 체인 심층 분석

### 요청 처리 흐름

```
┌─────────────────────────────────────────────────────────────┐
│ 클라이언트 요청 → NettyWebServer                            │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ HttpWebHandlerAdapter (ReactiveStreams 처리)               │
│ - 요청 인코딩 검증, 멀티파트 파싱                           │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ DispatcherHandler (웹 요청 디스패칭)                       │
│ - HandlerMapping 조회 (RoutePredicateHandlerMapping)        │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ RoutePredicateHandlerMapping (라우트 매칭)                 │
│ - Path, Method, Header, Query 기반 Route 선택              │
│ - 라우트에 등록된 필터 목록 준비                            │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ FilteringWebHandler (필터 체인 실행)                       │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Pre-Phase (요청 처리)                              │    │
│  │ ─────────────────────────────────────────────────  │    │
│  │ 1. GlobalFilter (정렬 순서: Ordered 구현)          │    │
│  │    - ReactiveLoadBalancerClientFilter (LB)         │    │
│  │    - WebsocketRoutingFilter (WS 라우팅)           │    │
│  │    - NettyRoutingFilter (HTTP 라우팅)              │    │
│  │                                                     │    │
│  │ 2. Route-specific GatewayFilter (정렬 순서)       │    │
│  │    - RewritePath, AddRequestHeader 등             │    │
│  │                                                     │    │
│  │ 3. RouteToRequestUrlFilter (최종 URL 구성)        │    │
│  └────────────────────────────────────────────────────┘    │
│                          ↓                                   │
│              ► 원본 서비스로 실제 요청 전송 ◄              │
│                          ↓                                   │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Post-Phase (응답 처리)                             │    │
│  │ ─────────────────────────────────────────────────  │    │
│  │ 1. 응답 상태 코드 검사                            │    │
│  │ 2. GlobalFilter (역순 실행)                       │    │
│  │    - WriteResponseFilter (응답 전송)              │    │
│  │                                                     │    │
│  │ 3. Route-specific GatewayFilter (역순)           │    │
│  │    - AddResponseHeader, RewriteResponseHeader     │    │
│  │                                                     │    │
│  │ 4. 응답 바디 인코딩, 압축 등                      │    │
│  └────────────────────────────────────────────────────┘    │
│                          ↓                                   │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ 클라이언트에 응답 반환                                     │
└─────────────────────────────────────────────────────────────┘
```

### 필터 순서 결정 메커니즘

Spring Cloud Gateway는 `Ordered` 인터페이스를 통해 필터 우선순위를 관리합니다:

```java
// org.springframework.core.Ordered
public interface Ordered {
    int getOrder();  // 숫자가 작을수록 먼저 실행
    
    int HIGHEST_PRECEDENCE = Integer.MIN_VALUE;  // -2147483648
    int LOWEST_PRECEDENCE = Integer.MAX_VALUE;   // 2147483647
}
```

**내장 GlobalFilter 순서 (일부):**
- `LoadBalancerClientFilter`: 10
- `RouteToRequestUrlFilter`: 10000
- `WriteResponseFilter`: -1 (마지막에 응답 전송)
- `ForwardRoutingFilter`: Integer.MAX_VALUE
- `WebsocketRoutingFilter`: Integer.MAX_VALUE

GatewayFilter는 Route별로 지정되며, 라우트 설정 순서대로 실행됩니다.

---

## 💻 커스텀 GlobalFilter 작성 및 요청·응답 바디 처리

```java
// CustomLoggingGlobalFilter.java
package com.example.gateway.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.io.ByteArrayOutputStream;
import java.net.URI;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

@Slf4j
@Component
public class CustomLoggingGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // Pre-phase: 요청 정보 기록
        long startTime = System.currentTimeMillis();
        
        ServerWebExchange modifiedExchange = exchange.mutate()
                .request(req -> {
                    // 요청 헤더 로깅
                    log.info("▶ [요청] {} {} | Headers: {}",
                            req.getMethod(),
                            req.getURI(),
                            req.getHeaders());
                    
                    // X-Request-ID 추가 (분산 추적용)
                    req.header("X-Request-ID", generateRequestId());
                    
                    // X-Gateway-Entry-Time 추가 (성능 측정용)
                    req.header("X-Gateway-Entry-Time", String.valueOf(startTime));
                })
                .build();
        
        // 라우팅 수행 후 응답 처리
        return chain.filter(modifiedExchange)
                .doFinally(signal -> {
                    // Post-phase: 응답 정보 기록 (신호 타입과 무관하게 항상 실행)
                    long duration = System.currentTimeMillis() - startTime;
                    
                    log.info("◀ [응답] Status: {} | Duration: {}ms | URI: {}",
                            modifiedExchange.getResponse().getStatusCode(),
                            duration,
                            modifiedExchange.getRequest().getURI());
                    
                    // 응답 헤더에 게이트웨이 처리 시간 추가
                    modifiedExchange.getResponse()
                            .getHeaders()
                            .add("X-Gateway-Process-Time", duration + "ms");
                });
    }
    
    @Override
    public int getOrder() {
        // 다른 필터보다 먼저 실행되도록 높은 우선순위 설정
        return Ordered.HIGHEST_PRECEDENCE + 10;
    }
    
    private String generateRequestId() {
        return "REQ-" + System.currentTimeMillis() + "-" + 
               Thread.currentThread().getId();
    }
}
```

### 요청/응답 바디 캐싱 (주의: 성능 저하)

```java
// RequestBodyCachingGlobalFilter.java
// 문제: HttpServerRequest의 바디는 단 한 번만 읽을 수 있음 (Reactive Streams)
// 해결: 바디를 메모리에 캐시하여 여러 필터가 접근 가능하게 함

@Slf4j
@Component
public class RequestBodyCachingGlobalFilter implements GlobalFilter, Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String contentType = exchange.getRequest()
                .getHeaders()
                .getFirst("Content-Type");
        
        // JSON/XML 요청만 캐싱 (바이너리는 스킵)
        if (contentType != null && 
            (contentType.contains("application/json") || 
             contentType.contains("application/xml"))) {
            
            return exchange.getRequest()
                    .getBody()
                    .collectList()
                    .flatMap(dataBuffers -> {
                        // 데이터 버퍼를 하나로 병합
                        byte[] body = dataBuffers.stream()
                                .map(buf -> {
                                    byte[] bytes = new byte[buf.readableByteCount()];
                                    buf.read(bytes);
                                    return bytes;
                                })
                                .reduce((a, b) -> {
                                    byte[] result = new byte[a.length + b.length];
                                    System.arraycopy(a, 0, result, 0, a.length);
                                    System.arraycopy(b, 0, result, a.length, b.length);
                                    return result;
                                })
                                .orElse(new byte[0]);
                        
                        // 원본 요청 바디 로깅
                        String bodyStr = new String(body, StandardCharsets.UTF_8);
                        log.debug("요청 바디: {}", bodyStr);
                        
                        // 캐시된 바디를 Exchange에 저장
                        exchange.getAttributes().put("CACHED_REQUEST_BODY", body);
                        
                        // 새로운 요청 객체 생성 (바디 포함)
                        ServerWebExchange modifiedExchange = exchange.mutate()
                                .request(req -> {
                                    // 여기서는 새 요청 객체를 반환해야 하는데,
                                    // Spring은 내부적으로 처리
                                })
                                .build();
                        
                        return chain.filter(modifiedExchange);
                    });
        }
        
        return chain.filter(exchange);
    }
    
    @Override
    public int getOrder() {
        // 라우팅 전에 실행 (다른 필터는 캐시된 바디에 접근 가능)
        return Ordered.HIGHEST_PRECEDENCE + 100;
    }
}
```

---

## 📊 패턴 비교

| 항목 | Nginx (리버스 프록시) | Spring Cloud Gateway | Kong | Istio Ingress |
|------|:---:|:---:|:---:|:---:|
| **기본 기능** | 트래픽 라우팅, 로드밸런싱 | 트래픽 라우팅 + 필터 체인 | 트래픽 라우팅 + 플러그인 | 트래픽 관리 + 사이드카 |
| **인증/인가** | 모듈 필요 (nginx-oauth2-proxy) | 내장 (Spring Security 통합) | 플러그인 | 정책 기반 |
| **Rate Limiting** | lua 스크립트 | GatewayFilter (Redis 지원) | 플러그인 | 정책 기반 |
| **요청 변환** | 제한적 (정규식) | 전체 Java 로직 가능 | 플러그인 | 제한적 |
| **응답 집계** | 불가능 | 가능 (GraphQL처럼) | 플러그인 | 불가능 |
| **메모리 사용** | 매우 낮음 (C) | 중간~높음 (JVM) | 중간 (Lua) | 높음 (Envoy) |
| **Java 통합** | 별도 구성 | 자연스러운 통합 | 스크립팅 필요 | Kubernetes 기반 |
| **개발 생산성** | 낮음 (설정 복잡) | 높음 (Java/YAML) | 중간 (Lua/플러그인) | 중간 (YAML) |
| **성능 (req/s)** | 50,000+ | 5,000~10,000 | 10,000~20,000 | 5,000~10,000 |

---

## ⚖️ 트레이드오프

### 장점 (Advantages)

✅ **세밀한 제어**: 필터 체인에서 요청/응답의 모든 측면을 Java 코드로 조작 가능
- JWT 검증 후 Custom 헤더 추가
- 요청 바디 재구성 (JSON 필드 암호화)
- 조건부 응답 변환 (Content-Type에 따라 다른 처리)

✅ **Spring 생태계 통합**: Spring Security, Spring Cloud Config, Spring Data 등 기존 라이브러리 활용 가능
- 자동 설정(Auto-Configuration) 지원
- 테스트 용이 (MockMvc, WebTestClient)

✅ **다중 인스턴스 자동 관리**: Spring Cloud Gateway 인스턴스들은 무상태(Stateless)이므로 수평 확장 용이

### 단점 (Disadvantages)

❌ **높은 메모리 사용**: 모든 요청이 JVM 힙에 적재되므로 대용량 파일 처리 시 문제
- 파일 업로드 (>100MB)는 Nginx 같은 경량 프록시를 별도로 사용하기도 함
- OutOfMemory 위험 (충분한 힙 메모리 사전 할당 필요)

❌ **낮은 처리량 (Throughput)**: Nginx의 10분의 1 수준
- 요청당 GC 오버헤드
- 스레드 풀 크기에 제약

❌ **복잡한 필터 순서 관리**: 순서 오류로 인한 미묘한 버그 발생 가능
- GlobalFilter vs GatewayFilter의 실행 타이밍이 직관적이지 않음
- 여러 필터가 같은 Exchange 속성을 수정하면 경쟁 조건(Race Condition) 발생 가능

❌ **재시작 비용**: 새 필터 배포 시 Gateway 인스턴스 재시작 필요
- Nginx는 설정 리로드로 무중단 가능 (`nginx -s reload`)

---

## 📌 핵심 정리

✅ **API Gateway는 MSA의 단일 진입점**
- 모든 클라이언트 요청이 먼저 도달하는 곳
- 인증, 인가, Rate Limiting 등 **크로스커팅 관심사**를 한곳에서 집중 관리

✅ **Spring Cloud Gateway 필터 체인 구조 이해 필수**
- Pre-phase (요청 처리) → Routing → Post-phase (응답 처리)
- GlobalFilter (모든 요청)와 GatewayFilter (라우트별) 구분
- 필터 순서는 `Ordered` 인터페이스의 `getOrder()` 값으로 결정 (작은 값이 먼저)

✅ **요청/응답 바디 캐싱 주의**
- Reactive Streams에서는 바디를 한 번만 읽을 수 있음
- 여러 필터가 바디에 접근해야 하면 명시적으로 메모리에 캐싱
- 캐싱하면 메모리 사용 급증 → 대용량 파일 처리 불가

✅ **필터 체인은 비동기 논리 흐름**
- `Mono<Void>`와 `chain.filter()` 호출로 Reactive 처리
- 동기 블로킹 코드(예: `Thread.sleep()`) 사용 금지 → 전체 시스템 지연
- 필터 내부 DB 조회는 `Mono`/`Flux`를 반환하는 라이브러리 사용

✅ **프로덕션 환경 Best Practices**
- 모든 필터에 타임아웃 설정 (무한 대기 방지)
- 요청 크기 제한 설정 (`RequestSize` 필터)
- 응답 바디 캐싱은 최소화 (필요한 경우만)
- 필터 체인 로깅으로 병목 지점 식별

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 문제점을 찾고, 올바른 필터 순서를 제시하세요.

```java
@Component
public class MyGlobalFilter1 implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        exchange.getRequest().getHeaders().add("X-Token", "secret123");
        return chain.filter(exchange);
    }
    @Override
    public int getOrder() { return 20; }
}

@Component
public class MyGlobalFilter2 implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("X-Token");
        log.info("Token: {}", token);
        return chain.filter(exchange);
    }
    @Override
    public int getOrder() { return 10; }  // MyGlobalFilter1보다 먼저 실행됨
}
```

<details><summary>해설 보기</summary>

**문제점:**
- `MyGlobalFilter2` (order=10)가 `MyGlobalFilter1` (order=20)보다 먼저 실행되므로, Token 헤더가 아직 추가되지 않은 상태에서 조회 시도
- 결과: `token == null` → "Token: null" 출력
- 또한 `exchange.getRequest().getHeaders().add()`는 **불변 객체**이므로 실제로 헤더가 추가되지 않을 수 있음

**올바른 해결:**
1. Filter2의 order를 30 이상으로 변경 (Filter1 이후 실행)
2. 헤더 추가는 `mutate()` 사용:
```java
return chain.filter(
    exchange.mutate()
        .request(req -> req.header("X-Token", "secret123"))
        .build()
);
```
3. GatewayFilter로 변경 (라우트별로 순서 명시) - 더 명확함
</details>

**Q2.** Spring Cloud Gateway에서 요청 바디를 로깅하고 싶은데, 다음과 같이 구현했을 때 무엇이 잘못되었고 어떻게 고칠 수 있을까요?

```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    exchange.getRequest().getBody()
            .map(buf -> {
                byte[] bytes = new byte[buf.readableByteCount()];
                buf.read(bytes);
                log.info("Request body: {}", new String(bytes));
                return buf;
            })
            .subscribe();  // ← 문제!
    return chain.filter(exchange);  // 바디를 아직 읽지 않은 상태로 라우팅
}
```

<details><summary>해설 보기</summary>

**문제점:**
1. `subscribe()`는 비동기 작업을 별도 스레드에서 수행하므로, 메인 흐름이 기다리지 않음
2. `chain.filter(exchange)` 호출 시점에 바디가 아직 읽혀지지 않았음
3. 후속 필터나 서비스는 이미 소비된 바디를 접근하려고 할 때 에러

**올바른 해결:**
```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    return exchange.getRequest().getBody()
            .collectList()  // 모든 DataBuffer 수집
            .flatMap(buffers -> {
                byte[] body = /* 버퍼들을 병합 */;
                log.info("Request body: {}", new String(body));
                
                // 캐시된 바디를 저장
                exchange.getAttributes().put("CACHED_REQUEST_BODY", body);
                
                // 이제 라우팅 수행 (바디가 완전히 읽힘)
                return chain.filter(exchange);
            });
}
```

**또는** 라이브러리 사용 (예: `spring-cloud-gateway-rsocket`에서 제공하는 utilities)
</details>

**Q3.** 다음 설정에서 1,000 req/s가 들어올 때, Rate Limiting 필터가 정확하게 작동하려면 어떤 조건이 필요할까요?

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: api-v1
          uri: lb://api-service
          predicates:
            - Path=/api/v1/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100    # 초당 100개 토큰 추가
                redis-rate-limiter.burstCapacity: 200    # 최대 200개 토큰 보유
                key-resolver: "#{@userKeyResolver}"      # 사용자별 제한
```

<details><summary>해설 보기</summary>

**필요한 조건:**

1. **Redis 서버의 고가용성**
   - 단일 Redis 인스턴스 장애 → 모든 Rate Limiting 작동 중지
   - 마스터-슬레이브 복제, 또는 Sentinel/Cluster 필수

2. **Redis Lua 스크립트의 원자성**
   - Spring Cloud Gateway는 내부적으로 Redis에서 Lua 스크립트 실행
   - 스크립트 실행 중 다른 클라이언트의 명령이 끼어들지 않음 (원자성 보장)

3. **여러 Gateway 인스턴스의 동기화**
   ```
   ┌─────────────────┐
   │ Gateway1        │ ──┐
   │ (100 req/s)     │   │
   ├─────────────────┤   ├──→ Redis (Rate Limiter)
   │ Gateway2        │   │    (단일 진실 공급원)
   │ (100 req/s)     │   │
   ├─────────────────┤   │
   │ Gateway3        │ ──┘
   │ (800 req/s)     │
   ```
   - 각 Gateway가 로컬 메모리를 사용하면 안 됨
   - Redis를 중앙 카운터로 사용해야 함

4. **Key Resolver의 정확한 구현**
   ```java
   @Bean
   public KeyResolver userKeyResolver() {
       return exchange -> Mono.just(
           exchange.getRequest()
               .getHeaders()
               .getFirst("X-User-ID")  // 또는 JWT에서 추출
               ?? "anonymous"
       );
   }
   ```
   - 사용자 ID가 없으면 익명 사용자로 취급
   - IP 기반 제한도 가능하지만 프록시 뒤에서는 신뢰 불가

5. **Clock Skew 고려**
   - Gateway 인스턴스들의 시계가 동기화되어 있어야 함
   - NTP(Network Time Protocol) 설정 필수

**결론:** Rate Limiter는 **Redis 기반 분산 제어**이므로, 클라우드 환경에서는 마이크로서비스 전체가 공유할 수 있는 중앙 Redis가 필수입니다.
</details>

---

<div align="center">

**[⬅️ 이전: Chapter 5 — 헬스체크와 자가 치유](../availability-patterns/06-health-check-and-self-healing.md)** | **[홈으로 🏠](../README.md)** | **[다음: 서비스 디스커버리 — Eureka vs Kubernetes Service ➡️](./02-service-discovery.md)**

</div>

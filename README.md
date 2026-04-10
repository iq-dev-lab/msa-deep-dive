<div align="center">

# 🏗️ MSA Deep Dive

**"서비스를 쪼개는 것과, 왜 그 경계에서 쪼개야 하는지 — 그리고 분리 이후 데이터 일관성을 어떻게 보장하는지 아는 것은 다르다"**

<br/>

> *"`@FeignClient`로 서비스 연결했으니 MSA겠지 — 와 — Bounded Context 기반으로 경계를 긋고, Saga로 분산 트랜잭션 없이 일관성을 달성하며, Circuit Breaker가 장애 전파를 차단하는 원리를 아는 것의 차이를 만드는 레포"*

서비스를 분리했지만 Shared Database가 남아있는 이유, 동기 호출 3단계가 가용성을 0.997로 낮추는 수학적 원리, Choreography Saga가 보상 트랜잭션을 체인으로 실행하는 흐름, API Gateway 필터 체인의 내부까지  
**왜 이렇게 설계됐는가** 라는 질문으로 MSA 아키텍처 전체를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![MSA](https://img.shields.io/badge/Architecture-MSA-0052CC?style=flat-square&logo=apachekafka&logoColor=white)](https://microservices.io/patterns/)
[![Spring](https://img.shields.io/badge/Spring_Cloud-2023.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://spring.io/projects/spring-cloud)
[![Kafka](https://img.shields.io/badge/Kafka-3.x-231F20?style=flat-square&logo=apachekafka&logoColor=white)](https://kafka.apache.org/)
[![Docs](https://img.shields.io/badge/Docs-41개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

MSA에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "서비스를 작게 나누면 됩니다" | Bounded Context 기반으로 어디서 경계를 그어야 하는가, 잘못 나눴을 때 분산 모놀리스가 만들어지는 이유 |
| "서비스끼리 REST로 통신합니다" | 동기 호출이 가용성을 `0.99 * 0.99 * 0.99 = 0.97`로 낮추는 원리, 어떤 경우에 비동기 이벤트를 선택해야 하는가 |
| "DB를 서비스마다 분리하세요" | Shared Database가 실제로 시스템을 어떻게 망가뜨리는가, 조인 없이 데이터를 조회하는 3가지 전략 |
| "분산 트랜잭션이 필요하면 Saga를 쓰세요" | Choreography Saga의 보상 트랜잭션 체인, Orchestrator의 상태 머신 설계, Dead Saga 감지와 수동 개입 |
| "Circuit Breaker를 추가하세요" | Closed/Open/Half-Open 상태 전이, Resilience4j 슬라이딩 윈도우 계산, Bulkhead 스레드 풀 격리 원리 |
| "API Gateway로 라우팅합니다" | Spring Cloud Gateway 필터 체인 내부 동작, Redis 기반 분산 Rate Limiting, JWT 검증 오프로딩 흐름 |
| 이론 나열 | 실행 가능한 Docker Compose 멀티 서비스 환경 + Spring Cloud + Kafka + Kubernetes 환경 고려 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Ch1](https://img.shields.io/badge/🔹_Ch1-MSA_설계_철학과_분해_전략-0052CC?style=for-the-badge)](./service-decomposition/01-msa-problem-and-cost.md)
[![Ch2](https://img.shields.io/badge/🔹_Ch2-서비스_간_통신_패턴-0052CC?style=for-the-badge)](./communication-patterns/01-sync-vs-async.md)
[![Ch3](https://img.shields.io/badge/🔹_Ch3-Database_per_Service-0052CC?style=for-the-badge)](./data-management/01-database-per-service.md)
[![Ch4](https://img.shields.io/badge/🔹_Ch4-Saga_분산_일관성-0052CC?style=for-the-badge)](./saga/01-distributed-transaction-problem.md)
[![Ch5](https://img.shields.io/badge/🔹_Ch5-가용성_패턴-0052CC?style=for-the-badge)](./availability-patterns/01-circuit-breaker.md)
[![Ch6](https://img.shields.io/badge/🔹_Ch6-API_Gateway-0052CC?style=for-the-badge)](./api-gateway/01-api-gateway-deep-dive.md)
[![Ch7](https://img.shields.io/badge/🔹_Ch7-관찰_가능성과_운영-0052CC?style=for-the-badge)](./observability/01-distributed-tracing.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: MSA 설계 철학과 분해 전략

> **핵심 질문:** MSA는 무엇을 해결하고 무엇을 대가로 가져가는가? 서비스 경계는 어떻게 결정하며, 잘못 나눴을 때 왜 모놀리스보다 나쁜 시스템이 만들어지는가?

<details>
<summary><b>모놀리스의 실제 문제부터 서비스 자율성 원칙까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. MSA가 해결하는 문제 — 모놀리스의 실제 한계](./service-decomposition/01-msa-problem-and-cost.md) | 독립 배포 불가·기술 스택 종속·팀 결합이 실제로 어떻게 개발 속도를 낮추는가, MSA가 공짜가 아닌 이유(운영 복잡도, 분산 시스템 문제 전부 감수), 언제 MSA가 정답이고 언제 모놀리스가 더 나은가 |
| [02. 서비스 분해 원칙 — Bounded Context와 단일 책임](./service-decomposition/02-service-decomposition-principles.md) | Bounded Context 기반 분해와 DDD 연결, 단일 책임 원칙(SRP)을 서비스 수준에서 적용하는 방법, Business Capability 기반 분해, "팀이 독립적으로 배포할 수 있는가" 기준으로 서비스 크기를 정하는 방법 |
| [03. 분산 모놀리스 안티패턴 — 쪼갰지만 여전히 결합된 이유](./service-decomposition/03-distributed-monolith-antipatterns.md) | 서비스를 나눴지만 강결합이 남은 이유, Shared Database 안티패턴이 스키마 변경을 불가능하게 만드는 시나리오, 동기 호출 체인이 가용성을 `0.99^N`으로 기하급수적으로 낮추는 수학적 증명 |
| [04. 서비스 분해 실전 — 전자상거래 도메인 예시](./service-decomposition/04-decomposition-in-practice.md) | 주문·배송·결제·상품 서비스를 어떤 기준으로 나누는가, 서비스 경계 재조정 전략(분리 후 합치기, 합친 것 다시 나누기), 실제 경계 설정 실수 사례와 수정 방법 |
| [05. MSA 도입 판단 기준 — Conway's Law와 팀 구조](./service-decomposition/05-msa-adoption-criteria.md) | Conway's Law(조직 구조가 시스템 구조를 결정한다)의 실증 사례, 팀 규모별 권장 아키텍처, 모놀리스 우선 전략과 Strangler Fig 패턴으로 점진적 전환하는 방법 |
| [06. 서비스 자율성 원칙 — 독립 배포·확장·기술·데이터](./service-decomposition/06-service-autonomy-principles.md) | 독립 배포 가능성·독립 확장 가능성·기술 스택 자율성·데이터 자율성의 4가지 기준, 각 기준을 위반했을 때 발생하는 실제 문제, 자율성 점검 체크리스트 |

</details>

<br/>

### 🔹 Chapter 2: 서비스 간 통신 패턴

> **핵심 질문:** 동기 통신과 비동기 통신 중 언제 무엇을 선택해야 하는가? REST, gRPC, 이벤트 기반 통신은 결합도와 가용성을 어떻게 다르게 만드는가?

<details>
<summary><b>동기 vs 비동기 선택 원칙부터 GraphQL Federation까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 동기 통신 vs 비동기 통신 — 선택 기준](./communication-patterns/01-sync-vs-async.md) | 즉시 응답 필요 여부·결합도 허용 여부에 따른 선택 기준, 동기 호출이 가용성을 낮추는 원리(`0.99 * 0.99 * 0.99 = 0.97`), 비동기 이벤트의 Eventual Consistency 비용과 수용 방법 |
| [02. REST API 설계 — 서비스 간 계약과 버저닝](./communication-patterns/02-rest-api-design.md) | 서비스 간 REST API 설계 원칙, URL 버저닝 vs Header 버저닝 트레이드오프, Consumer-Driven Contract Testing으로 Breaking Change를 배포 전에 감지하는 방법 |
| [03. gRPC 완전 분해 — Protocol Buffers와 HTTP/2](./communication-patterns/03-grpc-deep-dive.md) | Protocol Buffers 직렬화가 JSON보다 빠른 이유(바이너리 인코딩, 스키마 기반 생략), HTTP/2 멀티플렉싱으로 하나의 연결에서 여러 스트림을 처리하는 원리, 서버 스트리밍·양방향 스트리밍 활용, REST와의 선택 기준 |
| [04. 이벤트 기반 통신 — Kafka와 결합도 분리](./communication-patterns/04-event-driven-communication.md) | 이벤트 발행/구독이 결합도를 낮추는 원리, 이벤트 스키마 진화(Backward/Forward Compatibility)로 무중단 배포하는 방법, Kafka Topic 설계 원칙(서비스당 토픽 vs 이벤트 유형당 토픽) |
| [05. API Composition 패턴 — 여러 서비스 데이터 조합](./communication-patterns/05-api-composition.md) | 조인 없이 여러 서비스 데이터를 조합하는 방법, API Gateway에서 조합하는 방법과 BFF(Backend for Frontend) 패턴, 조합 중 하나의 서비스가 실패할 때 처리 방법 |
| [06. Service Mesh — Envoy Sidecar와 mTLS](./communication-patterns/06-service-mesh.md) | Envoy Sidecar가 애플리케이션 코드 수정 없이 트래픽을 가로채는 원리, mTLS로 서비스 간 암호화하는 방법, Istio Circuit Breaker가 Hystrix와 다른 점, 언제 Service Mesh가 필요한가 |
| [07. GraphQL Federation — 분산 스키마 통합](./communication-patterns/07-graphql-federation.md) | 여러 서비스의 GraphQL 스키마를 Supergraph로 합치는 방법, Subgraph가 자신의 타입을 소유하고 Federation이 쿼리를 라우팅하는 원리, REST vs GraphQL 선택 기준 |

</details>

<br/>

### 🔹 Chapter 3: 데이터 관리 — Database per Service

> **핵심 질문:** 왜 서비스마다 DB를 분리해야 하는가? 분리하면 발생하는 조인 문제와 일관성 문제를 어떻게 해결하는가?

<details>
<summary><b>Shared Database 안티패턴부터 서비스 간 외래키까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Database per Service 원칙 — 왜 DB를 분리해야 하는가](./data-management/01-database-per-service.md) | Shared Database가 실제로 스키마 변경을 불가능하게 만드는 시나리오, 서비스마다 DB를 분리했을 때 발생하는 조인 문제 해결 방향, 분리 수준(스키마 분리 vs 서버 분리) 선택 기준 |
| [02. Polyglot Persistence — 서비스별 DB 기술 선택](./data-management/02-polyglot-persistence.md) | 주문 서비스 MySQL + 상품 검색 Elasticsearch + 세션 Redis를 함께 설계하는 방법, 각 DB 기술의 특성에 맞는 서비스 유형, 폴리글랏 영속성의 운영 복잡도 비용 |
| [03. Join 없는 데이터 조회 전략 — API Composition, CQRS, 복제](./data-management/03-join-free-query-strategies.md) | API Composition(서비스 조합)·CQRS 읽기 모델·데이터 복제 허용 설계의 3가지 전략 비교, 각 전략의 일관성 트레이드오프, 서비스 패턴별 전략 선택 기준 |
| [04. 분산 데이터 일관성 — ACID vs BASE](./data-management/04-distributed-data-consistency.md) | ACID와 BASE의 실질적 차이, Eventual Consistency를 애플리케이션 레벨에서 보장하는 방법, 강한 일관성이 반드시 필요한 경우와 허용 가능한 경우를 구분하는 기준 |
| [05. 데이터 이관 전략 — 모놀리스 DB에서 서비스별 DB로](./data-management/05-data-migration-strategies.md) | 모놀리스 DB를 서비스별 DB로 점진적으로 분리하는 방법, Strangler Fig 데이터 이관 패턴, 이관 중 두 시스템이 동시에 데이터를 쓸 때 동기화 전략 |
| [06. 서비스 간 외래키 — 물리적 무결성 없이 일관성 보장](./data-management/06-cross-service-foreign-keys.md) | 물리적 외래키 대신 애플리케이션 레벨에서 참조 무결성을 보장하는 방법, 참조 무결성을 이벤트로 보장하는 패턴, 고아 데이터 발생 시 처리 전략 |

</details>

<br/>

### 🔹 Chapter 4: Saga — 분산 트랜잭션 없이 일관성 달성

> **핵심 질문:** 2PC 없이 여러 서비스에 걸친 데이터 일관성을 어떻게 달성하는가? Choreography와 Orchestration은 어떤 상황에서 각각 선택해야 하는가?

<details>
<summary><b>2PC의 문제부터 Saga 실패 모니터링까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 분산 트랜잭션의 문제 — 2PC와 Saga](./saga/01-distributed-transaction-problem.md) | 2PC(Two-Phase Commit)가 MSA에서 실용적으로 쓸 수 없는 이유(가용성 저하, 코디네이터 단일 실패점), CAP Theorem에서 가용성과 일관성의 트레이드오프, Saga가 2PC를 대체하는 원리 |
| [02. Choreography Saga — 이벤트 기반 자율 실행](./saga/02-choreography-saga.md) | 각 서비스가 이벤트에 반응하여 다음 단계를 스스로 진행하는 방법, 주문→결제→재고→배송 Choreography 흐름 전체, 중간 실패 시 보상 트랜잭션이 역방향으로 체인되는 시나리오 |
| [03. Orchestration Saga — 중앙 오케스트레이터 설계](./saga/03-orchestration-saga.md) | Saga Orchestrator가 상태 머신으로 각 단계를 중앙에서 지시하는 방법, Orchestrator 장애 시 상태를 복구하는 전략, Orchestrator의 책임 범위와 서비스 응집도 유지 방법 |
| [04. Choreography vs Orchestration — 선택 기준](./saga/04-choreography-vs-orchestration.md) | 결합도·복잡도·디버깅 난이도의 3가지 기준으로 두 방식 비교, 서비스 수와 Saga 단계 수에 따른 선택 가이드, 실무 권장 패턴과 혼합 사용 전략 |
| [05. 보상 트랜잭션 설계 — 멱등성과 취소 불가 단계](./saga/05-compensating-transaction.md) | 모든 Saga 단계가 취소 가능해야 하는 이유, 취소할 수 없는 단계(Pivot Transaction) 처리 방법, 보상 트랜잭션을 멱등하게 구현하지 않았을 때 발생하는 중복 실행 문제 |
| [06. Saga 실패 처리와 모니터링](./saga/06-saga-failure-and-monitoring.md) | Saga 진행 상태를 추적하는 방법, 중간 실패 시 재시작 전략(처음부터 vs 실패 지점부터), Dead Saga 감지 방법과 수동 개입 절차, Saga 상태 대시보드 구성 |

</details>

<br/>

### 🔹 Chapter 5: 가용성 패턴 — 장애 격리와 복원력

> **핵심 질문:** 하나의 서비스 장애가 어떻게 전체 시스템으로 전파되는가? Circuit Breaker, Bulkhead, Retry, Fallback을 어떻게 조합해야 장애 전파를 차단할 수 있는가?

<details>
<summary><b>Circuit Breaker 상태 전이부터 자가 치유까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Circuit Breaker 완전 분해 — Resilience4j 내부](./availability-patterns/01-circuit-breaker.md) | Closed/Open/Half-Open 상태 전이 조건, 실패율 임계값을 계산하는 슬라이딩 윈도우(Count 기반 vs Time 기반) 내부 구조, Spring Cloud CircuitBreaker 통합과 설정별 동작 차이 |
| [02. Bulkhead 패턴 — 스레드 풀 격리로 장애 전파 차단](./availability-patterns/02-bulkhead.md) | 하나의 서비스 장애가 스레드 풀을 고갈시켜 다른 서비스 호출을 멈추는 시나리오, Semaphore 기반 vs Thread Pool 기반 Bulkhead 차이, Resilience4j Bulkhead 구현과 설정 기준 |
| [03. Retry와 Exponential Backoff — 재시도가 장애를 키우는 경우](./availability-patterns/03-retry-and-backoff.md) | 재시도가 서버 부하를 폭발적으로 키우는 Thundering Herd 문제, Jitter 추가로 재시도를 분산하는 방법, 멱등성 보장 없이 재시도할 때의 중복 실행 위험 |
| [04. Timeout 전략 — 자원 고갈을 막는 타임아웃 설정](./availability-patterns/04-timeout-strategies.md) | 타임아웃 없이 서비스를 호출할 때 스레드가 고갈되는 원리, 적절한 Timeout 값을 결정하는 방법(P99 응답 시간 기반), Timeout + Retry + Circuit Breaker를 함께 설정할 때의 우선순위 |
| [05. Fallback 전략 — 기능 축소 운영 설계](./availability-patterns/05-fallback-strategies.md) | 서비스 장애 시 기본값 반환·캐시 데이터 반환·기능 축소 운영(Graceful Degradation) 3가지 Fallback 전략, Fallback 체인 설계(1차 Fallback 실패 시 2차), Fallback이 마스킹하면 안 되는 오류 유형 |
| [06. 헬스체크와 자가 치유 — Actuator와 Kubernetes Probe](./availability-patterns/06-health-check-and-self-healing.md) | Spring Boot Actuator 헬스체크가 실제로 무엇을 확인하는가, Kubernetes Liveness Probe vs Readiness Probe가 다른 이유(재시작 vs 트래픽 차단), 서비스 자동 복구 시나리오 |

</details>

<br/>

### 🔹 Chapter 6: API Gateway와 서비스 디스커버리

> **핵심 질문:** API Gateway는 단순 리버스 프록시와 무엇이 다른가? 서비스가 동적으로 늘어나는 환경에서 URL을 어떻게 관리하는가?

<details>
<summary><b>Gateway 필터 체인부터 BFF 패턴까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. API Gateway 완전 분해 — 필터 체인 내부](./api-gateway/01-api-gateway-deep-dive.md) | 단순 리버스 프록시가 아닌 이유(인증 오프로딩·Rate Limiting·요청 변환·라우팅), Spring Cloud Gateway의 GlobalFilter → GatewayFilter → Proxy 처리 흐름, 커스텀 필터를 작성하는 방법 |
| [02. 서비스 디스커버리 — Eureka vs Kubernetes Service](./api-gateway/02-service-discovery.md) | 하드코딩된 URL이 MSA에서 작동하지 않는 이유, Client-Side Discovery(Eureka + Ribbon)와 Server-Side Discovery(Kubernetes Service) 비교, 서비스가 등록·해제될 때 라우팅 테이블이 업데이트되는 타이밍 |
| [03. Rate Limiting 구현 — Token Bucket과 분산 카운터](./api-gateway/03-rate-limiting.md) | Token Bucket / Sliding Window 알고리즘의 작동 원리와 차이, Gateway에서 Redis로 분산 Rate Limiting을 구현하는 방법, 사용자별/클라이언트별/API별 한도를 다르게 설정하는 구조 |
| [04. 인증·인가 아키텍처 — JWT 오프로딩과 서비스 간 인증](./api-gateway/04-auth-architecture.md) | Gateway에서 JWT를 검증하고 각 서비스는 토큰을 신뢰하는 구조, OAuth2 + OIDC로 외부 인증을 통합하는 방법, 서비스 간 인증(Service Account, mTLS)과 사용자 인증의 분리 |
| [05. BFF 패턴 — 클라이언트별 최적화 API 레이어](./api-gateway/05-bff-pattern.md) | 웹·모바일·서드파티 클라이언트가 필요로 하는 데이터 형태가 다른 이유, 각 클라이언트 유형에 맞는 BFF를 두어 API 조합 책임을 위임하는 방법, GraphQL을 BFF로 사용하는 설계 |

</details>

<br/>

### 🔹 Chapter 7: 관찰 가능성과 운영

> **핵심 질문:** 요청이 10개의 서비스를 거칠 때 어디서 느려지는지 어떻게 찾는가? MSA 운영에서 반복적으로 나타나는 장애 패턴은 무엇인가?

<details>
<summary><b>분산 추적부터 운영 장애 패턴까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 분산 추적 — Trace ID로 서비스 흐름 추적](./observability/01-distributed-tracing.md) | Trace ID / Span ID가 요청의 전체 경로를 어떻게 기록하는가, OpenTelemetry + Zipkin/Jaeger 구성, 느린 서비스를 Waterfall 차트에서 핀포인트하는 방법, 샘플링 전략(전체 vs 확률적 샘플링) |
| [02. 중앙화 로깅 — Trace ID로 분산 로그 조합](./observability/02-centralized-logging.md) | 각 서비스의 로그를 Elasticsearch로 수집하는 구조(ELK / EFK Stack), Trace ID 기준으로 여러 서비스의 로그를 하나의 요청으로 조합하는 방법, 로그 레벨 전략과 구조화 로그(JSON) 포맷 |
| [03. 메트릭 모니터링 — RED 방법론과 Grafana 대시보드](./observability/03-metrics-monitoring.md) | 서비스별 Prometheus 메트릭 수집 구성, RED 방법론(Rate·Errors·Duration)으로 MSA 핵심 지표를 정의하는 방법, Grafana 대시보드로 서비스 상태를 한눈에 파악하는 구성 |
| [04. 배포 전략 — Blue/Green, Canary, Feature Flag](./observability/04-deployment-strategies.md) | Blue/Green 배포로 다운타임 없이 전환하는 방법, Canary 배포로 점진적 트래픽을 이전하며 위험을 낮추는 방법, Feature Flag로 배포와 기능 활성화를 분리하는 전략, Kubernetes Rolling Update 내부 동작 |
| [05. 운영 중 발생하는 장애 패턴 — 순환 의존성과 카스케이드 장애](./observability/05-operational-failure-patterns.md) | 서비스 간 순환 의존성 감지와 해소 방법, 카스케이드 장애가 퍼지는 경로 분석, 분산 데드락 발생 조건, Saga 실패를 추적하고 수동으로 개입하는 절차 |

</details>

---

## 🧪 실험 환경

이 레포의 모든 예시는 아래 Docker Compose 환경에서 실행됩니다.

```yaml
# docker-compose.yml
version: '3.8'
services:
  order-service:
    build: ./order-service
    ports: ["8081:8080"]
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql-order:3306/order_db
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:8761/eureka

  payment-service:
    build: ./payment-service
    ports: ["8082:8080"]
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql-payment:3306/payment_db
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092

  api-gateway:
    build: ./api-gateway
    ports: ["8080:8080"]

  eureka:
    build: ./eureka-server
    ports: ["8761:8761"]

  mysql-order:
    image: mysql:8.0
    environment: { MYSQL_DATABASE: order_db, MYSQL_ROOT_PASSWORD: root }

  mysql-payment:
    image: mysql:8.0
    environment: { MYSQL_DATABASE: payment_db, MYSQL_ROOT_PASSWORD: root }

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment: { ZOOKEEPER_CLIENT_PORT: 2181 }

  zipkin:
    image: openzipkin/zipkin:latest
    ports: ["9411:9411"]

  prometheus:
    image: prom/prometheus:latest
    ports: ["9090:9090"]

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
```

---

## 🔗 선행 학습 & 연결 레포

이 레포는 아래 레포들과 개념적으로 연결됩니다. 선행 학습 후 이 레포를 읽으면 훨씬 깊이 이해할 수 있습니다.

| 레포 | 연결 지점 |
|------|----------|
| **ddd-deep-dive** | Chapter 1 — Bounded Context 기반 서비스 분해 이해 필수 |
| **kafka-deep-dive** | Chapter 4 — Saga Choreography, Outbox Pattern 구현 |
| **spring-cloud-deep-dive** | Chapter 6 — Eureka, Gateway, Circuit Breaker 컴포넌트 이해 |
| **network-deep-dive** | Chapter 2 — REST/gRPC 통신 원리, TLS mTLS 이해 |

---

## 📚 참고 자료

- **Microservices Patterns** (Chris Richardson) — Saga, CQRS, 분산 데이터 패턴의 바이블
- **Building Microservices, 2nd Edition** (Sam Newman)
- **Designing Distributed Systems** (Brendan Burns)
- [microservices.io/patterns](https://microservices.io/patterns/) — Chris Richardson의 패턴 카탈로그
- [martinfowler.com/microservices](https://martinfowler.com/microservices/) — Martin Fowler의 MSA 아티클
- [netflixtechblog.com](https://netflixtechblog.com/) — Netflix 실전 MSA 사례

---

<div align="center">

**"분산 모놀리스를 만들지 않으려면, 서비스 경계보다 데이터 경계를 먼저 그어야 한다"**

</div>

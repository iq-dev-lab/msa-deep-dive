# 02. Polyglot Persistence — 서비스별 DB 기술 선택

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Polyglot Persistence가 무엇이고 왜 필요한가?
- 각 서비스에 어떤 DB 기술이 적합한지 어떻게 판단하는가?
- 주문, 검색, 세션 등 다양한 서비스에 최적의 DB 기술 조합은?
- Polyglot 환경의 운영 복잡성을 어떻게 관리할 것인가?
- CQRS와 Polyglot Persistence를 함께 사용할 때의 설계는?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

**Database per Service 원칙은 각 서비스가 독립적인 DB를 가져야 한다고 말합니다. 그렇다면 모두 같은 DB 기술을 써야 할까요?** 절대 아닙니다.

각 서비스는 **다른 특성의 데이터와 다른 접근 패턴**을 가집니다. 주문 서비스는 트랜잭션 무결성이 중요하지만, 상품 검색은 빠른 조회와 복잡한 쿼리가 중요합니다. 세션 저장소는 빠른 쓰기와 TTL(Time To Live)이 핵심입니다. 분석 서비스는 대용량 데이터 수집이 필요합니다. **같은 DB 기술로 모두 최적화할 수 없습니다.**

Polyglot Persistence는 **각 서비스의 요구사항에 맞는 최적의 DB 기술을 선택하는 것**입니다. 이는 Database per Service 원칙과 결합하면 진정한 마이크로서비스의 유연성과 확장성을 제공합니다.

**하지만 비용이 따릅니다.** 서로 다른 DB를 관리한다는 것은 DBA 팀이 여러 기술을 이해해야 하고, 백업 전략도 다르고, 모니터링도 다르고, 인프라도 복잡해집니다. 이 트레이드오프를 정확히 이해하고 결정해야 합니다.

---

## 😱 흔한 실수 (Before — 모든 서비스가 MySQL 사용)

```yaml
# application-monolith.yml
# 초기 설정: 모든 서비스가 MySQL 사용 (단순하지만 비효율)

# 주문 서비스 (Order Service)
order-service:
  datasource:
    url: jdbc:mysql://mysql.local:3306/orders_db
    # MySQL은 트랜잭션은 좋지만, 복잡한 쿼리 성능 떨어짐
  jpa:
    hibernate:
      dialect: org.hibernate.dialect.MySQL8Dialect

# 검색 서비스 (Search Service)
search-service:
  datasource:
    url: jdbc:mysql://mysql.local:3306/search_db
    # MySQL로 전체 상품 매칭, 필터링, 정렬을 구현해야 함
    # 수백만 상품에 대한 복잡한 쿼리 → 매우 느림
  jpa:
    hibernate:
      dialect: org.hibernate.dialect.MySQL8Dialect
```

이 구조의 문제점:

```java
// 검색 서비스가 MySQL에서 복잡한 쿼리를 실행
// 예: "가격 1~10만원, 별점 4점 이상, 배송 2일 이내"인 상품 검색

@Repository
public class ProductSearchRepository extends JpaRepository<Product, Long> {
    // MySQL에서 이런 쿼리는 테이블 스캔 + 정렬 → 매우 느림
    @Query("SELECT p FROM Product p " +
           "WHERE p.price BETWEEN :minPrice AND :maxPrice " +
           "AND p.rating >= :minRating " +
           "AND p.deliveryDays <= :maxDays " +
           "AND p.stock > 0 " +
           "ORDER BY p.rating DESC, p.createdAt DESC " +
           "LIMIT 100")
    List<Product> searchProducts(
        @Param("minPrice") int minPrice,
        @Param("maxPrice") int maxPrice,
        @Param("minRating") double minRating,
        @Param("maxDays") int maxDays
    );
    // 응답 시간: 2~5초 (사용자 체험 매우 나쁨)
}

// 세션 저장소도 MySQL에 저장
@Configuration
public class SessionConfig {
    @Bean
    public LettuceConnectionFactory connectionFactory() {
        // 실제로는 Redis를 사용하지만, 이 예시에서는 MySQL JDBC Session
        // 세션 조회마다 DISK I/O 발생 → 느림
        // TTL 구현이 복잡 (매번 DELETE 쿼리 필요)
        return null;
    }
}

// 분석 서비스도 MySQL에서 로그 데이터 저장
@Service
public class AnalyticsService {
    @Autowired
    private AnalyticsRepository analyticsRepository;
    
    @Async
    public void logUserEvent(UserEvent event) {
        // 매시간 수백만 건의 이벤트 저장 (INSERT 쓰레드 포화)
        analyticsRepository.save(event);  // MySQL INSERT 느림
    }
}

// 결과: 시스템 전체가 느려짐
// - 검색이 느려서 고객 이탈
// - 세션 조회가 느려서 로그인 지연
// - 분석 로그가 쌓여서 DB 공간 가득 참
```

**비용 분석:**
```
장점:
✅ 초기 구축 비용 낮음 (1개 DB 기술만 배우면 됨)
✅ 운영 단순함 (DBA 1명이 모든 걸 관리)

단점:
❌ 각 서비스 성능 저하 (모두 MySQL로 타협)
❌ 스케일링 어려움 (검색 성능 개선 → MySQL 튜닝 → 다른 서비스 영향)
❌ 업스트림 비용 (모든 성능 문제에 인프라 투자 필요)
```

---

## ✨ 올바른 접근 (After — Polyglot Persistence)

```yaml
# application-polyglot.yml
# 각 서비스가 최적의 DB 기술을 사용

# 주문 서비스 (Order Service) - PostgreSQL
# 이유: 트랜잭션 무결성, ACID 보장 중요
order-service:
  datasource:
    url: jdbc:postgresql://postgres.local:5432/orders_db
    username: orders_user
    password: ${ORDERS_DB_PASSWORD}
  jpa:
    hibernate:
      dialect: org.hibernate.dialect.PostgreSQL12Dialect
  properties:
    hibernate:
      jdbc:
        batch_size: 20
        fetch_size: 50

# 검색 서비스 (Search Service) - Elasticsearch
# 이유: 빠른 전문 검색(full-text search), 복잡한 집계 쿼리
search-service:
  elasticsearch:
    rest:
      uris: http://elasticsearch.local:9200
    username: elastic_user
    password: ${ELASTIC_PASSWORD}
  jackson:
    serialization:
      indent_output: false

# 캐시/세션 서비스 (Session Service) - Redis
# 이유: 밀리초 단위 응답, TTL 지원, 인메모리 성능
session-service:
  redis:
    host: redis.local
    port: 6379
    timeout: 2000  # 2초
    jedis:
      pool:
        max_active: 100
        max_idle: 50
        min_idle: 10

# 분석 서비스 (Analytics Service) - MongoDB
# 이유: 대용량 로그 저장, 유연한 스키마, 빠른 쓰기
analytics-service:
  data:
    mongodb:
      uri: mongodb://analytics-db.local:27017/analytics_db
      username: analytics_user
      password: ${ANALYTICS_PASSWORD}
  properties:
    mongdb:
      auto_index_creation: true

# 추천 서비스 (Recommendation Service) - Apache Cassandra
# 이유: 높은 쓰기 처리량, 분산 저장, 시계열 데이터
recommendation-service:
  cassandra:
    contact_points: cassandra-node1.local, cassandra-node2.local
    port: 9042
    keyspace_name: recommendation_ks
    local_datacenter: datacenter1
```

각 서비스의 특성에 맞는 설계:

```java
// 주문 서비스 - PostgreSQL (ACID 트랜잭션)
@Service
@Transactional
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    // 여러 엔티티의 일관성이 보장되어야 함
    public Order createOrder(OrderRequest request) {
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        order.setTotalAmount(request.getTotalAmount());
        
        // PostgreSQL의 ACID 트랜잭션 보장
        Order savedOrder = orderRepository.save(order);
        
        // 즉시 커밋 (강한 일관성 필요)
        request.getLineItems().forEach(item -> {
            OrderLineItem lineItem = new OrderLineItem();
            lineItem.setOrderId(savedOrder.getId());
            lineItem.setProductId(item.getProductId());
            lineItem.setQuantity(item.getQuantity());
            lineItemRepository.save(lineItem);
        });
        
        return savedOrder;
    }
}

// 검색 서비스 - Elasticsearch (빠른 검색)
@Service
public class ProductSearchService {
    
    @Autowired
    private RestHighLevelClient elasticsearchClient;
    
    public SearchResponse searchProducts(SearchQuery query) throws IOException {
        SearchRequest request = new SearchRequest("products");
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        
        // 복잡한 쿼리 (Elasticsearch는 이걸 밀리초 단위로 처리)
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        
        // 가격 범위 필터
        boolQuery.filter(QueryBuilders.rangeQuery("price")
            .gte(query.getMinPrice())
            .lte(query.getMaxPrice()));
        
        // 별점 필터
        boolQuery.filter(QueryBuilders.rangeQuery("rating")
            .gte(query.getMinRating()));
        
        // 텍스트 검색 (전문 검색)
        if (query.getKeyword() != null) {
            boolQuery.must(QueryBuilders.matchQuery("name", query.getKeyword())
                .boost(2.0));  // name 필드에 높은 가중치
        }
        
        sourceBuilder.query(boolQuery);
        sourceBuilder.sort("rating", SortOrder.DESC);
        sourceBuilder.from(query.getOffset());
        sourceBuilder.size(query.getLimit());
        
        request.source(sourceBuilder);
        SearchResponse response = elasticsearchClient.search(request, RequestOptions.DEFAULT);
        
        // 응답 시간: 10~50ms (MySQL에 비해 100배 빠름)
        return convertToSearchResponse(response);
    }
}

// 세션 서비스 - Redis (초고속 조회)
@Service
public class SessionService {
    
    @Autowired
    private RedisTemplate<String, Session> redisTemplate;
    
    public Session getSession(String sessionId) {
        // Redis: 메모리 접근 (~1ms)
        // MySQL: 디스크 접근 (~10ms)
        return redisTemplate.opsForValue().get(sessionId);
    }
    
    public void saveSession(String sessionId, Session session) {
        // Redis의 SETEX: 원자적 SET + TTL 지정
        redisTemplate.opsForValue().set(
            sessionId, 
            session, 
            Duration.ofHours(24)  // 24시간 후 자동 삭제
        );
    }
}

// 분석 서비스 - MongoDB (대용량 로그)
@Service
public class AnalyticsService {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    @Async
    public void logUserEvent(UserEvent event) {
        // MongoDB: 매우 빠른 쓰기 (밀리초)
        // 유연한 스키마 (각 이벤트가 다른 필드 가능)
        mongoTemplate.insert(event);
    }
    
    public List<UserEvent> getEventsByUser(Long userId) {
        Query query = Query.query(Criteria.where("userId").is(userId));
        return mongoTemplate.find(query, UserEvent.class);
    }
}
```

---

## 🔬 내부 동작 원리 — DB 기술 선택 의사결정 프레임워크

### 1. 데이터 특성별 DB 기술 매핑

```
┌─────────────────────────────────────────────────────────────┐
│                   데이터 특성                                │
└─────────────────────────────────────────────────────────────┘

1️⃣ 구조화된 데이터 + 트랜잭션 무결성 중요
   └─ PostgreSQL / MySQL (RDBMS)
   └─ 예: 주문, 결제, 재고

2️⃣ 빠른 검색 + 복잡한 쿼리
   └─ Elasticsearch / Solr (Search Engine)
   └─ 예: 상품 검색, 로그 분석

3️⃣ 빠른 쓰기/읽기 + TTL
   └─ Redis (In-Memory)
   └─ 예: 세션, 캐시, 레이트 리미팅

4️⃣ 대용량 + 유연한 스키마
   └─ MongoDB / Firebase (Document DB)
   └─ 예: 분석 로그, 사용자 정의 메타데이터

5️⃣ 높은 쓰기 처리량 + 시계열
   └─ Cassandra / InfluxDB (Time-Series DB)
   └─ 예: 사용자 활동 로그, 메트릭

6️⃣ 그래프 관계 + 복잡한 순회
   └─ Neo4j (Graph DB)
   └─ 예: 소셜 네트워크, 추천 알고리즘
```

### 2. DB 선택 결정 트리

```
서비스를 위한 DB 기술 선택
    │
    ├─ 구조화된 데이터인가?
    │  │
    │  ├─ YES → 트랜잭션 무결성이 중요한가?
    │  │        │
    │  │        ├─ YES → PostgreSQL / MySQL
    │  │        │
    │  │        └─ NO (읽기 우선) → Polyglot (쓰기는 PG, 읽기는 Elasticsearch)
    │  │
    │  └─ NO → 문서/로그 데이터인가?
    │           │
    │           ├─ YES → 유연한 스키마 필요? 
    │           │        ├─ YES → MongoDB
    │           │        └─ NO → Elasticsearch
    │           │
    │           └─ 시계열 데이터인가?
    │                    ├─ YES → Cassandra / InfluxDB
    │                    └─ NO → Redis (비정형, 빠른 접근)
    │
    └─ 검색 패턴이 특수한가?
       │
       ├─ 전문 검색 필요? → Elasticsearch
       │
       ├─ 빠른 쓰기/읽기 필요? → Redis
       │
       ├─ 관계 순회 필요? → Neo4j
       │
       └─ 기본 조회? → 위에서 선택
```

### 3. 실전 설계: 전자상거래 시스템 예시

```
┌─────────────────────────────────────────────────────────┐
│              전자상거래 시스템 DB 설계                   │
└─────────────────────────────────────────────────────────┘

주문 서비스
├─ PostgreSQL (orders_db)
│  └─ 트랜잭션 무결성 (결제 + 상태 변경)
├─ Redis (Order Cache)
│  └─ 최근 주문 조회 캐싱
└─ Elasticsearch (Order Log)
   └─ 주문 이력 검색 (고급 필터링)

상품 서비스
├─ PostgreSQL (products_db)
│  └─ 상품 마스터 데이터
├─ Elasticsearch (Product Search Index)
│  └─ 상품 검색 (전문 검색, 필터)
└─ Redis (Product Popularity)
   └─ 실시간 인기도 카운팅

사용자 서비스
├─ PostgreSQL (users_db)
│  └─ 사용자 정보
├─ Redis (Session)
│  └─ 로그인 세션 (TTL 24시간)
└─ Elasticsearch (User Activity)
   └─ 사용자 활동 로그

분석 서비스
├─ MongoDB (Analytics)
│  └─ 대용량 이벤트 로그 (유연한 스키마)
├─ Elasticsearch (Analytics Search)
│  └─ 분석 데이터 검색 및 집계
└─ Cassandra (Time-Series)
   └─ 시계별 메트릭 저장

추천 서비스
├─ Neo4j (User-Product Graph)
│  └─ 사용자-상품 상호작용 그래프
└─ Cassandra (Recommendation Cache)
   └─ 생성된 추천 결과 캐싱
```

---

## 💻 Spring Boot 다중 DB 설정

```java
// 주문 서비스: PostgreSQL
@Configuration
@EnableJpaRepositories(
    basePackages = "com.ecommerce.order.repository",
    entityManagerFactoryRef = "orderEntityManager",
    transactionManagerRef = "orderTransactionManager"
)
public class OrderDataSourceConfig {
    
    @Bean(name = "orderDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.order")
    public DataSource orderDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean(name = "orderEntityManager")
    public LocalContainerEntityManagerFactoryBean orderEntityManager(
            @Qualifier("orderDataSource") DataSource dataSource) {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan("com.ecommerce.order.entity");
        em.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
        em.setJpaProperties(jpaProperties());
        return em;
    }
    
    @Bean(name = "orderTransactionManager")
    public PlatformTransactionManager orderTransactionManager(
            @Qualifier("orderEntityManager") LocalContainerEntityManagerFactoryBean entityManager) {
        return new JpaTransactionManager(entityManager.getObject());
    }
    
    private Properties jpaProperties() {
        Properties properties = new Properties();
        properties.setProperty("hibernate.dialect", "org.hibernate.dialect.PostgreSQL12Dialect");
        properties.setProperty("hibernate.jdbc.batch_size", "20");
        properties.setProperty("hibernate.jdbc.fetch_size", "50");
        return properties;
    }
}

// 검색 서비스: Elasticsearch
@Configuration
@EnableElasticsearchRepositories(
    basePackages = "com.ecommerce.search.repository"
)
public class ElasticsearchConfig {
    
    @Bean
    public ClientConfiguration clientConfiguration() {
        return ClientConfiguration.builder()
            .connectedTo("elasticsearch.local:9200")
            .withConnectTimeout(5000)
            .withSocketTimeout(30000)
            .build();
    }
    
    @Bean
    public RestHighLevelClient elasticsearchClient() {
        final ClientConfiguration clientConfiguration = clientConfiguration();
        return RestClients.create(clientConfiguration).rest();
    }
}

// 세션 서비스: Redis
@Configuration
@EnableRedisRepositories
public class RedisSessionConfig {
    
    @Bean
    public LettuceConnectionFactory connectionFactory() {
        return new LettuceConnectionFactory();
    }
    
    @Bean
    public StringRedisTemplate stringRedisTemplate(
            LettuceConnectionFactory connectionFactory) {
        return new StringRedisTemplate(connectionFactory);
    }
    
    @Bean
    public RedisTemplate<String, Session> sessionRedisTemplate(
            LettuceConnectionFactory connectionFactory) {
        RedisTemplate<String, Session> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setDefaultSerializer(new GenericJackson2JsonRedisSerializer());
        template.setKeySerializer(new StringRedisSerializer());
        return template;
    }
}

// 분석 서비스: MongoDB
@Configuration
@EnableMongoRepositories(
    basePackages = "com.ecommerce.analytics.repository"
)
public class MongoDbConfig {
    
    @Bean
    public MongoClient mongoClient() {
        return MongoClients.create("mongodb://analytics-db.local:27017");
    }
    
    @Bean
    public MongoTemplate mongoTemplate(MongoClient mongoClient) {
        return new MongoTemplate(mongoClient, "analytics_db");
    }
}

// application.yml 설정
spring:
  datasource:
    order:
      url: jdbc:postgresql://postgres.local:5432/orders_db
      username: ${ORDERS_DB_USER}
      password: ${ORDERS_DB_PASSWORD}
      hikari:
        maximum-pool-size: 20
  
  elasticsearch:
    rest:
      uris: http://elasticsearch.local:9200
      username: elastic_user
      password: ${ELASTIC_PASSWORD}
  
  redis:
    host: redis.local
    port: 6379
    timeout: 2000ms
    jedis:
      pool:
        max-active: 100
  
  data:
    mongodb:
      uri: mongodb://analytics-db.local:27017/analytics_db
      username: ${MONGODB_USER}
      password: ${MONGODB_PASSWORD}
```

---

## 📊 Polyglot DB 기술 비교

| 기술 | 데이터 모델 | 강점 | 약점 | 사용 사례 |
|------|-----------|------|------|---------|
| **PostgreSQL** | 관계형 | ACID, JOIN, 강력한 쿼리 | 쓰기 성능 | 주문, 결제, 재고 |
| **Elasticsearch** | 문서 + 역인덱스 | 빠른 검색, 전문 검색 | 트랜잭션 없음 | 검색, 로그 분석 |
| **Redis** | Key-Value | 초고속 읽/쓰기, TTL | 데이터 크기 제한 | 세션, 캐시, 레이팅 |
| **MongoDB** | 문서 | 유연 스키마, 빠른 쓰기 | 약한 일관성 | 분석 로그, 메타데이터 |
| **Cassandra** | Wide Column | 높은 쓰기 처리량, 분산 | 복잡한 쿼리 불가 | 시계열, 활동 로그 |
| **Neo4j** | 그래프 | 관계 순회 빠름 | 수평 확장 어려움 | 추천, 소셜 네트워크 |

---

## ⚖️ 트레이드오프

```
기술 다양성의 이점 vs 운영 복잡도
├─ 이점:
│  ├─ ✅ 각 서비스 최적 성능 (검색 10배 빠름, 세션 응답 100배 빠름)
│  ├─ ✅ 스케일링 효율성 (각 기술 특성에 맞게 튜닝)
│  └─ ✅ 기술 선택 자유도
│
└─ 비용:
   ├─ ❌ DBA 전문성 분산 (PostgreSQL + Elasticsearch + Redis 다 알아야 함)
   ├─ ❌ 백업/복구 전략 다양화 (각 DB마다 다른 도구)
   ├─ ❌ 모니터링 복잡도 증가
   ├─ ❌ 라이선스 비용 (각 기술별 상용 버전)
   └─ ❌ 개발자 온보딩 시간 증가

초기 단계: 1~2가지 DB 기술 (PostgreSQL + Redis)
성장 단계: 3~4가지 (+ Elasticsearch)
성숙 단계: 5~6가지 (+ Cassandra, Neo4j)

언제 추가 기술을 도입할 것인가?
├─ 성능 문제 심각 (해결 불가능한 수준)
├─ 운영팀 역량 충분
├─ 기술 지원 인프라 준비 (모니터링, 배포, 복구)
└─ ROI 계산 (기술 도입 비용 < 성능 개선 이점)
```

---

## 📌 핵심 정리

```
✅ Polyglot Persistence란:
  - 각 서비스가 자신의 요구사항에 맞는 DB 기술을 선택
  - Database per Service 원칙의 자연스러운 확장

✅ 서비스별 DB 선택:
  - 주문/결제: PostgreSQL (ACID 트랜잭션)
  - 검색: Elasticsearch (빠른 검색, 복잡한 쿼리)
  - 세션/캐시: Redis (초고속, TTL)
  - 분석 로그: MongoDB (대용량, 유연 스키마)
  - 시계열: Cassandra (높은 쓰기)
  - 그래프: Neo4j (관계 중심)

✅ Polyglot의 이점:
  - 각 서비스 최적 성능 (검색 10배 빠름)
  - 스케일링 효율성
  - 기술 혁신 가능성

✅ Polyglot의 비용:
  - DBA 전문성 분산 필요
  - 운영 복잡도 증가
  - 백업/복구 전략 다양화
  - 개발자 온보딩 시간 증가

✅ 도입 시기:
  - 초기: 1~2가지 기술 (PostgreSQL + Redis)
  - 성능 문제 심각할 때 추가 기술 도입
  - 운영팀 역량 충분할 때 확대
```

---

## 🤔 생각해볼 문제

**Q1.** 당신의 팀이 모든 서비스를 PostgreSQL로 운영하고 있습니다. 상품 검색 기능이 매우 느려서 (응답 시간 2~3초) 고객이 불평입니다. Elasticsearch를 도입하려고 하는데, 기술 선택 외에 고려해야 할 조직 요소는 무엇일까요?

<details>
<summary>해설 보기</summary>

기술만의 문제가 아닙니다. Elasticsearch 도입 전 확인해야 할 것:

1. **운영팀 역량**:
   - Elasticsearch를 관리할 수 있는 DBA가 있는가?
   - 없으면 외부 교육/채용 필요
   - 훈련 기간 고려

2. **인프라**:
   - Elasticsearch 클러스터 구성 (최소 3 노드)
   - 충분한 디스크 공간 (인덱싱 오버헤드)
   - 네트워크 대역폭

3. **운영 프로세스**:
   - 모니터링 (Kibana 또는 다른 도구)
   - 백업/복구 절차
   - 성능 튜닝 가이드
   - 장애 응대 프로세스

4. **데이터 동기화**:
   - PostgreSQL에서 Elasticsearch로 어떻게 데이터 동기화?
   - 초기 색인 (Bulk 작업)
   - 증분 색인 (이벤트 기반 업데이트)
   - 데이터 검증

5. **비용 계산**:
   - 인프라 비용 (클러스터 운영)
   - 인원 비용 (DBA 시간)
   - 라이선스 비용 (상용 버전)
   - 총 비용 vs 성능 개선 효과

결론: **"기술 선택 = 조직 결정"**. Elasticsearch 도입은 기술 문제가 아니라 조직의 준비 수준을 확인하는 과정입니다.

</details>

**Q2.** Redis, Elasticsearch, MongoDB를 모두 운영하는 환경에서 갑자기 Elasticsearch 클러스터가 장애 상황입니다. 상품 검색이 안 됩니다. 어떻게 대응할 것인가?

<details>
<summary>해설 보기</summary>

Elasticsearch 장애 대응 전략:

1. **즉시 대응** (장애 인지 후 5분):
   - 모니터링 알람 확인 (무엇이 깨졌는가?)
   - Elasticsearch 클러스터 상태 확인 (노드 다운? 디스크 풀?)
   - 임시 우회 경로 활성화

2. **임시 우회** (5~30분):
   - 검색 요청을 PostgreSQL로 우회 (느리지만 작동)
   - 또는 Redis 캐시된 검색 결과 제공
   - 사용자에게 명시 ("검색 성능 저하 중입니다")

3. **근본 원인 파악** (30분~):
   - Elasticsearch 로그 분석
   - 클러스터 상태 (주황색/빨강색)
   - 노드 메모리/디스크 상태

4. **복구**:
   - 디스크 가득: 오래된 인덱스 삭제
   - 노드 다운: 노드 재시작
   - 메모리 부족: 힙 크기 증가

5. **검증** (복구 후):
   - 샘플 검색 쿼리 테스트
   - 응답 시간 확인
   - 검색 결과 정확도 검증

**결론**: Polyglot은 **여러 기술의 장애를 동시에 관리**해야 한다는 의미입니다. 각 기술의 장애 시나리오와 복구 계획이 사전에 있어야 합니다. "모두 우회 가능"한 구조 설계가 중요합니다.

</details>

**Q3.** CQRS 패턴과 Polyglot Persistence를 함께 사용한다면, 어떤 구조가 될까요? (쓰기는 PostgreSQL, 읽기는 Elasticsearch, 캐시는 Redis)

<details>
<summary>해설 보기</summary>

CQRS + Polyglot 조합 설계:

```
┌─────────────────────────────────────────────────────┐
│                    API 계층                         │
└─────────────────────────────────────────────────────┘
         │                                    │
         │ (쓰기)                             │ (읽기)
         ▼                                    ▼
    ┌────────────┐                    ┌────────────┐
    │ Command    │                    │ Query      │
    │ Handler    │                    │ Handler    │
    └────────────┘                    └────────────┘
         │                                    │
         │ 1. 데이터 저장                      │ 3. 캐시 확인
         ▼                                    ▼
    ┌────────────────────────────────────────────────┐
    │         PostgreSQL (Write Model)               │
    │  - 주문, 결제 정보 (트랜잭션 일관성)         │
    └────────────────────────────────────────────────┘
         │
         │ 2. Event 발행 (비동기)
         ▼
    ┌────────────────────────────────────────────────┐
    │      Redis Cache (Hot Data)                    │
    │  - 최근 주문 (TTL 24시간)                      │
    │  - 인기 검색어 (TTL 1시간)                     │
    └────────────────────────────────────────────────┘
         │
         │ 3. Event Subscriber (비동기 업데이트)
         ▼
    ┌────────────────────────────────────────────────┐
    │      Elasticsearch (Read Model Index)          │
    │  - 모든 주문 (검색 최적화)                     │
    │  - 전문 검색 인덱스                            │
    └────────────────────────────────────────────────┘

데이터 흐름:
1. 고객이 주문 생성 요청 (POST /orders)
2. CommandHandler가 PostgreSQL에 저장 (ACID 보장)
3. OrderCreatedEvent 발행
4. EventSubscriber1: Elasticsearch에 읽기 모델 인덱싱
5. EventSubscriber2: Redis에 캐시 저장
6. 고객이 주문 조회 (GET /orders?search=...)
7. QueryHandler가 Redis/Elasticsearch에서 조회 (빠름)

이점:
✅ 쓰기: PostgreSQL의 ACID 보장
✅ 읽기: Elasticsearch 빠른 검색 + Redis 초고속 캐시
✅ 확장성: 읽기 모델을 따로 스케일
✅ 성능: 쓰기와 읽기 경로 분리

복잡도:
❌ 최종 일관성 (이벤트 구독 지연)
❌ 이벤트 기반 아키텍처 필요
❌ 데이터 동기화 문제 처리 필요

</details>

---

<div align="center">

**[⬅️ 이전: Database per Service 원칙 — 왜 DB를 분리해야 하는가](./01-database-per-service.md)** | **[홈으로 🏠](../README.md)** | **[다음: Join 없는 데이터 조회 전략 — API Composition, CQRS, 복제 ➡️](./03-join-free-query-strategies.md)**

</div>

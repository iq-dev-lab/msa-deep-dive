# 06. Saga 실패 처리와 모니터링

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- "Dead Saga"는 무엇이고 어떻게 감지할 것인가?
- Saga 진행 상태를 실시간으로 추적할 수 있는 구조는?
- Backward Recovery와 Forward Recovery의 차이는 무엇이고, 각각 언제 쓰는가?
- Saga 실패 시 관리자는 무엇을 확인하고 어떻게 개입해야 하는가?
- 모니터링할 핵심 메트릭은 무엇인가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

Saga 패턴의 가장 큰 장점은 "높은 가용성"이지만, 이와 함께 "부분적 실패" 상황이 빈번해집니다. 한 서비스가 느리거나 장애 중이어도 다른 부분은 계속 실행되므로, **중간 상태에 갇혀 있는 Saga**가 늘어납니다.

예를 들어:
- OrderService는 응답했지만, PaymentService가 계속 오프라인
- Payment 완료 후 InventoryService로 요청 중인데 타임아웃
- Saga 상태를 DB에 저장했는데, 저장 직후 프로세스 장애

이런 상황에서 "지금 어디가 문제인가?", "이 Saga는 결국 어떻게 될 것인가?"를 파악하고 대응하는 것이 **운영 안정성**을 결정합니다. 따라서:

1. **죽은 Saga 자동 감지**: PENDING 상태로 24시간 이상 방치된 Saga 찾기
2. **실시간 모니터링**: 각 단계별 처리 시간, 실패율 추적
3. **수동 개입 도구**: 관리자가 특정 Saga의 상태를 조회하고 조정할 수 있는 API
4. **근본 원인 분석**: 왜 실패했는지 분산 로그에서 추적

이러한 것들이 없으면, 한번 고장난 Saga는 영원히 중간 상태로 남아 데이터 불일치를 유발합니다.

---

## 😱 흔한 실수 (Before)

```java
// ❌ 잘못된 접근 1: Saga 상태 추적이 전혀 없음
// 무엇이 실패했는지 알 수 없음

@Service
public class OrderSagaWithoutMonitoring {
    
    public void createOrder(OrderRequest request) {
        try {
            Order order = orderService.createOrder(request);
            Payment payment = paymentService.pay(order.getId(), request.getAmount());
            Inventory inventory = inventoryService.reserve(request.getProductId());
            Shipment shipment = shippingService.createShipment(order.getId());
            
        } catch (Exception e) {
            // ❌ 단순히 예외만 처리
            log.error("Order creation failed", e);
            // 어디서 실패했는지 알 수 없음
            // 이미 완료된 단계가 무엇인지 알 수 없음
            // 보상을 어디서부터 시작해야 하는지 알 수 없음
        }
    }
}

/*
❌ 결과:
- 고객: "내 주문이 어디까지 진행됐나?" → 전혀 알 수 없음
- 개발팀: "에러 로그에 뭐가 있나?" → 전체 스택 트레이스만 있음
- 운영팀: "이 고객 이슈를 어떻게 해결할 건가?" → 수동으로 DB 조회
- 결과: Saga가 부분 완료 상태로 영구 방치
*/
```

```java
// ❌ 잘못된 접근 2: Saga 로그는 있지만, 모니터링/복구 전략 없음

@Entity
@Table(name = "saga_logs")
public class SagaLog {
    @Id
    @GeneratedValue
    private Long id;
    
    private String sagaId;
    private String step;
    private String status;
    private LocalDateTime timestamp;
    
    // 문제: 이 테이블에 기록만 하고, 실패한 Saga를 찾는 쿼리 없음
    //      "지금 며칠 동안 PENDING 상태인 Saga는?" 조회 불가능
    //      "이 고객의 모든 Saga는?" 조회 불가능
}

@Service
public class SagaServiceWithoutRecovery {
    
    public void executeOrderSaga(String sagaId) {
        try {
            // Step 1, 2, 3 실행...
            // 각 단계마다 sagaLogRepository.save(log)
            
        } catch (Exception e) {
            // ❌ 실패 후 어떻게 할 것인가?
            // - 자동 재시도? → 타이밍은? 몇 회?
            // - 수동 개입? → 누가? 언제?
            // - 그냥 무시? → 영구적 불일치?
        }
    }
    
    // ❌ 이런 쿼리가 없음:
    // - "어제 이후로 PENDING 상태인 Saga들"
    // - "고객 123의 모든 Saga 조회"
    // - "결제 실패 Saga만 조회"
}
```

```yaml
# ❌ 잘못된 설정: 모니터링 설정이 없음

spring:
  jpa:
    hibernate:
      ddl-auto: validate

# 문제: 
# - 로그 수집 설정 없음 (ELK, Splunk 등)
# - 메트릭 수집 설정 없음 (Prometheus, Grafana 등)
# - 알림 설정 없음 (Slack, PagerDuty 등)
# - Saga 상태별 집계 쿼리 인덱스 없음
# → 운영팀이 문제를 발견했을 때 이미 손쓸 수 없는 상황
```

---

## ✨ 올바른 접근 (After)

```java
/**
 * Saga 모니터링 및 복구 전략
 * 
 * 1. SagaLog 테이블: 모든 단계 기록
 * 2. DeadSagaDetector: 주기적으로 미완료 Saga 감지
 * 3. SagaRecoveryService: 자동 또는 수동 복구
 * 4. SagaMonitoringDashboard: 실시간 현황 조회
 */

// Step 1: 상세한 Saga 로깅
@Entity
@Table(name = "saga_logs", indexes = {
    @Index(name = "idx_saga_id", columnList = "sagaId"),
    @Index(name = "idx_status", columnList = "status"),
    @Index(name = "idx_timestamp", columnList = "timestamp"),
    @Index(name = "idx_customer_id", columnList = "customerId")
})
public class SagaLog {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String sagaId;
    private Long customerId;
    private Long orderId;
    
    private String step;  // ORDER_CREATED, PAYMENT_COMPLETED, etc.
    private String status;  // SUCCESS, FAILED, PENDING, COMPENSATED
    
    private String errorMessage;
    private String errorCode;
    
    private LocalDateTime timestamp;
    private LocalDateTime updatedAt;
    
    private Long durationMs;  // 이 단계 처리 시간
    
    private String metadata;  // JSON: 추가 정보
}

// Step 2: SagaLog 쿼리 저장소
@Repository
public interface SagaLogRepository extends JpaRepository<SagaLog, Long> {
    
    // 특정 Saga의 모든 로그 (시간순)
    List<SagaLog> findBySagaIdOrderByTimestampAsc(String sagaId);
    
    // 고객의 모든 Saga 조회
    List<SagaLog> findByCustomerIdOrderByTimestampDesc(Long customerId);
    
    // PENDING 상태인 Saga들 (오래된 것 먼저)
    List<SagaLog> findByStatusOrderByTimestampAsc(String status);
    
    // 최근 N분 동안의 실패한 Saga들
    List<SagaLog> findByStatusAndTimestampAfter(
        String status,
        LocalDateTime timestamp
    );
    
    // 특정 단계에서 실패한 Saga들
    List<SagaLog> findByStepAndStatus(String step, String status);
    
    // 평균 처리 시간 (메트릭)
    @Query("SELECT AVG(sl.durationMs) FROM SagaLog sl WHERE sl.step = :step AND sl.status = 'SUCCESS'")
    Double getAverageDurationForStep(@Param("step") String step);
    
    // 실패율 (메트릭)
    @Query("SELECT COUNT(*) FROM SagaLog sl WHERE sl.step = :step AND sl.status = 'FAILED'")
    Long getFailureCountForStep(@Param("step") String step);
}

/**
 * Step 3: Dead Saga Detector (스케줄러)
 * 일정 시간 이상 미완료된 Saga를 감지하고 복구 시도
 */
@Service
@Slf4j
public class DeadSagaDetector {
    
    @Autowired
    private SagaRepository sagaRepository;
    
    @Autowired
    private SagaRecoveryService recoveryService;
    
    @Autowired
    private AlertingService alertingService;
    
    /**
     * 매 10분마다 실행
     * "30분 이상 PENDING 상태인 Saga" 감지
     */
    @Scheduled(fixedRate = 600000)  // 10분
    public void detectDeadSagas() {
        LocalDateTime threshold = LocalDateTime.now().minusMinutes(30);
        
        List<OrderSaga> deadSagas = sagaRepository.findByStateAndUpdatedAtBefore(
            SagaState.PENDING,
            threshold
        );
        
        if (!deadSagas.isEmpty()) {
            log.warn("Detected {} dead sagas", deadSagas.size());
            
            for (OrderSaga saga : deadSagas) {
                try {
                    log.info("Attempting to recover saga: {}", saga.getId());
                    
                    // 복구 시도: 마지막 성공 지점부터 재개
                    recoveryService.recoverFromLastSuccessfulStep(saga);
                    
                } catch (Exception e) {
                    log.error("Recovery failed for saga: {}", saga.getId(), e);
                    
                    // 자동 복구 실패 → 관리자에게 알림
                    alertingService.sendAlert(AlertLevel.CRITICAL,
                        String.format("Dead saga recovery failed: %s", saga.getId()),
                        saga
                    );
                }
            }
        }
    }
    
    /**
     * 매 시간마다 실행
     * Saga 상태별 통계 수집
     */
    @Scheduled(fixedRate = 3600000)  // 1시간
    public void collectSagaMetrics() {
        Map<SagaState, Long> stateCount = sagaRepository
            .groupByState();
        
        Map<String, Double> stepMetrics = new HashMap<>();
        List<String> allSteps = sagaLogRepository.findDistinctSteps();
        
        for (String step : allSteps) {
            // 각 단계의 평균 처리 시간, 실패율
            Double avgDuration = sagaLogRepository.getAverageDurationForStep(step);
            Long failureCount = sagaLogRepository.getFailureCountForStep(step);
            
            stepMetrics.put(step + ".avgDurationMs", avgDuration);
            stepMetrics.put(step + ".failureCount", failureCount.doubleValue());
        }
        
        // Prometheus 메트릭으로 수집
        metricsRegistry.recordSagaMetrics(stateCount, stepMetrics);
    }
}

/**
 * Step 4: Saga 복구 전략
 */
@Service
@Slf4j
public class SagaRecoveryService {
    
    @Autowired
    private OrderSagaOrchestrator orchestrator;
    
    @Autowired
    private SagaLogRepository sagaLogRepository;
    
    /**
     * Backward Recovery: 마지막 성공 지점부터 재개
     * 가장 안전한 방법 (하지만 느림)
     */
    public void recoverFromLastSuccessfulStep(OrderSaga saga) {
        // Step 1: 마지막 성공한 단계 조회
        List<SagaLog> logs = sagaLogRepository.findBySagaIdOrderByTimestampAsc(saga.getId());
        
        SagaLog lastSuccess = logs.stream()
            .filter(log -> "SUCCESS".equals(log.getStatus()))
            .reduce((a, b) -> b)  // 마지막 것
            .orElseThrow(() -> new SagaRecoveryException("No successful step found"));
        
        log.info("[Saga:{}] Last successful step: {}", saga.getId(), lastSuccess.getStep());
        
        // Step 2: 현재 상태를 마지막 성공 상태로 설정
        SagaState targetState = mapStepToState(lastSuccess.getStep());
        saga.setState(targetState);
        sagaRepository.save(saga);
        
        // Step 3: Saga 재개 (다음 단계 실행)
        orchestrator.executeSaga(saga.getId());
    }
    
    /**
     * Forward Recovery: 다음 단계로 강제 진행
     * 높은 위험성 (데이터 불일치 가능)
     * 용도: 이미 완료된 것으로 확인된 경우에만
     */
    public void forceProgressToNextStep(OrderSaga saga, SagaState nextState) {
        log.warn("[Saga:{}] Forcing progress to state: {}", saga.getId(), nextState);
        
        saga.setState(nextState);
        sagaRepository.save(saga);
        
        orchestrator.executeSaga(saga.getId());
    }
    
    /**
     * 수동 개입: 관리자가 특정 Saga를 특정 상태로 설정
     */
    @Transactional
    public void manuallySetSagaState(String sagaId, SagaState newState, String reason) {
        OrderSaga saga = sagaRepository.findById(sagaId)
            .orElseThrow();
        
        log.warn("[Saga:{}] Manually set to state: {} (Reason: {})",
            sagaId, newState, reason);
        
        saga.setState(newState);
        saga.setManualInterventionReason(reason);
        saga.setManualInterventionAt(LocalDateTime.now());
        sagaRepository.save(saga);
        
        // 감시 기록
        auditLog.record(AuditAction.SAGA_MANUAL_INTERVENTION,
            Map.of(
                "sagaId", sagaId,
                "newState", newState.toString(),
                "reason", reason
            )
        );
    }
}

/**
 * Step 5: Saga 모니터링 대시보드 API
 */
@RestController
@RequestMapping("/api/sagas")
public class SagaMonitoringController {
    
    @Autowired
    private SagaRepository sagaRepository;
    
    @Autowired
    private SagaLogRepository sagaLogRepository;
    
    /**
     * 특정 Saga의 상세 정보 조회
     */
    @GetMapping("/{sagaId}")
    public ResponseEntity<SagaDetailResponse> getSagaDetail(@PathVariable String sagaId) {
        OrderSaga saga = sagaRepository.findById(sagaId)
            .orElseThrow(() -> new EntityNotFoundException("Saga not found"));
        
        List<SagaLog> logs = sagaLogRepository.findBySagaIdOrderByTimestampAsc(sagaId);
        
        return ResponseEntity.ok(new SagaDetailResponse(saga, logs));
    }
    
    /**
     * Dead Saga 조회
     */
    @GetMapping("/dead")
    public ResponseEntity<List<SagaDetailResponse>> listDeadSagas(
        @RequestParam(defaultValue = "30") int minutesThreshold
    ) {
        LocalDateTime threshold = LocalDateTime.now().minusMinutes(minutesThreshold);
        List<OrderSaga> deadSagas = sagaRepository.findByStateAndUpdatedAtBefore(
            SagaState.PENDING,
            threshold
        );
        
        return ResponseEntity.ok(
            deadSagas.stream()
                .map(saga -> new SagaDetailResponse(
                    saga,
                    sagaLogRepository.findBySagaIdOrderByTimestampAsc(saga.getId())
                ))
                .collect(Collectors.toList())
        );
    }
    
    /**
     * 고객의 모든 Saga 조회
     */
    @GetMapping("/customer/{customerId}")
    public ResponseEntity<List<SagaDetailResponse>> getCustomerSagas(
        @PathVariable Long customerId
    ) {
        List<SagaLog> logs = sagaLogRepository.findByCustomerIdOrderByTimestampDesc(customerId);
        List<String> sagaIds = logs.stream()
            .map(SagaLog::getSagaId)
            .distinct()
            .collect(Collectors.toList());
        
        List<OrderSaga> sagas = sagaRepository.findAllById(sagaIds);
        
        return ResponseEntity.ok(
            sagas.stream()
                .map(saga -> new SagaDetailResponse(
                    saga,
                    sagaLogRepository.findBySagaIdOrderByTimestampAsc(saga.getId())
                ))
                .collect(Collectors.toList())
        );
    }
    
    /**
     * Saga 상태별 통계
     */
    @GetMapping("/metrics")
    public ResponseEntity<SagaMetricsResponse> getMetrics() {
        Map<SagaState, Long> stateCounts = sagaRepository.groupByState();
        
        List<String> steps = new ArrayList<>();  // ORDER_CREATED, PAYMENT_COMPLETED, ...
        Map<String, StepMetrics> stepMetrics = new HashMap<>();
        
        for (String step : steps) {
            double avgDuration = sagaLogRepository.getAverageDurationForStep(step);
            long failureCount = sagaLogRepository.getFailureCountForStep(step);
            
            stepMetrics.put(step, new StepMetrics(avgDuration, failureCount));
        }
        
        return ResponseEntity.ok(new SagaMetricsResponse(stateCounts, stepMetrics));
    }
    
    /**
     * Saga 수동 개입 (상태 변경)
     */
    @PostMapping("/{sagaId}/recover")
    public ResponseEntity<Void> recoverSaga(
        @PathVariable String sagaId,
        @RequestBody RecoveryRequest request
    ) {
        // 관리자 권한 확인 (생략)
        
        recoveryService.manuallySetSagaState(
            sagaId,
            request.getTargetState(),
            request.getReason()
        );
        
        return ResponseEntity.ok().build();
    }
}

// 응답 DTO
@Data
public class SagaDetailResponse {
    private String sagaId;
    private SagaState state;
    private Long orderId;
    private Long customerId;
    private List<SagaLogEntry> logs;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    public SagaDetailResponse(OrderSaga saga, List<SagaLog> logs) {
        this.sagaId = saga.getId();
        this.state = saga.getState();
        this.orderId = saga.getOrderId();
        this.customerId = saga.getCustomerId();
        this.createdAt = saga.getCreatedAt();
        this.updatedAt = saga.getUpdatedAt();
        this.logs = logs.stream()
            .map(log -> new SagaLogEntry(
                log.getStep(),
                log.getStatus(),
                log.getErrorMessage(),
                log.getDurationMs(),
                log.getTimestamp()
            ))
            .collect(Collectors.toList());
    }
}

@Data
public class SagaMetricsResponse {
    private Map<SagaState, Long> stateCounts;
    private Map<String, StepMetrics> stepMetrics;
    
    @Data
    public static class StepMetrics {
        private double averageDurationMs;
        private long failureCount;
    }
}
```

```yaml
# ✅ 올바른 모니터링 설정

spring:
  jpa:
    hibernate:
      ddl-auto: validate

# 로그 수집 (ELK Stack)
logging:
  level:
    com.example.saga: DEBUG
  pattern:
    console: "[%d{ISO8601}] %thread %level %logger{36} - %msg%n"
    file: "[%d{ISO8601}] %thread %level %logger{36} - %msg%n"
  file:
    name: logs/saga.log
    max-size: 100MB
    max-history: 30

# 메트릭 수집 (Prometheus)
management:
  metrics:
    export:
      prometheus:
        enabled: true
  endpoints:
    web:
      exposure:
        include: prometheus,health,metrics

# Saga 모니터링
app:
  saga:
    monitoring:
      # Dead Saga 감지 설정
      dead-saga-detection-interval-minutes: 10
      pending-threshold-minutes: 30  # 30분 이상 PENDING이면 dead
      
      # 자동 복구
      auto-recovery-enabled: true
      max-recovery-attempts: 3
      
      # 알림 설정
      alerting:
        enabled: true
        critical-threshold: 10  # 10개 이상 dead saga 발생 시 알림
        slack-webhook-url: ${SLACK_WEBHOOK_URL}
        pagerduty-enabled: false  # 운영 환경에서는 true
```

---

## 🔬 내부 동작 원리 — Saga 생명 주기 추적

### Saga 상태 흐름과 모니터링 지점

```
┌──────────────────────────────────────────────────────────────┐
│              Saga 생명 주기와 모니터링 지점                  │
└──────────────────────────────────────────────────────────────┘

[사용자 요청] → startOrderSaga()
    │
    ▼
  ┌─────────────┐
  │   PENDING   │  ← 모니터링: 1단계 대기 시간
  └──────┬──────┘
         │
         ▼
  ┌─────────────────┐
  │ ORDER_CREATED   │  ← 모니터링: 주문 생성 성공
  └──────┬──────────┘
         │
         ▼
  ┌──────────────────────┐
  │ PAYMENT_PROCESSING   │  ← 모니터링: 결제 중 (장시간 대기?)
  └──────┬───────────────┘
         │
    ┌────┴──────────────┬─────────────────────┐
    │ (정상)            │ (에러 발생)         │
    ▼                   ▼                      ▼
  PAYMENT_COMPLETED  ERROR: Timeout      COMPENSATION_STARTED
                     (→ 재시도?)         (→ 보상 시작)
  
  [정상 경로]                        [실패 경로]
    │                                  │
    ▼                                  ▼
  INVENTORY_RESERVED            PAYMENT_REFUNDED
    │                                  │
    ▼                                  ▼
  SHIPMENT_CREATED              ORDER_CANCELLED
    │                                  │
    ▼                                  ▼
  SAGA_COMPLETED                SAGA_FAILED
  (성공)                         (최종 실패)
  
  ↑                               ↑
  모니터링: 성공                   모니터링: 실패율, 원인

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

모니터링할 지점:
1. PENDING 상태 지속 시간
   → 30분 이상이면 "Dead Saga"로 표시
   
2. 각 단계별 처리 시간
   → PAYMENT_PROCESSING이 5분 이상 → 경고
   
3. 상태 전이 성공률
   → PAYMENT_PROCESSING → PAYMENT_COMPLETED 성공률 < 99% → 경고
   
4. 최종 상태 분포
   → SAGA_COMPLETED 비율이 낮음 → 시스템 문제
```

### Dead Saga 감지 타임라인

```
┌──────────────────────────────────────────────────────────┐
│              Dead Saga 감지 및 복구 절차                 │
└──────────────────────────────────────────────────────────┘

T0: 사용자가 주문 요청
    Saga 생성 (ID: saga-12345)
    State = PENDING
    createdAt = 2024-01-10 14:00:00

T1: OrderService 성공
    State = ORDER_CREATED
    updatedAt = 2024-01-10 14:00:05

T2: PaymentService 호출 → 타임아웃
    State = PENDING (변경 안 됨)
    updatedAt = 2024-01-10 14:00:05 (변경 안 됨)
    ← 여기서 멈춤!

T3-T30: 아무도 saga-12345를 만지지 않음

T31: DeadSagaDetector 실행 (T=2024-01-10 14:31:00)
    ├─ PENDING 상태이고
    ├─ updatedAt = 2024-01-10 14:00:05 (30분 전)
    └─ → "Dead Saga!"로 분류
    
    자동 복구 시작:
    └─ 마지막 성공: ORDER_CREATED
       → State = ORDER_CREATED로 설정
       → executeSaga() 호출
       → PaymentService 재시도

T32: PaymentService 재시도 성공 (이번엔 서버가 복구됨)
    State = PAYMENT_COMPLETED
    updatedAt = 2024-01-10 14:32:00

T33-T35: INVENTORY_RESERVED, SHIPMENT_CREATED 순차 완료

T36: Saga 최종 완료
    State = SAGA_COMPLETED
    updatedAt = 2024-01-10 14:36:00
    
    ← 자동 복구 성공! (관리자 개입 없음)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

만약 T32에서 PaymentService가 여전히 오프라인이었다면:
    → 재시도 실패 → 관리자 알림
       ├─ Slack 메시지: "saga-12345 recovery failed"
       ├─ 관리자: 대시보드에서 saga-12345 조회
       ├─ 관리자: "PaymentService 상태 확인"
       ├─ 관리자: PaymentService 복구 후 "Saga 재시도" 버튼 클릭
       ├─ Saga 재시도 → 성공
       └─ 문제 해결
```

---

## 💻 모니터링 대시보드 쿼리 예제

```sql
-- 1. Dead Saga 조회 (30분 이상 PENDING)
SELECT 
    s.id as saga_id,
    s.order_id,
    s.state,
    s.updated_at,
    DATEDIFF(MINUTE, s.updated_at, GETDATE()) as pending_minutes,
    (SELECT MAX(timestamp) FROM saga_logs WHERE saga_id = s.id AND status = 'SUCCESS') as last_success
FROM order_sagas s
WHERE s.state = 'PENDING'
  AND s.updated_at < DATEADD(MINUTE, -30, GETDATE())
ORDER BY s.updated_at ASC;

-- 2. 각 단계별 성공률 및 평균 처리 시간
SELECT 
    step,
    COUNT(*) as total,
    SUM(CASE WHEN status = 'SUCCESS' THEN 1 ELSE 0 END) as success_count,
    ROUND(100.0 * SUM(CASE WHEN status = 'SUCCESS' THEN 1 ELSE 0 END) / COUNT(*), 2) as success_rate,
    AVG(CASE WHEN status = 'SUCCESS' THEN duration_ms END) as avg_duration_ms,
    MAX(CASE WHEN status = 'FAILED' THEN error_message END) as latest_error
FROM saga_logs
WHERE timestamp > DATEADD(HOUR, -24, GETDATE())
GROUP BY step
ORDER BY success_rate ASC;

-- 3. 고객별 Saga 상태
SELECT 
    sl.customer_id,
    COUNT(DISTINCT sl.saga_id) as total_sagas,
    SUM(CASE WHEN os.state = 'SAGA_COMPLETED' THEN 1 ELSE 0 END) as completed,
    SUM(CASE WHEN os.state = 'PENDING' THEN 1 ELSE 0 END) as pending,
    SUM(CASE WHEN os.state = 'SAGA_FAILED' THEN 1 ELSE 0 END) as failed
FROM saga_logs sl
JOIN order_sagas os ON sl.saga_id = os.id
WHERE sl.customer_id = 12345
GROUP BY sl.customer_id;

-- 4. 실패 원인 분석 (최근 24시간)
SELECT 
    error_code,
    error_message,
    COUNT(*) as count,
    ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM saga_logs WHERE status = 'FAILED' AND timestamp > DATEADD(HOUR, -24, GETDATE())), 2) as percentage
FROM saga_logs
WHERE status = 'FAILED'
  AND timestamp > DATEADD(HOUR, -24, GETDATE())
GROUP BY error_code, error_message
ORDER BY count DESC;

-- 5. 단계별 병목 분석 (평균 처리 시간이 긴 단계)
SELECT TOP 10
    step,
    COUNT(*) as count,
    AVG(duration_ms) as avg_duration_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) as p95_duration_ms,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY duration_ms) as p99_duration_ms
FROM saga_logs
WHERE status = 'SUCCESS'
  AND timestamp > DATEADD(HOUR, -24, GETDATE())
GROUP BY step
ORDER BY avg_duration_ms DESC;
```

---

## 📊 모니터링 메트릭

| 메트릭 | 계산 방식 | 경고 임계값 | 액션 |
|------|------|------|------|
| **Saga 완료율** | SAGA_COMPLETED / (SAGA_COMPLETED + SAGA_FAILED) | < 95% | 시스템 정상성 확인 |
| **Dead Saga 수** | PENDING 상태 + updatedAt > 30분 | > 10개 | 관리자 알림, 자동 복구 |
| **평균 처리 시간** | 각 단계별 duration_ms의 평균 | 단계마다 다름 | 병목 분석, 성능 개선 |
| **에러율** | FAILED / 전체 Saga | > 5% | 근본 원인 분석 |
| **보상 성공률** | COMPENSATION_COMPLETED / (COMPENSATION_STARTED) | < 99% | 보상 로직 버그 의심 |
| **타임아웃 비율** | TimeoutException 발생 건수 / 전체 | > 2% | 네트워크/서비스 성능 이슈 |

---

## ⚖️ 트레이드오프

### 상세 로깅의 비용
- ✅ 각 단계마다 로그 저장 → 디버깅 용이
- ✅ 고객 ID, Order ID 등 메타데이터 → 추적 가능
- ✅ Error Code, Message → 원인 분석 용이
- ❌ 로그 저장소 용량 급증 (매달 수 GB)
- ❌ 쿼리 성능 저하 (대량의 로그)
- ❌ 저장소 비용 증가

### Dead Saga 검출의 트레이드오프
- ✅ 주기적 감지 → 자동 복구 가능
- ✅ 관리자 개입 최소화
- ❌ 검사 주기가 짧으면 (1분) → CPU 낭비
- ❌ 검사 주기가 길면 (1시간) → 복구 지연
- ❌ 거짓 양성 (실제로 진행 중인데 Dead로 판단)

### 자동 복구 vs 수동 개입
- ✅ **자동 복구**: 빠름, 관리자 부담 없음, 대부분의 경우 작동
- ❌ **자동 복구**: 데이터 불일치 위험, 예상치 못한 상황 발생 가능
- ✅ **수동 개입**: 안전함, 관리자가 상황을 명확히 파악 후 처리
- ❌ **수동 개입**: 느림 (관리자의 응답 시간), SLA 위반 가능

---

## 📌 핵심 정리

✅ **SagaLog 테이블**은 모든 단계를 기록합니다. 고객 ID, Order ID, 처리 시간 등으로 이후 분석을 가능하게 합니다.

✅ **Dead Saga Detector**는 일정 시간 이상 미완료된 Saga를 감지하고, 자동 복구를 시도합니다.

✅ **Backward Recovery**는 마지막 성공한 단계부터 재개하는 안전한 복구 방법입니다.

✅ **Forward Recovery**는 다음 단계로 강제 진행하는 위험한 방법입니다. 이미 완료된 것으로 확인된 경우에만 사용합니다.

✅ **수동 개입 API**를 통해 관리자가 특정 Saga의 상태를 조회하고 조정할 수 있습니다.

✅ **핵심 메트릭**: Saga 완료율, Dead Saga 수, 단계별 처리 시간, 에러율, 보상 성공률

---

## 🤔 생각해볼 문제

**Q1.** Dead Saga를 자동 복구할 때, 만약 PaymentService가 여전히 오프라인이라면? 무한 재시도 루프가 발생하지 않을까?

<details><summary>해설 보기</summary>
**문제**: 
- T1: DeadSagaDetector 실행 → 자동 복구 시도 → PaymentService 호출 → 실패
- T2 (10분 후): DeadSagaDetector 다시 실행 → 또 복구 시도 → 또 실패
- T3-T∞: 계속 반복...

**해결 방법**:

**1. 재시도 횟수 제한**
```java
OrderSaga saga = sagaRepository.findById(sagaId);

if (saga.getAutoRecoveryAttempts() >= 3) {
    log.warn("Max recovery attempts reached, need manual intervention");
    alertingService.sendAlert("MAX_RECOVERY_ATTEMPTS", sagaId);
    return;  // 더 이상 자동 복구 안 함
}

saga.incrementAutoRecoveryAttempts();
try {
    recoveryService.recover(saga);
} catch (Exception e) {
    sagaRepository.save(saga);
    // 다음 주기에 다시 시도 (최대 3회)
}
```

**2. 재시도 주기 확대 (exponential backoff)**
```
1차 실패 → 10분 후 재시도
2차 실패 → 30분 후 재시도
3차 실패 → 1시간 후 재시도
3회 이상 실패 → 관리자 개입
```

**3. 근본 원인 확인**
```java
// PaymentService가 정말 오프라인인지 확인
HealthStatus paymentHealth = healthCheckService.check("PaymentService");
if (!paymentHealth.isUp()) {
    // 아직 복구 중 → 자동 복구 스킵
    log.info("PaymentService is down, skipping auto recovery");
    return;
}

// PaymentService는 살아있는데 응답 느림?
if (paymentHealth.getResponseTime() > 30000) {
    // 타임아웃 설정 늘리기
    log.info("PaymentService slow, adjusting timeout");
}
```

**권장**: **3가지 조합**이 가장 실용적입니다.
- 최대 3회 자동 복구
- 각 재시도마다 시간 확대
- 3회 이상 실패하면 관리자 알림
</details>

**Q2.** Saga 로그가 매일 수 GB씩 쌓인다면 어떻게 할까? 저장소 비용이 너무 높아진다.

<details><summary>해설 보기</summary>
**문제**: 
- 일일 주문량: 100,000건
- 각 Saga당 로그: 5단계 × 4 필드 = 20개 이상 레코드
- 일일 로그: 2,000,000개 레코드 × 1KB = 2GB
- 월간: 60GB, 연간: 720GB (상당한 비용)

**해결 전략**:

**1. 로그 수준 차별화**
```java
// 주요 이벤트만 저장 (상세 로깅 아님)
if (isSignificantEvent(sagaLog)) {
    sagaLogRepository.save(sagaLog);  // 성공/실패의 최종 상태만
}

// 에러 발생 시에만 상세 로그
if (status == FAILED) {
    detailedLoggingService.save(
        sagaId,
        step,
        errorMessage,
        stackTrace  // 상세 정보
    );
}
```

**2. 로그 분리 저장**
```
- Hot Storage (고속, 비쌈): 최근 7일
- Warm Storage (중간): 8일~90일
- Cold Storage (저렴): 91일~2년

// 자동 이동
7일 지난 로그 → S3 Glacier (월 $1/100GB 정도)
```

**3. 로그 압축 및 집계**
```sql
-- 원본 로그
saga_logs (2,000,000건/일)

-- 집계 로그 (daily snapshot)
saga_daily_summary (100,000건/일)
- saga_id, state, final_status, total_duration_ms, error_code
- 이것만으로도 모니터링 충분

-- 기간별 조회가 필요할 때만 원본 로그 조회
-- (최근 7일만 유지)
```

**4. 로그 샘플링**
```java
// 성공한 경우만 10% 샘플링
if (status == SUCCESS && random() > 0.9) {
    return;  // 로그 안 저장
}

// 실패는 100% 저장
if (status == FAILED) {
    sagaLogRepository.save(sagaLog);
}
```

**권장**: **조합 접근**
```
1단계: 핵심만 저장 (최종 상태, 에러만)
    → 로그량 50% 감소

2단계: Hot/Warm/Cold Storage 분리
    → 월 비용 70% 절감 (장기 보관 필요시)

3단계: 일일 집계 테이블 생성
    → 대부분의 모니터링 쿼리 집계 테이블 사용
    → 원본 로그 조회 최소화
```

이 조합으로 연간 비용을 720GB → 100GB 수준으로 줄일 수 있습니다.
</details>

**Q3.** 관리자가 "Saga 상태를 SAGA_COMPLETED로 강제 설정"했는데, 실제로는 결제가 안 되었다면?

<details><summary>해설 보기</summary>
**위험한 상황**:
```
관리자: "Saga가 PENDING이니까 SAGA_COMPLETED로 설정해야지"
↓
결과: 
- Order: COMPLETED로 표시됨 (고객은 주문 완료로 봄)
- Payment: 실제로는 처리 안 됨
- Inventory: 재고는 감소됨
- 데이터 불일치 발생!
```

**해결 방법**:

**1. 관리자 개입 시 검증**
```java
@PostMapping("/{sagaId}/recover")
public ResponseEntity<Void> recoverSaga(
    @PathVariable String sagaId,
    @RequestBody RecoveryRequest request
) {
    OrderSaga saga = sagaRepository.findById(sagaId).orElseThrow();
    
    // 검증: 정말로 이 상태로 전환 가능한가?
    if (!isValidStateTransition(saga.getState(), request.getTargetState())) {
        throw new InvalidStateTransitionException(
            "Cannot transition from " + saga.getState() 
            + " to " + request.getTargetState()
        );
    }
    
    // 필수 정보 확인: "결제가 정말 되었나?"
    if (request.getTargetState() == SAGA_COMPLETED) {
        // 결제 확인
        Payment payment = paymentRepository.findByOrderId(saga.getOrderId());
        if (payment == null || !payment.isCompleted()) {
            throw new PaymentNotCompletedException(
                "Cannot complete saga without successful payment"
            );
        }
        
        // 재고 확인
        Inventory inventory = inventoryRepository.findByOrderId(saga.getOrderId());
        if (inventory == null || inventory.isAvailable() < 0) {
            throw new InventoryNotReservedException(
                "Inventory not properly reserved"
            );
        }
    }
    
    // 모든 검증 통과 → 상태 전환
    saga.setState(request.getTargetState());
    saga.setManualInterventionReason(request.getReason());
    sagaRepository.save(saga);
}
```

**2. 감시 로그 (Audit Trail)**
```java
auditLog.record(
    action = AuditAction.SAGA_STATE_CHANGE,
    actor = getCurrentUser(),  // 누가?
    sagaId = sagaId,
    fromState = saga.getState(),
    toState = request.getTargetState(),
    reason = request.getReason(),
    timestamp = now(),
    ipAddress = request.getIpAddress()
);
// 나중에 "누가 이걸 했나?" 조회 가능
```

**3. 제한된 권한**
```
- 일반 관리자: COMPENSATION_FAILED → 자동 복구 재시도만 가능
- 시니어 관리자: 상태 조회, 재시도
- 시스템 운영팀장: 상태 강제 변경 (최후의 수단)
```

**4. 사후 검증**
```java
@Scheduled(fixedRate = 3600000)  // 1시간마다
public void validateManualInterventions() {
    List<OrderSaga> manuallyChanged = sagaRepository
        .findByManualInterventionAtAfter(LocalDateTime.now().minusHours(24));
    
    for (OrderSaga saga : manuallyChanged) {
        // 상태가 실제와 일치하는가?
        boolean isConsistent = validateSagaConsistency(saga);
        
        if (!isConsistent) {
            log.error("Data inconsistency detected in saga: {}", saga.getId());
            alertingService.sendAlert(
                AlertLevel.CRITICAL,
                "Manual intervention caused data inconsistency",
                saga
            );
        }
    }
}
```

**권장**: **강제 변경 전에 자동 검증**, **감시 로그 기록**, **정기적 일관성 확인**
이 3가지를 모두 구현하면 관리자의 실수를 방지할 수 있습니다.
</details>

---

<div align="center">

**[⬅️ 이전: 보상 트랜잭션 설계 — 멱등성과 취소 불가 단계](./05-compensating-transaction.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — Circuit Breaker 완전 분해 ➡️](../availability-patterns/01-circuit-breaker.md)**

</div>

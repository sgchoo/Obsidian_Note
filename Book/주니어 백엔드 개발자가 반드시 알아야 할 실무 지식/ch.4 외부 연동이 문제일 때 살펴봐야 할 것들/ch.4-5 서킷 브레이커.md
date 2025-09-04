[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 4-5. 서킷 브레이커 (Circuit Breaker)

## 개요

**사용자 입장에도 수 초를 대기하다가 에러 화면을 보는 것보다 빠르게 에러 화면을 보는게 낫다.**

서킷 브레이커의 역할은 **외부 연동 서비스가 장애 상황일 때 빠르게 에러를, 정상일 때 연동 재개시키는 역할**을 한다.

### 왜 서킷 브레이커가 필요한가?

- **빠른 실패**: 장애 상황에서 긴 대기시간 없이 즉시 실패 응답
- **시스템 자원 보호**: 불필요한 요청으로 인한 리소스 낭비 방지
- **연쇄 장애 차단**: 외부 서비스 장애가 전체 시스템으로 전파되는 것을 방지
- **자동 복구**: 외부 서비스 정상화 시 자동으로 연동 재개

## 서킷 브레이커의 3가지 상태

**서킷 브레이커는 닫힘, 열림, 반열림 상태가 있다.**

### 1. 닫힌 상태 (CLOSED)

```
정상 동작 상태
┌─────────────┐
│   Request   │ ──────► ┌─────────────┐ ──────► ┌─────────────┐
│             │         │   Circuit   │         │  External   │
│             │         │   Breaker   │         │   Service   │
└─────────────┘ ◄────── └─────────────┘ ◄────── └─────────────┘
                         (CLOSED)
```

**특징:**

- 모든 요청이 외부 서비스로 전달
- 실패율과 응답시간을 지속적으로 모니터링
- 임계치 초과 시 열린 상태로 전환

```java
@Component
public class CircuitBreakerService {
    
    @CircuitBreaker(name = "external-service")
    public String callExternalService() {
        // 정상 상태: 모든 요청이 외부 서비스로 전달
        return externalApiClient.getData();
    }
}
```

### 2. 열린 상태 (OPEN)

```
서킷 차단 상태 - 즉시 실패 반환
┌─────────────┐
│   Request   │ ──────► ┌─────────────┐ ──X──   ┌─────────────┐
│             │         │   Circuit   │         │  External   │
│             │         │   Breaker   │         │   Service   │
└─────────────┘ ◄────── └─────────────┘         └─────────────┘
   Fail Fast             (OPEN)
```

**특징:**

- 모든 요청을 즉시 차단하고 Fallback 응답 반환
- 외부 서비스로 요청을 보내지 않음
- 일정 시간 후 반열림 상태로 전환

```java
@Component
public class CircuitBreakerService {
    
    @CircuitBreaker(name = "external-service", fallbackMethod = "fallback")
    public String callExternalService() {
        // 열린 상태에서는 이 메서드가 호출되지 않음
        return externalApiClient.getData();
    }
    
    public String fallback(Exception ex) {
        // 즉시 실패 응답 반환
        return "서비스 일시 중단. 잠시 후 다시 시도해주세요.";
    }
}
```

### 3. 반열린 상태 (HALF-OPEN)

```
테스트 요청 상태
┌─────────────┐
│   Request   │ ──────► ┌─────────────┐ ──?──► ┌─────────────┐
│             │         │   Circuit   │        │  External   │
│             │         │   Breaker   │        │   Service   │
└─────────────┘ ◄────── └─────────────┘ ◄─?─── └─────────────┘
                         (HALF-OPEN)
```

**특징:**

- 제한된 수의 요청만 외부 서비스로 전달
- 요청 결과에 따라 닫힌 상태 또는 열린 상태로 전환
- 외부 서비스 복구 여부를 판단하는 테스트 단계

## 실제 구현 예시

### 1. Resilience4j 기본 설정

```java
// application.yml
resilience4j:
  circuitbreaker:
    instances:
      external-service:
        failure-rate-threshold: 50          # 실패율 50% 초과시 OPEN
        minimum-number-of-calls: 5          # 최소 5번 호출 후 판단
        sliding-window-size: 10             # 최근 10개 요청으로 판단
        wait-duration-in-open-state: 30s    # OPEN 상태 30초 유지
        permitted-number-of-calls-in-half-open-state: 3  # HALF-OPEN에서 3개 테스트
        automatic-transition-from-open-to-half-open-enabled: true
```

```java
@Component
public class PaymentService {
    
    @CircuitBreaker(name = "payment-service", fallbackMethod = "paymentFallback")
    @TimeLimiter(name = "payment-service")
    public CompletableFuture<PaymentResponse> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            return paymentApiClient.processPayment(request);
        });
    }
    
    public CompletableFuture<PaymentResponse> paymentFallback(
            PaymentRequest request, Exception ex) {
        log.warn("결제 서비스 차단됨: {}", ex.getMessage());
        
        PaymentResponse fallbackResponse = new PaymentResponse();
        fallbackResponse.setStatus("PENDING");
        fallbackResponse.setMessage("결제 처리 중입니다. 잠시만 기다려주세요.");
        
        return CompletableFuture.completedFuture(fallbackResponse);
    }
}
```

### 2. 서비스별 차등 설정

```java
@Configuration
public class CircuitBreakerConfig {
    
    @Bean
    public CircuitBreakerConfigCustomizer paymentCircuitBreakerConfig() {
        return CircuitBreakerConfigCustomizer.of("payment-service", builder -> 
            builder
                .failureRateThreshold(30.0f)    // 결제: 30% 실패시 차단 (엄격)
                .waitDurationInOpenState(Duration.ofMinutes(2))
                .slidingWindowSize(20)
        );
    }
    
    @Bean
    public CircuitBreakerConfigCustomizer notificationCircuitBreakerConfig() {
        return CircuitBreakerConfigCustomizer.of("notification-service", builder -> 
            builder
                .failureRateThreshold(70.0f)    // 알림: 70% 실패시 차단 (관대)
                .waitDurationInOpenState(Duration.ofSeconds(30))
                .slidingWindowSize(10)
        );
    }
}
```

### 3. 상태 전환 모니터링

```java
@Component
public class CircuitBreakerEventListener {
    
    private final MeterRegistry meterRegistry;
    
    public CircuitBreakerEventListener(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    @EventListener
    public void onStateTransition(CircuitBreakerOnStateTransitionEvent event) {
        String serviceName = event.getCircuitBreakerName();
        CircuitBreaker.State fromState = event.getStateTransition().getFromState();
        CircuitBreaker.State toState = event.getStateTransition().getToState();
        
        log.info("서킷 브레이커 상태 변경: {} - {} → {}", 
            serviceName, fromState, toState);
        
        // 메트릭 기록
        Counter.builder("circuit.breaker.state.transition")
            .tag("service", serviceName)
            .tag("from", fromState.toString())
            .tag("to", toState.toString())
            .register(meterRegistry)
            .increment();
            
        // 알림 발송
        if (toState == CircuitBreaker.State.OPEN) {
            alertService.sendCircuitBreakerOpenAlert(serviceName);
        } else if (toState == CircuitBreaker.State.CLOSED) {
            alertService.sendCircuitBreakerRecoveredAlert(serviceName);
        }
    }
}
```

## 고급 서킷 브레이커 패턴

### 1. 계층형 서킷 브레이커

```java
@Service
public class HierarchicalCircuitBreakerService {
    
    // 개별 API용 서킷 브레이커
    @CircuitBreaker(name = "user-api")
    public UserInfo getUserInfo(String userId) {
        return userApiClient.getUser(userId);
    }
    
    @CircuitBreaker(name = "order-api") 
    public OrderInfo getOrderInfo(String orderId) {
        return orderApiClient.getOrder(orderId);
    }
    
    // 전체 외부 서비스용 서킷 브레이커
    @CircuitBreaker(name = "external-services", fallbackMethod = "allServicesFallback")
    public DashboardInfo getDashboardInfo(String userId) {
        CompletableFuture<UserInfo> userFuture = CompletableFuture
            .supplyAsync(() -> getUserInfo(userId));
        CompletableFuture<OrderInfo> orderFuture = CompletableFuture
            .supplyAsync(() -> getOrderInfo(userId));
            
        return DashboardInfo.builder()
            .user(userFuture.join())
            .orders(orderFuture.join())
            .build();
    }
    
    public DashboardInfo allServicesFallback(String userId, Exception ex) {
        return DashboardInfo.builder()
            .message("일부 정보를 불러올 수 없습니다")
            .build();
    }
}
```

### 2. 조건부 서킷 브레이커

```java
@Component
public class ConditionalCircuitBreakerService {
    
    private final Map<String, CircuitBreaker> circuitBreakers = new HashMap<>();
    
    public String callWithConditionalBreaker(String serviceType, String data) {
        CircuitBreaker circuitBreaker = getOrCreateCircuitBreaker(serviceType);
        
        return circuitBreaker.executeSupplier(() -> {
            if ("critical".equals(serviceType)) {
                // 중요한 서비스는 더 엄격한 타임아웃
                return callWithTimeout(data, 1000);
            } else {
                // 일반 서비스는 일반 타임아웃
                return callWithTimeout(data, 5000);
            }
        });
    }
    
    private CircuitBreaker getOrCreateCircuitBreaker(String serviceType) {
        return circuitBreakers.computeIfAbsent(serviceType, key -> {
            CircuitBreakerConfig config;
            if ("critical".equals(key)) {
                config = CircuitBreakerConfig.custom()
                    .failureRateThreshold(20.0f)    // 20% 실패시 차단
                    .waitDurationInOpenState(Duration.ofMinutes(5))
                    .build();
            } else {
                config = CircuitBreakerConfig.ofDefaults();
            }
            return CircuitBreaker.of(key, config);
        });
    }
}
```

### 3. 다단계 Fallback

```java
@Component
public class MultiLevelFallbackService {
    
    @CircuitBreaker(name = "primary-service", fallbackMethod = "secondaryServiceFallback")
    public ProductInfo getProductInfo(String productId) {
        return primaryApiClient.getProduct(productId);
    }
    
    @CircuitBreaker(name = "secondary-service", fallbackMethod = "cacheServiceFallback")
    public ProductInfo secondaryServiceFallback(String productId, Exception ex) {
        log.info("Primary 서비스 실패, Secondary 서비스 시도");
        return secondaryApiClient.getProduct(productId);
    }
    
    @CircuitBreaker(name = "cache-service", fallbackMethod = "defaultFallback")
    public ProductInfo cacheServiceFallback(String productId, Exception ex) {
        log.info("Secondary 서비스 실패, 캐시에서 조회");
        return cacheService.getProduct(productId);
    }
    
    public ProductInfo defaultFallback(String productId, Exception ex) {
        log.warn("모든 서비스 실패, 기본값 반환");
        return ProductInfo.builder()
            .id(productId)
            .name("상품 정보 로딩 중...")
            .available(false)
            .build();
    }
}
```

## 성능 최적화 및 튜닝

### 1. 동적 임계값 조정

```java
@Component
public class AdaptiveCircuitBreakerService {
    
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    @Scheduled(fixedRate = 60000) // 1분마다 실행
    public void adjustThresholds() {
        circuitBreakerRegistry.getAllCircuitBreakers().forEach(circuitBreaker -> {
            CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
            String serviceName = circuitBreaker.getName();
            
            // 최근 성능 지표 분석
            float currentFailureRate = metrics.getFailureRate();
            float avgResponseTime = getAvgResponseTime(serviceName);
            
            // 동적 임계값 계산
            float newThreshold = calculateOptimalThreshold(
                currentFailureRate, avgResponseTime
            );
            
            // 임계값 업데이트 (재생성 필요)
            if (Math.abs(newThreshold - getCurrentThreshold(serviceName)) > 10.0f) {
                recreateCircuitBreakerWithNewThreshold(serviceName, newThreshold);
                log.info("서킷브레이커 임계값 조정: {} -> {}%", 
                    serviceName, newThreshold);
            }
        });
    }
}
```

### 2. 비즈니스 로직 기반 판단

```java
@Component
public class BusinessAwareCircuitBreakerService {
    
    @CircuitBreaker(name = "business-service")
    public OrderResult processOrder(OrderRequest request) {
        return Supplier.of(() -> {
            try {
                OrderResult result = orderApiClient.processOrder(request);
                
                // 비즈니스 로직 기반 실패 판단
                if (result.getStatus().equals("BUSINESS_ERROR")) {
                    throw new BusinessException("비즈니스 규칙 위반");
                }
                
                return result;
            } catch (BusinessException ex) {
                // 비즈니스 에러는 서킷브레이커에 포함시키지 않음
                recordIgnoredFailure();
                throw ex;
            }
        }).get();
    }
}
```

## 모니터링 및 대시보드

### 1. 실시간 상태 모니터링

```java
@RestController
@RequestMapping("/admin/circuit-breakers")
public class CircuitBreakerMonitoringController {
    
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    @GetMapping("/status")
    public Map<String, Object> getCircuitBreakerStatus() {
        Map<String, Object> status = new HashMap<>();
        
        circuitBreakerRegistry.getAllCircuitBreakers().forEach(cb -> {
            CircuitBreaker.Metrics metrics = cb.getMetrics();
            
            Map<String, Object> cbStatus = new HashMap<>();
            cbStatus.put("state", cb.getState().toString());
            cbStatus.put("failureRate", metrics.getFailureRate());
            cbStatus.put("numberOfCalls", metrics.getNumberOfCalls());
            cbStatus.put("numberOfFailedCalls", metrics.getNumberOfFailedCalls());
            cbStatus.put("numberOfSuccessfulCalls", metrics.getNumberOfSuccessfulCalls());
            
            status.put(cb.getName(), cbStatus);
        });
        
        return status;
    }
    
    @PostMapping("/force-open/{name}")
    public ResponseEntity<String> forceOpen(@PathVariable String name) {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker(name);
        cb.transitionToOpenState();
        return ResponseEntity.ok("Circuit breaker forced to OPEN state");
    }
    
    @PostMapping("/force-close/{name}")
    public ResponseEntity<String> forceClose(@PathVariable String name) {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker(name);
        cb.transitionToClosedState();
        return ResponseEntity.ok("Circuit breaker forced to CLOSED state");
    }
}
```

### 2. 메트릭 수집

```java
@Component
public class CircuitBreakerMetricsCollector {
    
    @EventListener
    public void onCircuitBreakerEvent(CircuitBreakerEvent event) {
        String serviceName = event.getCircuitBreakerName();
        
        switch (event.getEventType()) {
            case SUCCESS:
                meterRegistry.counter("circuit.breaker.calls", 
                    "service", serviceName, "result", "success").increment();
                break;
                
            case ERROR:
                meterRegistry.counter("circuit.breaker.calls",
                    "service", serviceName, "result", "error").increment();
                break;
                
            case IGNORED_ERROR:
                meterRegistry.counter("circuit.breaker.calls",
                    "service", serviceName, "result", "ignored").increment();
                break;
                
            case NOT_PERMITTED:
                meterRegistry.counter("circuit.breaker.calls",
                    "service", serviceName, "result", "rejected").increment();
                break;
        }
    }
}
```

## 테스트 전략

### 1. 서킷 브레이커 동작 테스트

```java
@SpringBootTest
class CircuitBreakerServiceTest {
    
    @MockBean
    private ExternalApiClient externalApiClient;
    
    @Autowired
    private CircuitBreakerService circuitBreakerService;
    
    @Test
    @DisplayName("실패율 50% 초과시 서킷 브레이커가 열린다")
    void shouldOpenCircuitBreakerOnHighFailureRate() {
        // Given: 외부 서비스가 계속 실패하도록 설정
        when(externalApiClient.getData())
            .thenThrow(new RuntimeException("Service failure"));
        
        // When: 실패율이 임계치를 넘도록 여러 번 호출
        for (int i = 0; i < 10; i++) {
            try {
                circuitBreakerService.callExternalService();
            } catch (Exception e) {
                // 예외 무시
            }
        }
        
        // Then: 서킷 브레이커가 열림 상태가 되어 Fallback 호출
        String result = circuitBreakerService.callExternalService();
        assertThat(result).isEqualTo("서비스 일시 중단. 잠시 후 다시 시도해주세요.");
    }
    
    @Test
    @DisplayName("서킷 브레이커가 HALF-OPEN 상태에서 성공시 CLOSED로 전환")
    void shouldTransitionToClosedOnSuccessInHalfOpenState() {
        // Given: 서킷 브레이커를 HALF-OPEN 상태로 만듦
        forceHalfOpenState();
        when(externalApiClient.getData()).thenReturn("success");
        
        // When: 테스트 호출 성공
        String result = circuitBreakerService.callExternalService();
        
        // Then: CLOSED 상태로 전환
        assertThat(result).isEqualTo("success");
        assertThat(getCircuitBreakerState()).isEqualTo(CircuitBreaker.State.CLOSED);
    }
}
```

### 2. 부하 테스트

```java
@Test
@DisplayName("동시성 환경에서 서킷 브레이커 동작 검증")
void shouldHandleConcurrentRequestsProperly() throws InterruptedException {
    int threadCount = 100;
    CountDownLatch latch = new CountDownLatch(threadCount);
    ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
    
    // 외부 서비스를 실패하도록 설정
    when(externalApiClient.getData())
        .thenThrow(new RuntimeException("Service failure"));
    
    List<CompletableFuture<String>> futures = IntStream.range(0, threadCount)
        .mapToObj(i -> CompletableFuture.supplyAsync(() -> {
            try {
                return circuitBreakerService.callExternalService();
            } catch (Exception e) {
                return "error: " + e.getMessage();
            } finally {
                latch.countDown();
            }
        }, executorService))
        .collect(Collectors.toList());
    
    latch.await(10, TimeUnit.SECONDS);
    
    List<String> results = futures.stream()
        .map(CompletableFuture::join)
        .collect(Collectors.toList());
    
    // 서킷 브레이커가 적절히 동작하여 일정 시점부터 Fallback 응답
    long fallbackCount = results.stream()
        .filter(result -> result.contains("서비스 일시 중단"))
        .count();
    
    assertThat(fallbackCount).isGreaterThan(0);
}
```

## 실무 체크리스트

### 설계 시 고려사항

- [ ] 서비스별 적절한 실패율 임계치 설정 (20-70%)
- [ ] 비즈니스 중요도에 따른 대기시간 설정
- [ ] 의미있는 Fallback 응답 설계
- [ ] 계층형 서킷 브레이커 구조 검토
- [ ] 비즈니스 로직 기반 실패 판단 기준

### 구현 시 확인사항

- [ ] 모든 외부 서비스 호출에 서킷 브레이커 적용
- [ ] Fallback 메서드의 예외 처리
- [ ] 상태 전환 이벤트 모니터링
- [ ] 다단계 Fallback 체인 구현
- [ ] 테스트 시나리오 작성 및 검증

### 운영 시 모니터링

- [ ] 서킷 브레이커 상태 실시간 모니터링
- [ ] 실패율 및 응답시간 추이 분석
- [ ] OPEN 상태 전환 알림 설정
- [ ] 정기적인 임계치 검토 및 튜닝
- [ ] 부하 테스트를 통한 동작 검증
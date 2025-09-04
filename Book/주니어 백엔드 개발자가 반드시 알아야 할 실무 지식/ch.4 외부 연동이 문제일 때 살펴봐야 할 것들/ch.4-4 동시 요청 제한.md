[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 4-4. 동시 요청 제한 (Concurrent Request Limiting)

## 개요

**연동 서비스에 임계치 이상의 요청을 보내지 않는다.**

### 왜 동시 요청 제한이 필요한가?

- **연동 서비스 보호**: 과도한 요청으로 인한 외부 서비스 장애 방지
- **안정적인 관계 유지**: 파트너사와의 협력 관계 보호
- **비용 절약**: API 호출량 기반 과금 모델에서 비용 통제
- **전체 시스템 안정성**: 외부 서비스 장애가 우리 서비스로 전파되는 것을 방지

## 동시 요청 제한 방법

### 1. Connection Pool 크기 제한

#### RestTemplate 설정

```java
@Configuration
public class HttpClientConfig {
    
    @Bean
    public RestTemplate paymentRestTemplate() {
        // Connection Pool 설정
        PoolingHttpClientConnectionManager connectionManager = 
            new PoolingHttpClientConnectionManager();
        
        connectionManager.setMaxTotal(20);           // 전체 최대 연결 수
        connectionManager.setDefaultMaxPerRoute(5);  // 호스트당 최대 연결 수
        
        CloseableHttpClient httpClient = HttpClients.custom()
            .setConnectionManager(connectionManager)
            .build();
        
        HttpComponentsClientHttpRequestFactory factory = 
            new HttpComponentsClientHttpRequestFactory(httpClient);
        
        return new RestTemplate(factory);
    }
}
```

#### WebClient 설정 (Reactive)

```java
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient paymentWebClient() {
        ConnectionProvider connectionProvider = ConnectionProvider
            .builder("payment-pool")
            .maxConnections(20)        // 최대 연결 수
            .maxIdleTime(Duration.ofSeconds(30))
            .maxLifeTime(Duration.ofMinutes(5))
            .pendingAcquireMaxCount(50)  // 대기 요청 최대 수
            .build();
        
        HttpClient httpClient = HttpClient.create(connectionProvider);
        
        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}
```

### 2. Rate Limiter 구현

#### Google Guava RateLimiter

```java
@Component
public class ExternalApiService {
    
    // 초당 10개 요청으로 제한
    private final RateLimiter rateLimiter = RateLimiter.create(10.0);
    
    public String callExternalApi() {
        // 요청 전 Rate Limit 확인
        if (!rateLimiter.tryAcquire(1, 2, TimeUnit.SECONDS)) {
            throw new RateLimitExceededException("요청 제한 초과");
        }
        
        return externalApiClient.getData();
    }
}
```

#### Bucket4j (Token Bucket Algorithm)

```java
@Component
public class TokenBucketLimitService {
    
    private final Bucket bucket;
    
    public TokenBucketLimitService() {
        // 분당 100개 요청, 초기 토큰 10개
        Bandwidth limit = Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1)));
        this.bucket = Bucket.builder()
            .addLimit(limit)
            .build();
    }
    
    public String callWithTokenBucket() {
        if (bucket.tryConsume(1)) {
            return externalApiClient.getData();
        } else {
            throw new RateLimitExceededException("토큰 부족");
        }
    }
}
```

### 3. Semaphore를 이용한 동시 실행 제한

```java
@Component
public class SemaphoreBasedLimitService {
    
    // 동시에 최대 5개 요청만 허용
    private final Semaphore semaphore = new Semaphore(5);
    
    public String callWithSemaphore() {
        try {
            // 2초 내에 허가를 얻지 못하면 실패
            if (!semaphore.tryAcquire(2, TimeUnit.SECONDS)) {
                throw new ConcurrentLimitExceededException("동시 실행 제한 초과");
            }
            
            return externalApiClient.getData();
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("요청 처리 중단됨", e);
        } finally {
            semaphore.release();
        }
    }
}
```

### 4. Reactive Streams Backpressure

```java
@Service
public class ReactiveRateLimitService {
    
    public Flux<String> processWithBackpressure(Flux<String> requests) {
        return requests
            // 동시 처리 수 제한
            .flatMap(request -> callExternalApi(request), 3) // 최대 3개 동시 실행
            // Rate Limiting
            .delayElements(Duration.ofMillis(100)) // 100ms 간격
            // 버퍼 오버플로우 시 Drop
            .onBackpressureDrop(dropped -> 
                log.warn("요청 드랍됨: {}", dropped));
    }
    
    private Mono<String> callExternalApi(String request) {
        return webClient
            .get()
            .uri("/api/data")
            .retrieve()
            .bodyToMono(String.class)
            .timeout(Duration.ofSeconds(5));
    }
}
```

## 서비스별 임계치 설정 가이드

### 1. 연동 서비스 유형별 권장사항

|서비스 유형|초당 요청수|동시 연결수|비고|
|---|---|---|---|
|**결제 서비스**|5-10 TPS|3-5개|안정성 우선|
|**인증 서비스**|20-50 TPS|5-10개|빠른 응답 필요|
|**알림 서비스**|100-200 TPS|10-20개|대용량 처리|
|**파일 업로드**|2-5 TPS|2-3개|대용량 데이터|
|**조회 API**|50-100 TPS|10-15개|일반적인 처리량|

### 2. 설정 파일 기반 관리

```yaml
# application.yml
external-api:
  rate-limits:
    payment-service:
      requests-per-second: 10
      max-concurrent: 5
      timeout-seconds: 30
    
    notification-service:
      requests-per-second: 100
      max-concurrent: 15
      timeout-seconds: 5
    
    user-service:
      requests-per-second: 50
      max-concurrent: 10
      timeout-seconds: 10
```

```java
@ConfigurationProperties(prefix = "external-api.rate-limits")
@Data
public class RateLimitConfig {
    
    private Map<String, ServiceLimit> services = new HashMap<>();
    
    @Data
    public static class ServiceLimit {
        private int requestsPerSecond;
        private int maxConcurrent;
        private int timeoutSeconds;
    }
}
```

### 3. 동적 임계치 조정

```java
@Component
public class DynamicRateLimitService {
    
    private final Map<String, RateLimiter> rateLimiters = new ConcurrentHashMap<>();
    private final ExternalServiceMonitor serviceMonitor;
    
    @Scheduled(fixedRate = 30000) // 30초마다 조정
    public void adjustRateLimits() {
        serviceMonitor.getServiceMetrics().forEach((serviceName, metrics) -> {
            double currentTps = calculateOptimalTps(metrics);
            updateRateLimit(serviceName, currentTps);
        });
    }
    
    private double calculateOptimalTps(ServiceMetrics metrics) {
        // 응답시간, 에러율, CPU 사용률 등을 고려한 최적 TPS 계산
        if (metrics.getErrorRate() > 0.05) {
            return Math.max(1.0, metrics.getCurrentTps() * 0.8);
        } else if (metrics.getAvgResponseTime() < 100) {
            return Math.min(100.0, metrics.getCurrentTps() * 1.2);
        }
        return metrics.getCurrentTps();
    }
}
```

## 고급 제한 패턴

### 1. 우선순위 기반 제한

```java
@Component
public class PriorityBasedLimitService {
    
    private final Semaphore highPrioritySemaphore = new Semaphore(10);
    private final Semaphore lowPrioritySemaphore = new Semaphore(5);
    
    public String callWithPriority(RequestPriority priority, String data) {
        Semaphore targetSemaphore = (priority == RequestPriority.HIGH) 
            ? highPrioritySemaphore 
            : lowPrioritySemaphore;
            
        try {
            if (!targetSemaphore.tryAcquire(1, TimeUnit.SECONDS)) {
                throw new RateLimitExceededException("우선순위별 제한 초과");
            }
            
            return externalApiClient.callApi(data);
        } finally {
            targetSemaphore.release();
        }
    }
}
```

### 2. Circuit Breaker와 연계

```java
@Component
public class CircuitBreakerWithRateLimit {
    
    private final RateLimiter rateLimiter = RateLimiter.create(10.0);
    
    @CircuitBreaker(name = "external-service", fallbackMethod = "fallback")
    public String callWithCircuitBreaker() {
        // Rate Limit 먼저 확인
        if (!rateLimiter.tryAcquire()) {
            throw new RateLimitExceededException("Rate limit exceeded");
        }
        
        return externalApiClient.getData();
    }
    
    public String fallback(Exception ex) {
        if (ex instanceof RateLimitExceededException) {
            return "요청량 초과로 인한 지연";
        }
        return "서비스 일시 중단";
    }
}
```

### 3. 큐 기반 비동기 처리

```java
@Component
public class QueueBasedLimitService {
    
    private final BlockingQueue<ApiRequest> requestQueue = 
        new ArrayBlockingQueue<>(1000);
    
    @Async
    @Scheduled(fixedDelay = 100) // 100ms마다 처리
    public void processQueuedRequests() {
        List<ApiRequest> batch = new ArrayList<>();
        requestQueue.drainTo(batch, 10); // 최대 10개씩 처리
        
        if (!batch.isEmpty()) {
            processBatch(batch);
        }
    }
    
    public CompletableFuture<String> submitRequest(String data) {
        ApiRequest request = new ApiRequest(data);
        
        if (!requestQueue.offer(request)) {
            request.complete("큐 포화로 인한 요청 거부");
        }
        
        return request.getFuture();
    }
}
```

## 모니터링 및 알림

### 1. 제한 현황 메트릭

```java
@Component
public class RateLimitMetrics {
    
    private final Counter allowedRequests;
    private final Counter blockedRequests;
    private final Gauge currentPermits;
    
    public RateLimitMetrics(MeterRegistry meterRegistry) {
        this.allowedRequests = Counter.builder("rate.limit.allowed")
            .tag("service", "external")
            .register(meterRegistry);
            
        this.blockedRequests = Counter.builder("rate.limit.blocked")
            .tag("service", "external")
            .register(meterRegistry);
            
        this.currentPermits = Gauge.builder("rate.limit.permits")
            .tag("service", "external")
            .register(meterRegistry, rateLimiter, r -> r.getRate());
    }
    
    public void recordAllowed(String serviceName) {
        allowedRequests.increment(Tags.of("target", serviceName));
    }
    
    public void recordBlocked(String serviceName) {
        blockedRequests.increment(Tags.of("target", serviceName));
    }
}
```

### 2. 임계치 초과 알림

```java
@Component
public class RateLimitAlertService {
    
    private final AlertService alertService;
    
    @EventListener
    public void handleRateLimitExceeded(RateLimitExceededEvent event) {
        if (event.getBlockedRatio() > 0.1) { // 10% 이상 차단 시
            alertService.sendAlert(
                AlertLevel.WARNING,
                String.format("서비스 %s의 요청 차단율이 %.1f%%입니다.", 
                    event.getServiceName(), 
                    event.getBlockedRatio() * 100)
            );
        }
    }
}
```

### 3. 실시간 대시보드

```java
@RestController
@RequestMapping("/admin/rate-limits")
public class RateLimitDashboardController {
    
    @GetMapping("/status")
    public Map<String, Object> getRateLimitStatus() {
        Map<String, Object> status = new HashMap<>();
        
        rateLimitServices.forEach((serviceName, limiter) -> {
            Map<String, Object> serviceStatus = new HashMap<>();
            serviceStatus.put("currentRate", limiter.getRate());
            serviceStatus.put("availablePermits", limiter.getClass().getSimpleName());
            serviceStatus.put("queueSize", getQueueSize(serviceName));
            
            status.put(serviceName, serviceStatus);
        });
        
        return status;
    }
    
    @PostMapping("/adjust/{serviceName}")
    public ResponseEntity<String> adjustRateLimit(
            @PathVariable String serviceName,
            @RequestParam double newRate) {
        
        adjustRateLimit(serviceName, newRate);
        return ResponseEntity.ok("Rate limit adjusted successfully");
    }
}
```

## 장애 상황별 대응 전략

### 1. 연동 서비스 장애 시

```java
@Component
public class FailureAdaptiveRateLimit {
    
    @EventListener
    public void handleServiceFailure(ServiceFailureEvent event) {
        // 장애 발생 시 요청량을 50%로 감소
        String serviceName = event.getServiceName();
        RateLimiter limiter = rateLimiters.get(serviceName);
        
        if (limiter != null) {
            double newRate = limiter.getRate() * 0.5;
            limiter.setRate(Math.max(1.0, newRate));
            
            log.info("서비스 {} 장애로 인해 Rate Limit을 {}로 조정", serviceName, newRate);
        }
    }
    
    @EventListener
    public void handleServiceRecovery(ServiceRecoveryEvent event) {
        // 복구 시 점진적으로 요청량 증가
        scheduleGradualIncrease(event.getServiceName());
    }
}
```

### 2. 급작스러운 트래픽 증가

```java
@Component
public class TrafficSurgeHandler {
    
    @EventListener
    public void handleTrafficSurge(TrafficSurgeEvent event) {
        if (event.getTrafficIncrease() > 2.0) { // 2배 이상 증가
            // 임시로 요청 제한 강화
            temporaryRateReduce(event.getServiceName(), 0.7);
            
            // 10분 후 정상화
            scheduler.schedule(
                () -> restoreNormalRate(event.getServiceName()),
                10, TimeUnit.MINUTES
            );
        }
    }
}
```

## 실무 체크리스트

### 설계 시 고려사항

- [ ] 연동 서비스별 적절한 임계치 설정
- [ ] Connection Pool 크기 최적화
- [ ] Rate Limiter 알고리즘 선택 (Token Bucket vs Leaky Bucket)
- [ ] 우선순위별 차등 제한 필요성 검토
- [ ] 큐 기반 비동기 처리 검토

### 구현 시 확인사항

- [ ] 제한 초과 시 적절한 에러 응답
- [ ] Circuit Breaker와의 연계
- [ ] 메트릭 수집 및 모니터링 설정
- [ ] 동적 임계치 조정 메커니즘
- [ ] 장애 상황별 대응 로직

### 운영 시 모니터링

- [ ] 요청 허용/차단 비율 추적
- [ ] 연동 서비스별 응답시간 모니터링
- [ ] 큐 사이즈 및 대기시간 확인
- [ ] 임계치 초과 알림 설정
- [ ] 정기적인 임계치 검토 및 조정
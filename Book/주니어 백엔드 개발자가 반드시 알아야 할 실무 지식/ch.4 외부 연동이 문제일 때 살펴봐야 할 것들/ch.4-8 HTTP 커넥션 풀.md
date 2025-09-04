[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 4-8. HTTP 커넥션 풀 (HTTP Connection Pool)

## 개요

**HTTP 연결도 커넥션 풀을 사용하면 연결 시간을 줄일 수 있다.**

### HTTP 커넥션 풀의 이점

```
커넥션 풀 미사용 시:
Request 1: TCP 핸드셰이크(100ms) + HTTP 요청/응답(200ms) = 300ms
Request 2: TCP 핸드셰이크(100ms) + HTTP 요청/응답(200ms) = 300ms
Request 3: TCP 핸드셰이크(100ms) + HTTP 요청/응답(200ms) = 300ms
총 소요시간: 900ms

커넥션 풀 사용 시:
Request 1: TCP 핸드셰이크(100ms) + HTTP 요청/응답(200ms) = 300ms (최초)
Request 2: HTTP 요청/응답(200ms) = 200ms (재사용)
Request 3: HTTP 요청/응답(200ms) = 200ms (재사용)
총 소요시간: 700ms (22% 성능 향상)
```

## HTTP 커넥션 풀의 3가지 핵심 고려사항

**HTTP 커넥션 풀을 사용할 때는 다음 3가지를 고려해야한다.**

### 1. HTTP 커넥션 풀의 크기

**커넥션 풀의 크기를 설정할 때는 연동 서비스의 처리 능력을 고려해야한다.**

#### RestTemplate 설정

```java
@Configuration
public class HttpConnectionPoolConfig {
    
    @Bean
    public RestTemplate paymentServiceRestTemplate() {
        // Connection Manager 설정
        PoolingHttpClientConnectionManager connectionManager = 
            new PoolingHttpClientConnectionManager();
            
        // 전체 최대 커넥션 수
        connectionManager.setMaxTotal(100);
        
        // 호스트당 최대 커넥션 수 (연동 서비스 처리 능력 고려)
        connectionManager.setDefaultMaxPerRoute(20);
        
        // 특정 호스트별 세부 설정
        connectionManager.setMaxPerRoute(
            new HttpRoute(new HttpHost("payment-api.example.com", 443, "https")), 30);
        
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
public class WebClientConnectionPoolConfig {
    
    @Bean
    public WebClient paymentWebClient() {
        // Connection Provider 설정
        ConnectionProvider connectionProvider = ConnectionProvider.builder("payment-pool")
            .maxConnections(50)                          // 최대 커넥션 수
            .maxIdleTime(Duration.ofSeconds(30))         // 유휴 상태 최대 시간
            .maxLifeTime(Duration.ofMinutes(5))          // 커넥션 최대 생존 시간
            .pendingAcquireMaxCount(100)                 // 대기 큐 최대 크기
            .pendingAcquireTimeout(Duration.ofSeconds(3)) // 커넥션 획득 대기 시간
            .build();
        
        HttpClient httpClient = HttpClient.create(connectionProvider)
            .keepAlive(true)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000);
        
        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}
```

#### 서비스별 커넥션 풀 크기 가이드

|서비스 유형|처리 능력(TPS)|권장 풀 크기|비고|
|---|---|---|---|
|**결제 서비스**|50-100|10-20|안정적 처리 우선|
|**사용자 인증**|200-500|30-50|빠른 응답 필요|
|**알림 발송**|1000+|50-100|대용량 처리|
|**파일 업로드**|10-20|5-10|대용량 데이터|
|**외부 API 조회**|100-200|20-40|일반적 조회|

### 2. 풀에서 HTTP 커넥션을 가져올 때까지 대기하는 시간

**대기 시간이 길어지면 전체 응답 시간도 함께 늘어나므로 대기 시간은 짧게 가져가는게 좋다** **이 책의 저자는 1~5초를 추천한다.**

```java
@Configuration
public class ConnectionAcquisitionTimeoutConfig {
    
    @Bean
    public RestTemplate fastResponseRestTemplate() {
        PoolingHttpClientConnectionManager connectionManager = 
            new PoolingHttpClientConnectionManager();
        connectionManager.setMaxTotal(50);
        connectionManager.setDefaultMaxPerRoute(10);
        
        // RequestConfig 설정
        RequestConfig requestConfig = RequestConfig.custom()
            .setConnectionRequestTimeout(2000)  // 커넥션 획득 대기: 2초
            .setConnectTimeout(3000)            // 연결 타임아웃: 3초
            .setSocketTimeout(5000)             // 읽기 타임아웃: 5초
            .build();
        
        CloseableHttpClient httpClient = HttpClients.custom()
            .setConnectionManager(connectionManager)
            .setDefaultRequestConfig(requestConfig)
            .build();
            
        HttpComponentsClientHttpRequestFactory factory = 
            new HttpComponentsClientHttpRequestFactory(httpClient);
            
        return new RestTemplate(factory);
    }
    
    @Bean
    public RestTemplate tolerantRestTemplate() {
        // 중요하지 않은 서비스용 - 더 긴 대기 시간 허용
        RequestConfig requestConfig = RequestConfig.custom()
            .setConnectionRequestTimeout(5000)  // 커넥션 획득 대기: 5초
            .setConnectTimeout(5000)
            .setSocketTimeout(10000)
            .build();
            
        // ... 나머지 설정
        return new RestTemplate(factory);
    }
}
```

#### 서비스별 권장 대기 시간

```yaml
# application.yml
http-client:
  timeouts:
    critical-services:
      connection-request: 1000    # 1초 (결제, 인증 등)
      connect: 2000
      socket: 5000
    
    general-services:
      connection-request: 3000    # 3초 (일반 조회 등)
      connect: 3000
      socket: 10000
    
    background-services:
      connection-request: 5000    # 5초 (배치, 알림 등)
      connect: 5000
      socket: 30000
```

### 3. HTTP 커넥션을 유지할 시간

**유지 시간은 연동 서비스에 맞춰 유지 시간을 적절히 설정한다.** **클라이언트의 커넥션 풀도 이 값보다 더 오래 유지하면 안된다.**

#### 커넥션 생존 시간 설정

```java
@Configuration
public class ConnectionLifetimeConfig {
    
    @Bean
    public RestTemplate configuredRestTemplate() {
        PoolingHttpClientConnectionManager connectionManager = 
            new PoolingHttpClientConnectionManager();
            
        // Keep-Alive 전략 설정
        ConnectionKeepAliveStrategy keepAliveStrategy = (response, context) -> {
            HeaderElementIterator it = new BasicHeaderElementIterator(
                response.headerIterator(HTTP.CONN_KEEP_ALIVE));
                
            while (it.hasNext()) {
                HeaderElement he = it.nextElement();
                String param = he.getName();
                String value = he.getValue();
                
                if (value != null && param.equalsIgnoreCase("timeout")) {
                    try {
                        return Long.parseLong(value) * 1000; // 초를 밀리초로 변환
                    } catch (NumberFormatException ignore) {
                    }
                }
            }
            
            // 서버가 Keep-Alive 정보를 제공하지 않으면 기본값 사용
            return 30 * 1000; // 30초
        };
        
        CloseableHttpClient httpClient = HttpClients.custom()
            .setConnectionManager(connectionManager)
            .setKeepAliveStrategy(keepAliveStrategy)
            .evictExpiredConnections()                    // 만료된 커넥션 자동 제거
            .evictIdleConnections(60, TimeUnit.SECONDS)   // 60초 유휴 커넥션 제거
            .build();
            
        return new RestTemplate(new HttpComponentsClientHttpRequestFactory(httpClient));
    }
}
```

#### 서버별 Keep-Alive 설정 가이드

```java
@Component
public class ServiceSpecificConnectionConfig {
    
    private final Map<String, RestTemplate> restTemplates = new HashMap<>();
    
    @PostConstruct
    public void initializeRestTemplates() {
        // 결제 서비스 - 짧은 Keep-Alive
        restTemplates.put("payment", createRestTemplate(
            20,     // 최대 커넥션
            10,     // Keep-Alive 시간 (초)
            2000    // 커넥션 획득 대기
        ));
        
        // 사용자 서비스 - 중간 Keep-Alive  
        restTemplates.put("user", createRestTemplate(
            30,     // 최대 커넥션
            30,     // Keep-Alive 시간 (초)
            3000    // 커넥션 획득 대기
        ));
        
        // 알림 서비스 - 긴 Keep-Alive
        restTemplates.put("notification", createRestTemplate(
            50,     // 최대 커넥션
            60,     // Keep-Alive 시간 (초)
            5000    // 커넥션 획득 대기
        ));
    }
    
    private RestTemplate createRestTemplate(
            int maxConnections, int keepAliveSeconds, int connectionRequestTimeout) {
        
        PoolingHttpClientConnectionManager connectionManager = 
            new PoolingHttpClientConnectionManager();
        connectionManager.setMaxTotal(maxConnections * 2);
        connectionManager.setDefaultMaxPerRoute(maxConnections);
        
        RequestConfig requestConfig = RequestConfig.custom()
            .setConnectionRequestTimeout(connectionRequestTimeout)
            .setConnectTimeout(3000)
            .setSocketTimeout(10000)
            .build();
        
        CloseableHttpClient httpClient = HttpClients.custom()
            .setConnectionManager(connectionManager)
            .setDefaultRequestConfig(requestConfig)
            .setKeepAliveStrategy((response, context) -> keepAliveSeconds * 1000)
            .evictIdleConnections(keepAliveSeconds, TimeUnit.SECONDS)
            .build();
            
        return new RestTemplate(new HttpComponentsClientHttpRequestFactory(httpClient));
    }
}
```

## 동적 커넥션 풀 관리

### 1. 런타임 커넥션 풀 조정

```java
@Component
public class DynamicConnectionPoolManager {
    
    private final Map<String, PoolingHttpClientConnectionManager> connectionManagers;
    private final ServiceMonitor serviceMonitor;
    
    @Scheduled(fixedRate = 60000) // 1분마다 조정
    public void adjustConnectionPools() {
        connectionManagers.forEach((serviceName, manager) -> {
            ServiceMetrics metrics = serviceMonitor.getMetrics(serviceName);
            
            // 응답시간과 처리량을 기반으로 커넥션 풀 크기 조정
            int optimalPoolSize = calculateOptimalPoolSize(metrics);
            
            if (optimalPoolSize != manager.getDefaultMaxPerRoute()) {
                manager.setDefaultMaxPerRoute(optimalPoolSize);
                log.info("{}서비스 커넥션 풀 크기 조정: {}", serviceName, optimalPoolSize);
            }
        });
    }
    
    private int calculateOptimalPoolSize(ServiceMetrics metrics) {
        double avgResponseTime = metrics.getAverageResponseTime();
        int currentThroughput = metrics.getCurrentThroughput();
        double errorRate = metrics.getErrorRate();
        
        // 응답시간이 길면 커넥션 수 증가
        int baseSize = 10;
        if (avgResponseTime > 1000) {
            baseSize += 10;
        } else if (avgResponseTime > 500) {
            baseSize += 5;
        }
        
        // 에러율이 높으면 커넥션 수 감소
        if (errorRate > 0.1) {
            baseSize = Math.max(5, baseSize - 5);
        }
        
        // 처리량을 고려한 최종 조정
        return Math.min(50, Math.max(5, baseSize + currentThroughput / 10));
    }
}
```

### 2. 서킷 브레이커와 연계

```java
@Component
public class CircuitBreakerAwareConnectionPool {
    
    @EventListener
    public void handleCircuitBreakerOpen(CircuitBreakerOpenEvent event) {
        String serviceName = event.getServiceName();
        PoolingHttpClientConnectionManager manager = getConnectionManager(serviceName);
        
        // 서킷 브레이커가 열리면 커넥션 풀 크기 축소
        int currentSize = manager.getDefaultMaxPerRoute();
        int reducedSize = Math.max(2, currentSize / 2);
        
        manager.setDefaultMaxPerRoute(reducedSize);
        log.info("서킷 브레이커 OPEN으로 인한 커넥션 풀 축소: {} -> {}", 
            currentSize, reducedSize);
    }
    
    @EventListener
    public void handleCircuitBreakerClosed(CircuitBreakerClosedEvent event) {
        String serviceName = event.getServiceName();
        
        // 서킷 브레이커가 닫히면 점진적으로 커넥션 풀 복구
        scheduleGradualPoolRecovery(serviceName);
    }
}
```

## 성능 모니터링 및 튜닝

### 1. 커넥션 풀 메트릭 수집

```java
@Component
public class ConnectionPoolMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Map<String, PoolingHttpClientConnectionManager> connectionManagers;
    
    @Scheduled(fixedRate = 10000) // 10초마다
    public void collectMetrics() {
        connectionManagers.forEach((serviceName, manager) -> {
            PoolStats stats = manager.getTotalStats();
            
            // 사용 중인 커넥션 수
            Gauge.builder("http.connection.active")
                .tag("service", serviceName)
                .register(meterRegistry, stats, PoolStats::getLeased);
            
            // 대기 중인 커넥션 수  
            Gauge.builder("http.connection.available")
                .tag("service", serviceName)
                .register(meterRegistry, stats, PoolStats::getAvailable);
            
            // 최대 커넥션 수
            Gauge.builder("http.connection.max")
                .tag("service", serviceName)
                .register(meterRegistry, stats, PoolStats::getMax);
            
            // 대기 중인 요청 수
            Gauge.builder("http.connection.pending")
                .tag("service", serviceName)
                .register(meterRegistry, stats, PoolStats::getPending);
        });
    }
}
```

### 2. 커넥션 풀 상태 대시보드

```java
@RestController
@RequestMapping("/admin/connection-pools")
public class ConnectionPoolDashboardController {
    
    private final Map<String, PoolingHttpClientConnectionManager> connectionManagers;
    
    @GetMapping("/status")
    public Map<String, Object> getConnectionPoolStatus() {
        Map<String, Object> status = new HashMap<>();
        
        connectionManagers.forEach((serviceName, manager) -> {
            PoolStats totalStats = manager.getTotalStats();
            
            Map<String, Object> serviceStatus = new HashMap<>();
            serviceStatus.put("maxConnections", totalStats.getMax());
            serviceStatus.put("leasedConnections", totalStats.getLeased());
            serviceStatus.put("availableConnections", totalStats.getAvailable());
            serviceStatus.put("pendingRequests", totalStats.getPending());
            
            // 각 라우트별 상세 정보
            Map<String, Object> routeStats = new HashMap<>();
            for (HttpRoute route : manager.getRoutes()) {
                PoolStats routeStat = manager.getStats(route);
                routeStats.put(route.getTargetHost().toString(), Map.of(
                    "max", routeStat.getMax(),
                    "leased", routeStat.getLeased(),
                    "available", routeStat.getAvailable(),
                    "pending", routeStat.getPending()
                ));
            }
            serviceStatus.put("routes", routeStats);
            
            status.put(serviceName, serviceStatus);
        });
        
        return status;
    }
    
    @PostMapping("/resize/{serviceName}")
    public ResponseEntity<String> resizeConnectionPool(
            @PathVariable String serviceName,
            @RequestParam int newSize) {
        
        PoolingHttpClientConnectionManager manager = connectionManagers.get(serviceName);
        if (manager != null) {
            manager.setDefaultMaxPerRoute(newSize);
            manager.setMaxTotal(newSize * 2);
            return ResponseEntity.ok("Connection pool resized to " + newSize);
        } else {
            return ResponseEntity.notFound().build();
        }
    }
}
```

### 3. 자동 알림 시스템

```java
@Component
public class ConnectionPoolAlertService {
    
    @Scheduled(fixedRate = 30000) // 30초마다 체크
    public void checkConnectionPoolHealth() {
        connectionManagers.forEach((serviceName, manager) -> {
            PoolStats stats = manager.getTotalStats();
            
            // 커넥션 사용률 체크
            double usageRatio = (double) stats.getLeased() / stats.getMax();
            if (usageRatio > 0.8) {
                alertService.sendAlert(
                    AlertLevel.WARNING,
                    String.format("%s 서비스 커넥션 사용률 높음: %.1f%%", 
                        serviceName, usageRatio * 100)
                );
            }
            
            // 대기 요청 수 체크
            if (stats.getPending() > 10) {
                alertService.sendAlert(
                    AlertLevel.CRITICAL,
                    String.format("%s 서비스에 %d개 요청이 대기 중", 
                        serviceName, stats.getPending())
                );
            }
        });
    }
}
```

## Best Practices 및 안티패턴

### ✅ 권장 사항

```java
// 1. 서비스별 별도 커넥션 풀 사용
@Configuration
public class ServiceSpecificConnectionPools {
    
    @Bean("paymentRestTemplate")
    public RestTemplate paymentRestTemplate() {
        return createRestTemplate("payment", 20, 2000, 30);
    }
    
    @Bean("userRestTemplate") 
    public RestTemplate userRestTemplate() {
        return createRestTemplate("user", 30, 3000, 60);
    }
}

// 2. 적절한 타임아웃 설정
RequestConfig.custom()
    .setConnectionRequestTimeout(3000)  // 커넥션 획득 대기
    .setConnectTimeout(5000)            // 연결 타임아웃  
    .setSocketTimeout(10000)            // 읽기 타임아웃
    .build();

// 3. 커넥션 정리 활성화
HttpClients.custom()
    .evictExpiredConnections()          // 만료된 커넥션 자동 제거
    .evictIdleConnections(60, TimeUnit.SECONDS)  // 유휴 커넥션 제거
    .build();
```

### ❌ 피해야 할 안티패턴

```java
// ❌ 모든 서비스에 동일한 커넥션 풀 사용
@Bean
public RestTemplate sharedRestTemplate() {
    // 모든 외부 서비스에 동일한 설정 사용 - 비효율적
    return new RestTemplate();
}

// ❌ 너무 긴 커넥션 획득 대기 시간
RequestConfig.custom()
    .setConnectionRequestTimeout(60000)  // 60초 대기 - 너무 김!
    .build();

// ❌ 커넥션 정리 미설정
HttpClients.custom()
    .setConnectionManager(connectionManager)
    .build();
    // evictExpiredConnections() 없음 - 메모리 누수 가능
```

## 테스트 및 검증

### 1. 커넥션 풀 부하 테스트

```java
@Test
@DisplayName("동시 요청 처리 성능 테스트")
void shouldHandleConcurrentRequestsEfficiently() throws InterruptedException {
    int concurrentRequests = 100;
    CountDownLatch latch = new CountDownLatch(concurrentRequests);
    List<Long> responseTimes = Collections.synchronizedList(new ArrayList<>());
    
    ExecutorService executorService = Executors.newFixedThreadPool(50);
    
    for (int i = 0; i < concurrentRequests; i++) {
        executorService.submit(() -> {
            try {
                long startTime = System.currentTimeMillis();
                
                String response = restTemplate.getForObject(
                    "http://external-service/api/data", String.class);
                    
                long responseTime = System.currentTimeMillis() - startTime;
                responseTimes.add(responseTime);
                
            } finally {
                latch.countDown();
            }
        });
    }
    
    latch.await(30, TimeUnit.SECONDS);
    
    // 평균 응답시간이 기대치 이하인지 확인
    double averageResponseTime = responseTimes.stream()
        .mapToLong(Long::longValue)
        .average()
        .orElse(0);
        
    assertThat(averageResponseTime).isLessThan(1000); // 1초 이내
    
    // 95% 응답시간이 기대치 이하인지 확인
    Collections.sort(responseTimes);
    long p95ResponseTime = responseTimes.get((int) (responseTimes.size() * 0.95));
    assertThat(p95ResponseTime).isLessThan(2000); // 2초 이내
}
```

## 실무 체크리스트

### 설계 시 고려사항

- [ ] 서비스별 처리 능력 분석 및 문서화
- [ ] 커넥션 풀 크기 결정 (연동 서비스 TPS 고려)
- [ ] 커넥션 획득 대기 시간 설정 (1-5초 권장)
- [ ] Keep-Alive 시간 설정 (서버 설정과 일치)
- [ ] 서비스별 차등 설정 필요성 검토

### 구현 시 확인사항

- [ ] 서비스별 별도 커넥션 풀 구성
- [ ] 적절한 타임아웃 값 설정
- [ ] 커넥션 정리 메커니즘 활성화
- [ ] 메트릭 수집 및 모니터링 설정
- [ ] 동적 조정 로직 구현

### 운영 시 모니터링

- [ ] 커넥션 풀 사용률 실시간 추적
- [ ] 대기 요청 수 모니터링
- [ ] 평균/P95 응답시간 추적
- [ ] 커넥션 풀 고갈 알림 설정
- [ ] 정기적인 성능 테스트 및 튜닝
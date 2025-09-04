[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 4-9. 연동 서비스 이중화 (External Service Redundancy)

## 개요

**서비스가 대량 트래픽을 처리할 만큼 성장했다면, 연동 서비스의 이중화도 고려할 법하다.**

### 연동 서비스 이중화가 필요한 시점

```
서비스 성장 단계별 연동 전략
┌─────────────────────────────────────────────────────────────┐
│ Stage 1: 초기 (1k DAU)                                        │
│ ├─ 단일 연동 서비스                                          │
│ └─ 기본적인 에러 핸들링                                       │
├─────────────────────────────────────────────────────────────┤
│ Stage 2: 성장 (10k DAU)                                      │
│ ├─ 재시도, 타임아웃 최적화                                    │
│ └─ 서킷 브레이커 도입                                        │
├─────────────────────────────────────────────────────────────┤
│ Stage 3: 확장 (100k DAU) ← 이중화 고려 시점                   │
│ ├─ 연동 서비스 이중화                                        │
│ └─ 로드밸런싱, Failover                                      │
└─────────────────────────────────────────────────────────────┘
```

## 이중화 고려사항

**연동 서비스의 이중화를 진행할 경우 아래 2가지를 고려해야한다.**

### 1. 해당 기능이 서비스의 핵심인지

#### 핵심 기능 분류 매트릭스

|기능 유형|비즈니스 영향도|사용자 영향도|이중화 우선순위|예시|
|---|---|---|---|---|
|**Mission Critical**|높음|높음|🔴 **필수**|결제, 로그인 인증|
|**Business Critical**|높음|중간|🟡 **권장**|주문 처리, 재고 관리|
|**User Experience**|중간|높음|🟡 **권장**|알림, 추천 시스템|
|**Supporting**|중간|낮음|🟢 **선택**|로그 수집, 분석|
|**Nice-to-have**|낮음|낮음|⚫ **불필요**|A/B 테스트, 배너|

### 2. 이중화 비용이 감당 가능한 수준인지

#### 이중화 옵션별 비용 비교

| 이중화 방식              | 초기 비용    | 운영 비용 | 복잡도 | 가용성    | 적용 케이스            |
| ------------------- | -------- | ----- | --- | ------ | ----------------- |
| **Active-Active**   | 높음(2x)   | 높음    | 높음  | 99.99% | Mission Critical  |
| **Active-Standby**  | 중간(1.3x) | 중간    | 중간  | 99.9%  | Business Critical |
| **Multi-Vendor**    | 중간(1.5x) | 높음    | 높음  | 99.95% | 종속성 해결 필요 시       |
| **Circuit Breaker** | 낮음(0.1x) | 낮음    | 낮음  | 99%    | Supporting 서비스    |

## 이중화 패턴 구현

### 1. Active-Active 패턴

```java
@Component
public class ActiveActiveRedundantService {
    
    private final List<ExternalServiceClient> serviceClients;
    private final LoadBalancer loadBalancer;
    private final HealthChecker healthChecker;
    
    public <T> T callService(String operation, Object request, Class<T> responseType) {
        List<ExternalServiceClient> healthyClients = serviceClients.stream()
            .filter(healthChecker::isHealthy)
            .collect(Collectors.toList());
            
        if (healthyClients.isEmpty()) {
            throw new AllServicesUnavailableException("모든 연동 서비스 불가");
        }
        
        ExternalServiceClient selectedClient = loadBalancer.select(healthyClients);
        
        try {
            return selectedClient.call(operation, request, responseType);
        } catch (Exception e) {
            // 실패 시 다른 서비스로 재시도
            return fallbackToOtherService(operation, request, responseType, selectedClient);
        }
    }
    
    private <T> T fallbackToOtherService(
            String operation, Object request, Class<T> responseType, 
            ExternalServiceClient failedClient) {
        
        List<ExternalServiceClient> remainingClients = serviceClients.stream()
            .filter(client -> !client.equals(failedClient))
            .filter(healthChecker::isHealthy)
            .collect(Collectors.toList());
            
        for (ExternalServiceClient client : remainingClients) {
            try {
                return client.call(operation, request, responseType);
            } catch (Exception e) {
                log.warn("Fallback 실패: {}, 다음 서비스 시도", client.getName());
                continue;
            }
        }
        
        throw new AllServicesFailedException("모든 대체 서비스 실패");
    }
}
```

### 2. Active-Standby 패턴

```java
@Component
public class ActiveStandbyRedundantService {
    
    @Primary
    private final ExternalServiceClient primaryClient;
    
    @Qualifier("standby")
    private final ExternalServiceClient standbyClient;
    
    @CircuitBreaker(name = "primary-service", fallbackMethod = "callStandbyService")
    public <T> T callPrimaryService(String operation, Object request, Class<T> responseType) {
        try {
            return primaryClient.call(operation, request, responseType);
        } catch (Exception e) {
            log.error("Primary 서비스 호출 실패: {}", e.getMessage());
            throw e; // Circuit Breaker가 fallback 트리거
        }
    }
    
    public <T> T callStandbyService(String operation, Object request, 
                                   Class<T> responseType, Exception exception) {
        log.info("Standby 서비스로 전환: {}", exception.getMessage());
        
        try {
            return standbyClient.call(operation, request, responseType);
        } catch (Exception e) {
            log.error("Standby 서비스도 실패: {}", e.getMessage());
            throw new BothServicesFailedException("Primary/Standby 모두 실패", e);
        }
    }
    
    @Scheduled(fixedRate = 30000) // 30초마다 Primary 상태 확인
    public void monitorPrimaryHealth() {
        if (healthChecker.isHealthy(primaryClient)) {
            // Primary가 회복되면 Circuit Breaker 상태 확인 후 전환
            if (shouldSwitchBackToPrimary()) {
                circuitBreakerRegistry.circuitBreaker("primary-service")
                    .transitionToHalfOpenState();
                log.info("Primary 서비스 복구 감지, Half-Open으로 전환");
            }
        }
    }
}
```

### 3. Multi-Vendor 패턴

```java
@Component
public class MultiVendorRedundantService {
    
    private final Map<String, PaymentServiceClient> paymentVendors;
    private final VendorSelectorStrategy vendorSelector;
    
    public MultiVendorRedundantService() {
        this.paymentVendors = Map.of(
            "stripe", new StripePaymentClient(),
            "toss", new TossPaymentClient(),
            "kakao", new KakaoPaymentClient()
        );
    }
    
    public PaymentResponse processPayment(PaymentRequest request) {
        List<String> availableVendors = getAvailableVendors();
        
        for (String vendorName : vendorSelector.selectVendors(availableVendors, request)) {
            try {
                PaymentServiceClient vendor = paymentVendors.get(vendorName);
                PaymentResponse response = vendor.processPayment(request);
                
                if (response.isSuccess()) {
                    // 성공 시 벤더 정보 기록
                    recordSuccessfulVendor(vendorName, request, response);
                    return response;
                }
                
            } catch (Exception e) {
                log.warn("벤더 {} 처리 실패: {}", vendorName, e.getMessage());
                recordVendorFailure(vendorName, request, e);
                continue;
            }
        }
        
        throw new AllVendorsFailedException("모든 결제 벤더 처리 실패");
    }
    
    private List<String> getAvailableVendors() {
        return paymentVendors.entrySet().stream()
            .filter(entry -> healthChecker.isHealthy(entry.getValue()))
            .filter(entry -> !isVendorBlocked(entry.getKey()))
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());
    }
}
```

## 스마트 라우팅 전략

### 1. 부하 분산 기반 라우팅

```java
@Component
public class LoadBalancedRouter {
    
    private final ServiceMetricsCollector metricsCollector;
    
    public ExternalServiceClient selectService(List<ExternalServiceClient> availableServices) {
        return availableServices.stream()
            .min(Comparator.comparing(this::calculateScore))
            .orElseThrow(() -> new NoServiceAvailableException());
    }
    
    private double calculateScore(ExternalServiceClient service) {
        ServiceMetrics metrics = metricsCollector.getMetrics(service.getName());
        
        // 가중치 기반 점수 계산
        double responseTimeScore = normalizeResponseTime(metrics.getAvgResponseTime());
        double errorRateScore = metrics.getErrorRate();
        double loadScore = normalizeLoad(metrics.getCurrentLoad());
        
        return (responseTimeScore * 0.4) + (errorRateScore * 0.3) + (loadScore * 0.3);
    }
}
```

### 2. 지역 기반 라우팅

```java
@Component
public class GeographicRouter {
    
    public ExternalServiceClient selectNearestService(
            List<ExternalServiceClient> services, String userRegion) {
        
        return services.stream()
            .filter(service -> isServiceAvailable(service, userRegion))
            .min(Comparator.comparing(service -> calculateLatency(service, userRegion)))
            .orElse(selectGlobalService(services));
    }
    
    private double calculateLatency(ExternalServiceClient service, String userRegion) {
        // 지역별 네트워크 지연시간 계산
        Map<String, Double> regionLatency = service.getRegionLatencyMap();
        return regionLatency.getOrDefault(userRegion, Double.MAX_VALUE);
    }
}
```

### 3. 비용 기반 라우팅

```java
@Component 
public class CostOptimizedRouter {
    
    public ExternalServiceClient selectCostEffectiveService(
            List<ExternalServiceClient> services, TransactionContext context) {
        
        return services.stream()
            .filter(service -> meetsPerformanceThreshold(service))
            .min(Comparator.comparing(service -> calculateTotalCost(service, context)))
            .orElse(selectBestPerformanceService(services));
    }
    
    private double calculateTotalCost(ExternalServiceClient service, TransactionContext context) {
        double transactionCost = service.getTransactionCost(context.getAmount());
        double operationalCost = service.getOperationalCost();
        double penaltyCost = calculatePenaltyCost(service, context);
        
        return transactionCost + operationalCost + penaltyCost;
    }
}
```

## 모니터링 및 운영

### 1. 이중화 상태 모니터링

```java
@Component
public class RedundancyMonitoring {
    
    @Scheduled(fixedRate = 10000) // 10초마다
    public void monitorRedundancyHealth() {
        Map<String, List<ServiceHealthStatus>> serviceGroups = 
            redundancyGroupManager.getAllServiceGroups();
            
        serviceGroups.forEach((groupName, services) -> {
            long healthyCount = services.stre
```
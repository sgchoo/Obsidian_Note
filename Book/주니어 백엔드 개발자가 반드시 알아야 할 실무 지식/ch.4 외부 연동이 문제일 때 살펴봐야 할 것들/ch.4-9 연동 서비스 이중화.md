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

java

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

java

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

java

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

java

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

java

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

java

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

java

```java
@Component
public class RedundancyMonitoring {
    
    @Scheduled(fixedRate = 10000) // 10초마다
    public void monitorRedundancyHealth() {
        Map<String, List<ServiceHealthStatus>> serviceGroups = 
            redundancyGroupManager.getAllServiceGroups();
            
        serviceGroups.forEach((groupName, services) -> {
            long healthyCount = services.stream()
                .filter(ServiceHealthStatus::isHealthy)
                .count();
                
            double healthRatio = (double) healthyCount / services.size();
            
            // 메트릭 기록
            Gauge.builder("redundancy.health.ratio")
                .tag("group", groupName)
                .register(meterRegistry, () -> healthRatio);
            
            // 임계치 알림
            if (healthRatio < 0.5) {
                alertService.sendCriticalAlert(
                    String.format("이중화 그룹 %s의 가용 서비스가 50%% 미만입니다", groupName)
                );
            }
        });
    }
}
```

### 2. 자동 복구 시스템

java

```java
@Component
public class AutoRecoverySystem {
    
    @EventListener
    public void handleServiceRecovery(ServiceRecoveryEvent event) {
        String serviceName = event.getServiceName();
        RedundancyGroup group = redundancyGroupManager.getGroup(serviceName);
        
        if (group.getStrategy() == RedundancyStrategy.ACTIVE_STANDBY) {
            // Active-Standby에서는 Primary 복구 시 점진적 전환
            scheduleGradualSwitchback(serviceName);
        } else if (group.getStrategy() == RedundancyStrategy.ACTIVE_ACTIVE) {
            // Active-Active에서는 즉시 로드밸런싱에 포함
            loadBalancer.addService(serviceName);
        }
    }
    
    private void scheduleGradualSwitchback(String serviceName) {
        // 5분간 점진적으로 트래픽을 Primary로 전환
        scheduler.scheduleAtFixedRate(() -> {
            double currentRatio = trafficRouter.getPrimaryTrafficRatio(serviceName);
            if (currentRatio < 1.0) {
                trafficRouter.increasePrimaryTraffic(serviceName, 0.2); // 20%씩 증가
            }
        }, 1, 1, TimeUnit.MINUTES);
    }
}
```

## 테스트 전략

### 1. Chaos Engineering

java

```java
@Component
public class ChaosTestingService {
    
    @EventListener
    @ConditionalOnProperty("chaos.testing.enabled")
    public void simulateServiceFailure(ChaosTestEvent event) {
        String targetService = event.getTargetService();
        Duration failureDuration = event.getFailureDuration();
        
        log.info("Chaos 테스트 시작: {} 서비스 {} 동안 장애 시뮬레이션", 
            targetService, failureDuration);
        
        // 서비스 강제 실패
        serviceRegistry.markAsDown(targetService);
        
        // 지정된 시간 후 복구
        scheduler.schedule(() -> {
            serviceRegistry.markAsUp(targetService);
            log.info("Chaos 테스트 종료: {} 서비스 복구", targetService);
        }, failureDuration.toMillis(), TimeUnit.MILLISECONDS);
    }
}
```

### 2. 이중화 시나리오 테스트

java

```java
@SpringBootTest
class RedundancyScenarioTest {
    
    @Test
    @DisplayName("Primary 서비스 장애 시 Standby로 자동 전환")
    void shouldFailoverToStandbyOnPrimaryFailure() {
        // Given: Primary 서비스가 정상 상태
        when(primaryService.isHealthy()).thenReturn(true);
        when(standbyService.isHealthy()).thenReturn(true);
        
        // When: Primary 서비스 장애 발생
        when(primaryService.call(any())).thenThrow(new ServiceUnavailableException());
        
        // Then: Standby 서비스로 자동 전환
        TestRequest request = new TestRequest("test");
        TestResponse response = redundantService.call(request);
        
        verify(standbyService).call(request);
        assertThat(response).isNotNull();
    }
    
    @Test
    @DisplayName("모든 서비스 장애 시 적절한 예외 처리")
    void shouldThrowExceptionWhenAllServicesFail() {
        // Given: 모든 서비스 장애
        when(primaryService.call(any())).thenThrow(new ServiceUnavailableException());
        when(standbyService.call(any())).thenThrow(new ServiceUnavailableException());
        
        // When & Then: 적절한 예외 발생
        assertThatThrownBy(() -> redundantService.call(new TestRequest("test")))
            .isInstanceOf(AllServicesFailedException.class);
    }
}
```

## 실무 의사결정 가이드

### 이중화 도입 결정 플로우차트

```
서비스 이중화 검토 프로세스
┌─────────────────────────────────┐
│ 1. 트래픽 임계치 확인              │
│ (일 100만+ 요청)                 │
└─────────┬───────────────────────┘
          │ Yes
          ↓
┌─────────────────────────────────┐
│ 2. 핵심 기능 여부 확인            │
│ (Mission/Business Critical)     │
└─────────┬───────────────────────┘
          │ Yes
          ↓
┌─────────────────────────────────┐
│ 3. ROI 분석                     │
│ (비용 vs 장애 손실)              │
└─────────┬───────────────────────┘
          │ ROI > 30%
          ↓
┌─────────────────────────────────┐
│ 4. 이중화 방식 선택               │
│ - Active-Active                 │
│ - Active-Standby                │  
│ - Multi-Vendor                  │
└─────────────────────────────────┘
```

### 서비스별 이중화 전략 매트릭스

java

```java
@Component
public class RedundancyStrategyRecommender {
    
    public RedundancyStrategy recommend(ServiceProfile profile) {
        if (profile.getCriticalityLevel() == CriticalityLevel.MISSION_CRITICAL) {
            if (profile.getDailyTraffic() > 10_000_000) {
                return RedundancyStrategy.ACTIVE_ACTIVE;
            } else {
                return RedundancyStrategy.ACTIVE_STANDBY;
            }
        }
        
        if (profile.getCriticalityLevel() == CriticalityLevel.BUSINESS_CRITICAL) {
            if (profile.getVendorLockInRisk() > 0.7) {
                return RedundancyStrategy.MULTI_VENDOR;
            } else {
                return RedundancyStrategy.ACTIVE_STANDBY;
            }
        }
        
        return RedundancyStrategy.CIRCUIT_BREAKER_ONLY;
    }
}
```

## 실무 체크리스트

### 이중화 도입 검토

- [ ]  일일 트래픽이 임계치(100만 요청) 초과
- [ ]  서비스 핵심 기능 여부 분석 완료
- [ ]  ROI 분석 및 비용 정당화 검토
- [ ]  기술팀 역량 및 운영 복잡도 고려
- [ ]  비즈니스 승인 및 예산 확보

### 설계 시 고려사항

- [ ]  적절한 이중화 패턴 선택
- [ ]  라우팅 전략 정의
- [ ]  헬스체크 및 장애 감지 메커니즘
- [ ]  자동 복구 시나리오 설계
- [ ]  모니터링 및 알림 체계

### 구현 시 확인사항

- [ ]  각 서비스별 이중화 구현
- [ ]  로드밸런서 설정 및 테스트
- [ ]  Circuit Breaker와 연계 구현
- [ ]  메트릭 수집 및 대시보드 구성
- [ ]  장애 시나리오 테스트

### 운영 시 모니터링

- [ ]  이중화 그룹별 가용성 추적
- [ ]  자동 전환 성공률 모니터링
- [ ]  비용 효율성 정기 검토
- [ ]  Chaos Engineering 정기 실시
- [ ]  이중화 전략 최적화 및 튜닝
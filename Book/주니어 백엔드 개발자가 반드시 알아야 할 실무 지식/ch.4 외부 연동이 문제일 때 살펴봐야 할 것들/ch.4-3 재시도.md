[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 4-3. 재시도 (Retry)

## 개요

재시도를 통해 연동 실패를 줄일 수 있지만, **항상 재시도를 할 수 있는 것은 아니다**.

## 재시도 가능한 상황

### 1. 단순 조회 기능

```java
@Retryable(
    value = {ConnectException.class, SocketTimeoutException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 2.0)
)
public UserInfo getUserInfo(String userId) {
    return userApiClient.getUser(userId);
}
```

**특징:**

- 데이터 변경이 없으므로 안전하게 재시도 가능
- 네트워크 오류나 일시적 장애 시 효과적

### 2. 연결 타임아웃

```java
@Service
public class ConnectionRetryService {
    
    @Retryable(
        value = {ConnectTimeoutException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 500, multiplier = 1.5)
    )
    public String callExternalService() {
        try {
            return externalServiceClient.getData();
        } catch (ConnectTimeoutException e) {
            log.warn("연결 타임아웃 발생, 재시도 예정: {}", e.getMessage());
            throw e;
        }
    }
}
```

**특징:**

- 서버 연결 문제로 요청이 전송되지 않은 상황
- 요청이 서버에 도달하지 않았으므로 안전하게 재시도

### 3. 멱등성을 가진 변경 기능

```java
@Component
public class IdempotentOperationService {
    
    @Retryable(
        value = {HttpServerErrorException.class},
        maxAttempts = 2,
        backoff = @Backoff(delay = 2000)
    )
    public void updateUserStatus(@RequestHeader("Idempotency-Key") String key,
                                @RequestBody UserStatusRequest request) {
        userApiClient.updateStatus(key, request);
    }
}
```

**특징:**

- PUT, DELETE 등 멱등한 HTTP 메서드
- 동일한 요청을 여러 번 보내도 결과가 같음
- Idempotency Key를 활용한 중복 방지

## 재시도 불가능한 상황

### 1. 비멱등성 작업

```java
// ❌ 재시도 하면 안 되는 예시
public class NonIdempotentService {
    
    // 계좌 잔액 차감 - 재시도 시 중복 차감 위험
    public void deductBalance(String accountId, BigDecimal amount) {
        accountApiClient.deduct(accountId, amount);  // 재시도 금지
    }
    
    // 주문 생성 - 재시도 시 중복 주문 생성 위험  
    public OrderResponse createOrder(OrderRequest request) {
        return orderApiClient.createOrder(request);  // 재시도 금지
    }
}
```

### 2. 클라이언트 오류 (4xx)

```java
@Component
public class ErrorHandlingService {
    
    @Retryable(
        value = {HttpServerErrorException.class},  // 5xx만 재시도
        exclude = {HttpClientErrorException.class}, // 4xx는 재시도 안 함
        maxAttempts = 3
    )
    public String callApiWithValidation() {
        return externalApiClient.getData();
    }
}
```

## 재시도 전략: 횟수와 간격

### 1. 적절한 재시도 횟수

**대부분 2~3번이 적당하다.**

```java
@Configuration
@EnableRetry
public class RetryConfig {
    
    // 일반적인 경우: 최대 3회
    @Bean
    public RetryTemplate defaultRetryTemplate() {
        return RetryTemplate.builder()
            .maxAttempts(3)
            .fixedBackoff(1000)
            .retryOn(ConnectException.class)
            .retryOn(SocketTimeoutException.class)
            .build();
    }
    
    // 중요한 작업: 최대 5회
    @Bean
    public RetryTemplate criticalRetryTemplate() {
        return RetryTemplate.builder()
            .maxAttempts(5)
            .exponentialBackoff(1000, 2, 10000)
            .retryOn(HttpServerErrorException.class)
            .build();
    }
}
```

### 2. 재시도 간격 전략

#### Fixed Backoff (고정 간격)

```java
@Retryable(
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000)  // 1초 고정 간격
)
public String fixedBackoffRetry() {
    return externalApiClient.getData();
}
```

#### Exponential Backoff (지수적 증가)

**바로 재시도 하는 것보다 어느 정도 간격을 두고 재시도를 한다면, 성공할 가능성이 높아진다.**

```java
@Retryable(
    maxAttempts = 3,
    backoff = @Backoff(
        delay = 1000,      // 초기 대기시간: 1초
        multiplier = 2.0,  // 배수: 2배씩 증가
        maxDelay = 10000   // 최대 대기시간: 10초
    )
)
public String exponentialBackoffRetry() {
    return externalApiClient.getData();
}
```

**재시도 간격:**

- 1차 실패 → 1초 대기 → 2차 시도
- 2차 실패 → 2초 대기 → 3차 시도
- 3차 실패 → 포기

#### Random Jitter (랜덤 지연)

```java
@Component
public class JitterRetryService {
    
    private final RetryTemplate retryTemplate;
    
    public JitterRetryService() {
        this.retryTemplate = RetryTemplate.builder()
            .maxAttempts(3)
            .exponentialBackoff(1000, 2, 10000)
            .withJitter() // 랜덤 지연 추가
            .build();
    }
    
    public String callWithJitter() {
        return retryTemplate.execute(context -> {
            return externalApiClient.getData();
        });
    }
}
```

## 연동 서비스 성능 고려사항

**연동 서비스의 성능 상황도 함께 고려해야 한다.**

### 1. Circuit Breaker와 연계

```java
@Component
public class ResillientService {
    
    @CircuitBreaker(name = "external-service")
    @Retryable(
        maxAttempts = 2,  // Circuit이 OPEN되기 전 최소한의 재시도
        backoff = @Backoff(delay = 1000)
    )
    public String callWithCircuitBreaker() {
        return externalApiClient.getData();
    }
    
    @Recover
    public String recover(Exception ex) {
        log.error("모든 재시도 실패, Circuit Breaker 동작: {}", ex.getMessage());
        return "기본값";
    }
}
```

### 2. Rate Limiting 고려

```java
@Component
public class RateLimitedRetryService {
    
    private final RateLimiter rateLimiter = RateLimiter.create(10.0); // 초당 10회
    
    @Retryable(
        value = {RateLimitExceededException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 5000) // 5초 대기 (Rate Limit 고려)
    )
    public String callWithRateLimit() {
        if (!rateLimiter.tryAcquire()) {
            throw new RateLimitExceededException("Rate limit exceeded");
        }
        return externalApiClient.getData();
    }
}
```

### 3. 서비스별 맞춤 설정

```yaml
# application.yml
retry-config:
  payment-service:
    max-attempts: 5
    initial-delay: 2000
    multiplier: 2.0
    max-delay: 30000
  
  notification-service:
    max-attempts: 2
    initial-delay: 500
    multiplier: 1.5
    max-delay: 2000
  
  user-service:
    max-attempts: 3
    initial-delay: 1000
    multiplier: 2.0
    max-delay: 10000
```

## 고급 재시도 패턴

### 1. 조건부 재시도

```java
@Component
public class ConditionalRetryService {
    
    @Retryable(
        maxAttempts = 3,
        include = {HttpServerErrorException.class},
        condition = "#{root.cause.rawStatusCode >= 500 and root.cause.rawStatusCode < 600}"
    )
    public String conditionalRetry() {
        return externalApiClient.getData();
    }
}
```

### 2. 커스텀 재시도 로직

```java
@Component
public class CustomRetryService {
    
    public String smartRetry() {
        RetryPolicy retryPolicy = RetryPolicy.builder()
            .handle(ConnectException.class)
            .handle(SocketTimeoutException.class)
            .handleResultIf(result -> result == null)
            .withMaxRetries(3)
            .withBackoff(Duration.ofSeconds(1), Duration.ofSeconds(10), 2.0)
            .build();
            
        return Failsafe.with(retryPolicy)
            .onRetry(event -> log.info("재시도 시도: {}", event.getAttemptCount()))
            .onFailure(event -> log.error("모든 재시도 실패: {}", event.getFailure()))
            .get(() -> externalApiClient.getData());
    }
}
```

## 모니터링 및 메트릭

### 1. 재시도 현황 추적

```java
@Component
public class RetryMetrics {
    
    private final Counter retryCounter;
    private final Timer retryTimer;
    
    public RetryMetrics(MeterRegistry meterRegistry) {
        this.retryCounter = Counter.builder("retry.attempts")
            .tag("service", "external")
            .register(meterRegistry);
        
        this.retryTimer = Timer.builder("retry.duration")
            .tag("service", "external")
            .register(meterRegistry);
    }
    
    @EventListener
    public void handleRetryEvent(RetryEvent event) {
        retryCounter.increment(
            Tags.of(
                "attempt", String.valueOf(event.getAttemptCount()),
                "result", event.isSuccess() ? "success" : "failure"
            )
        );
    }
}
```

### 2. 대시보드 항목

- 재시도 발생률 (시간당/일당)
- 재시도 성공률
- 평균 재시도 횟수
- 재시도로 인한 지연시간

## 실무 체크리스트

### 재시도 적용 전 검토사항

- [ ] 해당 작업이 멱등성을 보장하는가?
- [ ] 재시도로 인한 부작용은 없는가?
- [ ] 적절한 재시도 횟수 설정 (2-3회)
- [ ] 지수적 백오프 패턴 적용
- [ ] 연동 서비스의 처리 능력 고려

### 재시도 구현 시 확인사항

- [ ] 재시도 대상 예외 명확히 정의
- [ ] 클라이언트 오류(4xx) 재시도 제외
- [ ] Circuit Breaker와 연계 구현
- [ ] 재시도 현황 모니터링 설정
- [ ] Fallback 메커니즘 구현

### 운영 시 모니터링

- [ ] 재시도 발생률 추적
- [ ] 재시도 성공률 모니터링
- [ ] 연동 서비스 부하 상황 확인
- [ ] 재시도 설정값 주기적 튜닝
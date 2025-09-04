[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 4-2. 타임아웃 (Timeout)

## 문제 상황

외부 서비스의 요청 시간이 **5분**이라고 예를 들면, 우리 모든 서비스의 응답 시간이 증가한다.

### 왜 문제가 되는가?

- **리소스 점유**: 긴 대기시간 동안 스레드가 블로킹됨
- **사용자 경험 악화**: 응답 지연으로 인한 서비스 품질 저하
- **시스템 부하 증가**: 대기 중인 요청들이 쌓여 전체 성능 저하

## 기능별 타임아웃 설정 전략

### 1. 결제 요청 (긴 타임아웃 필요)

결제 요청 등의 요청 시간이 오래 걸리는 API라면 타임아웃을 길게 가져가야 한다.

```java
// 결제 API - 30초 타임아웃
@Configuration
public class PaymentApiConfig {
    
    @Bean("paymentRestTemplate")
    public RestTemplate paymentRestTemplate() {
        HttpComponentsClientHttpRequestFactory factory = 
            new HttpComponentsClientHttpRequestFactory();
        factory.setConnectTimeout(5000);    // 연결: 5초
        factory.setReadTimeout(30000);      // 읽기: 30초
        
        return new RestTemplate(factory);
    }
}
```

### 2. 일반 조회 API (짧은 타임아웃)

```java
// 일반 조회 API - 3초 타임아웃
@Bean("defaultRestTemplate")
public RestTemplate defaultRestTemplate() {
    HttpComponentsClientHttpRequestFactory factory = 
        new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(1000);    // 연결: 1초
    factory.setReadTimeout(3000);       // 읽기: 3초
    
    return new RestTemplate(factory);
}
```

### 3. 실시간 알림 API (매우 짧은 타임아웃)

```java
// 실시간 알림 API - 1초 타임아웃
@Bean("notificationRestTemplate")
public RestTemplate notificationRestTemplate() {
    HttpComponentsClientHttpRequestFactory factory = 
        new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(500);     // 연결: 0.5초
    factory.setReadTimeout(1000);       // 읽기: 1초
    
    return new RestTemplate(factory);
}
```

## 기능 특성별 타임아웃 가이드

### 비즈니스 중요도에 따른 분류

|기능 유형|연결 타임아웃|읽기 타임아웃|예시|
|---|---|---|---|
|**결제/정산**|3초|30-60초|결제 처리, 정산 API|
|**인증/보안**|2초|10-15초|본인인증, OTP 검증|
|**파일 업로드**|5초|60-120초|이미지/문서 업로드|
|**일반 조회**|1초|3-5초|상품 정보, 사용자 정보|
|**실시간 데이터**|0.5초|1-2초|알림, 채팅|

### 설정 방법별 구현

#### 1. application.yml을 통한 설정

```yaml
# application.yml
external-api:
  timeout:
    payment:
      connect: 3000
      read: 30000
    general:
      connect: 1000
      read: 3000
    notification:
      connect: 500
      read: 1000
```

#### 2. WebClient 사용 시 (Spring WebFlux)

```java
@Service
public class ExternalApiService {
    
    public Mono<String> callPaymentApi() {
        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(
                HttpClient.create()
                    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
                    .responseTimeout(Duration.ofSeconds(30))
            ))
            .build()
            .get()
            .uri("/payment")
            .retrieve()
            .bodyToMono(String.class);
    }
}
```

#### 3. Feign Client 사용 시

```java
@FeignClient(
    name = "payment-service",
    configuration = PaymentFeignConfig.class
)
public interface PaymentClient {
    @GetMapping("/payment")
    PaymentResponse getPayment(@RequestParam String id);
}

@Configuration
public class PaymentFeignConfig {
    
    @Bean
    public Request.Options requestOptions() {
        return new Request.Options(
            3000,   // 연결 타임아웃
            30000   // 읽기 타임아웃
        );
    }
}
```

## 타임아웃 설정 시 고려사항

### 1. 네트워크 환경

- **내부 네트워크**: 짧은 타임아웃 (1-3초)
- **외부 네트워크**: 긴 타임아웃 (5-10초)
- **해외 서비스**: 더 긴 타임아웃 (10-30초)

### 2. 서비스 특성

- **동기 처리**: 사용자 대기시간 고려하여 짧게 설정
- **비동기 처리**: 좀 더 여유있게 설정
- **배치 처리**: 안정성 위주로 길게 설정

### 3. 장애 대응

```java
@Component
public class TimeoutHandlingService {
    
    @Retryable(
        value = {SocketTimeoutException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000)
    )
    public String callWithRetry() {
        try {
            return externalApiClient.getData();
        } catch (SocketTimeoutException e) {
            log.warn("타임아웃 발생, 재시도 예정: {}", e.getMessage());
            throw e;
        }
    }
}
```

## 모니터링 및 튜닝

### 1. 타임아웃 발생률 모니터링

```java
@Component
public class TimeoutMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Counter timeoutCounter;
    
    public TimeoutMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.timeoutCounter = Counter.builder("external.api.timeout")
            .tag("service", "payment")
            .register(meterRegistry);
    }
    
    public void recordTimeout(String apiName) {
        timeoutCounter.increment(Tags.of("api", apiName));
    }
}
```

### 2. 응답시간 히스토그램

```java
Timer.Sample sample = Timer.start(meterRegistry);
try {
    // API 호출
    String result = externalApi.call();
    return result;
} finally {
    sample.stop(Timer.builder("external.api.duration")
        .tag("service", "payment")
        .register(meterRegistry));
}
```

## 실무 체크리스트

- [ ] 기능별로 적절한 타임아웃 시간 설정
- [ ] 연결 타임아웃과 읽기 타임아웃 구분하여 설정
- [ ] 비즈니스 중요도에 따른 차등 적용
- [ ] 타임아웃 발생 시 재시도 로직 구현
- [ ] 타임아웃 발생률 모니터링 설정
- [ ] 주기적인 타임아웃 설정값 검토 및 튜닝
- [ ] 장애 상황 대비 Fallback 로직 준비
[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 4-7. 외부 연동이 느려질 때 DB 커넥션 문제 (DB Connection Pool Issues with Slow External Integration)

## 핵심 문제

**DB 트랜잭션 범위 안에서 외부 연동을 수행할 때, 외부 연동이 느려지면 DB 커넥션 풀이 부족해질 수 있다.**

### 문제 발생 시나리오

```java
// ❌ 문제가 되는 패턴
@Service
@Transactional  // DB 커넥션이 오래 점유됨
public class ProblematicOrderService {
    
    public OrderResult processOrder(OrderRequest request) {
        // 1. DB 커넥션 획득 및 트랜잭션 시작
        Order order = orderRepository.save(new Order(request));
        
        // 2. 외부 연동 (30초 소요) - DB 커넥션 계속 점유 중!
        PaymentResponse paymentResponse = paymentApiClient.processPayment(
            PaymentRequest.builder()
                .orderId(order.getId())
                .amount(order.getAmount())
                .build()
        );
        
        // 3. 외부 연동 완료 후에야 DB 커넥션 반환
        order.updatePaymentInfo(paymentResponse);
        return OrderResult.success(order);
        
        // 문제: 30초 동안 DB 커넥션이 다른 요청에서 사용 불가
    }
}
```

### DB 커넥션 풀 고갈 과정

```
DB Connection Pool (Size: 10)
┌─────────────────────────────────────────────────────────────┐
│ [Conn1] [Conn2] [Conn3] [Conn4] [Conn5] [Conn6] [Conn7] ... │
│   ↓       ↓       ↓       ↓       ↓       ↓       ↓         │
│ 외부API   외부API   외부API   외부API   외부API   외부API   외부API     │
│ 30초대기  30초대기  30초대기  30초대기  30초대기  30초대기  30초대기     │
└─────────────────────────────────────────────────────────────┘

결과: 11번째 요청부터 대기 상태 → 서비스 전체 지연 또는 장애
```

## 해결 방안 1: 트랜잭션 분리

**DB연동과 무관하게 외부 연동을 실행할 수 있다면, DB 커넥션을 사용 전이나 후에 실행하는 방안도 고려해볼 수 있다.**

### 1-1. 외부 연동 선행 처리

```java
@Service
public class OptimizedOrderService {
    
    public OrderResult processOrderWithPreIntegration(OrderRequest request) {
        // 1. 트랜잭션 범위 외부에서 외부 연동 먼저 수행
        PaymentResponse paymentResponse = processPaymentOutsideTransaction(request);
        
        if (!paymentResponse.isSuccess()) {
            return OrderResult.failure("결제 처리 실패");
        }
        
        // 2. 외부 연동 성공 후 짧은 DB 트랜잭션으로 데이터 저장
        return saveOrderInTransaction(request, paymentResponse);
    }
    
    private PaymentResponse processPaymentOutsideTransaction(OrderRequest request) {
        try {
            return paymentApiClient.processPayment(
                PaymentRequest.builder()
                    .temporaryOrderId(generateTempOrderId()) // 임시 ID 사용
                    .amount(request.getAmount())
                    .idempotencyKey(request.getIdempotencyKey())
                    .build()
            );
        } catch (Exception e) {
            log.error("외부 결제 처리 실패: {}", e.getMessage());
            throw new PaymentException("결제 처리 실패", e);
        }
    }
    
    @Transactional
    private OrderResult saveOrderInTransaction(
            OrderRequest request, PaymentResponse paymentResponse) {
        try {
            // 빠른 DB 작업만 수행 (100ms 이내)
            Order order = orderRepository.save(new Order(request));
            order.updatePaymentInfo(paymentResponse);
            
            return OrderResult.success(order);
            
        } catch (Exception e) {
            // DB 저장 실패 시 결제 취소 (비동기)
            asyncCompensationService.cancelPayment(
                paymentResponse.getPaymentId(), "DB 저장 실패");
            throw e;
        }
    }
}
```

### 1-2. 외부 연동 후행 처리 (이벤트 기반)

```java
@Service
public class EventDrivenOrderService {
    
    @Transactional
    public OrderResult processOrderWithPostIntegration(OrderRequest request) {
        // 1. 빠른 DB 트랜잭션으로 주문 생성
        Order order = orderRepository.save(
            Order.builder()
                .customerId(request.getCustomerId())
                .amount(request.getAmount())
                .status(OrderStatus.PAYMENT_PENDING)
                .build()
        );
        
        // 2. 트랜잭션 커밋 후 비동기로 외부 연동 수행
        applicationEventPublisher.publishEvent(
            new OrderCreatedEvent(order.getId(), request)
        );
        
        return OrderResult.pending(order);
    }
    
    @EventListener
    @Async
    public void handleOrderCreatedEvent(OrderCreatedEvent event) {
        try {
            // 외부 연동 수행 (DB 커넥션과 무관)
            PaymentResponse paymentResponse = paymentApiClient.processPayment(
                PaymentRequest.builder()
                    .orderId(event.getOrderId())
                    .amount(event.getRequest().getAmount())
                    .build()
            );
            
            // 결과에 따른 주문 상태 업데이트
            updateOrderStatus(event.getOrderId(), paymentResponse);
            
        } catch (Exception e) {
            // 외부 연동 실패 시 주문 취소
            handlePaymentFailure(event.getOrderId(), e);
        }
    }
    
    @Transactional
    private void updateOrderStatus(Long orderId, PaymentResponse paymentResponse) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException("주문을 찾을 수 없음"));
            
        if (paymentResponse.isSuccess()) {
            order.updateStatus(OrderStatus.PAID);
            order.updatePaymentInfo(paymentResponse);
        } else {
            order.updateStatus(OrderStatus.PAYMENT_FAILED);
        }
    }
}
```

## 후처리 시 보상 로직

**하지만 트랜잭션 커밋 후에 데이터를 되돌릴 수 없으므로 후처리를 반드시 고려해야한다.**

### 보상 트랜잭션 구현

**보상 트랜잭션 혹은 기능 특성에 따라 데이터 보정등을 고려해볼 수 있다.**

```java
@Component
public class OrderCompensationService {
    
    @EventListener
    @Async
    public void handlePaymentFailureEvent(PaymentFailureEvent event) {
        try {
            executeCompensation(event.getOrderId(), event.getFailureReason());
        } catch (Exception e) {
            log.error("보상 트랜잭션 실패: orderId={}, error={}", 
                event.getOrderId(), e.getMessage());
            handleCompensationFailure(event.getOrderId(), e);
        }
    }
    
    @Transactional
    private void executeCompensation(Long orderId, String failureReason) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException("주문을 찾을 수 없음"));
        
        switch (order.getStatus()) {
            case PAYMENT_PENDING:
                // 결제 대기 중 실패 → 주문 취소
                order.cancel(failureReason);
                
                // 재고 복구
                inventoryService.releaseReservation(order.getItems());
                
                // 고객 알림
                notificationService.sendOrderCancellationNotice(
                    order.getCustomerId(), order.getId(), failureReason);
                break;
                
            case PAID:
                // 결제 완료 후 문제 발생 → 환불 처리
                processRefund(order, failureReason);
                break;
                
            default:
                log.warn("보상 불가능한 주문 상태: orderId={}, status={}", 
                    orderId, order.getStatus());
        }
    }
    
    private void processRefund(Order order, String reason) {
        try {
            RefundResponse refundResponse = paymentApiClient.requestRefund(
                RefundRequest.builder()
                    .paymentId(order.getPaymentId())
                    .amount(order.getAmount())
                    .reason(reason)
                    .build()
            );
            
            order.updateRefundInfo(refundResponse);
            
        } catch (Exception e) {
            // 환불 실패 시 수동 처리를 위한 알림
            alertService.sendManualRefundAlert(order.getId(), reason, e.getMessage());
            order.markAsRequiringManualRefund(reason);
        }
    }
}
```

### 데이터 보정 로직

```java
@Component
public class DataReconciliationService {
    
    @Scheduled(fixedDelay = 300000) // 5분마다 실행
    @Transactional
    public void reconcileInconsistentData() {
        // 1. 결제 대기 상태가 너무 오래된 주문 찾기
        List<Order> staleOrders = orderRepository.findByStatusAndCreatedAtBefore(
            OrderStatus.PAYMENT_PENDING,
            LocalDateTime.now().minusMinutes(30)
        );
        
        for (Order order : staleOrders) {
            try {
                reconcileOrderPaymentStatus(order);
            } catch (Exception e) {
                log.error("주문 상태 보정 실패: orderId={}, error={}", 
                    order.getId(), e.getMessage());
            }
        }
        
        // 2. 결제는 성공했지만 주문 상태가 업데이트되지 않은 경우
        reconcileSuccessfulPayments();
    }
    
    private void reconcileOrderPaymentStatus(Order order) {
        if (order.hasPaymentInfo()) {
            // 결제 정보가 있으면 외부 서비스에서 실제 상태 확인
            PaymentStatus actualStatus = paymentApiClient
                .getPaymentStatus(order.getPaymentId());
                
            switch (actualStatus) {
                case SUCCESS:
                    order.updateStatus(OrderStatus.PAID);
                    log.info("지연된 결제 상태 보정 완료: orderId={}", order.getId());
                    break;
                    
                case FAILED:
                    executeCompensation(order.getId(), "결제 실패 확인됨");
                    break;
                    
                case PENDING:
                    // 계속 대기 중이면 타임아웃으로 처리
                    if (order.isPaymentTimeout()) {
                        executeCompensation(order.getId(), "결제 타임아웃");
                    }
                    break;
            }
        } else {
            // 결제 정보가 없으면 결제가 시작되지 않았으므로 주문 취소
            executeCompensation(order.getId(), "결제 미시작으로 인한 타임아웃");
        }
    }
}
```

## 커넥션 풀 최적화 전략

### 1. 동적 커넥션 풀 크기 조정

```yaml
# application.yml
spring:
  datasource:
    hikari:
      # 기본 설정
      minimum-idle: 5
      maximum-pool-size: 20
      idle-timeout: 300000       # 5분
      max-lifetime: 1200000      # 20분
      connection-timeout: 20000  # 20초
      
      # 외부 연동이 많은 서비스의 경우
      maximum-pool-size: 50      # 풀 크기 확대
      leak-detection-threshold: 30000  # 30초 이상 점유 시 감지
```

```java
@Configuration
public class DynamicDataSourceConfig {
    
    @Value("${app.external-integration.enabled:true}")
    private boolean externalIntegrationEnabled;
    
    @Bean
    @Primary
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        
        if (externalIntegrationEnabled) {
            // 외부 연동이 활성화된 경우 커넥션 풀 확대
            config.setMaximumPoolSize(50);
            config.setLeakDetectionThreshold(30000);
        } else {
            // 외부 연동이 없는 경우 기본 설정
            config.setMaximumPoolSize(20);
        }
        
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("user");
        config.setPassword("password");
        
        return new HikariDataSource(config);
    }
}
```

### 2. 커넥션 사용량 모니터링

```java
@Component
public class ConnectionPoolMonitoring {
    
    private final HikariDataSource dataSource;
    private final MeterRegistry meterRegistry;
    
    @Scheduled(fixedRate = 10000) // 10초마다
    public void monitorConnectionPool() {
        HikariPoolMXBean poolMXBean = dataSource.getHikariPoolMXBean();
        
        // 메트릭 수집
        Gauge.builder("db.connection.active")
            .register(meterRegistry, poolMXBean, HikariPoolMXBean::getActiveConnections);
            
        Gauge.builder("db.connection.idle")
            .register(meterRegistry, poolMXBean, HikariPoolMXBean::getIdleConnections);
            
        Gauge.builder("db.connection.total")
            .register(meterRegistry, poolMXBean, HikariPoolMXBean::getTotalConnections);
        
        // 경고 임계치 체크
        int activeConnections = poolMXBean.getActiveConnections();
        int totalConnections = poolMXBean.getTotalConnections();
        double usageRatio = (double) activeConnections / totalConnections;
        
        if (usageRatio > 0.8) { // 80% 초과 시 경고
            alertService.sendConnectionPoolAlert(
                String.format("DB 커넥션 사용률 높음: %.1f%% (%d/%d)", 
                    usageRatio * 100, activeConnections, totalConnections)
            );
        }
    }
}
```

## 비동기 처리 패턴

### 1. CompletableFuture를 활용한 비동기 처리

```java
@Service
public class AsyncIntegrationService {
    
    @Async("externalIntegrationExecutor")
    public CompletableFuture<OrderResult> processOrderAsync(OrderRequest request) {
        // 외부 연동을 별도 스레드풀에서 처리
        return CompletableFuture.supplyAsync(() -> {
            try {
                PaymentResponse paymentResponse = paymentApiClient
                    .processPayment(PaymentRequest.from(request));
                    
                return saveOrderAfterPayment(request, paymentResponse);
                
            } catch (Exception e) {
                log.error("비동기 주문 처리 실패: {}", e.getMessage());
                return OrderResult.failure(e.getMessage());
            }
        });
    }
    
    @Transactional
    private OrderResult saveOrderAfterPayment(
            OrderRequest request, PaymentResponse paymentResponse) {
        // 빠른 DB 작업만 수행
        Order order = orderRepository.save(new Order(request));
        order.updatePaymentInfo(paymentResponse);
        return OrderResult.success(order);
    }
}

@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean("externalIntegrationExecutor")
    public TaskExecutor externalIntegrationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("external-integration-");
        executor.initialize();
        return executor;
    }
}
```

### 2. 메시지 큐를 활용한 완전 분리

```java
@Service
public class MessageQueueOrderService {
    
    @Transactional
    public OrderResult submitOrder(OrderRequest request) {
        // 1. 빠른 DB 작업으로 주문 생성
        Order order = orderRepository.save(
            Order.builder()
                .customerId(request.getCustomerId())
                .amount(request.getAmount())
                .status(OrderStatus.PROCESSING)
                .build()
        );
        
        // 2. 메시지 큐에 외부 연동 작업 전송
        paymentProcessingQueue.send(
            PaymentProcessingMessage.builder()
                .orderId(order.getId())
                .paymentInfo(request.getPaymentInfo())
                .build()
        );
        
        return OrderResult.processing(order);
    }
    
    @RabbitListener(queues = "payment.processing.queue")
    public void processPayment(PaymentProcessingMessage message) {
        try {
            // 외부 연동 수행 (DB 커넥션과 완전 분리)
            PaymentResponse paymentResponse = paymentApiClient.processPayment(
                PaymentRequest.builder()
                    .orderId(message.getOrderId())
                    .paymentInfo(message.getPaymentInfo())
                    .build()
            );
            
            // 결과 업데이트
            updateOrderWithPaymentResult(message.getOrderId(), paymentResponse);
            
        } catch (Exception e) {
            handlePaymentProcessingFailure(message.getOrderId(), e);
        }
    }
}
```

## 테스트 및 성능 검증

### 1. 커넥션 풀 부하 테스트

```java
@SpringBootTest
class ConnectionPoolLoadTest {
    
    @Test
    @DisplayName("동시 요청 시 커넥션 풀 고갈 확인")
    void shouldHandleConcurrentRequestsWithoutPoolExhaustion() 
            throws InterruptedException {
        
        int concurrentRequests = 100;
        CountDownLatch latch = new CountDownLatch(concurrentRequests);
        ExecutorService executorService = Executors.newFixedThreadPool(100);
        
        List<CompletableFuture<OrderResult>> futures = IntStream
            .range(0, concurrentRequests)
            .mapToObj(i -> CompletableFuture.supplyAsync(() -> {
                try {
                    OrderRequest request = createTestOrderRequest(i);
                    return optimizedOrderService.processOrderWithPreIntegration(request);
                } finally {
                    latch.countDown();
                }
            }, executorService))
            .collect(Collectors.toList());
        
        // 모든 요청이 30초 내에 완료되어야 함
        boolean completed = latch.await(30, TimeUnit.SECONDS);
        assertThat(completed).isTrue();
        
        // 모든 요청이 성공적으로 처리되어야 함
        List<OrderResult> results = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
            
        long successCount = results.stream()
            .filter(OrderResult::isSuccess)
            .count();
            
        assertThat(successCount).isEqualTo(concurrentRequests);
    }
}
```

### 2. 메트릭 기반 성능 검증

```java
@Component
public class PerformanceValidator {
    
    @EventListener
    public void validatePerformance(OrderProcessedEvent event) {
        Duration processingTime = event.getProcessingTime();
        
        // DB 커넥션 점유 시간이 1초를 초과하면 경고
        if (processingTime.toMillis() > 1000) {
            log.warn("Long DB transaction detected: orderId={}, duration={}ms", 
                event.getOrderId(), processingTime.toMillis());
                
            alertService.sendPerformanceAlert(
                "Long Transaction", event.getOrderId(), processingTime);
        }
    }
}
```

## 실무 체크리스트

### 설계 시 고려사항

- [ ] 외부 연동과 DB 트랜잭션 분리 여부 결정
- [ ] 비동기 처리 도입 가능성 검토
- [ ] 보상 트랜잭션 시나리오 정의
- [ ] 데이터 일관성 요구사항 분석
- [ ] 메시지 큐 도입 검토

### 구현 시 확인사항

- [ ] DB 커넥션 풀 크기 적정성 검토
- [ ] 외부 연동 타임아웃 설정
- [ ] 비동기 처리를 위한 스레드풀 설정
- [ ] 커넥션 리크 감지 설정
- [ ] 보상 로직 구현

### 운영 시 모니터링

- [ ] DB 커넥션 풀 사용률 모니터링
- [ ] 장시간 트랜잭션 감지 및 알림
- [ ] 외부 연동 응답시간 추적
- [ ] 보상 트랜잭션 발생 빈도 확인
- [ ] 데이터 정합성 정기 검증
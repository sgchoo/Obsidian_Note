[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 4-6. 외부 연동과 DB 연동 (External Integration & DB Transaction)

## 개요

외부 연동과 DB 트랜잭션을 함께 다룰 때 발생할 수 있는 **분산 트랜잭션 문제**와 **데이터 일관성** 이슈를 해결하는 방법을 다룹니다.

## 트랜잭션 범위 내 외부 연동 문제

### 문제 상황 1: 외부 연동 실패 시 롤백

**트랜잭션 범위 안에서 외부 연동이 실패한 경우, 트랜잭션을 롤백할 수 있다.**

```java
@Service
@Transactional
public class PaymentOrderService {
    
    public OrderResponse processOrder(OrderRequest request) {
        // 1. DB에 주문 정보 저장
        Order order = orderRepository.save(new Order(request));
        
        try {
            // 2. 외부 결제 서비스 호출
            PaymentResponse paymentResponse = paymentApiClient.processPayment(
                PaymentRequest.builder()
                    .orderId(order.getId())
                    .amount(order.getAmount())
                    .build()
            );
            
            // 3. 결제 성공 시 주문 상태 업데이트
            order.updatePaymentInfo(paymentResponse);
            return OrderResponse.success(order);
            
        } catch (PaymentException e) {
            // 외부 연동 실패 시 전체 트랜잭션 롤백
            log.error("결제 처리 실패, 주문 롤백: {}", e.getMessage());
            throw new OrderProcessException("주문 처리 실패", e);
        }
    }
}
```

### 문제 상황 2: 타임아웃으로 인한 불확실성

**하지만 읽기 타임 아웃이 발생해 트랜잭션을 롤백할 땐, 실제로는 외부 서비스가 성공적으로 처리했을 가능성이 있다.**

```
시나리오: 결제 API 타임아웃
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Our App   │───►│   Payment   │───►│   Success   │
│             │    │   Service   │    │  (Unknown)  │
│             │◄─X─│             │    │             │
│  Timeout!   │    │   (30sec)   │    │             │
│   Rollback  │    │             │    │             │
└─────────────┘    └─────────────┘    └─────────────┘

문제: 실제로는 결제가 성공했을 수 있음!
```

## 타임아웃 발생 시 확인 방법

**이 것을 확인하는 방법은 2가지 중 하나를 검토해야한다.**

### 방법 1: 일정 주기로 확인한다

#### 스케줄링을 통한 주기적 확인

```java
@Component
public class PaymentReconciliationService {
    
    @Scheduled(fixedRate = 60000) // 1분마다 실행
    @Transactional
    public void reconcileTimeoutPayments() {
        List<Order> timeoutOrders = orderRepository
            .findByStatusAndCreatedAtBefore(
                OrderStatus.PAYMENT_TIMEOUT,
                LocalDateTime.now().minusMinutes(5)
            );
            
        for (Order order : timeoutOrders) {
            try {
                // 외부 결제 서비스에서 실제 결제 상태 확인
                PaymentStatus actualStatus = paymentApiClient
                    .getPaymentStatus(order.getPaymentId());
                
                if (actualStatus == PaymentStatus.SUCCESS) {
                    // 실제로는 성공한 경우 주문 상태 업데이트
                    order.updateStatus(OrderStatus.PAID);
                    log.info("타임아웃 주문 복구: {}", order.getId());
                    
                } else if (actualStatus == PaymentStatus.FAILED) {
                    // 실제로 실패한 경우 주문 취소
                    order.updateStatus(OrderStatus.CANCELLED);
                    log.info("타임아웃 주문 취소: {}", order.getId());
                }
                
            } catch (Exception e) {
                log.error("주문 상태 확인 실패: {}, {}", order.getId(), e.getMessage());
            }
        }
    }
}
```

#### 이벤트 기반 재확인

```java
@EventListener
@Async
public void handlePaymentTimeoutEvent(PaymentTimeoutEvent event) {
    // 타임아웃 발생 1분 후 재확인
    CompletableFuture
        .runAsync(() -> {
            try {
                Thread.sleep(60000); // 1분 대기
                reconcilePayment(event.getOrderId());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
}

private void reconcilePayment(Long orderId) {
    Order order = orderRepository.findById(orderId).orElse(null);
    if (order != null && order.getStatus() == OrderStatus.PAYMENT_TIMEOUT) {
        PaymentStatus status = paymentApiClient.getPaymentStatus(order.getPaymentId());
        updateOrderBasedOnPaymentStatus(order, status);
    }
}
```

### 방법 2: 성공 확인 API를 호출한다

**=> 연동 서비스가 해당 API를 지원해야한다.**

#### Idempotent 확인 API 활용

```java
@Service
public class IdempotentPaymentService {
    
    @Transactional
    public OrderResponse processOrderWithConfirmation(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        
        try {
            // 멱등성 키 생성
            String idempotencyKey = generateIdempotencyKey(order);
            
            // 결제 처리 with 멱등성 키
            PaymentResponse paymentResponse = paymentApiClient.processPayment(
                PaymentRequest.builder()
                    .orderId(order.getId())
                    .amount(order.getAmount())
                    .idempotencyKey(idempotencyKey)
                    .build()
            );
            
            order.updatePaymentInfo(paymentResponse);
            return OrderResponse.success(order);
            
        } catch (TimeoutException e) {
            // 타임아웃 발생 시 즉시 확인 API 호출
            return handlePaymentTimeout(order, e);
        }
    }
    
    private OrderResponse handlePaymentTimeout(Order order, TimeoutException e) {
        try {
            // 확인 API 호출
            PaymentConfirmResponse confirmResponse = paymentApiClient
                .confirmPayment(order.getPaymentId());
                
            if (confirmResponse.isSuccess()) {
                // 실제로 성공한 경우
                order.updatePaymentInfo(confirmResponse.getPaymentInfo());
                return OrderResponse.success(order);
            } else {
                // 실제로 실패한 경우
                throw new PaymentProcessException("결제 처리 실패");
            }
            
        } catch (Exception confirmException) {
            // 확인도 실패한 경우 - 보상 트랜잭션으로 처리
            order.updateStatus(OrderStatus.PAYMENT_PENDING);
            publishPaymentTimeoutEvent(order.getId());
            
            return OrderResponse.pending(order, "결제 상태 확인 중");
        }
    }
}
```

#### 2단계 확인 (Two-Phase Confirmation)

```java
@Service
public class TwoPhasePaymentService {
    
    @Transactional
    public OrderResponse processOrderTwoPhase(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        
        try {
            // 1단계: 결제 예약 (Hold)
            PaymentHoldResponse holdResponse = paymentApiClient.holdPayment(
                PaymentHoldRequest.builder()
                    .orderId(order.getId())
                    .amount(order.getAmount())
                    .holdDuration(Duration.ofMinutes(10)) // 10분간 유지
                    .build()
            );
            
            order.updateHoldInfo(holdResponse);
            
            // 2단계: 결제 확정 (Commit)
            PaymentCommitResponse commitResponse = paymentApiClient.commitPayment(
                PaymentCommitRequest.builder()
                    .holdId(holdResponse.getHoldId())
                    .build()
            );
            
            order.updatePaymentInfo(commitResponse);
            return OrderResponse.success(order);
            
        } catch (TimeoutException e) {
            // 타임아웃 시 확인 후 처리
            return handleTwoPhaseTimeout(order, e);
        }
    }
}
```

## 외부 연동 성공, DB 실패 시 처리

**외부 연동은 성공했으나, DB가 실패하면 취소 API를 호출한다.**

### 보상 트랜잭션 (Compensating Transaction) 구현

```java
@Service
public class CompensatingTransactionService {
    
    public OrderResponse processOrderWithCompensation(OrderRequest request) {
        PaymentResponse paymentResponse = null;
        
        try {
            // 1. 외부 결제 서비스 호출 (트랜잭션 범위 외부)
            paymentResponse = paymentApiClient.processPayment(
                PaymentRequest.builder()
                    .orderId(request.getOrderId())
                    .amount(request.getAmount())
                    .build()
            );
            
            // 2. DB 트랜잭션 시작
            return saveOrderWithCompensation(request, paymentResponse);
            
        } catch (Exception e) {
            // 외부 연동은 성공했지만 DB 저장 실패 시 보상
            if (paymentResponse != null && paymentResponse.isSuccess()) {
                compensatePayment(paymentResponse.getPaymentId());
            }
            throw new OrderProcessException("주문 처리 실패", e);
        }
    }
    
    @Transactional
    private OrderResponse saveOrderWithCompensation(
            OrderRequest request, PaymentResponse paymentResponse) {
        try {
            Order order = orderRepository.save(new Order(request));
            order.updatePaymentInfo(paymentResponse);
            
            return OrderResponse.success(order);
            
        } catch (DataIntegrityViolationException e) {
            // DB 저장 실패 시 결제 취소
            log.error("DB 저장 실패, 결제 취소 진행: {}", e.getMessage());
            throw e; // 상위에서 보상 트랜잭션 실행
        }
    }
    
    private void compensatePayment(String paymentId) {
        try {
            CancelResponse cancelResponse = paymentApiClient.cancelPayment(
                CancelRequest.builder()
                    .paymentId(paymentId)
                    .reason("DB 저장 실패")
                    .build()
            );
            
            log.info("결제 취소 완료: {}", cancelResponse.getCancelId());
            
        } catch (Exception e) {
            log.error("결제 취소 실패, 수동 처리 필요: {}", paymentId);
            // 알림 발송 또는 별도 처리 큐에 등록
            alertService.sendManualProcessAlert(paymentId, "결제 취소 실패");
        }
    }
}
```

### Saga 패턴 구현

```java
@Component
public class OrderSagaOrchestrator {
    
    public void processOrderSaga(OrderRequest request) {
        SagaTransaction saga = SagaTransaction.builder()
            .sagaId(UUID.randomUUID().toString())
            .build();
            
        try {
            // Step 1: 재고 확인 및 예약
            InventoryReservationResponse inventoryResponse = 
                executeStep(saga, () -> inventoryService.reserveItems(request.getItems()));
            
            // Step 2: 결제 처리
            PaymentResponse paymentResponse = 
                executeStep(saga, () -> paymentService.processPayment(request.getPaymentInfo()));
            
            // Step 3: 주문 생성
            OrderResponse orderResponse = 
                executeStep(saga, () -> orderService.createOrder(request, paymentResponse));
            
            // Step 4: 배송 요청
            DeliveryResponse deliveryResponse = 
                executeStep(saga, () -> deliveryService.requestDelivery(orderResponse.getOrder()));
            
            saga.markAsCompleted();
            
        } catch (SagaException e) {
            // 실패한 단계부터 역순으로 보상 트랜잭션 실행
            executeCompensations(saga);
        }
    }
    
    private <T> T executeStep(SagaTransaction saga, Supplier<T> step) {
        try {
            T result = step.get();
            saga.addStep(result);
            return result;
            
        } catch (Exception e) {
            saga.markAsFailed();
            throw new SagaException("Saga step failed", e);
        }
    }
    
    private void executeCompensations(SagaTransaction saga) {
        List<SagaStep> completedSteps = saga.getCompletedSteps();
        Collections.reverse(completedSteps);
        
        for (SagaStep step : completedSteps) {
            try {
                step.compensate();
            } catch (Exception e) {
                log.error("보상 트랜잭션 실패: {}", step.getStepName(), e);
            }
        }
    }
}
```

## 분산 트랜잭션 패턴별 비교

### 패턴별 특성 비교

|패턴|장점|단점|적용 상황|
|---|---|---|---|
|**동기 확인**|즉시 결과 확인|응답 시간 증가|실시간 처리 필요|
|**비동기 확인**|빠른 응답|결과 지연|사용자 대기 가능|
|**보상 트랜잭션**|구현 단순|데이터 불일치 기간 존재|단순한 비즈니스 로직|
|**Saga 패턴**|복잡한 워크플로우 지원|구현 복잡도 높음|다단계 프로세스|
|**2PC (Two-Phase Commit)**|강한 일관성|성능 저하, 가용성 문제|금융 거래 등|

### 실무 권장 패턴

```java
@Service
public class RecommendedTransactionService {
    
    // 권장: 외부 호출을 트랜잭션 외부로 분리
    public OrderResponse processOrderRecommended(OrderRequest request) {
        // 1. 외부 서비스 호출 (트랜잭션 외부)
        PaymentResult paymentResult = processPaymentOutsideTransaction(request);
        
        if (paymentResult.isSuccess()) {
            // 2. 성공 시에만 DB 트랜잭션 시작
            return saveOrderInTransaction(request, paymentResult);
        } else {
            return OrderResponse.failure("결제 처리 실패");
        }
    }
    
    private PaymentResult processPaymentOutsideTransaction(OrderRequest request) {
        try {
            PaymentResponse response = paymentApiClient.processPayment(
                PaymentRequest.from(request)
            );
            return PaymentResult.success(response);
            
        } catch (TimeoutException e) {
            // 타임아웃 시 확인 API 호출
            return confirmPaymentStatus(request.getOrderId());
            
        } catch (Exception e) {
            return PaymentResult.failure(e.getMessage());
        }
    }
    
    @Transactional
    private OrderResponse saveOrderInTransaction(
            OrderRequest request, PaymentResult paymentResult) {
        try {
            Order order = orderRepository.save(new Order(request));
            order.updatePaymentInfo(paymentResult.getPaymentInfo());
            
            return OrderResponse.success(order);
            
        } catch (Exception e) {
            // DB 실패 시 결제 취소
            asyncCompensationService.cancelPayment(
                paymentResult.getPaymentId(), "DB 저장 실패"
            );
            throw e;
        }
    }
}
```

## 모니터링 및 알림

### 1. 분산 트랜잭션 상태 추적

```java
@Entity
@Table(name = "distributed_transactions")
public class DistributedTransaction {
    
    @Id
    private String transactionId;
    
    @Enumerated(EnumType.STRING)
    private TransactionStatus status;
    
    private String paymentId;
    private String orderId;
    
    @CreatedDate
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    private String compensationStatus;
    private String errorMessage;
}

@Repository
public interface DistributedTransactionRepository 
        extends JpaRepository<DistributedTransaction, String> {
    
    List<DistributedTransaction> findByStatusAndCreatedAtBefore(
        TransactionStatus status, LocalDateTime cutoff);
}
```

### 2. 불일치 데이터 감지

```java
@Component
public class DataConsistencyChecker {
    
    @Scheduled(cron = "0 */10 * * * *") // 10분마다
    public void checkDataConsistency() {
        // 결제는 성공했지만 주문이 없는 경우
        List<String> orphanPayments = findOrphanPayments();
        orphanPayments.forEach(this::handleOrphanPayment);
        
        // 주문은 있지만 결제가 없는 경우
        List<String> orphanOrders = findOrphanOrders();
        orphanOrders.forEach(this::handleOrphanOrder);
    }
    
    private void handleOrphanPayment(String paymentId) {
        log.warn("고아 결제 발견: {}", paymentId);
        alertService.sendDataInconsistencyAlert(
            "Orphan Payment", paymentId);
    }
}
```

## 테스트 전략

### 1. 분산 트랜잭션 테스트

```java
@SpringBootTest
@Transactional
class DistributedTransactionTest {
    
    @Test
    @DisplayName("외부 연동 성공, DB 실패 시 보상 트랜잭션 실행")
    void shouldExecuteCompensatingTransactionOnDbFailure() {
        // Given
        when(paymentApiClient.processPayment(any()))
            .thenReturn(PaymentResponse.success("payment-123"));
        when(orderRepository.save(any()))
            .thenThrow(new DataIntegrityViolationException("DB error"));
        
        // When & Then
        assertThatThrownBy(() -> 
            compensatingTransactionService.processOrderWithCompensation(orderRequest))
            .isInstanceOf(OrderProcessException.class);
        
        // 보상 트랜잭션(결제 취소) 호출 확인
        verify(paymentApiClient).cancelPayment(any());
    }
    
    @Test
    @DisplayName("타임아웃 발생 시 확인 API 호출")
    void shouldCallConfirmApiOnTimeout() {
        // Given
        when(paymentApiClient.processPayment(any()))
            .thenThrow(new TimeoutException("Request timeout"));
        when(paymentApiClient.confirmPayment(any()))
            .thenReturn(PaymentConfirmResponse.success());
        
        // When
        OrderResponse response = idempotentPaymentService
            .processOrderWithConfirmation(orderRequest);
        
        // Then
        assertThat(response.isSuccess()).isTrue();
        verify(paymentApiClient).confirmPayment(any());
    }
}
```

### 2. 통합 테스트

```java
@SpringBootTest
@TestPropertySource(properties = {
    "external.payment.api.enabled=true",
    "spring.datasource.url=jdbc:h2:mem:testdb"
})
class IntegrationTest {
    
    @Test
    @DisplayName("전체 프로세스 통합 테스트")
    void shouldHandleCompleteOrderProcess() {
        // 실제 외부 서비스 호출을 포함한 통합 테스트
        // WireMock이나 TestContainers 활용
    }
}
```

## 실무 체크리스트

### 설계 시 고려사항

- [ ] 외부 호출을 트랜잭션 범위 외부로 분리 검토
- [ ] 타임아웃 발생 시 확인 방법 정의
- [ ] 보상 트랜잭션 시나리오 설계
- [ ] 데이터 일관성 보장 수준 결정
- [ ] 분산 트랜잭ション 패턴 선택

### 구현 시 확인사항

- [ ] 멱등성 키를 활용한 중복 방지
- [ ] 적절한 타임아웃 설정
- [ ] 보상 트랜잭션 로직 구현
- [ ] 트랜잭션 상태 추적 테이블 생성
- [ ] 에러 처리 및 알림 메커니즘

### 운영 시 모니터링

- [ ] 분산 트랜잭션 성공률 추적
- [ ] 보상 트랜잭션 발생 빈도 모니터링
- [ ] 데이터 불일치 상황 감지 및 알림
- [ ] 수동 개입이 필요한 케이스 추적
- [ ] 정기적인 데이터 일관성 검증
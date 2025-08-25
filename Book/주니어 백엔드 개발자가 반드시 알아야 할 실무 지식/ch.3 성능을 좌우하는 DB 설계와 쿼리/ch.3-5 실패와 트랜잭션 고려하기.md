[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 트랜잭션과 실패 처리 전략 가이드

## 1. 트랜잭션의 기본 원칙

### 1.1 트랜잭션이 필요한 이유

#### ACID 속성 보장

- **원자성(Atomicity)**: 모든 작업이 완전히 수행되거나 전혀 수행되지 않음
- **일관성(Consistency)**: 트랜잭션 실행 전후 데이터 무결성 유지
- **격리성(Isolation)**: 동시 실행되는 트랜잭션들이 서로 영향을 주지 않음
- **지속성(Durability)**: 완료된 트랜잭션의 결과는 영구적으로 저장

#### 문제 상황 예시

```java
// 🚫 트랜잭션 없이 데이터 변경 (위험한 패턴)
public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) {
    Account fromAccount = accountRepository.findById(fromAccountId);
    Account toAccount = accountRepository.findById(toAccountId);
    
    // 1. 출금 처리
    fromAccount.withdraw(amount);
    accountRepository.save(fromAccount); // 여기서 실패하면?
    
    // 2. 입금 처리  
    toAccount.deposit(amount);
    accountRepository.save(toAccount); // 여기서 실패하면 출금만 처리됨!
}
```

### 1.2 올바른 트랜잭션 구현

#### 선언적 트랜잭션 사용

```java
// ✅ 트랜잭션으로 안전하게 처리
@Service
@Transactional
public class AccountService {
    
    @Transactional(rollbackFor = Exception.class)
    public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        Account fromAccount = accountRepository.findById(fromAccountId)
            .orElseThrow(() -> new AccountNotFoundException("출금 계좌를 찾을 수 없습니다"));
        Account toAccount = accountRepository.findById(toAccountId)
            .orElseThrow(() -> new AccountNotFoundException("입금 계좌를 찾을 수 없습니다"));
        
        // 잔액 확인
        if (fromAccount.getBalance().compareTo(amount) < 0) {
            throw new InsufficientBalanceException("잔액이 부족합니다");
        }
        
        // 원자적으로 처리되는 작업들
        fromAccount.withdraw(amount);
        toAccount.deposit(amount);
        
        accountRepository.save(fromAccount);
        accountRepository.save(toAccount);
        
        // 거래 내역 기록
        transactionHistoryRepository.save(
            new TransactionHistory(fromAccountId, toAccountId, amount, "TRANSFER")
        );
        
        log.info("Transfer completed: {} -> {}, amount: {}", 
                fromAccountId, toAccountId, amount);
    }
}
```

---

## 2. 핵심 비즈니스와 부가 기능의 분리

### 2.1 트랜잭션 경계 설정 원칙

#### 핵심 비즈니스 로직과 부가 기능 구분

|구분|핵심 비즈니스 로직|부가 기능|
|---|---|---|
|**특징**|데이터 일관성에 직접적 영향|실패해도 비즈니스 로직에 영향 없음|
|**실패 시 대응**|전체 트랜잭션 롤백|로깅 후 계속 진행|
|**예시**|주문 생성, 결제 처리, 재고 차감|이메일 발송, SMS 알림, 로그 기록|

#### 잘못된 트랜잭션 구성

```java
// 🚫 문제가 있는 패턴: 이메일 발송 실패로 주문 생성까지 롤백됨
@Service
@Transactional
public class OrderService {
    
    public OrderResponse createOrder(CreateOrderRequest request) {
        // 1. 핵심 비즈니스 로직
        Order order = new Order(request);
        orderRepository.save(order);
        
        // 2. 재고 차감
        inventoryService.decreaseStock(request.getProductId(), request.getQuantity());
        
        // 3. 결제 처리
        paymentService.processPayment(request.getPaymentInfo());
        
        // 4. 이메일 발송 - 실패하면 위의 모든 작업이 롤백됨!
        emailService.sendOrderConfirmation(order); // SMTP 서버 오류 등으로 실패 가능
        
        return OrderResponse.from(order);
    }
}
```

### 2.2 올바른 트랜잭션 분리 전략

#### 패턴 1: Try-Catch로 부가 기능 보호

```java
// ✅ 개선된 패턴: 부가 기능 실패가 핵심 로직에 영향 주지 않음
@Service
@Transactional
public class OrderService {
    
    public OrderResponse createOrder(CreateOrderRequest request) {
        try {
            // 1. 핵심 비즈니스 로직 (트랜잭션 필수)
            Order order = new Order(request);
            orderRepository.save(order);
            
            // 2. 재고 차감
            inventoryService.decreaseStock(request.getProductId(), request.getQuantity());
            
            // 3. 결제 처리
            Payment payment = paymentService.processPayment(request.getPaymentInfo());
            order.setPayment(payment);
            
            // ✅ 핵심 로직 완료, 이제 커밋됨
            
            // 4. 부가 기능들 - 실패해도 주문은 유지됨
            handlePostOrderTasks(order);
            
            return OrderResponse.from(order);
            
        } catch (BusinessException e) {
            // 비즈니스 예외는 롤백
            log.error("Order creation failed: {}", e.getMessage());
            throw e;
        }
    }
    
    private void handlePostOrderTasks(Order order) {
        // 이메일 발송
        try {
            emailService.sendOrderConfirmation(order);
            log.info("Order confirmation email sent: orderId={}", order.getId());
        } catch (Exception e) {
            log.warn("Failed to send order confirmation email: orderId={}, error={}", 
                    order.getId(), e.getMessage());
            // 에러를 던지지 않음 - 주문 트랜잭션에 영향 주지 않음
        }
        
        // SMS 발송
        try {
            smsService.sendOrderNotification(order);
            log.info("Order SMS sent: orderId={}", order.getId());
        } catch (Exception e) {
            log.warn("Failed to send order SMS: orderId={}, error={}", 
                    order.getId(), e.getMessage());
        }
        
        // 외부 시스템 연동
        try {
            externalApiService.notifyOrderCreated(order);
            log.info("External API notified: orderId={}", order.getId());
        } catch (Exception e) {
            log.error("Failed to notify external system: orderId={}, error={}", 
                    order.getId(), e.getMessage());
            // 재시도 큐에 추가하거나 별도 처리
            retryQueueService.addFailedNotification(order.getId(), "ORDER_CREATED");
        }
    }
}
```

#### 패턴 2: 트랜잭션 분리 (별도 트랜잭션)

```java
@Service  
public class OrderService {
    
    @Autowired
    private OrderPostProcessor orderPostProcessor;
    
    @Transactional
    public OrderResponse createOrder(CreateOrderRequest request) {
        // 핵심 비즈니스 로직만 트랜잭션에 포함
        Order order = new Order(request);
        orderRepository.save(order);
        
        inventoryService.decreaseStock(request.getProductId(), request.getQuantity());
        Payment payment = paymentService.processPayment(request.getPaymentInfo());
        order.setPayment(payment);
        
        // 여기서 트랜잭션 커밋됨
        
        // 별도 트랜잭션으로 후처리 작업 수행
        orderPostProcessor.processAfterOrderCreation(order.getId());
        
        return OrderResponse.from(order);
    }
}

@Service
@Transactional(propagation = Propagation.REQUIRES_NEW)
public class OrderPostProcessor {
    
    public void processAfterOrderCreation(Long orderId) {
        Order order = orderRepository.findById(orderId).orElse(null);
        if (order == null) {
            log.warn("Order not found for post-processing: {}", orderId);
            return;
        }
        
        // 각 부가 기능을 독립적으로 처리
        processEmailNotification(order);
        processSmsNotification(order);
        processExternalApiNotification(order);
    }
    
    @Async
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void processEmailNotification(Order order) {
        try {
            emailService.sendOrderConfirmation(order);
            
            // 발송 성공 기록
            notificationHistoryRepository.save(
                NotificationHistory.success(order.getId(), "EMAIL", "ORDER_CONFIRMATION")
            );
        } catch (Exception e) {
            log.error("Email notification failed: orderId={}", order.getId(), e);
            
            // 실패 기록 저장
            notificationHistoryRepository.save(
                NotificationHistory.failure(order.getId(), "EMAIL", "ORDER_CONFIRMATION", e.getMessage())
            );
        }
    }
}
```

#### 패턴 3: 이벤트 기반 비동기 처리

```java
@Service
@Transactional
public class OrderService {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public OrderResponse createOrder(CreateOrderRequest request) {
        // 핵심 비즈니스 로직
        Order order = new Order(request);
        orderRepository.save(order);
        
        inventoryService.decreaseStock(request.getProductId(), request.getQuantity());
        paymentService.processPayment(request.getPaymentInfo());
        
        // 트랜잭션 커밋 후 이벤트 발행
        eventPublisher.publishEvent(new OrderCreatedEvent(order.getId()));
        
        return OrderResponse.from(order);
    }
}

@Component
public class OrderEventListener {
    
    @EventListener
    @Async
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void handleOrderCreated(OrderCreatedEvent event) {
        Order order = orderRepository.findById(event.getOrderId()).orElse(null);
        if (order == null) return;
        
        // 비동기로 부가 작업들 처리
        CompletableFuture.allOf(
            CompletableFuture.runAsync(() -> sendEmailNotification(order)),
            CompletableFuture.runAsync(() -> sendSmsNotification(order)),
            CompletableFuture.runAsync(() -> notifyExternalSystem(order))
        ).exceptionally(throwable -> {
            log.error("Some post-order tasks failed: orderId={}", order.getId(), throwable);
            return null;
        });
    }
    
    private void sendEmailNotification(Order order) {
        try {
            emailService.sendOrderConfirmation(order);
        } catch (Exception e) {
            log.error("Email notification failed: orderId={}", order.getId(), e);
            // 재시도 로직 또는 DLQ 전송
        }
    }
}
```

---
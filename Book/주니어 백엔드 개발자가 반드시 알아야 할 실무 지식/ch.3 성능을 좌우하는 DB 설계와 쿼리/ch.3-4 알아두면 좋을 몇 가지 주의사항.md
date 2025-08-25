[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# DB 운영 시 주의사항 및 모범 사례

## 1. 응답 지연과 재시도 메커니즘

### 1.1 재시도로 인한 서버 부하 악순환

#### 문제 상황

```javascript
// 🚫 위험한 재시도 패턴
async function fetchUserData(userId) {
    const maxRetries = 5;
    let attempt = 0;
    
    while (attempt < maxRetries) {
        try {
            const response = await api.get(`/users/${userId}`);
            return response.data;
        } catch (error) {
            attempt++;
            if (attempt === maxRetries) throw error;
            
            // 즉시 재시도 - 서버 부하 가중!
            continue;
        }
    }
}
```

#### 악순환 구조

```
1. DB 응답 지연 발생
    ↓
2. 클라이언트에서 타임아웃으로 인식
    ↓  
3. 재시도 요청 급증
    ↓
4. DB 서버 부하 증가
    ↓
5. 응답 지연 더욱 악화 (악순환 시작)
```

### 1.2 안전한 재시도 전략

#### 1) 지수 백오프(Exponential Backoff)

```javascript
// ✅ 안전한 재시도 패턴
class ApiClient {
    async fetchWithRetry(url, options = {}) {
        const { 
            maxRetries = 3, 
            baseDelay = 1000,
            maxDelay = 10000,
            jitter = true 
        } = options;
        
        for (let attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                return await this.fetch(url);
            } catch (error) {
                if (attempt === maxRetries) {
                    throw new Error(`Failed after ${maxRetries} retries: ${error.message}`);
                }
                
                // 지수적 백오프 + 지터
                let delay = Math.min(baseDelay * Math.pow(2, attempt), maxDelay);
                if (jitter) {
                    delay += Math.random() * 1000; // 0~1초 랜덤 지연
                }
                
                console.log(`Retry attempt ${attempt + 1} after ${delay}ms`);
                await this.sleep(delay);
            }
        }
    }
    
    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}
```

#### 2) 서킷 브레이커 패턴

```java
@Component
public class DatabaseCircuitBreaker {
    private static final int FAILURE_THRESHOLD = 5;
    private static final long TIMEOUT_DURATION = 60000; // 1분
    
    private int failureCount = 0;
    private long lastFailureTime = 0;
    private CircuitState state = CircuitState.CLOSED;
    
    public <T> T execute(Supplier<T> operation) throws Exception {
        if (state == CircuitState.OPEN) {
            if (System.currentTimeMillis() - lastFailureTime > TIMEOUT_DURATION) {
                state = CircuitState.HALF_OPEN;
            } else {
                throw new CircuitBreakerOpenException("Circuit breaker is OPEN");
            }
        }
        
        try {
            T result = operation.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            throw e;
        }
    }
    
    private void onSuccess() {
        failureCount = 0;
        state = CircuitState.CLOSED;
    }
    
    private void onFailure() {
        failureCount++;
        lastFailureTime = System.currentTimeMillis();
        
        if (failureCount >= FAILURE_THRESHOLD) {
            state = CircuitState.OPEN;
        }
    }
    
    enum CircuitState { CLOSED, OPEN, HALF_OPEN }
}
```

#### 3) 요청 제한 및 큐잉

```java
@Service
public class RateLimitedService {
    
    // Guava RateLimiter 사용
    private final RateLimiter rateLimiter = RateLimiter.create(100.0); // 초당 100개 요청
    
    public CompletableFuture<User> getUserAsync(Long userId) {
        if (!rateLimiter.tryAcquire()) {
            return CompletableFuture.failedFuture(
                new RateLimitExceededException("Too many requests")
            );
        }
        
        return CompletableFuture.supplyAsync(() -> userRepository.findById(userId));
    }
}
```

### 1.3 쿼리 타임아웃 설정

#### 서비스별 타임아웃 전략

|서비스 유형|권장 타임아웃|이유|
|---|---|---|
|**사용자 대화형 조회**|3-5초|사용자 경험 우선|
|**백그라운드 작업**|30-60초|안정성 우선|
|**배치 처리**|5-10분|완성도 우선|
|**실시간 API**|1-2초|응답성 최우선|

#### 데이터베이스별 타임아웃 설정

##### MySQL

```sql
-- 세션별 타임아웃 설정
SET SESSION max_execution_time = 5000; -- 5초

-- 글로벌 설정 (my.cnf)
max_execution_time = 30000  -- 30초
```

##### Spring Boot 설정

```yaml
# application.yml
spring:
  datasource:
    hikari:
      connection-timeout: 30000      # 커넥션 획득 타임아웃: 30초
      maximum-pool-size: 20
      max-lifetime: 1800000          # 커넥션 최대 생존 시간: 30분
  
  jpa:
    properties:
      hibernate:
        query:
          timeout: 5000              # 쿼리 타임아웃: 5초
```

#### 애플리케이션 레벨 타임아웃

```java
@Service
public class UserService {
    
    @Async
    @Timeout(value = 5, unit = TimeUnit.SECONDS)
    public CompletableFuture<List<User>> getActiveUsers() {
        return CompletableFuture.supplyAsync(() -> {
            return userRepository.findActiveUsers();
        });
    }
    
    // 타임아웃 예외 처리
    @ExceptionHandler(TimeoutException.class)
    public ResponseEntity<String> handleTimeout(TimeoutException e) {
        return ResponseEntity.status(HttpStatus.REQUEST_TIMEOUT)
                           .body("요청 처리 시간이 초과되었습니다. 잠시 후 다시 시도해주세요.");
    }
}
```

---

## 2. Primary/Replica 구조의 복제 지연 문제

### 2.1 복제 지연 문제점

#### 일반적인 문제 시나리오

```java
// 🚫 문제가 되는 패턴
@Service
@Transactional
public class OrderService {
    
    public void createOrder(CreateOrderRequest request) {
        // 1. Primary DB에 주문 생성
        Order order = new Order(request);
        orderRepository.save(order); // PRIMARY DB
        
        // 2. 즉시 주문 조회 (Replica에서 조회)
        Order savedOrder = orderRepository.findById(order.getId()); // REPLICA DB
        
        // 3. 복제 지연으로 인해 null 또는 이전 상태 반환 가능
        if (savedOrder == null) {
            throw new OrderNotFoundException("방금 생성한 주문을 찾을 수 없습니다");
        }
    }
}
```

#### 복제 지연의 원인

- **네트워크 지연**: Primary와 Replica 간 물리적 거리
- **부하 차이**: Primary의 쓰기 부하가 클 때
- **설정 문제**: 비동기 복제 모드 사용
- **하드웨어 성능**: Replica 서버의 성능 부족

### 2.2 해결 방법 1: 읽기 라우팅 전략

#### 명시적 Primary 읽기

```java
@Service
public class OrderService {
    
    @Autowired
    @Qualifier("primaryDataSource")
    private DataSource primaryDataSource;
    
    @Autowired  
    @Qualifier("replicaDataSource")
    private DataSource replicaDataSource;
    
    @Transactional
    public OrderResponse createOrder(CreateOrderRequest request) {
        // 1. Primary에 주문 생성
        Order order = new Order(request);
        orderRepository.save(order);
        
        // 2. 생성 직후 조회도 Primary에서 수행
        Order savedOrder = orderRepository.findByIdFromPrimary(order.getId());
        
        return OrderResponse.from(savedOrder);
    }
}

@Repository
public class OrderRepository {
    
    @ReadFromPrimary
    public Order findByIdFromPrimary(Long id) {
        // Primary DB에서 조회하도록 강제
        return entityManager.find(Order.class, id);
    }
    
    @ReadFromReplica
    public List<Order> findRecentOrders() {
        // Replica DB에서 조회 (일반적인 목록 조회)
        return entityManager.createQuery("...", Order.class).getResultList();
    }
}
```

#### Spring Boot에서 라우팅 구현

```java
// 데이터소스 라우팅 설정
@Configuration
public class DatabaseConfig {
    
    @Bean
    @Primary
    public DataSource routingDataSource() {
        ReplicationRoutingDataSource routingDataSource = new ReplicationRoutingDataSource();
        
        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put("primary", primaryDataSource());
        dataSourceMap.put("replica", replicaDataSource());
        
        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(replicaDataSource());
        
        return routingDataSource;
    }
}

// 커스텀 어노테이션으로 라우팅 제어
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ReadFromPrimary {
}

@Aspect
@Component
public class DatabaseRoutingAspect {
    
    @Before("@annotation(readFromPrimary)")
    public void routeToPrimary(ReadFromPrimary readFromPrimary) {
        DatabaseContextHolder.setDatabaseType(DatabaseType.PRIMARY);
    }
    
    @After("@annotation(readFromPrimary)")
    public void clearRouting() {
        DatabaseContextHolder.clearDatabaseType();
    }
}
```

### 2.3 해결 방법 2: 세션 일관성 보장

#### Sticky Session 패턴

```java
@Service
public class SessionConsistencyService {
    
    // 쓰기 작업 후 일정 시간 동안 Primary에서만 읽기
    private final LoadingCache<String, LocalDateTime> writeSessionCache = 
        Caffeine.newBuilder()
                .maximumSize(10000)
                .expireAfterWrite(Duration.ofSeconds(5)) // 5초 동안 Primary 읽기
                .build(key -> LocalDateTime.now());
    
    @Transactional
    public void updateUserProfile(String sessionId, UserProfile profile) {
        userRepository.save(profile);
        
        // 이 세션은 5초 동안 Primary에서 읽어야 함
        writeSessionCache.put(sessionId, LocalDateTime.now());
    }
    
    public UserProfile getUserProfile(String sessionId, Long userId) {
        LocalDateTime lastWrite = writeSessionCache.getIfPresent(sessionId);
        
        if (lastWrite != null && 
            lastWrite.isAfter(LocalDateTime.now().minusSeconds(5))) {
            // 최근 쓰기가 있었다면 Primary에서 읽기
            return userRepository.findByIdFromPrimary(userId);
        } else {
            // 일반적인 경우 Replica에서 읽기
            return userRepository.findById(userId);
        }
    }
}
```

#### 버전 기반 일관성 체크

```java
@Entity
public class Order {
    @Id
    private Long id;
    
    @Version
    private Long version; // JPA Optimistic Locking
    
    private LocalDateTime lastModified;
    
    // getters, setters...
}

@Service
public class ConsistentOrderService {
    
    @Transactional
    public OrderResponse updateOrder(Long orderId, UpdateOrderRequest request) {
        // Primary에서 수정
        Order order = orderRepository.findById(orderId);
        order.update(request);
        Order savedOrder = orderRepository.save(order);
        
        // 응답에 버전 정보 포함
        return OrderResponse.builder()
                .id(savedOrder.getId())
                .version(savedOrder.getVersion())
                .lastModified(savedOrder.getLastModified())
                .build();
    }
    
    public Order getOrderWithConsistencyCheck(Long orderId, Long expectedVersion) {
        Order order = orderRepository.findById(orderId); // Replica에서 조회
        
        if (expectedVersion != null && !expectedVersion.equals(order.getVersion())) {
            // 버전이 다르면 Primary에서 재조회
            order = orderRepository.findByIdFromPrimary(orderId);
        }
        
        return order;
    }
}
```

### 2.4 해결 방법 3: 비동기 처리 패턴

#### 이벤트 기반 아키텍처

```java
@Service
@Transactional
public class AsyncOrderService {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public void createOrder(CreateOrderRequest request) {
        // 1. Primary에 주문 생성
        Order order = new Order(request);
        Order savedOrder = orderRepository.save(order);
        
        // 2. 이벤트 발행 (비동기 처리)
        eventPublisher.publishEvent(new OrderCreatedEvent(savedOrder.getId()));
        
        // 3. 즉시 응답 (주문 ID만 반환)
        return CreateOrderResponse.builder()
                .orderId(savedOrder.getId())
                .status("PROCESSING")
                .message("주문이 생성되었습니다. 처리 상태는 별도로 확인해주세요.")
                .build();
    }
}

@EventListener
@Async
public class OrderEventHandler {
    
    public void handleOrderCreated(OrderCreatedEvent event) {
        // 복제가 완료된 후 후속 작업 수행
        try {
            Thread.sleep(1000); // 복제 완료 대기
            
            Order order = orderRepository.findById(event.getOrderId()); // Replica에서 안전하게 조회
            
            // 후속 작업: 이메일 발송, 재고 업데이트 등
            emailService.sendOrderConfirmation(order);
            inventoryService.updateStock(order);
            
        } catch (Exception e) {
            log.error("Order processing failed: {}", event.getOrderId(), e);
            // 재시도 로직 또는 에러 처리
        }
    }
}
```

---

## 3. 배치 처리 최적화

### 3.1 배치 처리의 장점과 활용 사례

#### 장점

- **처리량 최대화**: 한 번에 많은 데이터 처리
- **리소스 효율성**: DB 커넥션, 메모리 효율적 사용
- **트랜잭션 오버헤드 감소**: 개별 트랜잭션 대비 성능 향상
- **일관성 보장**: 관련 데이터를 함께 처리

#### 주요 활용 사례

|업무 영역|배치 처리 예시|처리 주기|
|---|---|---|
|**정산**|일일 매출 집계, 수수료 정산|일배치|
|**통계**|사용자 활동 분석, 리포트 생성|시간/일배치|
|**정리**|로그 아카이브, 만료 데이터 삭제|일/주배치|
|**동기화**|외부 시스템 연동, 데이터 복제|실시간/시간배치|

### 3.2 효율적인 배치 처리 구현

#### 1) JDBC 배치 업데이트

```java
@Service
public class UserBatchService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public void updateUserStatusBatch(List<UserStatusUpdate> updates) {
        String sql = "UPDATE users SET status = ?, updated_at = ? WHERE id = ?";
        
        jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                UserStatusUpdate update = updates.get(i);
                ps.setString(1, update.getStatus());
                ps.setTimestamp(2, Timestamp.valueOf(LocalDateTime.now()));
                ps.setLong(3, update.getUserId());
            }
            
            @Override
            public int getBatchSize() {
                return updates.size();
            }
        });
    }
    
    // 대용량 데이터 처리를 위한 청크 기반 배치
    public void processLargeDataset(List<Long> userIds) {
        int chunkSize = 1000;
        
        for (int i = 0; i < userIds.size(); i += chunkSize) {
            int end = Math.min(i + chunkSize, userIds.size());
            List<Long> chunk = userIds.subList(i, end);
            
            processUserChunk(chunk);
            
            // 메모리 정리 및 다른 작업에 CPU 양보
            if (i % 10000 == 0) {
                System.gc();
                Thread.sleep(100);
            }
        }
    }
}
```

#### 2) JPA 배치 처리

```java
@Service
@Transactional
public class JpaBatchService {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    public void saveBulkOrders(List<Order> orders) {
        int batchSize = 50; // hibernate.jdbc.batch_size와 맞춤
        
        for (int i = 0; i < orders.size(); i++) {
            entityManager.persist(orders.get(i));
            
            if (i % batchSize == 0 && i > 0) {
                // 배치 실행 및 메모리 정리
                entityManager.flush();
                entityManager.clear();
            }
        }
        
        // 마지막 배치 처리
        entityManager.flush();
        entityManager.clear();
    }
    
    // 벌크 업데이트 (단일 쿼리로 여러 레코드 업데이트)
    public int updateOrderStatusBulk(List<Long> orderIds, OrderStatus status) {
        return entityManager.createQuery(
            "UPDATE Order o SET o.status = :status, o.updatedAt = :now " +
            "WHERE o.id IN :orderIds")
            .setParameter("status", status)
            .setParameter("now", LocalDateTime.now())
            .setParameter("orderIds", orderIds)
            .executeUpdate();
    }
}
```

#### 3) Spring Batch 활용

```java
@Configuration
@EnableBatchProcessing
public class DataMigrationBatchConfig {
    
    @Bean
    public Job migrationJob() {
        return jobBuilderFactory.get("migrationJob")
                .start(migrationStep())
                .build();
    }
    
    @Bean
    public Step migrationStep() {
        return stepBuilderFactory.get("migrationStep")
                .<OldUser, NewUser>chunk(1000)  // 1000개씩 처리
                .reader(oldUserReader())
                .processor(userProcessor())
                .writer(newUserWriter())
                .faultTolerant()
                .skipLimit(100)  // 100개까지 스킵 허용
                .skip(Exception.class)
                .build();
    }
    
    @Bean
    public JdbcCursorItemReader<OldUser> oldUserReader() {
        return new JdbcCursorItemReaderBuilder<OldUser>()
                .dataSource(oldDataSource)
                .name("oldUserReader")
                .sql("SELECT id, name, email FROM old_users WHERE migrated = false")
                .rowMapper(new BeanPropertyRowMapper<>(OldUser.class))
                .build();
    }
    
    @Bean
    public ItemProcessor<OldUser, NewUser> userProcessor() {
        return oldUser -> {
            // 데이터 변환 로직
            return NewUser.builder()
                    .id(oldUser.getId())
                    .username(oldUser.getName())
                    .email(oldUser.getEmail().toLowerCase())
                    .createdAt(LocalDateTime.now())
                    .build();
        };
    }
    
    @Bean
    public JdbcBatchItemWriter<NewUser> newUserWriter() {
        return new JdbcBatchItemWriterBuilder<NewUser>()
                .dataSource(newDataSource)
                .sql("INSERT INTO new_users (id, username, email, created_at) " +
                     "VALUES (:id, :username, :email, :createdAt)")
                .beanMapped()
                .build();
    }
}
```

### 3.3 배치 처리 모니터링과 에러 처리

#### 진행 상황 모니터링

```java
@Service
public class BatchMonitoringService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    public void processBatchWithMonitoring(List<Data> dataList) {
        Timer.Sample sample = Timer.start(meterRegistry);
        Counter successCounter = meterRegistry.counter("batch.process.success");
        Counter errorCounter = meterRegistry.counter("batch.process.error");
        
        try {
            int processed = 0;
            for (Data data : dataList) {
                try {
                    processData(data);
                    successCounter.increment();
                    processed++;
                    
                    // 진행률 로깅
                    if (processed % 1000 == 0) {
                        double progress = (double) processed / dataList.size() * 100;
                        log.info("Batch progress: {}/{} ({:.1f}%)", 
                                processed, dataList.size(), progress);
                    }
                    
                } catch (Exception e) {
                    errorCounter.increment();
                    log.error("Failed to process data: {}", data.getId(), e);
                    
                    // 에러율이 너무 높으면 배치 중단
                    double errorRate = errorCounter.count() / processed * 100;
                    if (errorRate > 5.0) { // 5% 초과 시 중단
                        throw new BatchProcessingException(
                            "Error rate too high: " + errorRate + "%");
                    }
                }
            }
            
        } finally {
            sample.stop(Timer.builder("batch.process.duration")
                           .tag("status", "completed")
                           .register(meterRegistry));
        }
    }
}
```

#### 재시작 가능한 배치 설계

```java
@Service
public class ResumableBatchService {
    
    @Autowired
    private BatchCheckpointRepository checkpointRepository;
    
    public void processLargeDataset(String batchId, List<Long> dataIds) {
        // 체크포인트에서 마지막 처리 위치 확인
        BatchCheckpoint checkpoint = checkpointRepository
            .findByBatchId(batchId)
            .orElse(new BatchCheckpoint(batchId, 0));
        
        int startIndex = checkpoint.getLastProcessedIndex();
        int batchSize = 1000;
        
        log.info("Resuming batch {} from index {}", batchId, startIndex);
        
        try {
            for (int i = startIndex; i < dataIds.size(); i += batchSize) {
                int end = Math.min(i + batchSize, dataIds.size());
                List<Long> chunk = dataIds.subList(i, end);
                
                processChunk(chunk);
                
                // 체크포인트 업데이트
                checkpoint.setLastProcessedIndex(end);
                checkpoint.setLastProcessedAt(LocalDateTime.now());
                checkpointRepository.save(checkpoint);
                
                log.debug("Processed chunk {}-{}", i, end);
            }
            
            // 완료 표시
            checkpoint.setCompleted(true);
            checkpointRepository.save(checkpoint);
            
        } catch (Exception e) {
            log.error("Batch processing failed at index {}: {}", 
                     checkpoint.getLastProcessedIndex(), e.getMessage());
            throw e;
        }
    }
}
```

---

## 4. 종합 운영 가이드

### 4.1 성능 위기 상황 대응 절차

#### 1단계: 즉시 대응 (1-5분 내)

```bash
# 1. 현재 실행 중인 쿼리 확인
SHOW PROCESSLIST;

# 2. 슬로우 쿼리 확인  
SHOW STATUS LIKE 'Slow_queries';

# 3. 커넥션 상태 확인
SHOW STATUS LIKE 'Connections';
SHOW STATUS LIKE 'Threads_connected';

# 4. 필요시 문제 쿼리 강제 종료
KILL QUERY [process_id];
```

#### 2단계: 원인 분석 (5-15분 내)

- 애플리케이션 로그에서 에러 패턴 확인
- 모니터링 도구로 CPU, 메모리, I/O 확인
- 재시도 요청이나 무한 루프 의심 코드 점검
- 최근 배포나 설정 변경 사항 검토

#### 3단계: 근본 해결 (15분-1시간 내)

- 문제 쿼리 최적화 또는 비활성화
- 서킷 브레이커나 레이트 리미터 적용
- 긴급 스케일링 (임시 방편)
- 핫픽스 배포 검토

### 4.2 예방적 모니터링 설정

#### 핵심 지표 알람 설정

```yaml
# Prometheus/Grafana 알람 규칙 예시
groups:
  - name: database_alerts
    rules:
      - alert: SlowQueryIncrease
        expr: mysql_global_status_slow_queries > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow queries increasing"
          description: "Slow queries: {{ $value }}/sec"
      
      - alert: ConnectionPoolExhaustion  
        expr: hikaricp_connections_active / hikaricp_connections_max > 0.8
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Connection pool nearly exhausted"
          
      - alert: ReplicationLag
        expr: mysql_slave_lag_seconds > 5
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Replication lag is high: {{ $value }}s"
```

#### 자동화된 대응 스크립트

```bash
#!/bin/bash
# auto_recovery.sh - 자동 복구 스크립트

# CPU 사용률이 90% 이상이면
if [ $(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | sed 's/%us,//') -gt 90 ]; then
    echo "High CPU detected. Checking for problematic queries..."
    
    # 5초 이상 실행 중인 쿼리 확인
    LONG_QUERIES=$(mysql -e "SELECT COUNT(*) FROM information_schema.processlist WHERE Time > 5;" -s)
    
    if [ $LONG_QUERIES -gt 10 ]; then
        echo "Too many long-running queries. Activating circuit breaker..."
        # 서킷 브레이커 활성화 API 호출
        curl -X POST http://localhost:8080/actuator/circuitbreaker/enable
    fi
fi
```

### 4.3 운영 체크리스트

#### 일일 점검 항목

- [ ] 슬로우 쿼리 로그 검토
- [ ] 커넥션 풀 사용률 확인
- [ ] 복제 지연 상태 점검
- [ ] 배치 작업 성공/실패 현황 확인
- [ ] 디스크 사용량 및 증가율 모니터링

#### 주간 점검 항목

- [ ] 인덱스 사용 통계
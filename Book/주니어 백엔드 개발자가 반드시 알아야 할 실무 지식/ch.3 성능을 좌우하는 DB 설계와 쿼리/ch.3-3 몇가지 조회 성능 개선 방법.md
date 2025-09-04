[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 조회 성능 개선 실무 가이드

## 1. 집계 데이터 최적화: 비정규화 전략

### 1.1 문제 상황

게시글 목록에서 좋아요, 댓글 수, 조회수 등 집계 데이터를 실시간으로 계산하면 성능 문제가 발생합니다.

```sql
-- 🚫 성능이 나쁜 실시간 집계 쿼리
SELECT 
    p.id,
    p.title,
    p.content,
    COUNT(l.id) as like_count,           -- 실시간 계산
    COUNT(c.id) as comment_count,        -- 실시간 계산
    p.view_count
FROM posts p
LEFT JOIN likes l ON p.id = l.post_id
LEFT JOIN comments c ON p.id = c.post_id
GROUP BY p.id, p.title, p.content, p.view_count
ORDER BY p.created_at DESC
LIMIT 20;
```

### 1.2 해결 방법: 비정규화

#### 테이블 구조 개선

```sql
-- ✅ 비정규화된 posts 테이블
CREATE TABLE posts (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    author_id BIGINT NOT NULL,
    
    -- 비정규화된 집계 컬럼들
    like_count INT DEFAULT 0,
    comment_count INT DEFAULT 0,
    view_count INT DEFAULT 0,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_created_at (created_at),
    INDEX idx_author_like (author_id, like_count)
);
```

#### 최적화된 조회 쿼리

```sql
-- ✅ 빠른 조회 쿼리 (JOIN 불필요)
SELECT 
    id,
    title,
    author_id,
    like_count,
    comment_count,
    view_count,
    created_at
FROM posts 
ORDER BY created_at DESC 
LIMIT 20;
```

### 1.3 집계 데이터 업데이트 전략

#### 1) 트리거 방식

```sql
-- 좋아요 추가 시 집계 업데이트
DELIMITER //
CREATE TRIGGER update_like_count_after_insert
    AFTER INSERT ON likes
    FOR EACH ROW
BEGIN
    UPDATE posts 
    SET like_count = like_count + 1 
    WHERE id = NEW.post_id;
END //
DELIMITER ;

-- 좋아요 삭제 시 집계 업데이트
DELIMITER //
CREATE TRIGGER update_like_count_after_delete
    AFTER DELETE ON likes
    FOR EACH ROW
BEGIN
    UPDATE posts 
    SET like_count = like_count - 1 
    WHERE id = OLD.post_id;
END //
DELIMITER ;
```

#### 2) 애플리케이션 레벨 관리

```java
@Service
@Transactional
public class PostService {
    
    // 좋아요 추가
    public void addLike(Long postId, Long userId) {
        // 1. 좋아요 데이터 삽입
        Like like = new Like(postId, userId);
        likeRepository.save(like);
        
        // 2. 집계 데이터 업데이트
        postRepository.incrementLikeCount(postId);
    }
    
    // 좋아요 취소
    public void removeLike(Long postId, Long userId) {
        // 1. 좋아요 데이터 삭제
        likeRepository.deleteByPostIdAndUserId(postId, userId);
        
        // 2. 집계 데이터 업데이트
        postRepository.decrementLikeCount(postId);
    }
}
```

### 1.4 ⚠️ 동시성 주의사항

#### 문제점

여러 사용자가 동시에 좋아요를 누르면 집계 데이터가 정확하지 않을 수 있습니다.

#### 해결 방법 1: 데이터베이스 락 활용

```sql
-- 비관적 락 사용
UPDATE posts 
SET like_count = like_count + 1 
WHERE id = ? 
FOR UPDATE;
```

#### 해결 방법 2: 원자적 연산 활용

```java
// Redis를 활용한 원자적 증감
@Service
public class PostCountService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    public void incrementLikeCount(Long postId) {
        String key = "post:like:" + postId;
        redisTemplate.opsForValue().increment(key);
        
        // 주기적으로 DB 동기화
        scheduleDbSync(postId);
    }
}
```

#### 해결 방법 3: 이벤트 기반 비동기 처리

```java
@Component
public class PostEventListener {
    
    @EventListener
    @Async
    public void handleLikeEvent(LikeCreatedEvent event) {
        // 비동기로 집계 데이터 업데이트
        postRepository.incrementLikeCount(event.getPostId());
    }
}
```

---

## 2. 페이징 최적화: Cursor 기반 페이징

### 2.1 OFFSET의 문제점

#### 성능 저하 예시

```sql
-- 🚫 OFFSET이 클수록 성능 급격히 저하
SELECT * FROM posts 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 100000;  -- 10만 개 레코드를 건너뛰고 조회

-- 실행 시간 비교 (100만 건 데이터 기준)
-- OFFSET 0:      0.01초
-- OFFSET 10000:  0.1초
-- OFFSET 100000: 1.2초
-- OFFSET 500000: 6.8초
```

#### OFFSET 문제점
- **데이터가 많을수록 성능 급격히 저하**
- **앞선 모든 레코드를 내부적으로 처리**해야 함
- **메모리 사용량 증가**

### 2.2 Cursor 기반 페이징 (Keyset Pagination)

#### 기본 개념

마지막으로 조회한 레코드의 기준값(보통 ID나 timestamp)을 WHERE 조건으로 사용

#### 구현 예시

```sql
-- ✅ 첫 페이지 조회
SELECT id, title, content, created_at 
FROM posts 
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- ✅ 다음 페이지 조회 (last_created_at, last_id는 이전 페이지 마지막 값)
SELECT id, title, content, created_at 
FROM posts 
WHERE created_at < '2024-01-15 10:30:00' 
   OR (created_at = '2024-01-15 10:30:00' AND id < 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

#### 인덱스 설계

```sql
-- Cursor 페이징을 위한 복합 인덱스
CREATE INDEX idx_posts_cursor ON posts (created_at DESC, id DESC);
```

### 2.3 실무 구현 패턴

#### API 응답 구조

```json
{
  "data": [
    {
      "id": 12345,
      "title": "게시글 제목",
      "created_at": "2024-01-15T10:30:00Z"
    }
  ],
  "pagination": {
    "has_next": true,
    "next_cursor": "MjAyNC0wMS0xNVQxMDozMDowMFp8MTIzNDU"  // base64 인코딩된 cursor
  }
}
```

#### 서버 구현 (Spring Boot)

```java
@RestController
public class PostController {
    
    @GetMapping("/posts")
    public ResponseEntity<PagedResponse<Post>> getPosts(
            @RequestParam(required = false) String cursor,
            @RequestParam(defaultValue = "20") int size) {
        
        CursorRequest cursorRequest = CursorRequest.of(cursor, size);
        PagedResponse<Post> response = postService.getPosts(cursorRequest);
        
        return ResponseEntity.ok(response);
    }
}

@Service
public class PostService {
    
    public PagedResponse<Post> getPosts(CursorRequest request) {
        List<Post> posts;
        
        if (request.hasCursor()) {
            // cursor가 있으면 해당 지점부터 조회
            posts = postRepository.findPostsAfterCursor(
                request.getCreatedAt(), 
                request.getId(), 
                request.getSize() + 1  // has_next 판단을 위해 +1
            );
        } else {
            // 첫 페이지 조회
            posts = postRepository.findRecentPosts(request.getSize() + 1);
        }
        
        boolean hasNext = posts.size() > request.getSize();
        if (hasNext) {
            posts.remove(posts.size() - 1);  // 마지막 아이템 제거
        }
        
        String nextCursor = hasNext ? 
            createCursor(posts.get(posts.size() - 1)) : null;
        
        return new PagedResponse<>(posts, hasNext, nextCursor);
    }
}
```

### 2.4 Cursor 페이징의 한계와 대안

#### 한계점
- **임의 페이지 이동 불가** (1페이지, 10페이지로 바로 이동 불가)
- **전체 페이지 수 표시 어려움**
- **정렬 기준이 고유해야 함**

#### 하이브리드 접근법

```java
public class PostService {
    
    public PagedResponse<Post> getPosts(PageRequest request) {
        // 초기 페이지들은 OFFSET 사용 (사용자 경험)
        if (request.getPage() <= 10) {
            return getPostsWithOffset(request);
        }
        
        // 깊은 페이지는 Cursor 사용 (성능)
        return getPostsWithCursor(request);
    }
}
```

---

## 3. COUNT 쿼리 최적화

### 3.1 COUNT의 성능 문제

#### 문제 상황

```sql
-- 🚫 대용량 테이블에서 COUNT(*)는 매우 느림
SELECT COUNT(*) FROM posts WHERE category_id = 1;

-- 성능 비교 (데이터량에 따른 실행 시간)
-- 1만 건:    0.01초
-- 10만 건:   0.1초  
-- 100만 건:  2.3초
-- 1000만 건: 45초
```

#### COUNT가 느린 이유
- **전체 레코드를 스캔**해야 함
- **인덱스만으로는 해결 불가능한 경우** 많음
- **페이지 단위로 읽어야 하는 I/O 비용**

### 3.2 COUNT 최적화 전략

#### 1) 근사값 사용

```sql
-- 정확한 COUNT 대신 근사값 제공
SELECT 
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'size_mb',
    table_rows AS 'estimated_rows'
FROM information_schema.tables
WHERE table_schema = 'your_database' 
  AND table_name = 'posts';
```

#### 2) 별도 집계 테이블 운영

```sql
-- 카테고리별 게시글 수를 별도 관리
CREATE TABLE post_statistics (
    category_id INT PRIMARY KEY,
    post_count INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 빠른 조회
SELECT post_count 
FROM post_statistics 
WHERE category_id = 1;
```

#### 3) Redis 활용

```java
@Service
public class PostCountService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    // 게시글 생성 시 카운트 증가
    public void incrementPostCount(int categoryId) {
        String key = "category:post_count:" + categoryId;
        redisTemplate.opsForValue().increment(key);
    }
    
    // 빠른 카운트 조회
    public long getPostCount(int categoryId) {
        String key = "category:post_count:" + categoryId;
        String count = redisTemplate.opsForValue().get(key);
        return count != null ? Long.parseLong(count) : 0L;
    }
}
```

#### 4) 샘플링 기반 추정

```sql
-- 전체 데이터의 일부를 샘플링하여 추정
SELECT 
    COUNT(*) * 100 as estimated_total
FROM (
    SELECT * FROM posts 
    WHERE RAND() < 0.01  -- 1% 샘플링
    LIMIT 1000
) sample_data;
```

### 3.3 실무 적용 가이드

#### COUNT 대안 선택 기준

|데이터 규모|정확도 요구|권장 방법|
|---|---|---|
|1만 건 이하|높음|실시간 COUNT|
|10만 건 이하|높음|인덱스 최적화 후 COUNT|
|100만 건 이상|보통|집계 테이블 + 주기적 동기화|
|1000만 건 이상|낮음|Redis 카운터 + 근사값|

#### UI에서 COUNT 대안 표시

```javascript
// 정확한 수치 대신 범위 표시
function formatCount(count) {
    if (count < 1000) return count.toString();
    if (count < 10000) return Math.floor(count / 1000) + 'K+';
    if (count < 1000000) return Math.floor(count / 10000) + '만+';
    return Math.floor(count / 1000000) + 'M+';
}

// 예시: 1,234 → "1K+", 12,345 → "1만+", 1,234,567 → "1M+"
```

---

## 4. 데이터 생명주기 관리

### 4.1 불필요한 데이터의 성능 영향

#### 문제 상황

```sql
-- 🚫 로그인 시도 내역이 누적되어 테이블 크기 증가
CREATE TABLE login_attempts (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    ip_address VARCHAR(45),
    success BOOLEAN,
    attempted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_attempted (user_id, attempted_at)
);

-- 3년 후: 수억 건의 레코드로 인한 성능 저하
-- - 인덱스 크기 증가
-- - 쿼리 응답 시간 증가  
-- - 백업/복구 시간 증가
-- - 스토리지 비용 증가
```

### 4.2 데이터 정리 전략

#### 1) 자동 삭제 (TTL 설정)

```sql
-- MySQL 8.0+ 에서 자동 삭제 설정
CREATE TABLE login_attempts (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    ip_address VARCHAR(45),
    success BOOLEAN,
    attempted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 30일 후 자동 삭제
    expires_at TIMESTAMP GENERATED ALWAYS AS (attempted_at + INTERVAL 30 DAY),
    
    INDEX idx_user_attempted (user_id, attempted_at),
    INDEX idx_expires_at (expires_at)
);

-- 정기적으로 만료된 데이터 삭제
DELETE FROM login_attempts 
WHERE expires_at < NOW() 
LIMIT 10000;
```

#### 2) 파티셔닝을 통한 자동 관리

```sql
-- 월별 파티셔닝으로 오래된 파티션 삭제
CREATE TABLE login_attempts (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    ip_address VARCHAR(45),
    success BOOLEAN,
    attempted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) PARTITION BY RANGE (TO_DAYS(attempted_at)) (
    PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
    PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
    PARTITION p202403 VALUES LESS THAN (TO_DAYS('2024-04-01')),
    -- ...
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- 오래된 파티션 삭제 (매우 빠름)
ALTER TABLE login_attempts DROP PARTITION p202401;
```

#### 3) 아카이브 테이블로 이동

```sql
-- 아카이브 테이블 생성 (압축 적용)
CREATE TABLE login_attempts_archive (
    id BIGINT,
    user_id BIGINT,
    ip_address VARCHAR(45),
    success BOOLEAN,
    attempted_at TIMESTAMP,
    archived_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=MyISAM ROW_FORMAT=COMPRESSED;

-- 주기적으로 오래된 데이터 아카이브
INSERT INTO login_attempts_archive 
SELECT *, NOW() 
FROM login_attempts 
WHERE attempted_at < DATE_SUB(NOW(), INTERVAL 90 DAY);

DELETE FROM login_attempts 
WHERE attempted_at < DATE_SUB(NOW(), INTERVAL 90 DAY);
```

### 4.3 데이터 타입별 관리 전략

#### 로그성 데이터

```sql
-- 로그 테이블 설계 시 고려사항
CREATE TABLE api_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    endpoint VARCHAR(255),
    method VARCHAR(10),
    response_time INT,
    status_code INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 보관 기간별 인덱스
    INDEX idx_recent (created_at) -- 최근 데이터 조회용
) 
PARTITION BY RANGE (TO_DAYS(created_at))
-- 일별 파티셔닝으로 관리
```

#### 이벤트 데이터

```sql
-- 이벤트 데이터는 집계 후 상세 데이터 삭제
CREATE TABLE user_events (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    event_type VARCHAR(50),
    occurred_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_date (user_id, occurred_at)
);

-- 월별 집계 테이블로 요약
CREATE TABLE user_event_summary (
    user_id BIGINT,
    event_type VARCHAR(50),
    event_count INT,
    summary_month DATE,
    
    PRIMARY KEY (user_id, event_type, summary_month)
);
```

#### 세션/캐시 데이터

```sql
-- Redis를 활용한 자동 만료
SET user_session:12345 "session_data" EX 3600  -- 1시간 후 자동 삭제

-- 애플리케이션에서 관리
@Component
public class SessionManager {
    
    @Scheduled(cron = "0 0 * * * *")  // 매시간 실행
    public void cleanupExpiredSessions() {
        sessionRepository.deleteExpiredSessions(
            LocalDateTime.now().minusHours(2)
        );
    }
}
```

### 4.4 데이터 정리 자동화

#### 배치 작업 스케줄링

```java
@Component
public class DataCleanupScheduler {
    
    // 매일 새벽 2시에 실행
    @Scheduled(cron = "0 0 2 * * *")
    public void cleanupOldData() {
        
        // 1. 로그인 시도 내역 (30일 보관)
        cleanupLoginAttempts();
        
        // 2. API 로그 (7일 보관)
        cleanupApiLogs();
        
        // 3. 임시 파일 (1일 보관)
        cleanupTempFiles();
        
        log.info("Data cleanup completed");
    }
    
    private void cleanupLoginAttempts() {
        LocalDateTime cutoff = LocalDateTime.now().minusDays(30);
        int deleted = loginAttemptRepository.deleteOldRecords(cutoff);
        log.info("Deleted {} old login attempts", deleted);
    }
}
```

#### 점진적 삭제 (큰 테이블 대응)

```java
@Service
public class DataCleanupService {
    
    @Async
    public void cleanupLargeTable(String tableName, LocalDateTime cutoff) {
        int batchSize = 10000;
        int totalDeleted = 0;
        
        while (true) {
            int deleted = jdbcTemplate.update(
                "DELETE FROM " + tableName + " WHERE created_at < ? LIMIT " + batchSize,
                cutoff
            );
            
            totalDeleted += deleted;
            
            if (deleted < batchSize) {
                break; // 더 이상 삭제할 데이터 없음
            }
            
            // 다른 쿼리에 영향을 주지 않기 위해 잠시 대기
            Thread.sleep(100);
        }
        
        log.info("Cleanup completed. Total deleted: {}", totalDeleted);
    }
}
```

---

## 5. 종합 최적화 체크리스트

### 5.1 집계 데이터 최적화

- [ ] COUNT 쿼리를 비정규화된 컬럼으로 대체
- [ ] 집계 데이터 업데이트 시 동시성 문제 해결
- [ ] 트리거 또는 애플리케이션 레벨에서 집계 관리
- [ ] Redis 등을 활용한 실시간 카운터 구현

### 5.2 페이징 최적화

- [ ] OFFSET 대신 Cursor 기반 페이징 구현
- [ ] 적절한 복합 인덱스 생성
- [ ] API 응답에 next_cursor 포함
- [ ] 깊은 페이지에서 성능 테스트 완료

### 5.3 COUNT 성능 개선

- [ ] 대용량 테이블의 COUNT(*) 쿼리 식별
- [ ] 근사값 또는 집계 테이블 활용
- [ ] UI에서 "1만+" 같은 범위 표시 적용
- [ ] Redis 등을 활용한 실시간 카운터

### 5.4 데이터 생명주기 관리

- [ ] 불필요한 로그 데이터 식별
- [ ] 자동 삭제 정책 수립
- [ ] 파티셔닝 또는 아카이브 전략 적용
- [ ] 배치 작업으로 정기적 정리 자동화

### 5.5 성능 모니터링

- [ ] 슬로우 쿼리 로그 활성화
- [ ] 쿼리 실행 계획 정기 검토
- [ ] 테이블 크기 모니터링
- [ ] 인덱스 사용률 점검

---

## 마무리

조회 성능 최적화는 **데이터의 특성과 사용 패턴을 정확히 파악**하는 것에서 시작됩니다.

### 핵심 원칙

1. **실시간 계산을 피하고 미리 계산된 값 활용**
2. **OFFSET 기반 페이징보다 Cursor 기반 페이징 선호**
3. **정확한 COUNT보다 빠른 근사값 제공**
4. **불필요한 데이터는 적극적으로 정리**

각 최적화 기법을 적용할 때는 **트레이드오프**를 충분히 고려하고, **단계적으로 적용**하여 효과를 검증하시기 바랍니다.
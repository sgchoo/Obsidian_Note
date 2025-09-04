[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 데이터베이스 Primary Key 전략 가이드

## 📌 목차
1. [인덱스 기본 개념](https://claude.ai/chat/473a0096-0a35-450b-9107-ede1b320f3f4#1-%EC%9D%B8%EB%8D%B1%EC%8A%A4-%EA%B8%B0%EB%B3%B8-%EA%B0%9C%EB%85%90)
2. [Primary Key와 인덱스](https://claude.ai/chat/473a0096-0a35-450b-9107-ede1b320f3f4#2-primary-key%EC%99%80-%EC%9D%B8%EB%8D%B1%EC%8A%A4)
3. [UUID vs Auto Increment 성능 비교](https://claude.ai/chat/473a0096-0a35-450b-9107-ede1b320f3f4#3-uuid-vs-auto-increment-%EC%84%B1%EB%8A%A5-%EB%B9%84%EA%B5%90)
4. [이중 키 전략](https://claude.ai/chat/473a0096-0a35-450b-9107-ede1b320f3f4#4-%EC%9D%B4%EC%A4%91-%ED%82%A4-%EC%A0%84%EB%9E%B5)
5. [실무 최적화 패턴](https://claude.ai/chat/473a0096-0a35-450b-9107-ede1b320f3f4#5-%EC%8B%A4%EB%AC%B4-%EC%B5%9C%EC%A0%81%ED%99%94-%ED%8C%A8%ED%84%B4)
6. [결론 및 권장사항](https://claude.ai/chat/473a0096-0a35-450b-9107-ede1b320f3f4#6-%EA%B2%B0%EB%A1%A0-%EB%B0%8F-%EA%B6%8C%EC%9E%A5%EC%82%AC%ED%95%AD)

---

## 1. 인덱스 기본 개념

### 인덱스란?
- 데이터베이스에서 빠른 검색을 위한 **B+Tree 자료구조**
- "테이블"이 아닌 "자료구조"로 이해해야 함
- 책의 목차나 색인과 유사한 개념

### 인덱스 타입

#### Clustered Index (Primary Key)

```sql
-- 데이터 자체가 PK 순서로 물리적으로 정렬
-- 테이블당 1개만 존재
CREATE TABLE article (
    id INTEGER PRIMARY KEY,  -- 자동으로 Clustered Index
    title VARCHAR(200)
);
```

#### Secondary Index

```sql
-- 별도의 B+Tree 구조 생성
-- 여러 개 생성 가능
CREATE INDEX idx_category ON article(category);
```

### 데이터가 적을 때 인덱스의 역효과

```sql
-- 10건 데이터에서 조회

-- 인덱스 사용 시
1. 인덱스 페이지 읽기
2. 데이터 페이지 읽기
= 2번의 I/O

-- Full Scan 시
1. 데이터 페이지 읽기 (한 번에)
= 1번의 I/O
```

**결론**: 데이터 1,000건 이하에서는 인덱스가 오히려 느릴 수 있음

---

## 2. Primary Key와 인덱스

### PK는 자동 인덱싱

```sql
CREATE TABLE article (
    id INTEGER(10) PRIMARY KEY,  -- 자동으로 인덱스 생성됨
    category INTEGER(10),
    writerId INTEGER(10)
);

-- PK 검색은 항상 빠름 (Full Scan 하지 않음)
SELECT * FROM article WHERE id = 5;  -- O(log n)
```

### PK가 없으면?

- 중복 데이터 방지 불가
- 특정 row 식별 불가
- UPDATE/DELETE 시 Full Scan
- **절대 PK 없는 테이블 만들지 말 것!**

---

## 3. UUID vs Auto Increment 성능 비교

### 성능 벤치마크 (100만 건 기준)

|작업|AUTO_INCREMENT|UUID v4 (랜덤)|UUID v7 (시간순)|
|---|---|---|---|
|INSERT|10초|45초 (4.5배 느림)|25초 (2.5배 느림)|
|단일 조회|0.001초|0.002초|0.002초|
|JOIN|0.05초|0.8초 (16배 느림)|0.8초|
|인덱스 크기|20MB|180MB (9배)|180MB|

### UUID가 느린 이유

#### 1. 랜덤 삽입으로 인한 페이지 분할

```sql
-- AUTO_INCREMENT: 순차 삽입
[1][2][3][4][5] → [6] 끝에만 추가

-- UUID: 랜덤 삽입
[1xxx][5xxx][9xxx] → [3xxx] 중간에 삽입 → 페이지 분할
```

#### 2. 크기 차이

- INTEGER: 4 bytes
- BIGINT: 8 bytes
- UUID: 36 bytes (문자열) 또는 16 bytes (binary)

#### 3. JOIN 성능

```sql
-- BIGINT JOIN
WHERE orders.user_id = users.id  -- 8 bytes 비교

-- UUID JOIN  
WHERE orders.user_id = users.id  -- 36 bytes 비교
```

---

## 4. 이중 키 전략

### 개념

내부 성능과 외부 보안을 모두 만족시키는 설계 패턴

```sql
CREATE TABLE users (
    -- 내부용: 성능 최적화
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    
    -- 외부용: 보안, API 노출
    public_id CHAR(36) DEFAULT (UUID()),
    
    email VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE INDEX idx_public_id(public_id)
);
```

### 장점

#### 1. 보안

```javascript
// ❌ 정수 ID 노출 - 예측 가능
GET /api/users/12345
GET /api/users/12346  // 다음 유저 추측 가능

// ✅ UUID 노출 - 예측 불가능  
GET /api/users/550e8400-e29b-41d4
GET /api/users/??????  // 추측 불가능
```

#### 2. 성능

```sql
-- 내부 JOIN (빠름)
SELECT * FROM orders WHERE user_id = 12345;  -- BIGINT

-- 외부 API 조회 (인덱스로 충분히 빠름)
SELECT * FROM users WHERE public_id = 'uuid...';  -- INDEX 사용
```

#### 3. 비즈니스 정보 보호

```python
# 정수 ID: 비즈니스 규모 노출
"주문번호 10000 → 10500" = "일주일 500건"

# UUID: 정보 보호
"abc-123 → def-456" = "???"
```

### 실제 사용 예시

```javascript
// API 엔드포인트
app.get('/api/users/:publicId', async (req, res) => {
    // public_id로 조회
    const user = await db.query(
        'SELECT * FROM users WHERE public_id = ?',
        [req.params.publicId]
    );
    
    // 응답에도 public_id만 노출
    res.json({
        id: user.public_id,  // 내부 id는 숨김
        email: user.email
    });
});

// 내부 처리는 정수 ID 사용
async function getUserOrders(userId) {
    return await db.query(
        'SELECT * FROM orders WHERE user_id = ?',
        [userId]  // BIGINT로 빠른 조회
    );
}
```

---

## 5. 실무 최적화 패턴

### 패턴 1: 역정규화

```sql
-- UUID를 관련 테이블에 복사
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,              -- 내부 JOIN용
    user_public_id CHAR(36),     -- 외부 조회용
    
    INDEX idx_user_public_id(user_public_id)
);

-- JOIN 없이 바로 조회 가능
SELECT * FROM orders WHERE user_public_id = 'uuid...';
```

### 패턴 2: 캐싱

```javascript
// Redis로 UUID → ID 매핑 캐싱
async function getUserIdByPublicId(publicId) {
    const cached = await redis.get(`user:${publicId}`);
    if (cached) return cached;
    
    const result = await db.query(
        'SELECT id FROM users WHERE public_id = ?',
        [publicId]
    );
    
    await redis.set(`user:${publicId}`, result.id, 'EX', 3600);
    return result.id;
}
```

### 패턴 3: Bulk Insert 최적화

```sql
-- 대량 데이터 입력 시
ALTER TABLE users DISABLE KEYS;
-- 100만건 INSERT
ALTER TABLE users ENABLE KEYS;

-- 또는 Bulk INSERT 사용
INSERT INTO users (name, email) VALUES 
('user1', 'email1'),
('user2', 'email2'),
('user3', 'email3');
```

### 패턴 4: 애플리케이션 레벨 처리

```javascript
class UserService {
    async getUserWithRelations(publicId) {
        // 1회 조회로 ID 획득
        const user = await this.getUserByPublicId(publicId);
        
        // 정수 ID로 병렬 조회
        const [orders, reviews] = await Promise.all([
            this.getOrders(user.id),
            this.getReviews(user.id)
        ]);
        
        return { ...user, orders, reviews };
    }
}
```

---

## 6. 결론 및 권장사항

### 상황별 선택 가이드

#### 1. 일반 웹 서비스 (권장) ✅

```sql
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    public_id CHAR(36) DEFAULT (UUID()),
    UNIQUE INDEX idx_public_id(public_id)
);
```

#### 2. 초고속 INSERT 필요 (로그)

```sql
CREATE TABLE logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY
    -- UUID 없이 사용
);
```

#### 3. 완전 분산 시스템

```sql
CREATE TABLE distributed_data (
    id CHAR(36) PRIMARY KEY  -- UUID v7 사용
);
```

### 핵심 원칙

1. **항상 PK는 필수** - 없으면 성능/무결성 문제
2. **보안이 필요하면 이중 키** - 성능 손실 최소화
3. **대량 INSERT는 순차 키** - AUTO_INCREMENT 최고
4. **JOIN이 많으면 정수 키** - UUID JOIN은 매우 느림
5. **트레이드오프 이해** - 완벽한 해결책은 없음

### 체크리스트

- [ ] PK는 반드시 설정했는가?
- [ ] 외부 노출 API인가? → 이중 키 고려
- [ ] JOIN이 많은가? → 정수 PK 필수
- [ ] 대량 INSERT가 빈번한가? → AUTO_INCREMENT
- [ ] 분산 시스템인가? → UUID v7 또는 Snowflake ID
- [ ] 적절한 인덱스를 추가했는가?

### 대기업 사례

|서비스|전략|특징|
|---|---|---|
|Instagram|BIGINT + BASE64 인코딩|샤딩된 ID 사용|
|Twitter|Snowflake ID|시간+머신+시퀀스 조합|
|Stripe|BIGINT + prefix_random|`cus_Qs1Gd...` 형식|
|YouTube|BIGINT + 11자 문자열|BASE64 변형|

---

## 마무리

> "빠른 INSERT를 원한다면 AUTO_INCREMENT,  
> 보안을 원한다면 UUID,  
> 둘 다 원한다면 이중 키 전략!"

**Remember**: 데이터베이스 설계는 트레이드오프의 연속입니다. 상황에 맞는 최적의 선택을 하되, 나중에 변경하기 어려운 PK는 특히 신중하게 결정하세요.
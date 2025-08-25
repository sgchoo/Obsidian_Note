[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# DB 성능 향상 가이드

## 1. 성능 문제의 핵심 원인

### 1.1 DB가 성능 병목의 주요 원인

- 성능 문제 발생 시 **DB 조회를 가장 먼저 의심**해야 함
- 대부분의 성능 저하는 DB 사용 패턴에서 발생

### 1.2 일반적인 문제 패턴

- **풀스캔(Full Table Scan)**: 인덱스 없이 전체 테이블 스캔
- **N+1 쿼리 문제**: 반복적인 개별 조회
- **비효율적인 조인**: 잘못된 조인 전략
- **불필요한 데이터 조회**: 필요 이상의 컬럼이나 로우 조회

### 1.3 문제 해결 방향

> **핵심**: DB 자체의 문제보다는 **DB를 잘못 사용하는 경우**가 대부분
> 
> 따라서 DB 교체나 하드웨어 업그레이드보다 **쿼리 최적화**를 우선 검토

---

## 2. 조회 트래픽을 고려한 인덱스 설계

### 2.1 인덱스 설계 원칙

#### 기본 원칙

- **조회 패턴 기반 설계**: 실제 사용되는 WHERE 조건을 기준으로 인덱스 생성
- **풀스캔 방지**: 모든 주요 조회 쿼리에 인덱스 적용
- **데이터 규모 고려**: 데이터 양에 따른 전략 수립

#### 데이터 규모별 전략

|데이터 규모|권장 전략|비고|
|---|---|---|
|**수천 건**|단일 인덱스만으로도 충분|심각한 성능 문제 발생 가능성 낮음|
|**수만 건 이상**|복합 인덱스 설계 필요|단일 인덱스만으로는 성능 한계|
|**수십만 건 이상**|파티셔닝, 커버링 인덱스 고려|고급 최적화 기법 적용|

### 2.2 선택도(Selectivity)와 인덱스

#### 선택도의 정의

```
선택도 = DISTINCT 값의 수 / 전체 레코드 수
```

#### 선택도 기준

- **높은 선택도 (0.8 이상)**: 인덱스 효과 큼
    - 예: 사용자 ID, 이메일, 주문번호
- **낮은 선택도 (0.1 이하)**: 인덱스 효과 제한적
    - 예: 성별, 상태값, 카테고리

#### 예외 상황: 작업 큐(Job Queue) 테이블

```sql
-- 작업 큐 테이블 예시
CREATE TABLE job_queue (
    id BIGINT PRIMARY KEY,
    status ENUM('PENDING', 'PROCESSING', 'COMPLETED') NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_status_created (status, created_at)  -- 낮은 선택도지만 필요
);

-- 대기 중인 작업 조회 (자주 실행되는 쿼리)
SELECT * FROM job_queue 
WHERE status = 'PENDING' 
ORDER BY created_at 
LIMIT 10;
```

**이유**: 비록 `status` 컬럼의 선택도가 낮지만, 실제 조회되는 데이터는 소수이므로 인덱스 효과가 있음

### 2.3 복합 인덱스 설계

#### 컬럼 순서 결정 원칙

1. **선택도가 높은 컬럼 우선**
2. **등호 조건(=) 우선, 범위 조건 나중**
3. **ORDER BY 절 고려**

#### 실제 예시

```sql
-- 게시판 조회 쿼리
SELECT * FROM posts 
WHERE category_id = 1 
  AND status = 'PUBLISHED' 
  AND created_at >= '2024-01-01'
ORDER BY created_at DESC
LIMIT 20;

-- 최적 인덱스 설계
CREATE INDEX idx_posts_optimized ON posts (
    category_id,    -- 등호 조건, 높은 선택도
    status,         -- 등호 조건, 낮은 선택도
    created_at      -- 범위 조건 + ORDER BY
);
```

### 2.4 커버링 인덱스 (Covering Index)

#### 정의

- 쿼리 실행에 필요한 **모든 컬럼을 포함**한 인덱스
- **SELECT, WHERE, ORDER BY, GROUP BY**에 사용되는 모든 컬럼이 인덱스에 포함

#### 장점

- **실제 테이블 데이터 조회 불필요** → 디스크 I/O 감소
- **성능 대폭 향상** 가능

#### 구현 예시

```sql
-- 자주 실행되는 쿼리
SELECT user_id, name, email, created_at 
FROM users 
WHERE status = 'ACTIVE' 
  AND created_at >= '2024-01-01'
ORDER BY created_at DESC;

-- 커버링 인덱스 생성
CREATE INDEX idx_users_covering ON users (
    status,         -- WHERE 조건
    created_at,     -- WHERE 조건 + ORDER BY
    user_id,        -- SELECT 컬럼
    name,           -- SELECT 컬럼
    email           -- SELECT 컬럼
);
```

#### 주의사항

- **인덱스 크기 증가** → 메모리 사용량 증가
- **INSERT/UPDATE 성능 저하** → 인덱스 유지 비용 증가
- **필요한 경우에만 선택적 적용**

---

## 3. 서버 성능 최적화

### 3.1 스케일링 전략

#### 긴급 상황 대응

- **Scale-up (수직 확장)**: 즉시 응답시간↓, 처리량↑
- **Scale-out (수평 확장)**: 비용 효율적, TPS 향상

#### ⚠️ 중요한 주의사항

> **DB에 문제가 있는 상황에서 서버만 추가하면 불에 기름을 붓는 격**
> 
> 반드시 DB 성능 문제를 먼저 해결한 후 스케일링 진행

### 3.2 DB 커넥션 풀 관리

#### 커넥션 풀의 필요성

- 직접 연결 시 **연결/해제 시간이 0.5~1초**로 매우 비효율적
- 커넥션 풀 사용으로 연결 재사용

#### 주요 설정 매개변수

|설정|권장값|설명|
|---|---|---|
|**풀 크기**|CPU 코어 수 × 2~4|DB 서버 상태 고려하여 조정|
|**대기 시간**|1~3초|짧게 설정하여 빠른 실패 처리|
|**최대 유휴시간**|DB 설정의 80%|DB 자동 연결 해제 방지|
|**유효성 검사**|활성화 필수|끊어진 연결 감지|

#### 커넥션 처리 프로세스

```
1. 풀에서 커넥션 요청
2. 유효성 검사 실행
3. 사용 불가 시 새 커넥션 생성
4. DB 쿼리 실행
5. 커넥션을 풀에 반환
```

### 3.3 캐시 전략

#### 캐시 종류별 특성

|캐시 타입|장점|단점|적용 사례|
|---|---|---|---|
|**로컬 캐시**|빠른 응답 속도<br>네트워크 오버헤드 없음|서버별 독립 관리<br>데이터 일관성 문제|설정 정보<br>정적 데이터|
|**리모트 캐시**|데이터 공유 가능<br>일관성 유지|네트워크 오버헤드<br>추가 인프라 필요|사용자 세션<br>실시간 데이터|

#### 캐시 전략 선택 가이드

- **읽기 전용 데이터**: 로컬 캐시 우선 고려
- **자주 변경되는 데이터**: 리모트 캐시 또는 캐시 TTL 짧게 설정
- **대용량 데이터**: 파티셔닝된 리모트 캐시

#### 🔴 캐시 관리의 핵심

> **적절한 시점에 캐시를 무효화(삭제)**하는 것이 가장 중요
> 
> 캐시 무효화 전략을 먼저 설계한 후 캐시 도입

---

## 4. 메모리 및 리소스 최적화

### 4.1 가비지 컬렉션(GC) 최적화

#### GC의 양면성

- **장점**: 메모리 관리 자동화
- **단점**: GC 실행 중 애플리케이션 일시 중단

#### 메모리 사용량 최적화 방법

##### 1. 페이지네이션

```sql
-- Before: 전체 데이터 조회 (메모리 과부하)
SELECT * FROM large_table WHERE condition;

-- After: 페이지네이션 적용
SELECT * FROM large_table 
WHERE condition 
ORDER BY id 
LIMIT 100 OFFSET 0;
```

##### 2. 스트림 처리

```java
// Before: 전체 파일을 메모리에 로드
byte[] fileContent = Files.readAllBytes(filePath);

// After: 스트림 방식으로 처리
try (InputStream inputStream = Files.newInputStream(filePath);
     OutputStream outputStream = response.getOutputStream()) {
    inputStream.transferTo(outputStream);
}
```

##### 3. 배치 처리

```java
// Before: 한 번에 전체 처리
List<Data> allData = repository.findAll();
processAllData(allData);

// After: 배치 단위로 처리
int batchSize = 1000;
int offset = 0;
List<Data> batch;

do {
    batch = repository.findBatch(offset, batchSize);
    processBatch(batch);
    offset += batchSize;
} while (batch.size() == batchSize);
```

### 4.2 응답 데이터 압축

#### 압축 대상 파일

- **압축 효과 큰 파일**: HTML, CSS, JavaScript, JSON, XML
- **압축 제외 파일**: JPEG, PNG, ZIP, PDF (이미 압축된 파일)

#### 압축 효과

- **파일 크기 약 70% 감소**
- 네트워크 대역폭 절약
- 전송 시간 단축

#### 설정 예시 (Nginx)

```nginx
# gzip 압축 설정
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_types
    text/plain
    text/css
    text/xml
    text/javascript
    application/json
    application/javascript
    application/xml
    application/rss+xml;
```

---

## 5. 실무 적용 가이드

### 5.1 인덱스 생성 시 주의사항

#### 언제 인덱스를 추가하지 말아야 하는가?

- **소규모 테이블**: 수천 건 이하의 데이터
- **읽기보다 쓰기가 많은 테이블**: INSERT/UPDATE 성능 저하
- **이미 충분한 인덱스가 있는 경우**: 중복 인덱스 방지

#### 인덱스 성능 모니터링

```sql
-- MySQL: 인덱스 사용 통계 확인
SELECT 
    TABLE_SCHEMA,
    TABLE_NAME,
    INDEX_NAME,
    CARDINALITY,
    SUB_PART
FROM information_schema.STATISTICS 
WHERE TABLE_SCHEMA = 'your_database';

-- 사용되지 않는 인덱스 찾기
SHOW INDEX FROM your_table;
```

### 5.2 성능 측정 및 모니터링

#### 쿼리 성능 분석

```sql
-- 실행 계획 확인
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- 슬로우 쿼리 로그 설정
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 1초 이상 쿼리 기록
```

#### 주요 모니터링 지표

- **응답 시간**: 평균, 95th percentile
- **처리량**: QPS (Queries Per Second)
- **DB 커넥션 사용률**: 풀 사용량 모니터링
- **캐시 히트율**: 캐시 효율성 측정

### 5.3 점진적 개선 전략

#### 성능 개선 우선순위

1. **🔍 원인 분석**: DB vs 애플리케이션 문제 구분
2. **📊 슬로우 쿼리 식별**: 가장 문제가 되는 쿼리 찾기
3. **📈 인덱스 최적화**: 풀스캔 쿼리부터 해결
4. **💾 캐시 도입**: 변경이 적은 데이터부터 캐시 적용
5. **🔄 커넥션 풀 조정**: 적절한 크기로 조정
6. **🗜️ 응답 압축**: 즉시 적용 가능한 최적화
7. **⚖️ 스케일링**: 마지막 수단으로 하드웨어 확장

### 5.4 체크리스트

#### DB 최적화 체크리스트

- [ ] 주요 조회 쿼리에 적절한 인덱스 존재
- [ ] 복합 인덱스 컬럼 순서가 최적화됨
- [ ] 불필요한 인덱스 제거
- [ ] 슬로우 쿼리 로그 모니터링 설정
- [ ] 실행 계획 정기적 검토

#### 애플리케이션 최적화 체크리스트

- [ ] 커넥션 풀 크기 적절히 설정
- [ ] 커넥션 대기 시간 짧게 설정
- [ ] 유효성 검사 기능 활성화
- [ ] 페이지네이션 적용
- [ ] 파일 처리 시 스트림 방식 사용
- [ ] 텍스트 응답 압축 활성화
- [ ] 캐시 무효화 전략 수립
- [ ] 트래픽 제한(Rate Limiting) 설정

---

## 6. 고급 최적화 기법

### 6.1 데이터베이스 파티셔닝

#### 수평 파티셔닝 (샤딩)

```sql
-- 날짜 기준 파티셔닝
CREATE TABLE orders_2024_01 (
    id BIGINT,
    order_date DATE,
    -- 기타 컬럼들
) PARTITION BY RANGE (YEAR(order_date) * 100 + MONTH(order_date));
```

#### 수직 파티셔닝

- 자주 조회되지 않는 컬럼을 별도 테이블로 분리
- BLOB, TEXT 등 대용량 데이터 분리

### 6.2 읽기 전용 복제본 활용

#### Master-Slave 구조

```java
// 읽기 쿼리는 Slave로 분산
@Transactional(readOnly = true)
public List<User> findActiveUsers() {
    return userRepository.findByStatus("ACTIVE");
}

// 쓰기 쿼리는 Master로
@Transactional
public User createUser(User user) {
    return userRepository.save(user);
}
```

### 6.3 쿼리 최적화 고급 기법

#### 서브쿼리 최적화

```sql
-- Before: 비효율적인 서브쿼리
SELECT * FROM users 
WHERE id IN (
    SELECT user_id FROM orders 
    WHERE created_at >= '2024-01-01'
);

-- After: JOIN으로 최적화
SELECT DISTINCT u.* FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE o.created_at >= '2024-01-01';
```

---

## 마무리

DB 성능 향상은 **점진적이고 체계적인 접근**이 필요합니다. 하드웨어 업그레이드나 DB 교체보다는 **올바른 사용법과 최적화**가 우선입니다.

### 핵심 원칙

1. **측정 없는 최적화 금지**: 성능 지표를 먼저 측정
2. **병목 지점 우선 해결**: 가장 큰 문제부터 해결
3. **단계별 적용**: 한 번에 여러 최적화를 적용하지 않음
4. **지속적 모니터링**: 최적화 후 효과 검증

이 가이드를 참고하여 체계적으로 DB 성능을 개선하시기 바랍니다.


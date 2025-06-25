[[Code - 찰스 펫졸드]]

# Code - Chapter 6: 전신과 릴레이

## 📖 개요

Chapter 6에서는 불 대수(Boolean Algebra)의 기초가 되는 집합론을 통해 논리적 사고의 수학적 기반을 설명합니다. 이는 데이터베이스의 관계 대수와 논리 게이트의 이론적 토대를 마련하는 중요한 장입니다.

## 🔢 집합론의 기초

### 집합의 기본 개념

**기초 수학의 집합을 이용해서 불 대수를 설명**

집합(Set)은 명확히 정의된 객체들의 모임:

```
집합 A = {1, 2, 3, 4, 5}
집합 B = {3, 4, 5, 6, 7}
전체집합 U = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
```

### 집합 연산

#### 1. 합집합 (Union) - OR 연산의 기초

```
A ∪ B = {1, 2, 3, 4, 5, 6, 7}
```

- 두 집합 중 하나라도 포함된 원소들
- 논리의 OR 연산과 대응

#### 2. 교집합 (Intersection) - AND 연산의 기초

```
A ∩ B = {3, 4, 5}
```

- 두 집합 모두에 포함된 원소들
- 논리의 AND 연산과 대응

#### 3. 여집합 (Complement) - NOT 연산의 기초

```
A' = U - A = {6, 7, 8, 9, 10}
```

- 전체집합에서 해당 집합을 뺀 원소들
- 논리의 NOT 연산과 대응

## 🔗 불 대수로의 확장

### 집합에서 불 대수로

집합 연산 → 불 대수 연산 → 논리 게이트:

```
집합론          불 대수        논리 게이트
합집합 (∪)   →   OR (+)    →   OR 게이트
교집합 (∩)   →   AND (·)   →   AND 게이트  
여집합 (')   →   NOT (¯)   →   NOT 게이트
```

### 불 대수의 기본 법칙

#### 1. 교환법칙 (Commutative Law)

```
집합: A ∪ B = B ∪ A
불 대수: X + Y = Y + X
```

#### 2. 결합법칙 (Associative Law)

```
집합: (A ∪ B) ∪ C = A ∪ (B ∪ C)
불 대수: (X + Y) + Z = X + (Y + Z)
```

#### 3. 분배법칙 (Distributive Law)

```
집합: A ∩ (B ∪ C) = (A ∩ B) ∪ (A ∩ C)
불 대수: X · (Y + Z) = (X · Y) + (X · Z)
```

#### 4. 드모르간 법칙 (De Morgan's Law)

```
집합: (A ∪ B)' = A' ∩ B'
불 대수: (X + Y)' = X' · Y'
```

## 🗃️ 데이터베이스와 관계 대수

### 집합의 확장: 테이블 구조

**집합을 확장시키면 DB 테이블과도 같다**

```
집합 (1차원):
A = {1, 2, 3, 4, 5}

릴레이션 (2차원 테이블):
학생 = {
  (1, "김철수", 20),
  (2, "이영희", 21), 
  (3, "박민수", 19)
}
```

### 관계 대수의 연산들

**DB에도 관계 대수라고 하는 집합의 개념이 존재한다**

#### 1. 선택 (Selection) - σ

sql

```sql
-- 나이가 20 이상인 학생들
SELECT * FROM 학생 WHERE 나이 >= 20;

-- 집합 표기법
σ(나이>=20)(학생)
```

#### 2. 투영 (Projection) - π

sql

```sql
-- 이름만 조회
SELECT 이름 FROM 학생;

-- 집합 표기법  
π(이름)(학생)
```

#### 3. 합집합 (Union) - ∪

sql

```sql
-- 두 테이블의 합집합
SELECT * FROM 학생1 
UNION 
SELECT * FROM 학생2;
```

#### 4. 교집합 (Intersection) - ∩

sql

```sql
-- 두 테이블의 교집합
SELECT * FROM 학생1
INTERSECT
SELECT * FROM 학생2;
```

#### 5. 차집합 (Difference) - -

sql

```sql
-- 학생1에는 있지만 학생2에는 없는 레코드
SELECT * FROM 학생1
EXCEPT  
SELECT * FROM 학생2;
```

#### 6. 카티션 곱 (Cartesian Product) - ×

sql

```sql
-- 모든 조합 생성
SELECT * FROM 학생, 과목;
```

#### 7. 조인 (Join) - ⋈

sql

```sql
-- 조건에 맞는 레코드들을 결합
SELECT * FROM 학생 s
JOIN 수강 c ON s.학번 = c.학번;
```

## ⚡ 논리 게이트의 이론적 기초

### 집합론 → 불 대수 → 논리 게이트

**추후 논리게이트 설명의 밑작업 같다**

#### 1. 기본 게이트와 집합 연산의 대응

```
AND 게이트:
입력 A, B → 출력 C = A ∩ B
A=1, B=1 → C=1 (교집합이 존재)
A=1, B=0 → C=0 (교집합 없음)

OR 게이트:  
입력 A, B → 출력 C = A ∪ B
A=1, B=0 → C=1 (합집합에 포함)
A=0, B=0 → C=0 (합집합 공집합)

NOT 게이트:
입력 A → 출력 C = A'
A=1 → C=0 (여집합)
A=0 → C=1 (여집합)
```

#### 2. 복합 게이트의 집합적 해석

```
NAND 게이트 = NOT(AND)
C = (A ∩ B)' = A' ∪ B'  (드모르간 법칙)

NOR 게이트 = NOT(OR)  
C = (A ∪ B)' = A' ∩ B'  (드모르간 법칙)

XOR 게이트 = 배타적 OR
C = (A ∪ B) - (A ∩ B)  (대칭차집합)
```

## 💻 현대 시스템에서의 응용

### 프로그래밍에서의 집합 연산

#### Python 예시

python

```python
# 집합 정의
set_a = {1, 2, 3, 4, 5}
set_b = {3, 4, 5, 6, 7}

# 합집합 (OR)
union = set_a | set_b        # {1, 2, 3, 4, 5, 6, 7}

# 교집합 (AND)  
intersection = set_a & set_b  # {3, 4, 5}

# 차집합 (NOT)
difference = set_a - set_b    # {1, 2}

# 대칭차집합 (XOR)
symmetric_diff = set_a ^ set_b # {1, 2, 6, 7}
```

#### SQL에서의 집합 연산

sql

```sql
-- 집합 기반 쿼리 최적화
SELECT DISTINCT 컬럼 FROM 테이블  -- 중복 제거 (집합의 고유성)

-- 존재 확인 (부분집합 관계)
SELECT * FROM A WHERE EXISTS (SELECT 1 FROM B WHERE A.id = B.id)

-- 포함 관계 확인
SELECT * FROM A WHERE id IN (SELECT id FROM B)
```

### 검색 엔진과 집합 연산

```
검색어: "프로그래밍 AND 파이썬"
결과 = {프로그래밍 문서들} ∩ {파이썬 문서들}

검색어: "자바 OR 파이썬"  
결과 = {자바 문서들} ∪ {파이썬 문서들}

검색어: "프로그래밍 NOT 자바"
결과 = {프로그래밍 문서들} - {자바 문서들}
```

## 🔄 실용적 응용 사례

### 1. 권한 관리 시스템

python

```python
# 사용자 권한을 집합으로 관리
admin_permissions = {"read", "write", "delete", "manage"}
user_permissions = {"read", "write"}

# 권한 확인 (부분집합 관계)
can_delete = "delete" in user_permissions  # False

# 권한 추가 (합집합)
new_permissions = user_permissions | {"create"}

# 권한 제거 (차집합)  
limited_permissions = admin_permissions - {"delete"}
```

### 2. 캐시 시스템

python

```python
# 캐시된 데이터와 요청된 데이터의 교집합
cached_keys = {"user:1", "user:2", "post:1"}  
requested_keys = {"user:1", "user:3", "post:1"}

# 캐시 히트 (교집합)
cache_hits = cached_keys & requested_keys  # {"user:1", "post:1"}

# 캐시 미스 (차집합)
cache_misses = requested_keys - cached_keys  # {"user:3"}
```

### 3. 추천 시스템

python

```python
# 사용자 선호도를 집합으로 표현
user_a_likes = {"액션", "SF", "스릴러"}
user_b_likes = {"SF", "로맨스", "드라마"}

# 공통 관심사 (교집합)
common_interests = user_a_likes & user_b_likes  # {"SF"}

# 추천 후보 (차집합)
recommendations = user_b_likes - user_a_likes  # {"로맨스", "드라마"}
```
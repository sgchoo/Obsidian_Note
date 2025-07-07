[[part4. 기초대수학]]

# 집합의 연산 (Set Operations)

## 기본 집합 연산

### 1. 여집합 (Complement)

- **정의**: A에 포함되지 않은 원소들을 모은 집합
- **표기**: A<sup>c</sup> 또는 A'
- **설명**: 전체집합 U에서 집합 A를 제외한 모든 원소들의 집합

### 2. 교집합 (Intersection)

- **정의**: 집합 A, B에 모두 포함되는 원소들의 집합
- **표기**: A ∩ B
- **설명**: 두 집합에 공통으로 속하는 원소들만을 포함하는 집합

### 3. 합집합 (Union)

- **정의**: 집합 A 또는 B에 포함되는 원소들을 모은 집합
- **표기**: A ∪ B
- **설명**: 두 집합 중 적어도 하나에 속하는 모든 원소들의 집합

## 집합 연산의 대수학적 특징

### 교환법칙 (Commutative Law)

- A ∪ B = B ∪ A
- A ∩ B = B ∩ A

### 결합법칙 (Associative Law)

- (A ∪ B) ∪ C = A ∪ (B ∪ C)
- (A ∩ B) ∩ C = A ∩ (B ∩ C)

### 분배법칙 (Distributive Law)

- A ∪ (B ∩ C) = (A ∪ B) ∩ (A ∪ C)
- A ∩ (B ∪ C) = (A ∩ B) ∪ (A ∩ C)

### 드모르간의 법칙 (De Morgan's Law)

- (A ∪ B)<sup>c</sup> = A<sup>c</sup> ∩ B<sup>c</sup>
- (A ∩ B)<sup>c</sup> = A<sup>c</sup> ∪ B<sup>c</sup>

## 기타 중요한 성질

### 항등원소 (Identity Elements)

- A ∪ ∅ = A (공집합은 합집합의 항등원소)
- A ∩ U = A (전체집합은 교집합의 항등원소)

### 지배원소 (Dominating Elements)

- A ∪ U = U
- A ∩ ∅ = ∅

### 멱등법칙 (Idempotent Law)

- A ∪ A = A
- A ∩ A = A

### 여집합의 성질

- (A<sup>c</sup>)<sup>c</sup> = A
- A ∪ A<sup>c</sup> = U
- A ∩ A<sup>c</sup> = ∅
[[part.1 기초 프로그래밍]]

# 파이썬 비교 연산자와 조건문

## 비교 연산자

파이썬에서는 다양한 비교 연산자를 사용하여 값을 비교할 수 있습니다:

|연산자|의미|
|---|---|
|`==`|같음|
|`!=`|같지 않음|
|`>`|크다|
|`<`|작다|
|`>=`|크거나 같다|
|`<=`|작거나 같다|

## 조건문 구조

### 1. 단순 if 조건문

```python
temp = int(input("CPU Temperature: "))

if temp >= 30:
    print("CPU Fan is Operating")
```

### 2. if-else 조건문

두 가지 경우로 나누어 처리할 때 사용합니다.

```python
grade = float(input("수능 평균 등급: "))

if grade <= 2:
    print("수능 최저 기준을 만족합니다")
    print("합격입니다.")
else:
    print("수능 최저 기준을 만족하지 않습니다.")
    print("불합격입니다.")
```

### 3. if-elif-else 조건문

세 가지 이상의 조건으로 나누어 처리할 때 사용합니다.

```python
score = int(input("시험 점수를 입력하세요: "))

if score >= 90:
    print("학점: A")
elif score >= 80:
    print("학점: B")
elif score >= 70:
    print("학점: C")
elif score >= 60:
    print("학점: D")
else:
    print("학점: F")
```

## 논리 연산자

조건을 결합하거나 확장하기 위해 논리 연산자를 사용할 수 있습니다:

|연산자|의미|회로 비유|
|---|---|---|
|`and`|두 조건이 모두 참이어야 참|직렬 연결|
|`or`|적어도 하나의 조건이 참이면 참|병렬 연결|
|`not`|조건의 결과를 반대로 뒤집음|반전|

### 예시

```python
temp = int(input("CPU Temperature: "))
fan_status = input("Fan Status (on/off): ")

# and 연산자 - 두 조건이 모두 참이어야 실행됨 (직렬 연결)
if temp >= 30 and fan_status == "off":
    print("경고: 온도가 높은데 팬이 꺼져 있습니다!")

# or 연산자 - 적어도 하나의 조건이 참이면 실행됨 (병렬 연결)
if temp >= 40 or fan_status == "off":
    print("시스템 점검이 필요합니다.")
```
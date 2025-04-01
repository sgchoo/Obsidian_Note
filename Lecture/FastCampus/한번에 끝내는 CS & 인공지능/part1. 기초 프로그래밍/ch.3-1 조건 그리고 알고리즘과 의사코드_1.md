[[part.1 기초 프로그래밍]]

# 조건문과 순서도

## 조건이란?

조건문은 프로그램의 실행 흐름을 제어하는 구문입니다. 특정 조건(논리 연산)이 참(True)인지 거짓(False)인지에 따라 다른 코드 블록을 실행합니다.

## 순서도 기본 기호

순서도(Flowchart)는 알고리즘이나 프로세스의 흐름을 시각적으로 표현하는 다이어그램입니다.

![순서도][https://mblogthumb-phinf.pstatic.net/MjAyMDExMjNfMTEz/MDAxNjA2MTAwMzAwOTM0.HvMxWrpKKh2FnrwJ6p6Whj9QHuAfcjtqAkmejtgkCS8g.f7ecuGQZ7Gsv6pJHfwmmUBlnx8OhSlZJiNiKDJpjCiIg.PNG.pk3152/Screenshot_11.png?type=w800]

## 의사코드(Pseudo Code)

의사코드는 알고리즘을 자연어로 표현한 것으로, 실제 프로그래밍 언어의 문법에 얽매이지 않고 논리적 흐름을 설명합니다.

### 예시: 성인 여부 확인 프로그램

**의사코드:**

```
시작
    나이를 입력받는다
    IF 나이가 18 이상이면
        "성인입니다"를 출력한다
    ELSE
        "미성년자입니다"를 출력한다
종료
```

**실제 Python 코드:**

```python
# 시작
# 나이를 입력받는다
age = int(input("나이를 입력하세요: "))

# 만약 나이가 18 이상이면
if age >= 18:
    # "성인입니다"를 출력한다
    print("성인입니다")
# 그렇지 않으면
else:
    # "미성년자입니다"를 출력한다
    print("미성년자입니다")
# 종료
```

### 예시: 학점 계산 프로그램

**의사코드:**

```
시작
    점수를 입력받는다
    IF 점수가 90점 이상이면
        학점 A를 부여한다
    ELSE IF 점수가 80점 이상이면
        학점 B를 부여한다
    ELSE IF 점수가 70점 이상이면
        학점 C를 부여한다
    ELSE IF 점수가 60점 이상이면
        학점 D를 부여한다
    그렇지 않으면
        학점 F를 부여한다
    학점을 출력한다
종료
```

**실제 Python 코드:**

```python
# 시작
# 점수를 입력받는다
score = int(input("점수를 입력하세요: "))

# 조건에 따라 학점 부여
if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
elif score >= 70:
    grade = "C"
elif score >= 60:
    grade = "D"
else:
    grade = "F"

# 학점을 출력한다
print(f"당신의 학점은 {grade}입니다.")
# 종료
```

이처럼 의사코드는 프로그램의 논리 구조를 명확하게 표현하면서도, 특정 프로그래밍 언어의 문법에 구애받지 않는 장점이 있습니다.
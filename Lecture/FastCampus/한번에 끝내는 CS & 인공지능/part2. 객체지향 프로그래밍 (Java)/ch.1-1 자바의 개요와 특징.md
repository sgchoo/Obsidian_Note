[[part2. 객체지향 프로그래밍 (Java)]]

# JVM과 JDK

## JVM (Java Virtual Machine)

JVM은 자바 가상 머신(Java Virtual Machine)의 약자로, 자바 바이트코드를 실행하는 가상 머신입니다.

### JVM의 주요 특징

1. **플랫폼 독립성**
    
    - "Write Once, Run Anywhere" 철학 구현
    - 자바 프로그램은 운영체제에 상관없이 JVM이 설치된 모든 환경에서 동일하게 실행 가능
2. **바이트코드 실행**
    
    - 자바 소스코드(.java)는 컴파일러에 의해 바이트코드(.class)로 변환
    - JVM은 이 바이트코드를 해당 운영체제에 맞게 해석하여 실행
3. **메모리 관리**
    
    - 가비지 컬렉션(Garbage Collection)을 통한 자동 메모리 관리
    - 개발자가 명시적으로 메모리 할당/해제를 처리할 필요가 없음
4. **JVM 구조**
    
    - 클래스 로더(Class Loader): 클래스 파일을 로드
    - 실행 엔진(Execution Engine): 바이트코드를 실행
    - 런타임 데이터 영역(Runtime Data Area): 메모리 영역
        - 메서드 영역(Method Area)
        - 힙 영역(Heap Area)
        - 스택 영역(Stack Area)
        - PC 레지스터(PC Register)
        - 네이티브 메서드 스택(Native Method Stack)

## JDK (Java Development Kit)

JDK는 자바 개발 키트(Java Development Kit)의 약자로, 자바 애플리케이션을 개발하는 데 필요한 도구의 모음입니다.

### JDK의 주요 구성 요소

1. **JRE (Java Runtime Environment)**
    
    - JVM + 자바 클래스 라이브러리
    - 자바 프로그램을 실행하는 데 필요한 최소 환경
2. **개발 도구**
    
    - javac: 자바 컴파일러 (소스코드 → 바이트코드)
    - java: 자바 인터프리터 (바이트코드 실행)
    - javadoc: 문서 생성기
    - jar: 자바 아카이브 도구
    - javap: 클래스 파일 디스어셈블러
    - 디버거 등 다양한 개발 유틸리티
3. **API (Application Programming Interface)**
    
    - 자바 프로그래밍에 사용되는 클래스 라이브러리
    - java.lang, java.util, java.io 등 다양한 패키지 포함

### JDK vs JRE vs JVM

![JDK-JRE-JVM 관계](https://example.com/jdk-jre-jvm.png)

- **JDK**: 개발 + 실행 환경 (개발자용)
- **JRE**: 실행 환경 (일반 사용자용)
- **JVM**: 자바 프로그램을 실행하는 가상 머신 (JRE의 일부)

## 자바 버전별 JDK

|버전|출시년도|주요 특징|
|---|---|---|
|Java 8 (LTS)|2014|람다 표현식, 스트림 API, 새로운 날짜/시간 API|
|Java 11 (LTS)|2018|HTTP 클라이언트 표준화, 람다 파라미터 지역변수 문법|
|Java 17 (LTS)|2021|봉인 클래스, 패턴 매칭, 레코드 클래스|
|Java 21 (LTS)|2023|가상 스레드, 패턴 매칭 for-switch, 레코드 패턴|

## JVM 언어

JVM 위에서 실행되는 자바 외 프로그래밍 언어들:

- Kotlin
- Scala
- Groovy
- Clojure
- JRuby
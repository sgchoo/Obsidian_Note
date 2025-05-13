[[part2. 객체지향 프로그래밍 (Java)]]

# 상속의 개념과 추상화

## 객체지향 프로그래밍에서의 상속

상속(Inheritance)은 객체지향 프로그래밍(OOP)의 핵심 개념 중 하나로, 기존 클래스의 속성과 메소드를 새로운 클래스가 물려받아 재사용하고 확장할 수 있게 하는 메커니즘입니다. 이를 통해 코드 재사용성을 높이고 계층적인 관계를 구성할 수 있습니다.

### 상속의 기본 개념

- **상속의 정의**: 객체지향에서 상속은 기존 클래스(부모 클래스/슈퍼 클래스)를 기반으로 새로운 클래스(자식 클래스/서브 클래스)를 만드는 과정입니다.
    
- **상속의 목적**:
    
    - 코드 재사용성 향상
    - 유지보수 용이성 증대
    - 프로그램의 구조화 및 계층화
    - 다형성 구현의 기반
- **상속 관계**:
    
    - **is-a 관계**: 자식 클래스는 부모 클래스의 특수한 형태입니다. (예: "고양이는 동물이다")
    - 자식 클래스 객체는 부모 클래스 타입으로 참조될 수 있습니다.

## Java에서의 상속 구현: extends 키워드

Java에서는 `extends` 키워드를 사용하여 상속을 구현합니다.

```java
// 부모 클래스
public class Animal {
    protected String name;
    
    public Animal(String name) {
        this.name = name;
    }
    
    public void eat() {
        System.out.println(name + " is eating.");
    }
    
    public void sleep() {
        System.out.println(name + " is sleeping.");
    }
}

// 자식 클래스
public class Cat extends Animal {
    private String breed;
    
    public Cat(String name, String breed) {
        super(name); // 부모 클래스의 생성자 호출
        this.breed = breed;
    }
    
    // 메소드 오버라이딩: 부모 클래스의 메소드를 재정의
    @Override
    public void eat() {
        System.out.println(name + " the " + breed + " cat is eating fish.");
    }
    
    // 자식 클래스만의 새로운 메소드
    public void meow() {
        System.out.println(name + " says: Meow!");
    }
}
```

### 상속의 특징

1. **단일 상속**: Java는 클래스의 다중 상속을 지원하지 않습니다. 한 클래스는 하나의 클래스만 직접 상속받을 수 있습니다.
    
2. **메소드 오버라이딩(Method Overriding)**: 자식 클래스에서 부모 클래스의 메소드를 재정의할 수 있습니다.
    
3. **super 키워드**: 자식 클래스에서 부모 클래스의 멤버에 접근할 때 사용합니다.
    
4. **생성자 상속**: 생성자는 상속되지 않습니다. 자식 클래스는 자신의 생성자를 정의해야 합니다.
    
5. **멤버 접근 제어**: 상속받은 멤버의 접근 권한은 유지되지만, private 멤버는 자식 클래스에서 직접 접근할 수 없습니다.
    

## 추상 메소드와 추상 클래스

### 추상 메소드 (Abstract Method)

추상 메소드는 선언만 있고 구현부(body)가 없는 메소드입니다. 메소드의 시그니처(이름, 매개변수, 반환 타입)만 정의하고, 구체적인 구현은 이를 상속받는 서브 클래스에서 해야 합니다.

```java
public abstract void calculateArea(); // 추상 메소드 선언
```

### 추상 클래스 (Abstract Class)

추상 클래스는 하나 이상의 추상 메소드를 포함하는 클래스이며, `abstract` 키워드로 선언됩니다. 추상 클래스의 주요 특징:

- 추상 클래스는 인스턴스화할 수 없습니다. (객체 생성 불가)
- 추상 클래스를 상속받는 클래스는 모든 추상 메소드를 구현해야 합니다. (아니면 해당 클래스도 추상 클래스가 되어야 함)
- 추상 클래스는 일반 메소드와 추상 메소드를 모두 가질 수 있습니다.
- 클래스의 공통 기능을 정의하는 데 사용됩니다.

```java
public abstract class Shape {
    protected String color;
    
    // 생성자
    public Shape(String color) {
        this.color = color;
    }
    
    // 일반 메소드
    public String getColor() {
        return color;
    }
    
    // 추상 메소드
    public abstract double calculateArea();
    public abstract double calculatePerimeter();
}

// 구체적인 자식 클래스
public class Circle extends Shape {
    private double radius;
    
    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }
    
    // 추상 메소드 구현
    @Override
    public double calculateArea() {
        return Math.PI * radius * radius;
    }
    
    @Override
    public double calculatePerimeter() {
        return 2 * Math.PI * radius;
    }
}
```

## 추상 클래스 vs 인터페이스

상속과 관련하여 Java에서는 추상 클래스와 인터페이스라는 두 가지 추상화 메커니즘을 제공합니다. 둘의 차이점:

|특성|추상 클래스|인터페이스|
|---|---|---|
|키워드|`abstract class`|`interface`|
|상속|단일 상속|다중 구현 가능|
|멤버 변수|모든 타입 가능|상수만 가능 (public static final)|
|메소드 구현|일반 메소드와 추상 메소드 혼합 가능|Java 8부터 default, static 메소드 구현 가능|
|접근 제어자|모든 접근 제어자 사용 가능|모든 멤버는 public|
|용도|관련성이 높은 클래스 간의 코드 공유|서로 관련성이 없는 클래스에 공통 동작 강제|

## 상속의 장단점

### 장점

1. **코드 재사용**: 기존 클래스의 기능을 재사용하여 개발 시간 단축
2. **확장성**: 기존 코드를 수정하지 않고 기능 확장 가능
3. **다형성**: 같은 타입이지만 다양한 구현을 가능하게 함
4. **일관된 인터페이스**: 표준화된 인터페이스를 통한 통합성 보장

### 단점

1. **강한 결합**: 부모 클래스와 자식 클래스 간의 강한 결합 발생
2. **취약한 기반 클래스 문제**: 부모 클래스 변경 시 자식 클래스에 영향
3. **깊은 상속 계층**: 계층이 깊어질수록 이해와 디버깅 난이도 증가
4. **불필요한 기능 상속**: 필요하지 않은 기능까지 상속될 수 있음

## 상속의 대안과 보완

상속의 단점을 보완하기 위해 다음과 같은 대안이나 보완 기법을 사용할 수 있습니다:

1. **합성(Composition)**: "has-a" 관계를 구현하여 객체를 클래스의 멤버로 포함
2. **위임(Delegation)**: 특정 작업을 다른 클래스의 객체에게 위임
3. **인터페이스 활용**: 다중 구현을 통한 유연성 확보
4. **전략 패턴(Strategy Pattern)**: 알고리즘을 캡슐화하고 교체 가능하게 만들기

## 결론

상속은 객체지향 프로그래밍의 강력한 도구이지만, 적절하게 사용해야 합니다. "상속보다 합성을 선호하라"는 객체지향 설계의 원칙을 기억하고, 상속은 명확한 "is-a" 관계가 성립할 때 사용하는 것이 좋습니다. 추상 클래스와 인터페이스를 적절히 활용하여 코드의 유연성, 재사용성, 유지보수성을 높이는 것이 중요합니다.
[[Nest.js]]

# Java 핵심 개념: 오버라이드, 애노테이션, 상속

## 오버라이드(Override)

오버라이드는 객체지향 프로그래밍의 중요한 특징으로, 상위 클래스에서 이미 정의된 메서드를 하위 클래스에서 재정의하는 것을 말합니다.

### 핵심 특징

- **메서드 시그니처 유지**: 오버라이드할 때는 메서드 이름, 매개변수 타입, 개수, 반환 타입이 동일해야 합니다.
- **접근 제어자**: 상위 클래스 메서드보다 더 제한적이지 않아야 합니다(예: private → public 가능, public → private 불가능).
- **예외 처리**: 상위 클래스 메서드보다 더 많은 체크 예외를 선언할 수 없습니다.

### 예시 코드

```java
class Animal {
    public void makeSound() {
        System.out.println("동물이 소리를 냅니다.");
    }
}

class Dog extends Animal {
    @Override  // 오버라이드 애노테이션
    public void makeSound() {
        System.out.println("멍멍!");  // 기능 변경
    }
}

class Cat extends Animal {
    @Override
    public void makeSound() {
        System.out.println("야옹!");  // 기능 변경
    }
}
```

### 활용 케이스

- **다형성 구현**: 같은 메서드 호출로 객체별 다른 동작 수행
- **기존 기능 확장**: 기본 동작을 유지하면서 새로운 기능 추가
- **특화된 동작 구현**: 일반적인 동작을 특정 클래스에 맞게 커스터마이즈

```java
// 다형성 예시
Animal myPet = new Dog();
myPet.makeSound();  // 출력: "멍멍!"

myPet = new Cat();
myPet.makeSound();  // 출력: "야옹!"
```

## 애노테이션(Annotation)

애노테이션은 코드에 메타데이터를 추가하는 방법으로, 컴파일러나 런타임에 특별한 처리를 지시합니다.

### 주요 특징

- **코드 메타데이터**: 소스 코드에 추가 정보를 제공
- **컴파일/런타임 처리**: 컴파일러나 프레임워크에게 특별한 처리 지시
- **문서화**: API 문서에 정보 제공

### 주요 내장 애노테이션

- **@Override**: 메서드가 상위 클래스 메서드를 오버라이드함을 명시
- **@Deprecated**: 해당 요소가 더 이상 권장되지 않음을 표시
- **@SuppressWarnings**: 특정 경고 메시지 억제
- **@FunctionalInterface**: 함수형 인터페이스임을 명시(하나의 추상 메서드만 포함)

### 커스텀 애노테이션 만들기

```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)  // 런타임에도 유지
@Target(ElementType.METHOD)          // 메서드에만 적용 가능
public @interface CustomAnnotation {
    String value() default "기본값";  // 파라미터 정의
    int count() default 1;
}

class Example {
    @CustomAnnotation(value = "테스트", count = 5)
    public void testMethod() {
        // 메서드 구현
    }
}
```

### 애노테이션 처리기

```java
// 런타임에 애노테이션 정보 읽기
Method method = Example.class.getMethod("testMethod");
CustomAnnotation annotation = method.getAnnotation(CustomAnnotation.class);
System.out.println(annotation.value());  // 출력: "테스트"
System.out.println(annotation.count());  // 출력: 5
```

## 상속(Inheritance)

상속은 기존 클래스의 특성(속성과 메서드)을 새로운 클래스가 물려받는 것을 의미합니다.

### 핵심 특징

- **코드 재사용**: 기존 코드를 재사용하여 개발 효율성 증가
- **계층 구조**: 객체 간 계층적 관계 형성
- **확장성**: 기존 클래스의 기능을 확장하거나 특화 가능

### 구현 방법

Java에서는 `extends` 키워드를 사용해 단일 상속을 구현합니다.

```java
// 상위 클래스
class Vehicle {
    private String brand;
    
    public Vehicle(String brand) {
        this.brand = brand;
    }
    
    public void start() {
        System.out.println("차량이 시동을 겁니다.");
    }
    
    public String getBrand() {
        return brand;
    }
}

// 하위 클래스
class Car extends Vehicle {
    private int doors;
    
    public Car(String brand, int doors) {
        super(brand);  // 상위 클래스 생성자 호출
        this.doors = doors;
    }
    
    // 새로운 메서드 추가
    public void honk() {
        System.out.println("빵빵!");
    }
    
    // 메서드 오버라이드
    @Override
    public void start() {
        super.start();  // 상위 클래스 메서드 호출
        System.out.println(getBrand() + " 자동차의 엔진이 부드럽게 작동합니다.");
    }
}
```

### super 키워드

`super` 키워드는 하위 클래스에서 상위 클래스의 멤버(메서드, 변수)에 접근할 때 사용합니다.

#### 주요 용도

1. **상위 클래스 생성자 호출**:
    
    ```java
    super();  // 기본 생성자
    super(param1, param2);  // 매개변수 있는 생성자
    ```
    
2. **상위 클래스 메서드 호출**:
    
    ```java
    super.methodName();  // 상위 클래스 메서드 호출
    ```
    
3. **상위 클래스 변수 접근**:
    
    ```java
    super.variableName;  // 상위 클래스 변수 접근
    ```
    

### 상속의 제한사항

- **final 클래스**: `final` 키워드가 붙은 클래스는 상속할 수 없음
- **private 멤버**: 상위 클래스의 `private` 멤버는 직접 접근 불가능
- **단일 상속**: Java는 클래스의 다중 상속을 지원하지 않음(인터페이스는 다중 구현 가능)

## 실제 활용 예시

아래 예시는 오버라이드, 애노테이션, 상속을 함께 활용한 코드입니다:

```java
// 기본 도형 클래스
abstract class Shape {
    private String color;
    
    public Shape(String color) {
        this.color = color;
    }
    
    public String getColor() {
        return color;
    }
    
    // 추상 메서드 - 하위 클래스에서 반드시 구현해야 함
    public abstract double calculateArea();
    
    // 일반 메서드 - 오버라이드 가능
    public void display() {
        System.out.println("도형을 그립니다. 색상: " + color);
    }
}

// 원 클래스
class Circle extends Shape {
    private double radius;
    
    public Circle(String color, double radius) {
        super(color);  // 상위 클래스 생성자 호출
        this.radius = radius;
    }
    
    @Override  // 오버라이드 애노테이션
    public double calculateArea() {
        return Math.PI * radius * radius;
    }
    
    @Override
    public void display() {
        super.display();  // 상위 클래스 메서드 호출
        System.out.println("원을 그립니다. 반지름: " + radius);
    }
    
    // 새로운 메서드 추가
    @Deprecated  // 더 이상 사용을 권장하지 않는 메서드
    public double getPerimeter() {
        return 2 * Math.PI * radius;
    }
}

// 사각형 클래스
class Rectangle extends Shape {
    private double width;
    private double height;
    
    public Rectangle(String color, double width, double height) {
        super(color);
        this.width = width;
        this.height = height;
    }
    
    @Override
    public double calculateArea() {
        return width * height;
    }
    
    @Override
    public void display() {
        super.display();
        System.out.println("사각형을 그립니다. 가로: " + width + ", 세로: " + height);
    }
}

// 메인 클래스
class Main {
    public static void main(String[] args) {
        // 다형성 활용
        Shape circle = new Circle("빨강", 5.0);
        Shape rectangle = new Rectangle("파랑", 4.0, 6.0);
        
        // 동일한 메서드 호출이지만 다른 동작 수행(다형성)
        System.out.println("원 면적: " + circle.calculateArea());
        System.out.println("사각형 면적: " + rectangle.calculateArea());
        
        circle.display();  // 오버라이드된 메서드 호출
        rectangle.display();  // 오버라이드된 메서드 호출
    }
}
```

이 예시는 각 개념이 어떻게 함께 작동하는지 보여줍니다:

- **상속**: Shape 클래스를 Circle과 Rectangle이 상속
- **오버라이드**: calculateArea()와 display() 메서드 재정의
- **애노테이션**: @Override, @Deprecated 사용
- **super**: 상위 클래스 생성자와 메서드 호출

## 결론

오버라이드, 애노테이션, 상속은 객체지향 프로그래밍의 핵심 개념으로, 코드의 재사용성과 확장성을 높이는 데 중요한 역할을 합니다. 이러한 개념들을 적절히 활용하면 유지보수가 용이하고 확장 가능한 코드를 작성할 수 있습니다.
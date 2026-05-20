[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

## 가상 스레드로 자원 효율 높이기

일반적인 Spring MVC + Tomcat 구조에서는 사용자의 요청이 들어올 때마다 요청을 처리할 스레드가 하나 할당된다.

```text
사용자 요청
  ↓
Tomcat Thread 할당
  ↓
Controller
  ↓
Service
  ↓
Repository / DB / 외부 API
  ↓
응답 반환
````

이 방식은 이해하기 쉽고 코드도 단순하지만, 문제가 있다.  
DB 조회나 외부 API 호출처럼 오래 걸리는 IO 작업이 발생하면, 해당 스레드는 응답이 올 때까지 대기한다.

```text
Thread-1 → DB 응답 대기
Thread-2 → 외부 API 응답 대기
Thread-3 → Redis 응답 대기
```

이때 스레드는 CPU를 적극적으로 사용하고 있지 않지만, 다른 요청을 처리하지 못한 채 점유된다.  
동시 요청이 많아지면 스레드 풀이 고갈되고, 이후 요청은 사용 가능한 스레드가 생길 때까지 기다려야 한다.

---

### 기존 플랫폼 스레드의 한계

기존 Java의 플랫폼 스레드는 OS 스레드와 강하게 연결되어 있다.  
따라서 요청 하나마다 플랫폼 스레드를 오래 점유하면 메모리 사용량과 스케줄링 비용이 커진다.

특히 웹 서버의 작업은 대부분 CPU 연산보다 IO 대기가 많다.

```text
실제 CPU 작업: 5ms
DB / 외부 API 대기: 95ms
총 응답 시간: 100ms
```

이 경우 95ms 동안 스레드는 실질적으로 쉬고 있지만, 다른 요청을 처리할 수는 없다.  
이것이 전통적인 thread-per-request 모델의 주요 한계다.

---

### 가상 스레드의 핵심 아이디어

Java의 Virtual Thread는 기존 플랫폼 스레드보다 훨씬 가벼운 스레드다.

가상 스레드는 요청당 스레드 모델을 유지하면서도, IO 대기 중에는 실제 OS 스레드인 carrier thread를 붙잡고 있지 않을 수 있다.

```text
요청 도착
  ↓
Virtual Thread에서 Controller 실행
  ↓
DB 요청
  ↓
Virtual Thread 일시 중단
  ↓
Carrier Thread 반납
  ↓
DB 응답 도착
  ↓
Virtual Thread 재개
  ↓
응답 반환
```

즉, 개발자는 기존처럼 동기 코드로 작성할 수 있다.

```java
User user = userRepository.findById(id);
return UserResponse.from(user);
```

하지만 내부적으로는 IO 대기 중에 비싼 OS 스레드가 묶이지 않도록 동작한다.

---

### 가상 스레드가 개선하는 것

가상 스레드는 개별 DB 쿼리나 외부 API 호출 자체를 빠르게 만들어주지는 않는다.  
DB 쿼리가 500ms 걸린다면, 그 요청은 여전히 비슷한 시간이 걸린다.

대신 개선되는 부분은 다음과 같다.

|항목|설명|
|---|---|
|스레드 점유 감소|IO 대기 중 플랫폼 스레드가 오래 묶이지 않음|
|메모리 효율 개선|많은 요청을 더 가벼운 스레드로 처리 가능|
|동시 처리량 증가|같은 자원으로 더 많은 동시 요청 처리 가능|
|코드 단순성 유지|기존 동기 코드 스타일을 유지할 수 있음|
|기존 Spring MVC 구조와 궁합|Controller-Service-Repository 구조를 크게 바꾸지 않아도 됨|

---

### Spring Boot에서의 사용 예시

Spring Boot에서는 Java 21 이상 환경에서 다음 설정으로 가상 스레드 사용을 활성화할 수 있다.

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

이 설정을 켠다고 동기 코드가 논블로킹 코드로 바뀌는 것은 아니다.  
기존 코드는 여전히 동기 코드처럼 실행된다.

```java
User user = userRepository.findById(id);
return UserResponse.from(user);
```

다만 이 요청 처리 흐름이 기존 플랫폼 스레드가 아니라 가상 스레드 위에서 실행될 수 있다.

---

### 논블로킹 IO와의 차이

가상 스레드와 논블로킹 IO는 모두 IO 대기 중 스레드 낭비를 줄이기 위한 방법이다.  
하지만 접근 방식이 다르다.

#### 가상 스레드

```java
User user = userRepository.findById(id);
return UserResponse.from(user);
```

- 동기 코드 스타일을 유지한다.
    
- 요청마다 가상 스레드를 사용한다.
    
- IO 대기 중에는 carrier thread를 반납할 수 있다.
    
- 기존 Spring MVC, JDBC, JPA 구조와 잘 어울린다.
    

#### 논블로킹 IO

```java
return userRepository.findById(id)
    .map(UserResponse::from);
```

- 코드 자체를 비동기 흐름으로 작성한다.
    
- 응답을 기다리지 않고, 응답이 오면 실행할 작업을 등록한다.
    
- 이벤트 루프나 selector 기반으로 동작한다.
    
- WebFlux, R2DBC, Reactive Redis, WebClient 같은 논블로킹 생태계와 함께 사용할 때 효과가 크다.
    

---

### Kotlin Coroutine, Go Goroutine과의 관계

Java Virtual Thread, Kotlin Coroutine, Go Goroutine은 모두 비슷한 문제를 해결하려는 기술이다.

> IO 대기 때문에 비싼 OS 스레드가 낭비되는 문제를 줄이고, 더 많은 동시 작업을 효율적으로 처리하는 것

다만 구현 방식은 다르다.

|구분|Java Virtual Thread|Go Goroutine|Kotlin Coroutine|
|---|---|---|---|
|실행 단위|JVM의 가벼운 스레드|Go 런타임의 가벼운 실행 단위|suspend 가능한 비동기 작업|
|코드 스타일|동기 코드 유지|동기 코드 유지|동기처럼 보이는 비동기 코드|
|스케줄링 주체|JVM|Go Runtime|Coroutine Dispatcher|
|IO 대기 처리|carrier thread 반납 가능|goroutine을 park하고 OS thread 재활용|suspend 지점에서 중단 후 재개|
|특징|기존 Java 코드와 궁합이 좋음|언어 기본 동시성 모델|async/await와 유사한 감각|

Kotlin의 `suspend`는 “실행 중간에 멈췄다가 나중에 다시 이어질 수 있다”는 의미다.

```kotlin
suspend fun getUser(id: Long): UserResponse {
    val user = userRepository.findById(id)
    return UserResponse.from(user)
}
```

코루틴은 suspend 지점에서 실행 상태를 저장하고, 스레드를 반납한 뒤, 나중에 응답이 오면 저장된 지점부터 다시 실행된다.

다만 JPA/JDBC처럼 블로킹 기반 라이브러리를 코루틴 안에서 사용한다고 해서 자동으로 논블로킹이 되는 것은 아니다.  
이 경우에는 보통 `Dispatchers.IO` 같은 별도 IO 스레드 풀로 분리해서 처리한다.

---

### 정리

가상 스레드는 기존의 요청당 스레드 모델을 버리는 기술이 아니다.  
오히려 요청당 스레드 모델을 유지하면서, 각 스레드의 비용을 크게 낮추는 방식이다.

기존 방식에서는 IO 대기 중에도 플랫폼 스레드가 계속 점유된다.  
반면 가상 스레드는 IO 대기 중에 carrier thread를 반납하고, 응답이 도착하면 같은 가상 스레드의 실행 흐름을 이어간다.

따라서 일반적인 Spring MVC + JPA 기반 API 서버에서는 논블로킹 IO로 전체 구조를 바꾸기 전에, 가상 스레드를 먼저 고려할 수 있다.

> 가상 스레드는 동기 코드의 단순함을 유지하면서, IO 대기 중심 서버의 자원 효율과 동시 처리량을 높이기 위한 방법이다.
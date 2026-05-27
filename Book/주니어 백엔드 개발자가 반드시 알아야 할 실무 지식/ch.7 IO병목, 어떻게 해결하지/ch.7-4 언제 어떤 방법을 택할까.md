[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 논블로킹 I/O와 가상 스레드 적용 검토 기준

## 1. 왜 먼저 검토가 필요한가?

논블로킹 I/O나 가상 스레드는 성능 문제를 해결하기 위한 수단이다.

하지만 성능 문제가 없는 상황에서 무리하게 적용하면 오히려 유지보수 난이도만 올라갈 수 있다.

따라서 적용하기 전에 다음을 먼저 검토해야 한다.

```text
1. 실제로 문제가 있는가?
2. 문제가 있다면 I/O 관련 성능 문제인가?
3. 구현 변경이 가능한가?
````

---

## 2. 성능 문제가 있는가?

먼저 현재 시스템에 실제 성능 문제가 있는지 확인해야 한다.

예를 들어 다음과 같은 문제가 있는지 본다.

```text
- 응답 시간이 길어지고 있는가?
- p95, p99 latency가 증가하고 있는가?
- 요청 대기열이 쌓이는가?
- 서버 메모리 사용률이 급격히 증가하는가?
- 스레드 수가 과도하게 증가하는가?
- CPU 사용률은 낮은데 요청 처리가 밀리는가?
- 트래픽 증가 시 서버가 버티지 못하는가?
```

성능 문제가 없다면 논블로킹 I/O나 가상 스레드를 적용할 이유가 약하다.

이런 변경은 구조 복잡도, 디버깅 난이도, 유지보수 비용을 높일 수 있기 때문이다.

---

## 3. 문제가 I/O 관련인가?

성능 문제가 있다고 해서 바로 논블로킹 I/O나 가상 스레드를 적용하면 안 된다.

그 문제가 정말 I/O 대기와 관련된 문제인지 확인해야 한다.

### 적용 효과가 있을 가능성이 높은 경우

```text
- 요청 수가 늘어나면서 실행 중인 스레드 수가 급증한다.
- 많은 스레드가 DB, Redis, 외부 API, 파일 I/O 응답을 기다리고 있다.
- CPU 사용률은 아직 여유가 있다.
- 메모리 사용률이 스레드 증가와 함께 올라간다.
- thread dump에서 WAITING, TIMED_WAITING, socket read 상태가 많다.
- Tomcat thread pool이나 executor queue가 자주 가득 찬다.
```

이런 경우는 플랫폼 스레드가 I/O 대기 때문에 많이 묶여 있는 상황일 수 있다.

이때는 가상 스레드나 논블로킹 I/O가 효과를 볼 가능성이 있다.

---

### 적용 효과가 낮은 경우

```text
- CPU 사용률이 90~100%에 가깝다.
- DB 쿼리 자체가 느리다.
- 인덱스 문제, 락 경합, N+1 문제가 있다.
- DB connection pool이 부족하다.
- 외부 API 자체가 느리거나 rate limit에 걸린다.
- GC pause가 길다.
- 메모리 누수가 있다.
```

이런 경우에는 논블로킹 I/O나 가상 스레드를 적용해도 근본 문제가 해결되지 않는다.

예를 들어 DB 쿼리가 3초 걸리는 상황이라면, 가상 스레드를 적용해도 쿼리가 300ms가 되지는 않는다.

다만 기존 플랫폼 스레드가 3초 동안 묶여 있던 비용을 줄여 더 많은 요청을 받아낼 수는 있다.

---

## 4. 가상 스레드 적용 가능 여부 판단 기준

가상 스레드를 적용할 수 있는지는 단순히 Java 버전만으로 판단하지 않는다.

다음 세 가지를 함께 봐야 한다.

```text
1. 기술적으로 적용 가능한가?
2. 적용했을 때 효과가 나는 구조인가?
3. 적용했을 때 부작용을 감당할 수 있는가?
```

---

## 5. 기술적으로 적용 가능한가?

먼저 런타임과 프레임워크가 가상 스레드를 지원하는지 확인한다.

Spring Boot 기준으로는 대략 다음 조건을 본다.

```text
- Java 21 이상을 사용할 수 있는가?
- Spring Boot에서 virtual thread 옵션을 사용할 수 있는가?
- 운영 배포 환경에서 JDK 버전 변경이 가능한가?
- WAS, Tomcat, Jetty, Executor 설정과 충돌하지 않는가?
```

Spring Boot에서는 다음 옵션으로 가상 스레드를 활성화할 수 있다.

```properties
spring.threads.virtual.enabled=true
```

하지만 이 옵션을 켤 수 있다고 해서 바로 적용 가능한 것은 아니다.

반드시 부하 테스트와 런타임 지표 확인이 필요하다.

---

## 6. 효과가 나는 구조인가?

가상 스레드는 특히 기존 코드가 동기 blocking 방식일 때 효과를 보기 쉽다.

대표적으로 다음 구조와 잘 맞는다.

```text
- Spring MVC
- Servlet 기반 서버
- JDBC 기반 DB 접근
- RestTemplate 또는 blocking HTTP client
- 요청 1개당 스레드 1개를 사용하는 thread-per-request 구조
```

예를 들면 다음과 같은 코드이다.

```java
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) {
    User user = userRepository.findById(id);
    Order order = orderClient.getRecentOrder(id);

    return UserResponse.from(user, order);
}
```

이 코드는 동기적으로 보이지만 내부에서 DB I/O, 외부 API I/O가 발생한다.

기존 플랫폼 스레드는 이 I/O가 끝날 때까지 대기한다.

가상 스레드를 사용하면 이 대기 비용을 줄일 수 있다.

---

## 7. 효과가 낮은 구조

다음과 같은 경우에는 가상 스레드 효과가 낮을 수 있다.

```text
- 이미 WebFlux, Reactor 기반 논블로킹 구조를 사용 중이다.
- Kotlin Coroutine 기반으로 이미 비동기 처리를 하고 있다.
- Netty 기반 논블로킹 서버 구조이다.
- CPU 연산이 병목이다.
- DB 쿼리 자체가 느린 것이 핵심 문제이다.
```

가상 스레드는 CPU 연산을 빠르게 만드는 기술이 아니다.

따라서 CPU 병목이거나 DB 자체가 느린 문제라면 다른 개선이 우선이다.

---

## 8. Thread Dump로 근거 찾기

가상 스레드 적용 가능성을 판단하려면 thread dump를 확인해야 한다.

확인할 내용은 다음과 같다.

```text
- http-nio-* 스레드가 많이 쌓여 있는가?
- tomcat-* 스레드가 많이 쌓여 있는가?
- executor-* 스레드가 많이 쌓여 있는가?
- 대부분 RUNNABLE 상태인가?
- 대부분 WAITING, TIMED_WAITING 상태인가?
- socket read, DB read 상태가 많은가?
- Hikari connection 획득 대기가 많은가?
```

판단 기준은 다음과 같다.

```text
대부분 CPU 연산 중이다
→ 가상 스레드 효과 낮음

대부분 DB, HTTP, Redis, 파일 I/O 대기 중이다
→ 가상 스레드 적용 후보

대부분 Hikari connection 획득 대기 중이다
→ 가상 스레드보다 DB connection pool, 쿼리, DB 처리량 확인 우선
```

---

## 9. APM과 메트릭으로 확인할 지표

APM이나 모니터링 도구에서는 다음 지표를 확인한다.

```text
- CPU 사용률
- 메모리 사용률
- 전체 스레드 수
- 요청 처리량
- p95 / p99 응답 시간
- DB query time
- DB connection acquire time
- Hikari active connections
- Hikari pending threads
- 외부 API 응답 시간
- executor queue size
- GC pause time
```

가상 스레드가 효과적일 가능성이 있는 상황은 다음과 같다.

```text
CPU 사용률은 낮거나 중간이다.
메모리 사용률은 높다.
플랫폼 스레드 수가 많다.
요청 대기열이 증가한다.
응답 시간이 증가한다.
I/O 대기 시간이 많다.
```

반대로 다음 상황이면 다른 병목을 먼저 봐야 한다.

```text
CPU 사용률이 매우 높다.
DB query time 자체가 길다.
connection pool wait time이 길다.
GC pause가 길다.
특정 외부 API 응답이 매우 느리다.
```

---

## 10. DB Connection Pool 병목 확인

가상 스레드를 적용하면 더 많은 요청을 가볍게 받을 수 있다.

하지만 DB connection pool 크기가 그대로라면 실제 DB 동시 처리량은 그대로 제한된다.

예를 들어 다음 상황을 보자.

```text
기존 플랫폼 스레드: 200개
DB connection pool: 30개
동시 요청: 1000개
```

가상 스레드를 적용하면 1000개 요청을 더 적은 스레드 비용으로 받아낼 수 있다.

하지만 DB에 동시에 접근할 수 있는 요청은 여전히 connection pool 크기에 제한된다.

따라서 다음 지표를 반드시 봐야 한다.

```text
- Hikari active connections
- Hikari idle connections
- Hikari pending threads
- connection acquire time
- connection timeout 발생 여부
- DB CPU 사용률
- slow query
- lock wait
```

판단은 다음과 같이 한다.

```text
DB connection 획득 대기가 병목이다
→ 가상 스레드만으로 해결 어렵다.
→ 쿼리 튜닝, pool 크기 조정, DB 확장, 캐시 검토가 우선이다.

서버 스레드가 I/O 대기로 많이 묶여 있고 DB는 여유가 있다
→ 가상 스레드 적용 효과가 있을 수 있다.
```

---

## 11. Pinning 여부 확인

가상 스레드 적용 시 주의해야 할 개념 중 하나가 pinning이다.

가상 스레드는 I/O 대기 시 carrier thread에서 분리되어 다른 작업이 실행될 수 있게 한다.

하지만 특정 상황에서는 가상 스레드가 carrier thread에 붙잡혀 내려오지 못할 수 있다.

이를 pinning이라고 한다.

대표적인 원인은 다음과 같다.

```text
- synchronized 블록 안에서 blocking I/O 수행
- synchronized 메서드 안에서 DB 호출
- synchronized 메서드 안에서 외부 API 호출
- native method 호출
- foreign function 호출
```

위험한 예시는 다음과 같다.

```java
public synchronized UserResult process() {
    User user = userRepository.findById(id);

    PaymentResult result = paymentClient.approve(...);

    return UserResult.from(user, result);
}
```

이 코드는 `synchronized` 안에서 DB I/O와 외부 API I/O를 수행한다.

가상 스레드 적용 시 pinning 문제가 발생할 수 있다.

개선 방향은 다음과 같다.

```java
public UserResult process() {
    User user = userRepository.findById(id);
    PaymentResult result = paymentClient.approve(...);

    synchronized (this) {
        return updateState(user, result);
    }
}
```

핵심은 긴 I/O 작업을 `synchronized` 밖으로 빼는 것이다.

---

## 12. JFR로 확인할 것

가상 스레드를 테스트 환경에서 활성화한 뒤 JFR을 통해 다음 이벤트를 확인한다.

```text
jdk.VirtualThreadPinned
```

확인할 내용은 다음과 같다.

```text
- VirtualThreadPinned 이벤트가 많이 발생하는가?
- pinning 시간이 긴가?
- pinning 발생 지점이 요청의 핵심 경로인가?
- 특정 라이브러리나 코드에서 반복적으로 발생하는가?
- synchronized 안에서 I/O가 발생하고 있는가?
```

pinning이 적고 짧다면 감당 가능할 수 있다.

반대로 pinning이 많고 긴 시간이 발생한다면 가상 스레드 적용 효과가 떨어질 수 있다.

---

## 13. 적용 가능성이 높은 경우

다음 조건에 많이 해당하면 가상 스레드 적용 가능성이 높다.

```text
- Java 21 이상으로 올릴 수 있다.
- Spring MVC, Servlet, JDBC 기반의 동기 blocking 구조이다.
- 요청당 하나의 스레드를 점유하는 thread-per-request 구조이다.
- 트래픽 증가 시 플랫폼 스레드 수와 메모리 사용량이 급증한다.
- CPU 사용률은 아직 여유가 있다.
- thread dump상 많은 스레드가 DB, HTTP, Redis 등 I/O 대기 상태이다.
- JFR 테스트에서 VirtualThreadPinned 이벤트가 많지 않다.
- synchronized 안에서 긴 I/O를 수행하지 않는다.
- DB connection pool, 외부 API rate limit 같은 하위 병목도 함께 관리할 수 있다.
```

---

## 14. 적용 가능성이 낮은 경우

다음 조건에 해당하면 가상 스레드 적용 우선순위가 낮다.

```text
- Java 21 이상으로 올릴 수 없다.
- 이미 WebFlux, Reactor 기반 논블로킹 구조이다.
- CPU 연산이 병목이다.
- DB 쿼리 자체가 느린 것이 주원인이다.
- DB connection pool 고갈이 주원인이다.
- synchronized 안에서 긴 I/O를 많이 수행한다.
- native library 호출이 hot path에 많다.
- ThreadLocal에 무거운 객체를 캐싱하는 구조가 많다.
- 부하 테스트에서 처리량 개선 없이 DB wait만 증가한다.
- pinning 이벤트가 많이 발생한다.
```

---

## 15. 가상 스레드 적용 판단 절차

실무에서는 다음 순서로 판단하는 것이 좋다.

```text
1. 현재 병목 지표 확인
   - CPU
   - 메모리
   - thread count
   - request queue
   - p95 / p99 latency
   - DB query time
   - connection pool wait time

2. thread dump 확인
   - CPU 실행 중인지
   - I/O 대기 중인지
   - DB connection 대기 중인지

3. 코드 구조 확인
   - Spring MVC / Servlet / JDBC 기반인가?
   - thread-per-request 구조인가?
   - 이미 논블로킹 구조는 아닌가?

4. 위험 코드 확인
   - synchronized 안에서 I/O 수행
   - native method 호출
   - 무거운 ThreadLocal 사용
   - 고정 thread pool에 강하게 의존하는 로직

5. 테스트 환경에서 가상 스레드 활성화
   - spring.threads.virtual.enabled=true

6. 부하 테스트 비교
   - 기존 플랫폼 스레드 버전
   - 가상 스레드 적용 버전

7. 결과 비교
   - 처리량
   - p95 / p99 응답 시간
   - 메모리 사용량
   - platform thread 수
   - virtual thread 수
   - DB connection wait time
   - VirtualThreadPinned 이벤트
```

---

## 16. 논블로킹 I/O와 가상 스레드 선택 기준

둘 다 I/O 대기 문제를 완화할 수 있지만 접근 방식이 다르다.

|구분|가상 스레드|논블로킹 I/O|
|---|---|---|
|코드 스타일|동기 코드 유지|비동기/이벤트 기반 코드|
|대표 구조|Spring MVC + JDBC|WebFlux + Reactor|
|장점|기존 코드 변경이 적음|적은 스레드로 높은 동시성|
|단점|pinning, pool 병목 주의|코드 복잡도 증가|
|적합한 경우|기존 동기 서버 개선|처음부터 비동기 구조 설계|
|주의점|DB pool, synchronized, ThreadLocal|콜백/체인 복잡도, 디버깅 난이도|

---

## 17. 최종 정리

논블로킹 I/O나 가상 스레드는 성능 문제가 있을 때 검토해야 한다.

성능 문제가 없다면 적용 자체가 유지보수 비용만 증가시킬 수 있다.

성능 문제가 있더라도 그 원인이 CPU 연산, DB 쿼리 자체, connection pool 고갈, GC 문제라면 논블로킹 I/O나 가상 스레드가 근본 해결책이 되기 어렵다.

가상 스레드는 특히 동기 blocking I/O 기반의 thread-per-request 구조에서 효과를 보기 쉽다.

하지만 적용 전에는 반드시 thread dump, APM 지표, DB connection pool 지표, JFR의 VirtualThreadPinned 이벤트, 부하 테스트 결과를 통해 실제 개선 가능성과 부작용을 확인해야 한다.

---

## 한 줄 요약

가상 스레드 적용 가능 여부는  
`Java 21을 쓸 수 있는가?`만으로 판단하지 않는다.

현재 병목이 `I/O 대기 중인 플랫폼 스레드 비용`인지,  
애플리케이션 구조가 `동기 blocking thread-per-request 구조`인지,  
그리고 `pinning, DB pool, 외부 API 제한` 같은 부작용이 없는지를  
thread dump, APM, JFR, 부하 테스트로 검증해서 판단해야 한다.
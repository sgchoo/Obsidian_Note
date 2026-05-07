[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 동시성(Concurrency) 처리

## 개요

여러 스레드나 프로세스가 동시에 같은 데이터를 수정할 때 발생하는 동시성 문제는 전형적인 실수 중 하나다. 동시성 문제는 **프로세스 수준**과 **DB 수준** 양쪽 모두에서 검토해야 한다. 어느 한쪽만 방어해도 데이터 정합성이 깨질 수 있기 때문이다.

> 예시: 단일 프로세스에서는 `synchronized`로 보호하고 있더라도, 다중 인스턴스로 스케일 아웃되는 순간 프로세스 단위 잠금은 무력화된다. 반대로 DB 트랜잭션만 신뢰하면 격리 수준에 따라 lost update가 발생할 수 있다.

---

## 1. 프로세스 수준의 동시성 제어

### 1.1 잠금(Lock)을 이용한 접근 제어

가장 일반적이고 직관적인 방법이다. 일반적인 흐름은 다음과 같다.

1. 잠금을 획득(acquire)
2. 공유 자원에 접근 — **임계 영역(critical section)**
3. 잠금을 해제(release)

#### Lock vs Semaphore

|구분|동시 진입 가능 스레드 수|주요 용도|
|---|---|---|
|**Lock (Mutex)**|1개|임계 영역 보호, 상호 배제|
|**Semaphore**|N개(설정 가능)|자원 풀(connection pool, rate limit 등) 제한|

- **Lock**은 한 번에 1개의 스레드만 잠금을 획득할 수 있다.
- **Semaphore**는 동시에 임계 영역에 진입할 수 있는 스레드 수를 N개로 제한할 수 있다.

#### 추가로 알아둘 잠금 종류

요약본에 빠진 부분 중 실무에서 자주 쓰이는 것들을 보완한다.

- **ReentrantLock**: 같은 스레드가 이미 획득한 잠금을 재획득할 수 있다. `synchronized`도 재진입을 지원하지만 `ReentrantLock`은 `tryLock(timeout)`, 공정성 옵션, 조건 변수(`Condition`)를 별도로 제공한다.
- **ReadWriteLock**: 읽기는 공유, 쓰기는 배타. 읽기가 압도적으로 많은 워크로드에서 유리하다.
- **StampedLock (Java 8+)**: 낙관적 읽기(optimistic read)를 지원해 `ReadWriteLock`보다 처리량이 높을 수 있다. 단, 재진입을 지원하지 않는다.

### 1.2 잠금을 사용하지 않는 동시성 제어

잠금은 컨텍스트 스위칭과 대기 비용을 유발한다. 단순 카운터처럼 작은 임계 영역이라면 더 가벼운 수단이 있다.

#### 원자적 타입 (Atomic Types)

- Java의 `AtomicInteger`, `AtomicLong`, `AtomicReference` 등이 대표적이다.
- 내부적으로 **CAS(Compare-And-Swap)** 연산을 사용한다.
    - CAS는 "현재 값이 기대값과 같으면 새 값으로 교체"하는 단일 CPU 명령(예: x86의 `CMPXCHG`)이다.
    - 실패하면 재시도하는 **lock-free** 알고리즘으로 동작한다.
- 경합이 적을 때는 잠금보다 빠르지만, 경합이 매우 심하면 재시도 비용이 누적되어 오히려 느려질 수 있다. 이때는 `LongAdder`처럼 분산 누적 방식을 쓰는 편이 낫다.

#### 동시성 컬렉션 (Concurrent Collections)

- Java의 `ConcurrentHashMap`, `CopyOnWriteArrayList`, `ConcurrentLinkedQueue` 등.
- 내부적으로 세분화된 잠금(lock striping), CAS, 불변 스냅샷 등을 조합해 사용한다.
- **주의**: 컬렉션 자체의 단일 연산은 스레드 안전하지만, **여러 연산을 묶은 복합 동작은 스레드 안전하지 않다.**
    
    ```java
    // ❌ get-then-put은 원자적이지 않다if (!map.containsKey(key)) {    map.put(key, value);}// ✅ 원자적 메서드를 사용해야 한다map.putIfAbsent(key, value);// 또는map.computeIfAbsent(key, k -> compute(k));
    ```
    

---

## 2. DB 수준의 동시성 제어

DB에서 발생하는 동시성 문제는 **비관적 잠금**과 **낙관적 잠금** 두 가지 전략으로 해결한다.

### 2.1 비관적 잠금 (Pessimistic Lock, 선점 잠금)

- 충돌이 **자주 일어날 것**으로 가정하고 데이터를 읽는 시점에 미리 잠금을 건다.
- 구현: `SELECT ... FOR UPDATE` (배타 잠금), `SELECT ... FOR SHARE` (공유 잠금).
- **장점**: 충돌 시 재시도가 필요 없어 로직이 단순하다.
- **단점**: 트랜잭션이 길어지면 다른 트랜잭션이 모두 대기하므로 처리량이 떨어진다. 교착 상태(deadlock) 위험도 커진다.
- **적합한 상황**: 충돌 빈도가 높은 핫 데이터(예: 재고 차감, 좌석 예약).

### 2.2 낙관적 잠금 (Optimistic Lock, 비선점 잠금)

- 충돌이 **드물 것**으로 가정하고 잠금 없이 진행한 뒤, 커밋 시점에 충돌 여부를 검사한다.
- 구현: 보통 **버전 컬럼(version)** 또는 **타임스탬프**를 사용.
    
    ```sql
    UPDATE item   SET stock = stock - 1, version = version + 1 WHERE id = ? AND version = ?;-- affected rows = 0 이면 다른 트랜잭션이 먼저 수정한 것 → 재시도 또는 실패 처리
    ```
    
- JPA에서는 `@Version` 어노테이션으로 간단히 구현할 수 있다.
- **장점**: 잠금이 없어 처리량이 높다.
- **단점**: 충돌 시 재시도 로직이 필요하며, 충돌이 많으면 오히려 비효율적이다.
- **적합한 상황**: 충돌 빈도가 낮은 일반적인 CRUD.

### 2.3 증분 쿼리 (Atomic Increment Query)

잠금 없이 수치를 증가시키는 방법이다.

```sql
UPDATE counter SET value = value + 1 WHERE id = ?;
```

- DB가 **행 단위 잠금**을 자동으로 걸기 때문에 일반적으로 안전하다.
- 다만 **DB 엔진과 격리 수준에 따라 원자성이 달라질 수 있어** 사용 전 반드시 확인이 필요하다.
    - MySQL InnoDB, PostgreSQL 등 주요 RDBMS의 `UPDATE ... SET col = col + 1`은 행 단위 X-lock으로 안전하다.
    - 일부 NoSQL이나 분산 DB에서는 별도의 원자적 연산 API(예: MongoDB의 `$inc`, Redis의 `INCR`)를 사용해야 한다.

### 2.4 격리 수준(Isolation Level)도 함께 검토할 것

요약본에는 빠져 있지만 동시성 이야기에서 빼놓을 수 없는 부분이다.

|격리 수준|Dirty Read|Non-repeatable Read|Phantom Read|
|---|---|---|---|
|READ UNCOMMITTED|O|O|O|
|READ COMMITTED|X|O|O|
|REPEATABLE READ|X|X|O (MySQL InnoDB는 갭락으로 거의 차단)|
|SERIALIZABLE|X|X|X|

격리 수준이 낮으면 **lost update**(갱신 분실)가 발생할 수 있고, 이때는 비관적/낙관적 잠금으로 보완해야 한다.

---

## 3. 주의 사항

### 3.1 잠금 해제하기

- 예외가 발생해도 잠금이 반드시 해제되도록 보장해야 한다.
    
    ```java
    lock.lock();try {    // critical section} finally {    lock.unlock(); // 반드시 finally에서 해제}
    ```
    
- Kotlin/Java에서는 `synchronized` 블록을 쓰면 자동 해제되지만, `ReentrantLock`처럼 명시적 잠금은 `try-finally`가 필수다.

### 3.2 대기 시간 지정하기

- 무한 대기(`lock()`, `acquire()`)는 시스템 전체가 멈출 위험이 있다.
- `tryLock(timeout)`처럼 **타임아웃이 있는 API**를 사용해 일정 시간 안에 잠금을 얻지 못하면 실패 처리하도록 한다.
- DB에서도 `innodb_lock_wait_timeout`, `SET LOCAL lock_timeout` 등으로 대기 시간을 제한할 수 있다.

### 3.3 교착 상태 (Deadlock)

두 개 이상의 스레드/트랜잭션이 서로가 가진 잠금을 기다리며 영원히 진행하지 못하는 상태.

**발생 조건 (Coffman 조건)**

1. 상호 배제(Mutual Exclusion)
2. 점유 대기(Hold and Wait)
3. 비선점(No Preemption)
4. 순환 대기(Circular Wait)

**예방·완화 방법**

- **잠금 순서 통일**: 모든 코드 경로에서 자원을 동일한 순서로 획득한다 (가장 효과적).
- **타임아웃**: `tryLock(timeout)`으로 순환 대기를 깨뜨릴 수 있게 한다.
- **잠금 범위 최소화**: 임계 영역을 짧게 유지해 충돌 가능성 자체를 줄인다.
- **DB 차원**: 대부분의 DB는 데드락을 자동 감지하고 한쪽 트랜잭션을 롤백시킨다(MySQL InnoDB의 `Deadlock found` 오류). 애플리케이션에서는 이 예외를 잡아 **재시도 로직**을 갖춰두는 편이 안전하다.

### 3.4 보완: 분산 환경에서의 동시성

요약본 범위를 넘지만 실무에서는 거의 항상 만나게 된다.

- **분산 잠금(Distributed Lock)**: 여러 인스턴스에서 공통된 잠금이 필요할 때 Redis(Redlock), ZooKeeper, etcd 등을 활용.
- **멱등성(Idempotency)**: 재시도가 자연스럽게 일어나는 분산 환경에서는, 잠금보다 **요청을 여러 번 처리해도 같은 결과**가 되도록 설계하는 것이 더 견고할 때가 많다.
- **메시지 큐 직렬화**: 같은 키의 작업을 같은 컨슈머/파티션으로 보내 자연스럽게 순차 처리되도록 만드는 방식도 동시성 문제를 회피하는 좋은 패턴이다.

---

## 정리

|레벨|도구|사용 시점|
|---|---|---|
|프로세스|`synchronized`, `ReentrantLock`|전통적 임계 영역 보호|
|프로세스|`Atomic*`, CAS|단순 수치/참조 갱신|
|프로세스|`ConcurrentHashMap` 등|동시 접근 컬렉션|
|DB|비관적 잠금 (`FOR UPDATE`)|충돌 빈도 높음|
|DB|낙관적 잠금 (`@Version`)|충돌 빈도 낮음|
|DB|증분 쿼리 (`col = col + 1`)|단순 카운터|
|분산|Redis/ZooKeeper 분산 잠금, 멱등 설계|다중 인스턴스|

핵심은 "**한 가지 수단에 의존하지 말고 레벨별로 검토할 것**"이다. 프로세스 잠금은 단일 인스턴스 안에서만 유효하고, DB 잠금은 트랜잭션 경계 안에서만 유효하다. 시스템 구조에 맞는 조합을 선택해야 한다.
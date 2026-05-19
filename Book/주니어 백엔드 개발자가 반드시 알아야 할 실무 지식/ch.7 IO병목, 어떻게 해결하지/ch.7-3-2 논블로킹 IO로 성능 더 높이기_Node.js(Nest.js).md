[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]
## 1. Node.js / NestJS 진영에서는 어떻게 볼 수 있는가?

Java NIO에서는 개발자가 `Selector`, `Channel`, `SelectionKey`를 직접 다룬다.

반면 Node.js / NestJS에서는 개발자가 Java NIO의 Selector 같은 코드를 직접 작성하지 않는다.

Node.js 내부에서 다음 요소들이 그 역할을 수행한다.

```text
Node.js 런타임
libuv
Event Loop
OS의 I/O 이벤트 감시 기능
```

Node.js는 운영체제의 I/O 감시 기능을 직접 JavaScript 코드에서 다루게 하지 않고, 런타임이 감싸서 처리한다.

Nest 개발자는 보통 다음처럼 작성한다.

```ts
const user = await this.userRepository.findOne(id);
```

또는 다음처럼 작성한다.

```ts
const result = await axios.get(url);
```

이 코드에서 `await`를 사용하면 현재 async 함수의 흐름은 잠시 멈춘다.

하지만 Node의 메인 스레드 전체가 I/O 완료까지 블로킹되는 것은 아니다.

흐름은 대략 다음과 같다.

```text
Nest 코드
  ↓
DB 또는 외부 API 요청
  ↓
I/O 작업을 런타임/libuv/OS에 위임
  ↓
현재 async 함수는 대기 상태
  ↓
Node 이벤트 루프는 다른 요청을 계속 처리
  ↓
I/O 완료 이벤트 도착
  ↓
Promise resolve/reject
  ↓
await 이후 코드 실행
```

---

## 2. Promise는 Selector인가?

Promise는 Selector가 아니다.

Selector의 역할은 다음이다.

```text
여러 I/O 채널의 준비 상태를 감시한다.
```

Promise의 역할은 다음이다.

```text
비동기 작업이 완료되었을 때 성공/실패 결과를 담고,
그 결과를 JavaScript 코드에서 이어받을 수 있게 한다.
```

즉, 역할을 나누면 다음과 같다.

|역할|담당|
|---|---|
|I/O 준비 상태 감시|OS + libuv + Node Event Loop|
|완료 이벤트 처리|Node Event Loop|
|완료 결과를 JS 코드로 표현|Promise|
|동기 코드처럼 이어 쓰는 문법|async/await|

따라서 Node/Nest에서 Java NIO의 Selector 역할을 하는 코드를 직접 작성할 필요는 거의 없다.

그 역할은 Node 런타임과 libuv가 내부적으로 수행한다.

---

## 3. Promise는 이벤트인가?

Promise를 이벤트처럼 생각하면 이해에는 도움이 된다.

예를 들어 다음 코드는 다음과 같은 의미이다.

```ts
db.user.findMany().then((users) => {
  console.log(users);
});
```

```text
DB 조회가 끝나면 이 함수를 실행해줘.
```

이 점에서는 이벤트와 비슷하다.

하지만 Promise 자체가 이벤트인 것은 아니다.

이벤트는 보통 여러 번 발생할 수 있다.

```ts
socket.on("data", (chunk) => {
  console.log(chunk);
});
```

`data` 이벤트는 소켓에서 데이터가 올 때마다 여러 번 발생할 수 있다.

반면 Promise는 한 번만 완료된다.

```text
pending → fulfilled
```

또는

```text
pending → rejected
```

한 번 resolve 또는 reject되면 상태가 다시 바뀌지 않는다.

따라서 Promise는 다음처럼 이해하는 것이 더 정확하다.

> Promise는 이벤트 그 자체가 아니라, 비동기 작업의 완료/실패 신호와 결과를 담는 객체이다.

---

## 4. await의 의미

`await`는 Promise가 완료될 때까지 현재 async 함수의 다음 실행을 미룬다.

```ts
const user = await this.userService.findUser();

console.log(user);
```

이 코드는 다음과 같은 흐름이다.

```text
1. findUser() 비동기 작업 시작
2. 결과가 올 때까지 현재 async 함수의 다음 줄 실행 보류
3. 그동안 Node 이벤트 루프는 다른 요청을 처리할 수 있음
4. Promise가 resolve됨
5. user에 결과가 담김
6. console.log(user) 실행
```

중요한 점은 다음이다.

> await는 현재 async 함수의 흐름을 멈추는 것이지, Node의 메인 스레드 전체를 막는 것이 아니다.

---

## 5. Promise를 await하지 않으면?

Promise를 `await`하지 않으면 현재 코드 흐름은 그 Promise의 완료를 기다리지 않는다.

```ts
const promise = this.userService.findUser();

console.log("다음 코드 실행");
```

이 경우 `findUser()`의 비동기 작업은 시작될 수 있지만, 현재 함수는 결과를 기다리지 않고 다음 줄로 넘어간다.

즉 다음과 같다.

```text
비동기 작업 요청은 보냄
하지만 결과를 기다리지 않음
다음 코드 실행
```

실제 결과를 사용하려면 `await`하거나 `.then()`을 사용해야 한다.

```ts
const user = await this.userService.findUser();
```

또는

```ts
this.userService.findUser().then((user) => {
  console.log(user);
});
```

---

## 6. Nest에서 return Promise는 괜찮은가?

Nest 컨트롤러에서는 Promise를 그대로 반환하는 코드는 일반적으로 괜찮다.

```ts
@Get(":id")
findOne(@Param("id") id: string) {
  return this.userService.findOne(id);
}
```

이 코드는 `await`를 사용하지 않았지만 Promise를 호출자에게 반환한다.

Nest는 이 Promise가 완료될 때까지 기다렸다가 HTTP 응답을 보낸다.

따라서 단순 반환만 한다면 아래 코드와 동작상 거의 비슷하다.

```ts
@Get(":id")
async findOne(@Param("id") id: string) {
  return await this.userService.findOne(id);
}
```

단순히 반환만 할 때는 `return await`를 굳이 사용할 필요가 없는 경우가 많다.

---

## 7. 주의할 점: fire-and-forget

다음 코드는 이메일 발송 작업을 기다리지 않고 바로 응답한다.

```ts
async createUser(dto: CreateUserDto) {
  this.emailService.sendWelcomeEmail(dto.email);

  return {
    success: true,
  };
}
```

이 구조는 의도적이라면 fire-and-forget 패턴이다.

하지만 실수라면 문제가 될 수 있다.

```text
회원 생성 응답은 바로 반환됨
이메일 발송 성공/실패는 현재 요청 흐름에서 모름
이메일 발송 실패 시 에러 처리를 놓칠 수 있음
```

최소한 다음처럼 에러 처리는 붙이는 것이 좋다.

```ts
void this.emailService.sendWelcomeEmail(dto.email).catch((error) => {
  this.logger.error("Welcome email failed", error);
});
```

더 안정적인 방식은 큐를 사용하는 것이다.

```text
회원 생성 완료
  ↓
이메일 발송 Job 등록
  ↓
Worker가 비동기로 처리
```

예를 들면 다음과 같은 도구를 사용할 수 있다.

```text
BullMQ
RabbitMQ
Kafka
SQS
별도 Worker 서버
```

---

## 18. Node/Nest에서 조심해야 할 블로킹 코드

Node/Nest가 논블로킹 I/O 기반이라고 해서 모든 코드가 자동으로 논블로킹이 되는 것은 아니다.

I/O 대기는 논블로킹으로 처리할 수 있지만, CPU 연산은 여전히 이벤트 루프를 막을 수 있다.

예를 들어 다음 코드는 위험하다.

```ts
for (let i = 0; i < 10_000_000_000; i++) {
  // heavy CPU work
}
```

또는 다음처럼 Promise로 감싸도 내부가 동기 CPU 작업이면 이벤트 루프를 막는다.

```ts
await new Promise((resolve) => {
  heavyCpuWork();
  resolve(true);
});
```

Promise를 사용했다고 해서 CPU 작업이 자동으로 다른 스레드에서 실행되는 것은 아니다.

따라서 Node/Nest에서 중요한 것은 다음이다.

```text
I/O 작업은 비동기 API를 사용한다.
CPU가 오래 걸리는 작업은 이벤트 루프에서 분리한다.
```

CPU 작업은 다음과 같은 구조로 분리할 수 있다.

```text
Worker Threads
별도 Queue Worker
별도 Microservice
Kubernetes Replica
Node Cluster / PM2 Cluster
```

---

## 19. Java NIO와 Node/Nest 비교

|개념|Java NIO|Node/Nest|
|---|---|---|
|I/O 감시|Selector를 직접 사용|libuv/Event Loop가 내부 처리|
|채널|SocketChannel, ServerSocketChannel|Socket, Stream, HTTP 요청 등으로 추상화|
|준비 이벤트|SelectionKey로 확인|이벤트 루프가 내부 처리|
|결과 처리|read/write 직접 호출|Promise, callback, event, async/await|
|개발자 관심사|Selector/Channel 관리|비동기 API 사용, 이벤트 루프 블로킹 방지|
|병렬성 확장|Multi Selector / EventLoopGroup|Cluster, Worker Threads, Queue Worker, Replica|

---

## 20. 최종 정리

논블로킹 I/O는 다음 문제를 해결한다.

> I/O가 끝날 때까지 스레드가 대기하며 낭비되는 문제

I/O Multiplexing은 다음 문제를 해결한다.

> 하나의 스레드가 하나의 I/O만 기다리는 비효율

Multi Selector 구조는 다음 문제를 완화한다.

> 하나의 Selector 스레드가 모든 Channel의 이벤트 처리까지 순차적으로 담당하는 한계

Node/Nest의 Promise와 async/await는 다음을 제공한다.

> 비동기 작업의 완료 결과를 JavaScript 코드에서 자연스럽게 이어받는 방법

가장 중요한 구분은 다음이다.

```text
Non-blocking I/O
= I/O 준비가 안 됐을 때 스레드가 기다리지 않는 방식

I/O Multiplexing
= 하나의 스레드가 여러 I/O 채널의 준비 상태를 감시하는 방식

Selector
= Java NIO에서 여러 Channel의 I/O 준비 상태를 감시하는 객체

EventLoop
= 이벤트를 감시하고, 준비된 작업의 콜백/후속 처리를 실행하는 루프

Promise
= 비동기 작업의 완료/실패 결과를 담는 객체

await
= Promise가 완료될 때까지 현재 async 함수의 다음 실행을 미루는 문법
```

따라서 Java 진영과 Node/Nest 진영을 연결해서 이해하면 다음과 같다.

```text
Java NIO:
Channel → Selector → SelectionKey → read/write

Node/Nest:
I/O 요청 → libuv/Event Loop → Promise resolve/reject → await 이후 실행
```

결론적으로, Node/Nest에서 개발자가 Java NIO의 Selector 같은 코드를 직접 작성하는 것은 일반적이지 않다.

Node/Nest 개발자에게 더 중요한 것은 다음이다.

```text
1. 비동기 I/O API를 제대로 사용하기
2. await와 Promise의 흐름을 정확히 이해하기
3. 이벤트 루프를 막는 CPU/동기 작업을 피하기
4. 오래 걸리는 작업은 Queue, Worker, 별도 프로세스로 분리하기
```
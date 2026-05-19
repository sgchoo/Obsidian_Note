[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]


# 논블로킹 I/O, I/O 멀티플렉싱, Promise와 이벤트 루프 정리

## 1. 경량 스레드와 논블로킹 I/O

- 가상 스레드와 고루틴 같은 경량 스레드를 사용하면 I/O 중심 작업을 하는 서버의 처리량을 높일 수 있다.
- 하지만 경량 스레드가 많아질수록 더 많은 메모리를 사용하고, 스케줄링에도 더 많은 시간을 사용하게 된다.
- 즉, 사용자가 폭발적으로 증가하면 어느 순간 경량 스레드로도 한계가 온다.
- 이때는 서버의 I/O 구현 방식을 구조적으로 변경해야 한다.
- 이때 사용하는 대표적인 방식이 **논블로킹 I/O**이다.

---

## 2. 논블로킹 I/O 동작 개요

논블로킹 I/O는 입출력이 끝날 때까지 스레드가 대기하지 않는다.

예를 들어 Java NIO에서 다음 코드는 데이터를 읽을 때까지 대기하지 않는다.

```java
int byteReads = channel.read(buffer);
````

논블로킹 모드에서 읽을 데이터가 없다면 `channel.read()`는 기다리지 않고 바로 `0`을 반환할 수 있다.

즉, 블로킹 I/O와 논블로킹 I/O의 차이는 다음과 같다.

|방식|동작|
|---|---|
|Blocking I/O|데이터가 올 때까지 스레드가 대기한다.|
|Non-blocking I/O|데이터가 없으면 기다리지 않고 바로 반환한다.|

하지만 실제로 논블로킹 I/O를 사용할 때는 무작정 `read()`를 계속 호출하지 않는다.

대신 다음과 같이 처리한다.

```text
1. 여러 Channel을 감시한다.
2. 그중 읽기/쓰기 가능한 Channel을 찾는다.
3. 준비된 Channel에 대해서만 read/write를 실행한다.
```

이때 Java NIO에서는 `Selector`가 이 역할을 담당한다.

---

## 3. Selector의 역할

`Selector`는 여러 개의 `Channel`을 감시하다가, 어떤 Channel이 I/O 연산을 수행할 준비가 되었는지 알려준다.

예를 들어 여러 클라이언트가 서버에 연결되어 있다고 하자.

```text
Client A ┐
Client B ├── Selector
Client C ┤
Client D ┘
```

Selector는 다음과 같은 질문에 답해주는 역할을 한다.

```text
지금 읽을 수 있는 Channel은 무엇인가?
지금 쓸 수 있는 Channel은 무엇인가?
새 연결을 받을 수 있는 Channel은 무엇인가?
```

예를 들어 A와 C만 데이터를 보냈다면 Selector는 다음처럼 알려준다.

```text
Readable:
- Channel A
- Channel C
```

그러면 서버는 A와 C에 대해서만 `read()`를 실행한다.

```java
if (key.isReadable()) {
    SocketChannel channel = (SocketChannel) key.channel();
    channel.read(buffer);
}
```

즉, 핵심은 다음과 같다.

> 스레드가 각 소켓 앞에서 기다리는 것이 아니라, Selector가 여러 소켓을 한 번에 감시하고 준비된 소켓만 알려준다.

---

## 4. I/O Multiplexing이란?

이 구조를 **I/O Multiplexing**이라고 볼 수 있다.

I/O Multiplexing은 하나의 스레드가 하나의 I/O만 기다리는 방식이 아니라, 하나의 스레드가 여러 I/O 채널의 상태를 함께 감시하는 방식이다.

### Blocking I/O 방식

```text
Client A → Thread A
Client B → Thread B
Client C → Thread C
Client D → Thread D
```

각 클라이언트마다 스레드가 하나씩 붙고, 각 스레드는 자신의 클라이언트에서 데이터가 올 때까지 대기한다.

### Non-blocking I/O + Selector 방식

```text
Client A ┐
Client B ├── Selector ── Thread 1
Client C ┤
Client D ┘
```

하나의 스레드가 Selector를 통해 여러 Channel을 감시한다.

읽기 가능한 Channel만 골라서 처리하므로, 데이터가 없는 Channel 앞에서 스레드가 낭비되지 않는다.

---

## 5. 단일 Selector 구조의 한계

단일 Selector 구조에서는 하나의 스레드가 여러 Channel을 감시할 수 있다.

```text
Thread-1
  └─ Selector-1
      ├─ Channel A
      ├─ Channel B
      ├─ Channel C
      ├─ Channel D
      ├─ Channel E
      └─ Channel F
```

이 구조에서 A, C, F가 동시에 읽기 가능한 상태가 되었다고 하자.

```text
Readable:
- Channel A
- Channel C
- Channel F
```

Selector는 A, C, F가 읽기 가능하다고 알려준다.

하지만 실제 처리 코드는 하나의 스레드에서 실행되므로 순차적으로 처리된다.

```text
Thread-1:
  1. Channel A read 처리
  2. Channel C read 처리
  3. Channel F read 처리
```

즉, 논블로킹 I/O는 스레드가 I/O 대기 상태에 빠지는 것을 줄여주지만, 하나의 스레드가 동시에 여러 개의 처리 코드를 실행할 수 있는 것은 아니다.

정리하면 다음과 같다.

> 논블로킹 I/O는 대기 효율을 높이는 기술이고, 병렬 처리를 자동으로 만들어주는 기술은 아니다.

---

## 6. 채널을 N개 그룹으로 나누는 구조

단일 Selector의 한계를 완화하기 위해 여러 개의 Selector와 여러 개의 Worker Thread를 둔다.

이때 “채널을 N개 그룹으로 나눈다”는 말은 여러 Channel을 각각 다른 Selector에 나누어 등록한다는 뜻이다.

예를 들어 Worker Thread를 3개 둔다면 다음과 같은 구조가 된다.

```text
Thread-1
  └─ Selector-1
      ├─ Channel A
      └─ Channel B

Thread-2
  └─ Selector-2
      ├─ Channel C
      └─ Channel D

Thread-3
  └─ Selector-3
      ├─ Channel E
      └─ Channel F
```

이때 A, C, F가 동시에 읽기 가능한 상태가 되면 다음처럼 처리될 수 있다.

```text
Thread-1: Channel A 처리
Thread-2: Channel C 처리
Thread-3: Channel F 처리
```

즉, 서로 다른 Selector 그룹에 속한 Channel은 서로 다른 스레드에서 병렬로 처리될 수 있다.

---

## 7. 연결 담당 스레드와 읽기/쓰기 담당 스레드

서버 구조를 조금 더 구체적으로 보면 다음과 같다.

```text
[Acceptor Thread]
        |
        | 새 연결 accept
        v
  SocketChannel 생성
        |
        | 라운드로빈 또는 부하 기준으로 분배
        v

[Worker Thread 1]                  [Worker Thread 2]
  └─ Selector 1                      └─ Selector 2
      ├─ Channel A                       ├─ Channel D
      ├─ Channel B                       ├─ Channel E
      └─ Channel C                       └─ Channel F
```

여기서 역할은 다음과 같다.

|구성 요소|역할|
|---|---|
|Acceptor Thread|새 클라이언트 연결을 받는다.|
|Worker Thread|연결된 Channel의 read/write 이벤트를 처리한다.|
|Selector|여러 Channel의 I/O 준비 상태를 감시한다.|
|Channel|클라이언트와 연결된 통신 통로이다.|

Netty식 용어로 보면 다음과 비슷하다.

```text
BossGroup
  └─ accept 담당

WorkerGroup
  ├─ EventLoop 1
  │   ├─ Channel A
  │   └─ Channel B
  │
  ├─ EventLoop 2
  │   ├─ Channel C
  │   └─ Channel D
  │
  └─ EventLoop 3
      ├─ Channel E
      └─ Channel F
```

여기서 `EventLoop`는 대략 다음 조합으로 이해할 수 있다.

```text
EventLoop = Thread + Selector + 담당 Channel들
```

---

## 8. Channel은 언제 그룹에 배정되는가?

보통 새 클라이언트 연결이 `accept`될 때 특정 Worker에 배정된다.

예를 들어 다음처럼 라운드로빈으로 분배할 수 있다.

```text
1번 연결 → Worker Thread 1
2번 연결 → Worker Thread 2
3번 연결 → Worker Thread 3
4번 연결 → Worker Thread 1
5번 연결 → Worker Thread 2
6번 연결 → Worker Thread 3
```

코드 느낌으로는 다음과 같다.

```java
List<EventLoop> eventLoops = List.of(
    new EventLoop("worker-1"),
    new EventLoop("worker-2"),
    new EventLoop("worker-3")
);

int index = 0;

while (true) {
    SocketChannel channel = serverSocketChannel.accept();

    EventLoop selected = eventLoops.get(index);
    selected.register(channel);

    index = (index + 1) % eventLoops.size();
}
```

이렇게 등록된 이후에는 하나의 Channel은 보통 하나의 Worker Thread에 고정된다.

```text
Channel A → Worker Thread 1
Channel B → Worker Thread 2
Channel C → Worker Thread 3
```

이유는 하나의 Channel을 여러 스레드가 동시에 읽고 쓰면 버퍼 상태나 패킷 처리 순서가 꼬일 수 있기 때문이다.

---

## 9. 예시 코드: 단일 Selector Echo Server

아래 코드는 하나의 Selector가 여러 클라이언트 Channel을 감시하는 단순 Echo Server 예시이다.

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;

public class SingleSelectorEchoServer {

    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();

        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(9000));
        serverChannel.configureBlocking(false);

        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("Server started on port 9000");

        while (true) {
            selector.select();

            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();

            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();

                if (key.isAcceptable()) {
                    ServerSocketChannel server = (ServerSocketChannel) key.channel();
                    SocketChannel clientChannel = server.accept();

                    clientChannel.configureBlocking(false);
                    clientChannel.register(selector, SelectionKey.OP_READ);

                    System.out.println("Client connected: " + clientChannel.getRemoteAddress());
                }

                if (key.isReadable()) {
                    SocketChannel clientChannel = (SocketChannel) key.channel();

                    ByteBuffer buffer = ByteBuffer.allocate(1024);
                    int bytesRead = clientChannel.read(buffer);

                    if (bytesRead == -1) {
                        System.out.println("Client disconnected: " + clientChannel.getRemoteAddress());
                        clientChannel.close();
                        continue;
                    }

                    if (bytesRead > 0) {
                        buffer.flip();

                        String message = new String(buffer.array(), 0, buffer.limit());
                        System.out.println("Received: " + message.trim());

                        clientChannel.write(ByteBuffer.wrap(("echo: " + message).getBytes()));
                    }
                }
            }
        }
    }
}
```

이 코드의 핵심은 다음이다.

```java
selector.select();
```

이 코드는 등록된 여러 Channel 중에서 I/O 이벤트가 발생할 때까지 기다린다.

그리고 다음 부분에서 준비된 Channel만 처리한다.

```java
if (key.isReadable()) {
    SocketChannel clientChannel = (SocketChannel) key.channel();
    clientChannel.read(buffer);
}
```

즉, 모든 Channel에 대해 무작정 `read()`를 호출하는 것이 아니라 읽기 가능한 Channel에 대해서만 `read()`를 호출한다.

---

## 10. 예시 코드: Multi Selector 구조

아래 구조는 Acceptor Thread가 새 연결을 받고, Worker Thread들이 Channel을 나눠 담당하는 방식이다.

```text
Acceptor Thread
  └─ 새 연결 accept

Worker-1
  └─ Selector-1
      ├─ Client A
      └─ Client D

Worker-2
  └─ Selector-2
      ├─ Client B
      └─ Client E

Worker-3
  └─ Selector-3
      ├─ Client C
      └─ Client F
```

핵심 코드 흐름은 다음과 같다.

```java
int selectedIndex = index.getAndIncrement() % workerCount;
Worker selectedWorker = workers[selectedIndex];

selectedWorker.register(clientChannel);
```

즉, 새로 연결된 `clientChannel`을 여러 Worker 중 하나에 등록한다.

Worker는 내부적으로 자기 Selector를 가지고 있다.

```java
static class Worker extends Thread {
    private final Selector selector;

    public void run() {
        while (true) {
            selector.select();

            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();

            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();

                if (key.isReadable()) {
                    handleRead(key);
                }
            }
        }
    }
}
```

이 구조에서는 각 Worker가 자신에게 등록된 Channel만 감시하고 처리한다.

---

## 11. Node.js / NestJS 진영에서는 어떻게 볼 수 있는가?

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

## 12. Promise는 Selector인가?

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

## 13. Promise는 이벤트인가?

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

## 14. await의 의미

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

## 15. Promise를 await하지 않으면?

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

## 16. Nest에서 return Promise는 괜찮은가?

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

## 17. 주의할 점: fire-and-forget

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
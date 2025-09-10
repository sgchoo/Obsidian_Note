[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# 📝 5-2. 별도 스레드(Thread)로 실행하기

비동기 연동의 개념을 가장 단순하게 구현하는 방법은 별도의 **스레드(Thread)**를 생성하여 작업을 위임하는 것입니다. 메인 로직을 처리하는 일꾼(메인 스레드) 외에, 시간이 걸리는 작업을 처리할 보조 일꾼(새로운 스레드)을 하나 더 고용하는 것과 같습니다.

---

## 어떻게 구현할까? 👨‍🍳

메인 셰프(메인 스레드)가 주문을 받아 요리를 하다가, 오래 걸리는 채소 다지기(시간이 걸리는 작업)를 보조 셰프(새로운 스레드)에게 맡기고 자신은 바로 다음 요리(응답 반환)를 준비하는 상황입니다.

Java

```
// Main Thread (메인 셰프)
public void handleUserRequest() {
    System.out.println("주문 접수 완료! 메인 요리를 시작합니다.");

    // 새로운 스레드(보조 셰프)에게 오래 걸리는 작업을 시킴
    Thread sideJob = new Thread(() -> {
        // 이 코드는 별도의 스레드에서 실행됩니다.
        sendWelcomeEmailAsync("user@example.com");
    });
    sideJob.start(); // "start!" 라고 말하면 보조 셰프는 자기 일을 시작함 (Fire and Forget)

    System.out.println("메인 요리 완료! 손님에게 서빙합니다."); // 보조 셰프를 기다리지 않고 바로 응답
}

// 비동기로 실행되는 메서드
public void sendWelcomeEmailAsync(String email) {
    System.out.println("보조 셰프: 이메일 발송 준비 중...");
    try {
        // 5초가 걸리는 이메일 발송 작업 시뮬레이션
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        // 스레드 중단 시 처리
    }
    System.out.println("보조 셰프: 이메일 발송 완료!");
}
```

- **명명 규칙(Naming Convention):** 코드만 보고도 비동기로 동작한다는 것을 알 수 있도록, `sendWelcomeEmailAsync`처럼 메서드 이름 뒤에 **`Async`**와 같은 접미사를 붙이는 것이 좋은 관례입니다.
    

---

## 💣 치명적인 함정: 예외 처리(Exception Handling)

가장 중요하고, 주니어 개발자가 가장 많이 실수하는 부분입니다. **별도 스레드에서 발생한 예외는 원래의 `try/catch` 문으로 잡을 수 없습니다.**

메인 셰프가 보조 셰프에게 일을 시킨 뒤, 그 자리에서 "혹시 사고가 나면 내가 처리할게"(`try/catch`)라고 말해봤자 소용없습니다. 보조 셰프는 이미 다른 곳에서 독립적으로 일하고 있기 때문에, 거기서 칼에 손을 베이는 사고(예외 발생)가 나도 메인 셰프는 알 수 없습니다.

#### 잘못된 예외 처리 예시

Java

```
// ❌ 이렇게 짜면 절대 안 됩니다!
try {
    Thread sideJob = new Thread(() -> {
        // 이메일 주소가 null이라 NullPointerException이 발생한다고 가정
        String email = null;
        email.toLowerCase(); // 여기서 예외 발생!
    });
    sideJob.start();
} catch (Exception e) {
    // 💥 이 catch 블록은 절대 실행되지 않습니다!
    // 메인 스레드는 예외가 발생한 사실조차 모릅니다.
    log.error("이메일 발송 실패!", e);
}
```

---

## 올바른 예외 처리 방법 ✅

예외 처리는 **작업이 실제로 실행되는 코드 블록, 즉 새로운 스레드 내부**에서 직접 해주어야 합니다.

Java

```
// ✅ 올바른 방법
Thread sideJob = new Thread(() -> {
    try {
        String email = null;
        email.toLowerCase(); // 예외 발생!
    } catch (Exception e) {
        // 👍 예외는 바로 여기, 스레드 내부에서 잡아야 합니다.
        log.error("이메일 발송 스레드에서 심각한 오류 발생!", e);
        // 실패 사실을 DB에 기록하는 등의 후속 조치를 할 수 있습니다.
    }
});
sideJob.start();
```

---

## "가장 쉬운 방법"의 진짜 의미 🤔

`new Thread()`는 코드를 작성하기는 쉽지만, 실무에서 사용하기에는 매우 위험한 **저수준(low-level) 방식**입니다.

- **자원 낭비:** 요청이 있을 때마다 새 스레드를 만들고 버리는 것은 비용이 매우 비쌉니다. 사용자가 몰리면 순식간에 서버가 다운될 수 있습니다.
    
- **관리의 어려움:** 생성된 스레드의 개수를 제어하기 어렵고, 작업 실패 시 재시도 같은 고급 기능을 구현하기 복잡합니다.
    
- **치명적 오류:** 스레드 내에서 처리되지 않은(uncaught) 예외는 전체 애플리케이션을 중단시킬 수도 있습니다.
    

> **결론:** 주니어 개발자는 실무에서 **직접 `new Thread()`를 사용하여 비동기 처리를 구현하는 것을 피해야 합니다.** 대신, 언어나 프레임워크가 제공하는 더 안전하고 효율적인 고수준의 비동기 처리 도구를 사용해야 합니다.

- **Java/Spring:** `ExecutorService` 또는 `@Async` 어노테이션
    
- **Python:** `asyncio` 라이브러리
    
- **C# / .NET:** `async/await` 키워드
    

이러한 도구들은 **스레드 풀(Thread Pool)**이라는 기술을 사용해 스레드를 효율적으로 재사용하고, 예외 처리를 훨씬 쉽게 도와줍니다
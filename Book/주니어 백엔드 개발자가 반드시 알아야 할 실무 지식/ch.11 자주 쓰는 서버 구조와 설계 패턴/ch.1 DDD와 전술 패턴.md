[[주니어 백엔드 개발자가 반드시 알아야 할 실무 지식]]

# DDD 전술 패턴(Tactical Pattern) 정리

## 1. 왜 DDD 전술 패턴을 쓰는가

로직이 복잡한 도메인일수록 "데이터 구조 + 서비스 계층에 절차형 로직"으로 구현하면 다음 문제가 생긴다.

- 비즈니스 규칙(불변식)이 여러 서비스 메서드에 중복/분산된다.
- 어떤 상태 변경이 허용되는 조합인지 코드만 보고 파악하기 어렵다.
- 트랜잭션 범위와 일관성 경계가 명확하지 않아, 반쯤 갱신된 상태가 생길 위험이 있다.

DDD는 이 문제를 "도메인 모델이 스스로 규칙을 지키게 만드는 것"으로 해결한다. 아래 구성 요소는 서로 독립적인 게 아니라, **Aggregate를 중심으로 유기적으로 연결된 하나의 세트**로 이해해야 한다.

---

## 2. 도메인 모델 구성 요소

### 2.1 Entity (엔티티)

- 고유 식별자(Identity)로 구분되며, 내부 상태가 바뀌어도 식별자는 불변이다.
- 식별자가 같으면 다른 필드 값이 모두 달라도 "같은 객체"로 취급한다(동등성 판단 기준 = ID).
- 예: `Order`(주문번호로 식별), `Member`(회원번호로 식별)

```typescript
// NestJS/TS 예시
export class Order {
  private readonly id: OrderId; // 식별자, 불변
  private orderStatus: OrderStatus;
  private orderLines: OrderLine[];

  changeShippingAddress(newAddress: ShippingAddress): void {
    if (!this.orderStatus.isChangeable()) {
      throw new OrderCannotBeChangedException(this.id);
    }
    this.shippingAddress = newAddress;
  }
}
```

```kotlin
// Spring Boot/Kotlin 예시
class Order(
    val id: OrderId, // 식별자, val로 불변 보장
    private var orderStatus: OrderStatus,
    private val orderLines: MutableList<OrderLine>
) {
    fun changeShippingAddress(newAddress: ShippingAddress) {
        check(orderStatus.isChangeable()) { "이미 배송이 시작된 주문은 변경할 수 없습니다: $id" }
        this.shippingAddress = newAddress
    }
}
```

### 2.2 Value (밸류)

- 식별자가 없고, **속성 값 자체**로 동등성을 판단한다(`Money(1000) == Money(1000)`).
- 불변(Immutable) 객체로 구현하는 것이 원칙 — 값이 바뀌어야 하면 새 인스턴스로 교체한다.
- 예: `Money`, `ShippingAddress`, `OrderLine`(주문 항목 자체도 Order Aggregate 내부에서는 식별자 없이 값으로 다루는 경우가 많음)

```typescript
export class Money {
  private readonly amount: number;
  constructor(amount: number) {
    if (amount < 0) throw new InvalidMoneyException();
    this.amount = amount;
  }
  add(other: Money): Money {
    return new Money(this.amount + other.amount); // 새 인스턴스 반환
  }
  equals(other: Money): boolean {
    return this.amount === other.amount;
  }
}
```

```kotlin
data class Money(val amount: BigDecimal) {
    init { require(amount >= BigDecimal.ZERO) { "금액은 음수일 수 없습니다" } }
    operator fun plus(other: Money) = Money(amount + other.amount)
}
// Kotlin data class는 equals/hashCode를 값 기준으로 자동 생성 → Value 구현에 적합
```

> **보완 포인트**: Value는 "타입"으로서의 의미도 크다. `String`으로 주소를 표현하면 형식 검증이 여기저기 흩어지지만, `ShippingAddress` 타입으로 감싸면 생성 시점에 검증을 강제할 수 있다(Value Object = 타입 안전성 + 자기 검증).

### 2.3 Aggregate (애그리거트)

- 연관된 Entity와 Value를 묶어 **하나의 일관성(consistency) 경계**를 이룬다.
- 예: `Order` Aggregate = `Order`(Root Entity) + `OrderLine`(Value 목록) + `ShippingAddress`(Value)

**보완 포인트 — 원문에서 빠진 핵심 개념들:**

1. **Aggregate Root**: Aggregate 내부에는 여러 객체가 있지만, 외부에서는 반드시 **Root 하나를 통해서만** 접근해야 한다. `OrderLine`을 외부에서 직접 수정하는 코드가 있으면 안 되고, 반드시 `order.changeOrderLine(...)`처럼 Root의 메서드를 거쳐야 한다.
2. **불변식(Invariant) 관리**: Aggregate의 존재 이유는 "이 경계 안의 규칙은 항상 참이어야 한다"는 것을 보장하는 것이다. 예: "주문 총액은 항상 OrderLine 합계와 일치해야 한다", "배송 시작 후에는 주소를 바꿀 수 없다."
3. **트랜잭션 경계 = Aggregate 경계**: 하나의 트랜잭션에서는 원칙적으로 **하나의 Aggregate만** 수정한다. 여러 Aggregate를 한 트랜잭션에서 같이 수정하고 싶어지면, 그건 Aggregate 경계를 잘못 나눴다는 신호일 수 있다. (연관된 다른 Aggregate는 ID로만 참조하고, 객체 참조로 직접 물고 있지 않는다.)

```typescript
export class Order {
  private readonly id: OrderId;
  private orderLines: OrderLine[]; // Aggregate 내부 요소
  private totalAmount: Money;

  // 외부는 이 메서드를 통해서만 orderLines를 변경 (Root를 통한 접근)
  addOrderLine(line: OrderLine): void {
    this.orderLines.push(line);
    this.totalAmount = this.calculateTotal(); // 불변식 재계산
  }

  private calculateTotal(): Money {
    return this.orderLines.reduce((sum, l) => sum.add(l.amount), Money.ZERO);
  }
}
```

```kotlin
class Order(val id: OrderId) {
    private val orderLines = mutableListOf<OrderLine>()
    var totalAmount: Money = Money.ZERO
        private set // setter를 외부에 노출하지 않음 → Root를 통한 접근만 허용

    fun addOrderLine(line: OrderLine) {
        orderLines.add(line)
        totalAmount = orderLines.fold(Money.ZERO) { acc, l -> acc + l.amount }
    }
}
```

### 2.4 Repository (레포지토리)

- 도메인 모델과 물리적 저장소(DB 등) 사이의 **컬렉션처럼 보이는 인터페이스**를 제공한다.
- 핵심: **저장/조회 단위는 Aggregate 전체**다. `OrderRepository`는 있지만 `OrderLineRepository`는 만들지 않는다(OrderLine은 Order를 통해서만 조회/저장).
- 인터페이스는 도메인 계층에 두고, 구현체(JPA, TypeORM 등)는 인프라 계층에 둔다(의존성 역전).

```typescript
// 도메인 계층
export interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;
}

// 인프라 계층 (NestJS + TypeORM 구현)
@Injectable()
export class TypeOrmOrderRepository implements OrderRepository {
  // ...구현
}
```

```kotlin
// 도메인 계층
interface OrderRepository {
    fun findById(id: OrderId): Order?
    fun save(order: Order)
}

// 인프라 계층 (Spring Data JPA 구현)
@Repository
class JpaOrderRepository(
    private val jpaRepository: OrderJpaRepository
) : OrderRepository {
    override fun findById(id: OrderId): Order? = jpaRepository.findById(id.value)
        .map { it.toDomain() }.orElse(null)
    override fun save(order: Order) { jpaRepository.save(OrderEntity.fromDomain(order)) }
}
```

### 2.5 Domain Service (도메인 서비스)

- **특정 Aggregate에 속하지 않는** 로직, 특히 여러 Aggregate에 걸친 계산/정책을 담당한다.
- 예: "여러 상품의 재고와 할인 정책을 조합해 최종 결제 금액을 계산" — `Order`나 `Product` 어느 한쪽 책임으로 보기 어려운 로직.

> **보완 포인트**: Domain Service는 "일단 로직을 넣을 곳이 마땅치 않아서" 만드는 만능 서비스가 되기 쉽다. Entity/Value에 넣을 수 있는 로직인데 습관적으로 Domain Service로 빼면 **빈약한 도메인 모델(Anemic Domain Model)** 안티패턴이 된다. Domain Service는 "정말로 여러 Aggregate/외부 정책을 조합해야 하는 경우"에만 최후 수단으로 사용한다.

```kotlin
// 여러 Aggregate(Order, DiscountPolicy)를 조합하는 도메인 서비스
class OrderPricingService(
    private val discountPolicyRepository: DiscountPolicyRepository
) {
    fun calculateFinalAmount(order: Order, member: Member): Money {
        val policy = discountPolicyRepository.findApplicablePolicy(member.grade)
        return policy.apply(order.totalAmount)
    }
}
```

### 2.6 Domain Event (도메인 이벤트)

- 도메인에서 의미 있는 일이 발생했음을 표현 (`OrderPlaced`, `OrderCancelled`, `PaymentCompleted`)
- 활용 목적:
    1. **Aggregate 간 결합도 낮추기**: `Order`가 직접 `Inventory`를 호출하는 대신, `OrderPlaced` 이벤트를 발행하고 재고 모듈이 구독한다.
    2. **트랜잭션 경계 분리**: 이벤트 발행 → 별도 트랜잭션/비동기 처리로 다른 Aggregate를 "결과적 일관성(eventual consistency)"으로 갱신.
    3. **사이드 이펙트의 명시화**: "주문 완료 시 알림 발송" 같은 부가 로직을 Order 엔티티 안에 넣지 않고 이벤트 구독자로 분리.

```typescript
export class OrderPlacedEvent {
  constructor(
    readonly orderId: OrderId,
    readonly occurredAt: Date = new Date()
  ) {}
}

export class Order {
  private domainEvents: OrderPlacedEvent[] = [];

  place(): void {
    this.orderStatus = OrderStatus.PLACED;
    this.domainEvents.push(new OrderPlacedEvent(this.id));
  }

  pullDomainEvents(): OrderPlacedEvent[] {
    const events = this.domainEvents;
    this.domainEvents = [];
    return events;
  }
}
```

```kotlin
data class OrderPlacedEvent(val orderId: OrderId, val occurredAt: Instant = Instant.now())

class Order(val id: OrderId) {
    private val domainEvents = mutableListOf<Any>()

    fun place() {
        status = OrderStatus.PLACED
        domainEvents.add(OrderPlacedEvent(id))
    }

    fun pullDomainEvents(): List<Any> {
        val events = domainEvents.toList()
        domainEvents.clear()
        return events
    }
}
// Spring: @DomainEvents / @AfterDomainEventPublication 어노테이션으로 자동 발행도 가능
```

---

## 3. Bounded Context (바운디드 컨텍스트)

- 전술 패턴이 **모델 내부**를 정리하는 도구라면, Bounded Context는 **모델 간 경계**를 정리하는 도구다.
- 같은 단어("주문", "상품")라도 컨텍스트마다 의미와 속성이 다를 수 있다.
    - 주문 컨텍스트의 `Product`: 가격, 옵션, 재고 등 판매 관점
    - 배송 컨텍스트의 `Product`: 무게, 부피, 포장 방식 등 물류 관점
    - 하나의 `Product` 모델로 통합하려 하면 오히려 두 컨텍스트 모두에 안 맞는 모델이 된다.
- 각 Bounded Context는 보통 하나의 **Ubiquitous Language(통용어)**를 가지며, 실무에서는 대체로 하나의 서비스/모듈 경계와 일치시킨다(마이크로서비스라면 서비스 단위, 모놀리식이라면 패키지/모듈 단위).

**보완 포인트 — Context 간 통합 전략(Context Mapping)**: Bounded Context는 나누는 것 자체보다, **경계 간 관계를 어떻게 정의할지**가 더 중요하다.

|관계 유형|설명|예시|
|---|---|---|
|Shared Kernel|두 컨텍스트가 일부 모델을 공유|공통 `Money`, `UserId` 타입 공유 라이브러리|
|Customer-Supplier|한쪽이 다른 쪽의 요구사항에 맞춰줌|결제 컨텍스트가 주문 컨텍스트 요구에 맞춰 API 제공|
|Conformist|상위 컨텍스트 모델을 그대로 따름|외부 PG사 API 모델을 그대로 수용|
|Anti-Corruption Layer(ACL)|외부 모델을 변환 계층으로 격리, 내부 모델 오염 방지|레거시 시스템 연동 시 어댑터로 변환|

```typescript
// Anti-Corruption Layer 예시: 외부 PG사 응답을 내부 도메인 모델로 변환
export class PaymentGatewayAdapter {
  constructor(private readonly externalClient: ExternalPgClient) {}

  async requestPayment(order: Order): Promise<PaymentResult> {
    const externalResponse = await this.externalClient.pay({
      merchantOrderId: order.id.value,
      amount: order.totalAmount.amount,
    });
    // 외부 응답 포맷 → 내부 도메인 모델로 변환 (오염 차단)
    return PaymentResult.fromExternal(externalResponse);
  }
}
```

---

## 4. 정리: Aggregate 설계 체크리스트

실제로 Aggregate를 나눌 때 아래 질문으로 점검하면 도움이 된다.

1. **이 경계 안의 불변식은 무엇인가?** (항상 참이어야 하는 규칙을 먼저 정의)
2. **Root는 무엇이고, 외부는 오직 Root를 통해서만 접근하는가?**
3. **한 트랜잭션에서 이 Aggregate 하나만 수정하는가?** (다른 Aggregate 수정이 필요하면 Domain Event로 분리 검토)
4. **다른 Aggregate를 참조할 때 객체 참조가 아니라 ID로 참조하는가?** (참조 그래프가 커지는 것 방지)
5. **Aggregate가 너무 커서 동시성 충돌(낙관적 락 실패)이 잦지 않은가?** → 너무 크면 쪼개는 것을 고려

> Aggregate는 "일관성이 필요한 최소 단위"로 최대한 작게 설계하는 것이 원칙이다. 크게 만들수록 안전해 보이지만, 실제로는 동시성 문제와 성능 저하의 원인이 된다.
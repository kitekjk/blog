# DDD (Domain-Driven Design) 학습 가이드

> 물류 이커머스 도메인 기준 · 신입 AI Native Engineer 대상

---

## 목차

1. [DDD란 무엇인가](#1-ddd란-무엇인가)
2. [Strategic Design (전략적 설계)](#2-strategic-design-전략적-설계)
3. [전략에서 전술로 — 설계의 연결](#3-전략에서-전술로--설계의-연결)
4. [Tactical Design (전술적 설계)](#4-tactical-design-전술적-설계)
5. [Aggregate 설계 원칙](#5-aggregate-설계-원칙)
6. [Domain Event 활용](#6-domain-event-활용)
7. [실전 예제: 배송비 계산 도메인 모델링](#7-실전-예제-배송비-계산-도메인-모델링)
8. [흔한 실수와 안티패턴](#8-흔한-실수와-안티패턴)
9. [AI와 DDD](#9-ai와-ddd)
10. [참고 자료](#10-참고-자료)

---

## 용어 매핑 — DDD ↔ Hexagonal Architecture

이 문서와 [Hexagonal Architecture 학습 가이드](./hexagonal-architecture-learning-guide.md)는 같은 물류 도메인을 다룬다. 두 문서에서 사용하는 용어의 대응 관계를 먼저 정리한다.

| DDD 용어 | Hexagonal 용어 | 설명 |
|----------|---------------|------|
| Repository 인터페이스 | **Outbound Port** | Aggregate 저장/조회를 추상화하는 계약 |
| Repository 구현체 | **Driven Adapter** | JPA, MongoDB 등 실제 기술 구현 |
| Application Service | **UseCase (Inbound Port 구현)** | 유즈케이스 오케스트레이션 |
| Domain Service | (Domain 계층에 위치) | 특정 Entity에 속하지 않는 도메인 로직 |
| Domain Event Publisher | **Outbound Port** | 이벤트 발행을 추상화 |
| Anti-Corruption Layer | **Adapter** | 외부 모델을 내부 모델로 변환 |

---

## 1. DDD란 무엇인가

### 정의

DDD(Domain-Driven Design)는 **비즈니스 도메인의 복잡성을 소프트웨어 설계의 중심에 놓는 접근법**이다. Eric Evans가 2003년 동명의 책에서 제안했다.

핵심 철학: **코드가 비즈니스를 그대로 반영해야 한다.**

### 왜 필요한가

물류 이커머스를 예로 들면:

- "주문"이라는 단어가 CS팀, 물류팀, 정산팀에서 각각 다른 의미로 쓰인다
- 배송비 계산 규칙이 수십 가지이고, 비즈니스 담당자도 헷갈린다
- 신규 기능 추가할 때마다 예상치 못한 곳이 깨진다

DDD는 이런 **복잡한 비즈니스 로직을 체계적으로 다루는 방법론**이다.

### 전통 개발방식 vs DDD

| 구분 | 전통 방식 (데이터 중심) | DDD (도메인 중심) |
|------|------------------------|-------------------|
| 설계 출발점 | DB 테이블 설계 | 비즈니스 도메인 모델 |
| 비즈니스 로직 위치 | Service 클래스에 절차적으로 | Domain 객체 안에 |
| 팀 소통 언어 | 개발자끼리 기술 용어 | 비즈니스 전문가와 공통 용어 |
| 모델 변경 비용 | 테이블 변경 = 대공사 | 도메인 모델 중심이라 유연 |
| 적합한 상황 | 단순 CRUD 앱 | 복잡한 비즈니스 규칙이 있는 시스템 |

> 💡 **신입 팁**: 모든 프로젝트에 DDD를 쓸 필요는 없다. 비즈니스 로직이 단순한 CRUD라면 오버엔지니어링이다. 물류처럼 규칙이 복잡한 도메인에서 빛난다.

---

## 2. Strategic Design (전략적 설계)

전략적 설계는 **큰 그림을 그리는 단계**다. 코드를 쓰기 전에 먼저 해야 한다.

### 2.1 Ubiquitous Language (보편 언어)

개발팀과 비즈니스팀이 **같은 단어를 같은 의미로** 사용하는 것.

```
❌ 나쁜 예
- 개발자: "OrderEntity의 status 필드를 UPDATE 해서..."
- 기획자: "주문 상태를 '배송중'으로 바꿔주세요"
→ 서로 다른 언어를 쓰고 있다

✅ 좋은 예
- 모두: "주문을 출고 처리하면, 주문 상태가 '배송중'으로 변경된다"
→ 코드에서도 order.dispatch(), OrderStatus.IN_DELIVERY
```

### 2.2 Bounded Context (바운디드 컨텍스트)

**같은 단어가 다른 의미를 갖는 경계를 명확히 나누는 것.**

물류 이커머스에서 "상품"이라는 단어:

| 컨텍스트 | "상품"의 의미 | 관심사 |
|---------|-------------|--------|
| **상품 카탈로그** | 판매 상품 정보 (이름, 설명, 이미지) | 노출, 검색 |
| **재고** | 물리적 재고 단위 (SKU, 수량, 위치) | 입출고, 재고 수량 |
| **주문** | 주문 항목 (가격, 수량, 할인) | 결제, 취소 |
| **배송** | 배송해야 할 물건 (무게, 부피, 포장) | 물류, 운송 |
| **정산** | 정산 대상 (판매가, 수수료, 정산액) | 셀러 정산 |

#### 물류 도메인 Context Map

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   주문 (Order)  │────▶│  배송 (Shipping) │────▶│  정산 (Settlement) │
│             │     │             │     │             │
│ - 주문 생성   │     │ - 배송 생성    │     │ - 정산 생성    │
│ - 결제 처리   │     │ - 택배사 연동  │     │ - 수수료 계산  │
│ - 주문 취소   │     │ - 배송 추적    │     │ - 셀러 지급    │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   ▲
       ▼                   │
┌─────────────┐     ┌─────────────┐
│  재고 (Inventory) │     │  상품 (Catalog)  │
│             │     │             │
│ - 재고 차감   │     │ - 상품 등록    │
│ - 입고 처리   │     │ - 가격 관리    │
│ - 재고 조회   │     │ - 카테고리     │
└─────────────┘     └─────────────┘
```

### 2.3 Context Map — 컨텍스트 간 관계

컨텍스트 간 관계를 정의하는 패턴:

| 패턴 | 설명 | 물류 예시 |
|------|------|----------|
| **Upstream/Downstream** | 한쪽이 제공, 한쪽이 소비 | 주문(Upstream) → 배송(Downstream) |
| **Anti-Corruption Layer** | 외부 모델 오염 방지 번역 계층 | 택배사 API 응답을 내부 모델로 변환 |
| **Published Language** | 공식 교환 형식 | 주문 이벤트 JSON 스키마 |
| **Shared Kernel** | 두 컨텍스트가 공유하는 모델 | Money, Address 같은 공통 VO |

> 💡 **조직과 컨텍스트**: 실무에서 Bounded Context는 **팀 소유권**과 밀접하다. 주문 컨텍스트는 주문팀, 배송 컨텍스트는 물류팀이 소유한다. 컨텍스트 경계를 정할 때 "이 영역을 한 팀이 독립적으로 개발/배포할 수 있는가?"를 기준으로 삼으면 좋다. (참고: Conway의 법칙)

### 2.4 Event Storming — Bounded Context를 도출하는 실전 기법

**Event Storming**은 도메인 전문가와 개발자가 함께 모여 **비즈니스 이벤트를 중심으로 도메인을 탐색하는 워크숍 기법**이다. Alberto Brandolini가 고안했다.

#### 왜 필요한가

- Bounded Context를 "감"으로 나누면 나중에 경계가 어긋난다
- 비즈니스 전문가가 직접 참여해야 진짜 Ubiquitous Language가 나온다
- **오렌지색 포스트잇으로 시작해서**, 자연스럽게 도메인 모델이 도출된다

#### 진행 방법 (간략 버전)

```
1. 준비: 넓은 벽, 포스트잇(색상별), 마커, 도메인 전문가 + 개발자
2. 도메인 이벤트 나열 (🟧 오렌지)
   → "주문이 생성됨", "결제가 완료됨", "배송이 출고됨"
   → 과거 시제로 작성!
3. 시간순 정렬 (왼→오)
4. 커맨드 추가 (🟦 파랑)
   → "주문 생성하기", "결제 처리하기"
5. Aggregate 식별 (🟨 노랑)
   → 커맨드를 받아 이벤트를 발생시키는 주체: Order, Payment, Shipment
6. Bounded Context 경계 긋기
   → 관련 이벤트/커맨드/Aggregate를 그룹핑하면 자연스럽게 경계가 보인다
```

#### 물류 도메인 Event Storming 결과 예시

```
시간 →

🟦 주문 생성    🟦 결제 요청    🟦 배송 생성    🟦 출고 처리     🟦 배송 완료
     ↓              ↓             ↓              ↓              ↓
🟨 Order       🟨 Payment    🟨 Shipment   🟨 Shipment    🟨 Shipment
     ↓              ↓             ↓              ↓              ↓
🟧 주문생성됨   🟧 결제완료됨  🟧 배송생성됨  🟧 출고됨       🟧 배송완료됨

├── 주문 Context ──┤├─ 결제 Context ─┤├────── 배송 Context ──────────┤
```

> 💡 **신입 팁**: Event Storming은 2~3시간이면 충분하다. 처음엔 "주문이 생성됨" 같은 이벤트를 마구 쏟아내고, 나중에 정렬하면서 정리하는 방식이다. 완벽하게 하려고 하지 마라. **빠르게 여러 번** 반복하는 것이 핵심이다.

---

## 3. 전략에서 전술로 — 설계의 연결

전략적 설계에서 Bounded Context와 이벤트 흐름이 정해졌다면, 이제 **각 컨텍스트 내부를 설계**할 차례다.

### 3.1 연결 과정

```
Event Storming 결과          Tactical Design

🟨 Aggregate: Order    →    Order (Aggregate Root)
                             ├── OrderItem (Entity)
                             ├── Money (Value Object)
                             └── Address (Value Object)

🟧 이벤트: 주문생성됨   →    OrderPlaced (Domain Event)

🟦 커맨드: 주문 생성    →    PlaceOrderUseCase (Application Service)
```

### 3.2 Application Service vs Domain Service

신입이 가장 헷갈리는 구분 중 하나:

| 구분 | Application Service | Domain Service |
|------|-------------------|----------------|
| **역할** | 유즈케이스 오케스트레이션 (흐름 조율) | 특정 Entity에 속하지 않는 도메인 로직 |
| **예시** | `PlaceOrderUseCase` — 주문 생성 → 저장 → 이벤트 발행 | `ShippingFeeCalculator` — 배송비 계산 규칙 |
| **외부 의존** | Repository, EventPublisher 등 Port 사용 | 없음 (순수 도메인 로직) |
| **Hex에서 위치** | Inbound Port 구현 (UseCase) | Domain 계층 |
| **비유** | 식당 매니저 (주문 접수 → 주방 전달 → 서빙) | 셰프 (실제 요리) |

> 💡 **판별 기준**: "이 로직이 DB나 외부 시스템과 무관하게, 순수하게 비즈니스 규칙인가?" → Yes면 Domain Service. "여러 단계를 조율하는 흐름인가?" → Yes면 Application Service.

---

## 4. Tactical Design (전술적 설계)

전술적 설계는 **각 Bounded Context 내부의 코드 구조**를 다룬다.

### 4.1 Entity (엔티티)

**고유 식별자(ID)를 가지며, 생명주기 동안 상태가 변하는 객체.**

```kotlin
// 배송 엔티티 — ID로 식별되고, 상태가 변한다
class Shipment(
    val id: ShipmentId,
    val orderId: OrderId,
    val recipient: Recipient,
    private var status: ShipmentStatus = ShipmentStatus.CREATED,
    private var trackingNumber: TrackingNumber? = null,
) {
    fun dispatch(trackingNumber: TrackingNumber) {
        require(status == ShipmentStatus.CREATED) {
            "이미 출고된 배송입니다: $id"
        }
        this.status = ShipmentStatus.DISPATCHED
        this.trackingNumber = trackingNumber
    }

    fun complete() {
        require(status == ShipmentStatus.DISPATCHED) {
            "출고 상태에서만 배송 완료 처리 가능합니다: $id"
        }
        this.status = ShipmentStatus.DELIVERED
    }

    // 두 Shipment가 같은지는 ID로만 판단
    override fun equals(other: Any?): Boolean =
        other is Shipment && id == other.id

    override fun hashCode(): Int = id.hashCode()
}
```

### 4.2 Value Object (값 객체)

**식별자 없이, 값 자체로 동등성을 판단하는 불변 객체.**

```kotlin
// 주소 — 값이 같으면 같은 것. 불변.
data class Address(
    val zipCode: String,
    val city: String,
    val street: String,
    val detail: String,
) {
    init {
        require(zipCode.matches(Regex("\\d{5}"))) {
            "우편번호는 5자리 숫자여야 합니다: $zipCode"
        }
    }

    fun isRemoteArea(): Boolean {
        // 도서산간 지역 판별
        return REMOTE_AREA_ZIP_PREFIXES.any { zipCode.startsWith(it) }
    }

    companion object {
        private val REMOTE_AREA_ZIP_PREFIXES = listOf("63", "69") // 제주, 도서산간
    }
}

// 금액 — 통화와 함께 관리
data class Money(
    val amount: Long,
    val currency: Currency = Currency.KRW,
) {
    init {
        require(amount >= 0) { "금액은 0 이상이어야 합니다: $amount" }
    }

    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "통화가 다릅니다" }
        return Money(amount + other.amount, currency)
    }

    operator fun minus(other: Money): Money {
        require(currency == other.currency) { "통화가 다릅니다" }
        return Money(amount - other.amount, currency)
    }

    fun isZero(): Boolean = amount == 0L
}

enum class Currency { KRW, USD }

// 무게
data class Weight(val grams: Int) {
    init { require(grams > 0) { "무게는 0보다 커야 합니다" } }

    fun toKg(): Double = grams / 1000.0
    fun isHeavy(): Boolean = grams > 30_000 // 30kg 초과
}
```

> 💡 **Kotlin 문법 팁 — `data class`**: `data class`는 `equals()`, `hashCode()`, `toString()`, `copy()`를 자동 생성해준다. Value Object에 딱 맞는 문법이다. 반면 Entity는 ID로만 동등성을 판단해야 하므로 `data class`를 쓰지 않고 직접 `equals()`를 구현한다.

### 4.3 Aggregate & Aggregate Root

**Aggregate는 하나의 트랜잭션 단위로 일관성을 보장하는 객체 묶음.** Aggregate Root가 유일한 진입점이다.

```kotlin
// Order가 Aggregate Root
// OrderItem은 Order를 통해서만 접근/변경 가능
class Order private constructor(
    val id: OrderId,
    val buyerId: BuyerId,
    private val _items: MutableList<OrderItem>,
    private var status: OrderStatus = OrderStatus.PLACED,
    private var shippingAddress: Address,
) {
    val items: List<OrderItem> get() = _items.toList()

    // ──────────────────────────────────────────────────────
    // 도메인 이벤트 수집 패턴
    // Aggregate 내부에서 이벤트를 리스트에 모아두고,
    // Application Service가 저장 후 한꺼번에 발행한다.
    // 이렇게 하면 "저장 성공 → 이벤트 발행" 순서를 보장할 수 있다.
    // ──────────────────────────────────────────────────────
    private val _events = mutableListOf<DomainEvent>()
    val domainEvents: List<DomainEvent> get() = _events.toList()

    companion object {
        // 팩토리 메서드: 새로운 주문 생성 시에만 이벤트 발생
        fun place(
            id: OrderId,
            buyerId: BuyerId,
            items: List<OrderItem>,
            shippingAddress: Address,
        ): Order {
            require(items.isNotEmpty()) { "주문 항목이 비어있습니다" }

            val order = Order(id, buyerId, items.toMutableList(), shippingAddress = shippingAddress)
            order._events.add(
                OrderPlaced(
                    orderId = id,
                    buyerId = buyerId,
                    items = items,
                    shippingAddress = shippingAddress,
                    occurredAt = Instant.now(),
                )
            )
            return order
        }

        // DB에서 복원할 때 사용하는 팩토리 — 이벤트를 발생시키지 않는다!
        // (Hexagonal 가이드의 Repository Adapter에서 사용)
        fun reconstitute(
            id: OrderId,
            buyerId: BuyerId,
            items: List<OrderItem>,
            status: OrderStatus,
            shippingAddress: Address,
        ): Order {
            return Order(id, buyerId, items.toMutableList(), status, shippingAddress)
        }
    }

    fun cancel(reason: String) {
        require(status == OrderStatus.PLACED) {
            "결제 완료 전 주문만 취소 가능합니다"
        }
        status = OrderStatus.CANCELLED
        _events.add(OrderCancelled(id, reason, Instant.now()))
    }

    fun totalPrice(): Money =
        _items.fold(Money(0)) { acc, item -> acc + item.totalPrice() }

    fun clearEvents() = _events.clear()
}

// OrderItem은 독립적으로 존재하지 않음 — Order에 종속
data class OrderItem(
    val productId: ProductId,
    val productName: String,
    val unitPrice: Money,
    val quantity: Int,
) {
    init { require(quantity > 0) { "수량은 1 이상이어야 합니다" } }
    fun totalPrice(): Money = Money(unitPrice.amount * quantity)
}
```

> 💡 **`reconstitute` vs `place`**: `place()`는 새로운 주문 생성 시 이벤트를 발행한다. `reconstitute()`는 DB에서 기존 데이터를 복원할 때 사용하며, **이벤트를 생성하지 않는다.** 이 구분이 없으면 DB에서 조회할 때마다 이벤트가 중복 생성되는 버그가 발생한다. [Hexagonal 가이드 4.7절](./hexagonal-architecture-learning-guide.md#47-adapter--persistence-jpa)에서 실제 사용 예시를 확인하자.

### 4.4 Domain Event (도메인 이벤트)

**도메인에서 발생한 의미 있는 사건.**

```kotlin
// 마커 인터페이스
interface DomainEvent {
    val occurredAt: Instant
}

// 주문 생성됨
data class OrderPlaced(
    val orderId: OrderId,
    val buyerId: BuyerId,
    val items: List<OrderItem>,
    val shippingAddress: Address,
    override val occurredAt: Instant,
) : DomainEvent

// 주문 취소됨
data class OrderCancelled(
    val orderId: OrderId,
    val reason: String,
    override val occurredAt: Instant,
) : DomainEvent

// 배송 생성됨
data class ShipmentCreated(
    val shipmentId: ShipmentId,
    val orderId: OrderId,
    override val occurredAt: Instant,
) : DomainEvent
```

### 4.5 Domain Service (도메인 서비스)

**특정 Entity에 속하지 않는 도메인 로직.** (Application Service와의 차이는 [3.2절](#32-application-service-vs-domain-service) 참조)

```kotlin
// 배송비 계산 — 어떤 Entity에도 자연스럽게 속하지 않는 로직
// 순수 비즈니스 규칙만 담고 있다. DB, 외부 API 호출 없음.
class ShippingCostCalculator {

    fun calculate(
        items: List<OrderItem>,
        address: Address,
        totalOrderPrice: Money,
    ): Money {
        // 무료배송 기준
        if (totalOrderPrice.amount >= 50_000) {
            return Money(0)
        }

        val baseCost = Money(3_000)

        // 도서산간 추가 요금
        val remoteSurcharge = if (address.isRemoteArea()) Money(3_000) else Money(0)

        return baseCost + remoteSurcharge
    }
}
```

### 4.6 Repository (리포지토리)

**Aggregate의 저장/조회를 추상화하는 인터페이스.** 도메인 계층에 인터페이스를, 인프라 계층에 구현을 둔다.

Hexagonal Architecture에서 Repository 인터페이스는 **Outbound Port**에 해당한다.

```kotlin
// 도메인 계층 — 인터페이스만 정의 (= Outbound Port)
interface OrderRepository {
    fun findById(id: OrderId): Order?
    fun save(order: Order)
    fun nextId(): OrderId
}

// 인프라 계층 — JPA 구현 (= Driven Adapter)
// → Hexagonal Architecture 가이드 4.7절에서 자세히 다룸
```

---

## 5. Aggregate 설계 원칙

### 원칙 1: 트랜잭션 경계 = Aggregate 경계

하나의 트랜잭션에서 **하나의 Aggregate만** 수정한다.

```
✅ 맞음: 주문 생성 트랜잭션에서 Order Aggregate만 저장
✅ 맞음: 재고 차감은 별도 트랜잭션 (이벤트로 처리)

❌ 틀림: 하나의 트랜잭션에서 Order + Inventory + Shipment 모두 수정
```

> 💡 **최종 일관성(Eventual Consistency)**: Aggregate 간에는 즉시 일관성 대신 **최종 일관성**을 허용한다. 주문이 생성되면 이벤트를 통해 재고가 차감되고, 배송이 생성된다. 모두 같은 트랜잭션에 넣지 않아도 결과적으로 데이터가 일관성을 가지게 된다. 이것이 이벤트 기반 아키텍처의 핵심 아이디어다.

### 원칙 2: 불변식(Invariant) 보호

Aggregate는 항상 유효한 상태를 유지해야 한다.

```kotlin
class Order {
    fun addItem(item: OrderItem) {
        require(_items.size < MAX_ITEMS) {
            "주문 항목은 최대 ${MAX_ITEMS}개까지 가능합니다"
        }
        require(!_items.any { it.productId == item.productId }) {
            "이미 같은 상품이 담겨있습니다"
        }
        _items.add(item)
    }

    companion object {
        private const val MAX_ITEMS = 50
    }
}
```

### 원칙 3: 작게 유지하기

**Aggregate는 가능한 작게 설계한다.**

#### ❌ 잘못된 Aggregate — God Aggregate

```kotlin
// 나쁜 예: 주문이 모든 것을 품고 있다
class Order(
    val id: OrderId,
    val items: MutableList<OrderItem>,
    val payment: Payment,           // ← 결제는 별도 Aggregate여야
    val shipment: Shipment,         // ← 배송은 별도 Aggregate여야
    val invoice: Invoice,           // ← 정산도 별도 Aggregate여야
    val reviews: MutableList<Review>, // ← 리뷰까지?!
) {
    // 주문 변경할 때마다 결제, 배송, 정산, 리뷰까지 다 로딩해야 함
    // 동시성 충돌도 자주 발생
}
```

#### ✅ 올바른 Aggregate — 작고 독립적

```kotlin
// 주문 Aggregate — 주문 자체만 관심
class Order(
    val id: OrderId,
    val buyerId: BuyerId,
    private val _items: MutableList<OrderItem>,
    private var status: OrderStatus,
    private var shippingAddress: Address,
)

// 배송 Aggregate — orderId로 참조만
class Shipment(
    val id: ShipmentId,
    val orderId: OrderId,  // ← ID로만 참조! 객체 참조 아님
    val recipient: Recipient,
    private var status: ShipmentStatus,
)

// 결제 Aggregate — 마찬가지로 ID 참조
class Payment(
    val id: PaymentId,
    val orderId: OrderId,  // ← ID로만 참조
    val amount: Money,
    private var status: PaymentStatus,
)
```

> 💡 **핵심**: Aggregate 간에는 **ID로만 참조**한다. 객체 참조를 하면 경계가 무너진다.

### Aggregate 설계 리뷰 체크리스트

코드 리뷰 시 이 질문들을 확인하자:

- [ ] 하나의 트랜잭션에서 하나의 Aggregate만 수정하는가?
- [ ] Aggregate 내부 상태가 외부에서 직접 변경 불가능한가? (`private set`)
- [ ] 다른 Aggregate를 객체 참조하지 않고 ID로만 참조하는가?
- [ ] Aggregate가 항상 유효한 상태를 유지하는가? (`require` 블록)
- [ ] 이벤트가 Aggregate 내부에서만 생성되는가?

---

## 6. Domain Event 활용

### 이벤트 기반 컨텍스트 간 통신

Bounded Context 간에 직접 호출하면 강결합이 된다. **이벤트를 통해 느슨하게 연결**한다.

### OrderPlaced → ShipmentCreated 전체 흐름

```
주문 컨텍스트                          배송 컨텍스트
┌──────────────┐                    ┌──────────────┐
│ Order.place() │                    │              │
│   ↓           │   OrderPlaced     │              │
│ OrderPlaced   │──── 이벤트 발행 ──→│ 이벤트 수신    │
│ 이벤트 생성    │                    │   ↓           │
│              │                    │ Shipment 생성  │
└──────────────┘                    │ ShipmentCreated│
                                    │ 이벤트 생성    │
                                    └──────────────┘
                                           │
                                    재고 컨텍스트
                                    ┌──────────────┐
                                    │ 재고 차감      │
                                    └──────────────┘
```

### 이벤트 수집-발행 패턴 상세

Aggregate 내부에서 이벤트가 어떻게 수집되고 발행되는지 단계별로 살펴보자:

```
1. Aggregate 메서드 호출 (order.cancel())
   → _events 리스트에 OrderCancelled 추가 (아직 발행 안 됨!)

2. Application Service가 Repository로 저장
   → orderRepository.save(order) — DB에 상태 저장

3. Application Service가 이벤트 발행
   → order.domainEvents.forEach { eventPublisher.publish(it) }

4. 이벤트 리스트 초기화
   → order.clearEvents()
```

**왜 이렇게 하나?** 만약 이벤트를 즉시 발행하면, DB 저장이 실패해도 이벤트는 이미 나간 상태가 된다. 수집 후 저장 성공 후 발행하면 이 문제를 방지할 수 있다.

#### 코드 예시

```kotlin
// 1. 주문 UseCase (Application Service) — 주문 생성 후 이벤트 발행
class PlaceOrderUseCase(
    private val orderRepository: OrderRepository,
    private val eventPublisher: DomainEventPublisher,
) {
    fun execute(command: PlaceOrderCommand): OrderId {
        val order = Order.place(
            id = orderRepository.nextId(),
            buyerId = command.buyerId,
            items = command.items,
            shippingAddress = command.shippingAddress,
        )

        orderRepository.save(order)

        // Aggregate에서 수집된 이벤트를 발행
        order.domainEvents.forEach { eventPublisher.publish(it) }
        order.clearEvents()

        return order.id
    }
}

// 2. 배송 컨텍스트의 이벤트 핸들러
@Component
class OrderPlacedEventHandler(
    private val createShipmentUseCase: CreateShipmentUseCase,
) {
    @EventListener // 또는 @KafkaListener 등
    fun handle(event: OrderPlaced) {
        createShipmentUseCase.execute(
            CreateShipmentCommand(
                orderId = event.orderId,
                recipientAddress = event.shippingAddress,
                items = event.items,
            )
        )
    }
}

// 3. 이벤트 발행 인터페이스 (= Outbound Port)
interface DomainEventPublisher {
    fun publish(event: DomainEvent)
}

// 4. Spring ApplicationEvent 기반 구현 (= Driven Adapter)
@Component
class SpringDomainEventPublisher(
    private val applicationEventPublisher: ApplicationEventPublisher,
) : DomainEventPublisher {
    override fun publish(event: DomainEvent) {
        applicationEventPublisher.publishEvent(event)
    }
}
```

### 동기 vs 비동기 이벤트 발행

| 방식 | 구현 | 장점 | 단점 | 사용 시점 |
|------|------|------|------|----------|
| **동기 (In-Process)** | Spring `@EventListener` | 간단, 같은 트랜잭션 가능 | 강결합, 성능 병목 | 같은 서비스 내 컨텍스트 간 |
| **비동기 (메시지 브로커)** | Kafka, RabbitMQ | 느슨한 결합, 확장성 | 복잡성 증가, 이벤트 유실 가능 | 서비스 간, MSA |
| **Transactional Outbox** | DB 테이블 + 폴링/CDC | 이벤트 유실 방지 | 추가 인프라 | 신뢰성이 중요한 경우 |

> 💡 **실전 팁**: 처음에는 Spring `@EventListener` + `@TransactionalEventListener`로 시작하고, 서비스가 분리될 때 Kafka 등 메시지 브로커로 전환하는 것이 현실적이다.

### 멱등성(Idempotency)과 재시도

비동기 환경에서는 **같은 이벤트가 두 번 이상 도착**할 수 있다. 핸들러는 멱등하게 만들어야 한다:

```kotlin
@Component
class OrderPlacedEventHandler(
    private val createShipmentUseCase: CreateShipmentUseCase,
    private val shipmentRepository: ShipmentRepository,
) {
    @EventListener
    fun handle(event: OrderPlaced) {
        // 멱등성 체크: 이미 해당 주문의 배송이 생성되었는지 확인
        if (shipmentRepository.existsByOrderId(event.orderId)) {
            return // 이미 처리됨 — 무시
        }

        createShipmentUseCase.execute(
            CreateShipmentCommand(orderId = event.orderId, /* ... */)
        )
    }
}
```

---

## 7. 실전 예제: 배송비 계산 도메인 모델링

### 요구사항

물류 이커머스의 배송비 계산 규칙:

1. **기본 배송비**: 3,000원
2. **무료배송**: 주문 금액 5만원 이상
3. **도서산간 추가**: 제주(63xxx), 도서산간(69xxx) 지역은 +3,000원
4. **무거운 상품**: 30kg 초과 시 +5,000원
5. **묶음배송**: 같은 셀러 상품은 묶어서 배송, 배송비 1회만
6. **셀러별 무료배송 기준**: 셀러가 설정한 무료배송 금액 기준 적용

### 도메인 모델 설계

```kotlin
// === Value Objects ===

data class ShippingFee(
    val baseFee: Money,
    val remoteSurcharge: Money,
    val weightSurcharge: Money,
) {
    val total: Money get() = baseFee + remoteSurcharge + weightSurcharge

    companion object {
        fun free() = ShippingFee(Money(0), Money(0), Money(0))
    }
}

data class ShippingPolicy(
    val sellerId: SellerId,
    val freeShippingThreshold: Money, // 셀러별 무료배송 기준액
    val baseShippingFee: Money,       // 셀러별 기본 배송비
)

// === Domain Service ===

class ShippingFeeCalculator {

    fun calculate(
        items: List<ShippingItem>,
        destination: Address,
        sellerPolicy: ShippingPolicy,
    ): ShippingFee {
        val totalPrice = items.fold(Money(0)) { acc, item ->
            acc + item.totalPrice()
        }

        // 규칙 1: 셀러 무료배송 기준 충족
        if (totalPrice.amount >= sellerPolicy.freeShippingThreshold.amount) {
            // 도서산간은 무료배송이어도 추가 요금 발생
            return ShippingFee(
                baseFee = Money(0),
                remoteSurcharge = calculateRemoteSurcharge(destination),
                weightSurcharge = Money(0),
            )
        }

        // 규칙 2: 기본 배송비 + 추가 요금
        return ShippingFee(
            baseFee = sellerPolicy.baseShippingFee,
            remoteSurcharge = calculateRemoteSurcharge(destination),
            weightSurcharge = calculateWeightSurcharge(items),
        )
    }

    private fun calculateRemoteSurcharge(address: Address): Money =
        if (address.isRemoteArea()) Money(3_000) else Money(0)

    private fun calculateWeightSurcharge(items: List<ShippingItem>): Money {
        val totalGrams = items.sumOf { it.weight.grams * it.quantity }
        return if (totalGrams > 30_000) Money(5_000) else Money(0)
    }
}

data class ShippingItem(
    val productId: ProductId,
    val sellerId: SellerId,
    val unitPrice: Money,
    val quantity: Int,
    val weight: Weight,
) {
    fun totalPrice(): Money = Money(unitPrice.amount * quantity)
}

// === 묶음배송 처리: 셀러별로 그룹핑 ===

class OrderShippingFeeCalculator(
    private val calculator: ShippingFeeCalculator,
    private val policyRepository: ShippingPolicyRepository,
) {
    fun calculateTotal(
        items: List<ShippingItem>,
        destination: Address,
    ): Money {
        // 셀러별로 그룹핑 → 각 그룹의 배송비 계산 → 합산
        return items
            .groupBy { it.sellerId }
            .map { (sellerId, sellerItems) ->
                val policy = policyRepository.findBySellerId(sellerId)
                    ?: throw IllegalStateException(
                        "셀러 $sellerId 의 배송 정책이 등록되어 있지 않습니다"
                    )
                calculator.calculate(sellerItems, destination, policy)
            }
            .fold(Money(0)) { acc, fee -> acc + fee.total }
    }
}

interface ShippingPolicyRepository {
    fun findBySellerId(sellerId: SellerId): ShippingPolicy?
}
```

> 💡 **실무 팁**: 기본 배송비를 코드에 하드코딩하지 않는다. `ShippingPolicyRepository`에서 셀러별로 관리하고, 정책이 없는 셀러는 예외를 던져 빠르게 발견한다.

### 테스트

```kotlin
class ShippingFeeCalculatorTest {

    private val calculator = ShippingFeeCalculator()
    private val defaultPolicy = ShippingPolicy(
        sellerId = SellerId("seller-1"),
        freeShippingThreshold = Money(50_000),
        baseShippingFee = Money(3_000),
    )

    @Test
    fun `5만원 이상이면 기본 배송비 무료`() {
        val items = listOf(
            ShippingItem(ProductId("p1"), SellerId("seller-1"), Money(60_000), 1, Weight(500))
        )
        val address = Address("06000", "서울", "강남대로", "1층")

        val fee = calculator.calculate(items, address, defaultPolicy)

        assertEquals(Money(0), fee.total)
    }

    @Test
    fun `도서산간은 무료배송이어도 추가요금 발생`() {
        val items = listOf(
            ShippingItem(ProductId("p1"), SellerId("seller-1"), Money(60_000), 1, Weight(500))
        )
        val jejuAddress = Address("63000", "제주", "제주로", "1층")

        val fee = calculator.calculate(items, jejuAddress, defaultPolicy)

        assertEquals(Money(3_000), fee.total) // 도서산간 추가만 발생
    }

    @Test
    fun `30kg 초과시 무게 추가요금`() {
        val items = listOf(
            ShippingItem(ProductId("p1"), SellerId("seller-1"), Money(10_000), 1, Weight(35_000))
        )
        val address = Address("06000", "서울", "강남대로", "1층")

        val fee = calculator.calculate(items, address, defaultPolicy)

        assertEquals(Money(8_000), fee.total) // 기본 3000 + 무게 5000
    }
}
```

> 💡 **Domain 테스트에는 DB도, Spring도, Mock도 필요 없다!** 순수 Kotlin 객체이기 때문이다. 테스트 전략에 대한 자세한 내용은 [Hexagonal Architecture 가이드 5장](./hexagonal-architecture-learning-guide.md#5-테스트-전략)을 참조하자.

---

## 8. 흔한 실수와 안티패턴

### 8.1 빈약한 도메인 모델 (Anemic Domain Model)

**Entity가 getter/setter만 있고, 비즈니스 로직은 전부 Service에 있는 패턴.**

```kotlin
// ❌ 빈약한 도메인 모델 — Entity가 데이터 주머니
data class Order(
    var id: String? = null,
    var status: String = "PLACED",
    var items: MutableList<OrderItem> = mutableListOf(),
    var totalPrice: Long = 0,
)

// 로직이 전부 Service에...
class OrderService {
    fun cancelOrder(order: Order) {
        if (order.status != "PLACED") {
            throw IllegalStateException("취소 불가")
        }
        order.status = "CANCELLED"
        // 불변식 검증도 Service가 해야 하고...
        // 다른 Service에서도 order.status를 직접 바꿀 수 있고...
    }
}
```

```kotlin
// ✅ 풍부한 도메인 모델 — Entity가 자신의 규칙을 보호
class Order(
    val id: OrderId,
    private var status: OrderStatus = OrderStatus.PLACED,
) {
    fun cancel(reason: String) {
        require(status == OrderStatus.PLACED) {
            "결제 완료 전 주문만 취소 가능합니다"
        }
        status = OrderStatus.CANCELLED
        // 이 로직은 Order 안에만 존재 → 일관성 보장
    }
}
```

### 8.2 God Aggregate

**Aggregate 하나가 너무 많은 것을 포함하는 패턴.** → [5장 Aggregate 설계 원칙](#원칙-3-작게-유지하기) 참조

### 8.3 도메인 로직이 Service에 새어나간 경우

```kotlin
// ❌ 배송 가능 여부 판단이 Service에 있음
class ShipmentService {
    fun canDispatch(shipment: Shipment): Boolean {
        return shipment.status == "CREATED"
            && shipment.trackingNumber != null
            && shipment.items.isNotEmpty()
    }
}

// ✅ 배송 Entity가 자기 규칙을 직접 관리
class Shipment {
    fun dispatch(trackingNumber: TrackingNumber) {
        require(status == ShipmentStatus.CREATED) { "출고 불가 상태" }
        require(items.isNotEmpty()) { "배송 상품이 없습니다" }
        this.status = ShipmentStatus.DISPATCHED
        this.trackingNumber = trackingNumber
    }
}
```

> 💡 **판별 기준**: "이 로직이 없으면 객체가 잘못된 상태가 될 수 있는가?" → Yes면 Entity 안에 있어야 한다.

---

## 9. AI와 DDD

### 9.1 AI한테 도메인 모델 설계를 시키는 프롬프트 패턴

#### 패턴 1: 요구사항 → 도메인 모델 도출

```
너는 물류 이커머스 시스템의 DDD 전문가야.

아래 요구사항을 기반으로 도메인 모델을 설계해줘:

요구사항:
- 셀러가 상품을 등록하면 관리자 승인 후 판매 가능
- 구매자가 주문하면 재고가 차감되고 결제가 진행됨
- 결제 완료 후 배송이 생성됨

다음을 포함해줘:
1. Bounded Context 식별
2. 각 컨텍스트의 Aggregate, Entity, VO
3. 컨텍스트 간 Domain Event
4. Kotlin 코드
```

#### 패턴 2: 기존 코드 리팩토링

```
아래 Kotlin 코드는 빈약한 도메인 모델(Anemic Domain Model)이야.
DDD 원칙에 맞게 리팩토링해줘.

규칙:
- Entity는 자신의 불변식을 보호해야 해
- 비즈니스 로직은 도메인 객체 안에
- Aggregate Root를 통해서만 내부 객체 변경
- var 대신 private set, 외부에서 직접 상태 변경 불가

[코드 붙여넣기]
```

#### 패턴 3: DDD 모델 → Hexagonal 배치 (두 문서 연결)

```
아래 DDD 도메인 모델을 Hexagonal Architecture에 배치해줘.

도메인 모델:
[Order Aggregate, ShippingFeeCalculator 등 DDD 코드 붙여넣기]

요구사항:
- Inbound Port (UseCase 인터페이스) 정의
- Outbound Port (Repository 인터페이스) 정의
- Application Service (UseCase 구현) 작성
- 패키지 구조: com.example.order
- Kotlin + Spring Boot
```

### 9.2 AI가 잘하는 / 못하는 DDD 영역

| 영역 | AI가 잘하는 것 | AI가 못하는 것 |
|------|--------------|---------------|
| **모델링** | 패턴에 맞는 코드 생성 | 비즈니스 맥락 이해 (실제 물류 현장 경험) |
| **전략적 설계** | Bounded Context 분리 제안 | 조직 구조와 팀 역학 반영 |
| **전술적 설계** | Entity, VO, Aggregate 코드 생성 | Aggregate 경계를 "적절히" 잡는 것 |
| **이벤트** | 이벤트 흐름 설계 | 실제 운영 환경의 이벤트 순서/유실 대응 |
| **네이밍** | 관례적 이름 제안 | 회사 내부 용어(Ubiquitous Language) |

> 💡 **실전 팁**: AI는 "틀을 잡아주는 것"에 탁월하다. 하지만 **Ubiquitous Language는 비즈니스 전문가와 함께 정해야** 한다. AI가 만든 모델을 기획자에게 보여주고 "이 용어가 맞나요?"라고 확인하는 습관을 들이자.

---

## 10. 참고 자료

### 📚 책

| 책 | 설명 | 난이도 |
|---|------|-------|
| **도메인 주도 설계 핵심** (반 버논) | DDD 입문서로 최적. 얇고 핵심만 | ⭐ 입문 |
| **도메인 주도 설계** (에릭 에반스) | 원서. DDD의 바이블. 두껍다. | ⭐⭐⭐ 고급 |
| **도메인 주도 설계 구현** (반 버논) | 실제 구현에 초점. Aggregate 설계 상세 | ⭐⭐ 중급 |
| **오브젝트** (조영호) | 한국어, OOP 기초를 탄탄히. DDD 전에 읽으면 좋다 | ⭐ 입문 |

### 🎓 강의 / 영상

- [만들면서 배우는 클린 아키텍처 (인프런)](https://www.inflearn.com/) — Kotlin + Spring Boot 기반
- [조영호 - 우아한객체지향 (YouTube)](https://www.youtube.com/) — 도메인 모델링 사고방식
- [GOTO Conferences - DDD 영상들 (YouTube)](https://www.youtube.com/) — Eric Evans, Vaughn Vernon 발표
- [Alberto Brandolini - Event Storming (YouTube)](https://www.youtube.com/) — Event Storming 창시자 발표

### 🌐 블로그 / 글

- [Martin Fowler - DDD 관련 글들](https://martinfowler.com/tags/domain%20driven%20design.html)
- [우아한형제들 기술 블로그](https://techblog.woowahan.com/) — 실전 DDD 적용 사례
- [카카오페이 기술 블로그](https://tech.kakaopay.com/) — DDD + 이벤트 기반 아키텍처
- [EventStorming.com](https://www.eventstorming.com/) — Event Storming 공식 사이트

---

> 📌 **다음 단계**: 이 문서와 함께 [Hexagonal Architecture 학습 가이드](./hexagonal-architecture-learning-guide.md)를 읽으면, DDD 도메인 모델을 어떤 아키텍처에 배치할지 알 수 있다. 특히 이 문서의 Repository 인터페이스가 Hexagonal의 Outbound Port로, Application Service가 UseCase 구현으로 어떻게 매핑되는지 확인하자.

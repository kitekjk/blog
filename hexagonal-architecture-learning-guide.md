# Hexagonal Architecture 학습 가이드

> 물류 이커머스 도메인 기준 · 신입 AI Native Engineer 대상

---

## 목차

1. [Hexagonal Architecture란](#1-hexagonal-architecture란)
2. [핵심 개념](#2-핵심-개념)
3. [전통적 Layered Architecture와 비교](#3-전통적-layered-architecture와-비교)
4. [물류 도메인 전체 구현 예시](#4-물류-도메인-전체-구현-예시-kotlin--spring-boot)
5. [매핑 전략 — Domain ↔ JPA Entity 변환](#5-매핑-전략--domain--jpa-entity-변환)
6. [트랜잭션 처리 가이드](#6-트랜잭션-처리-가이드)
7. [Adapter 에러 처리와 복원력](#7-adapter-에러-처리와-복원력)
8. [테스트 전략](#8-테스트-전략)
9. [AI와 Hexagonal Architecture](#9-ai와-hexagonal-architecture)
10. [흔한 실수와 안티패턴](#10-흔한-실수와-안티패턴)
11. [참고 자료](#11-참고-자료)

---

## 용어 매핑 — Hexagonal ↔ DDD

이 문서와 [DDD 학습 가이드](./ddd-learning-guide.md)는 같은 물류 도메인을 다룬다. 용어 대응 관계:

| Hexagonal 용어 | DDD 용어 | 설명 |
|---------------|----------|------|
| **Outbound Port** | Repository 인터페이스 | Aggregate 저장/조회를 추상화하는 계약 |
| **Driven Adapter** | Repository 구현체 | JPA, MongoDB 등 실제 기술 구현 |
| **UseCase (Inbound Port 구현)** | Application Service | 유즈케이스 오케스트레이션 |
| **Domain 계층** | Entity, VO, Domain Service | 순수 비즈니스 로직 |
| **Driving Adapter** | (해당 없음) | Controller, Consumer 등 외부 진입점 |

---

## 1. Hexagonal Architecture란

### 정의

Hexagonal Architecture(헥사고날 아키텍처)는 **비즈니스 로직(Core)을 외부 기술(DB, API, UI)로부터 격리하는 아키텍처 패턴**이다.

2005년 Alistair Cockburn이 제안했으며, **Ports & Adapters** 패턴이라고도 부른다.

### 다른 이름들

| 이름 | 강조점 |
|------|--------|
| Hexagonal Architecture | 육각형 다이어그램에서 유래 |
| Ports & Adapters | 핵심 구조를 직관적으로 설명 |
| Clean Architecture (Uncle Bob) | 유사 개념, 의존성 규칙 강조 |
| Onion Architecture | 계층이 양파처럼 감싸는 구조 |

> 이름은 다르지만 핵심 원칙은 같다: **안쪽(도메인)이 바깥쪽(기술)을 모른다.**

### 왜 필요한가

물류 시스템을 예로 들면:

```
현실에서 자주 일어나는 일:
- DB를 MySQL → PostgreSQL로 바꿔야 함
- 택배사 API가 변경됨 (CJ → 한진)
- REST API → gRPC로 전환
- 배치 처리 추가 (같은 비즈니스 로직을 다른 트리거로)
```

이때 비즈니스 로직이 DB나 API에 직접 의존하면? **전부 다 바꿔야 한다.** Hexagonal Architecture를 쓰면 **Adapter만 교체**하면 된다.

---

## 2. 핵심 개념

### 전체 구조

```
                    Driving Side (외부 → 안쪽)
                    
        REST Controller ─┐
        CLI Command ─────┤
        Kafka Consumer ──┘
                         │
                    ┌────▼────┐
                    │  Inbound │
                    │   Port   │  ← UseCase 인터페이스
                    ├─────────┤
                    │         │
                    │  Core   │  ← Domain + Application
                    │         │
                    ├─────────┤
                    │ Outbound │  ← Repository, 외부 API 인터페이스
                    │   Port   │
                    └────┬────┘
                         │
        JPA Repository ──┤
        Redis Cache ─────┤    Driven Side (안쪽 → 외부)
        택배사 API ───────┘
```

### 2.1 Port (포트)

**Core와 외부 세계 사이의 인터페이스(계약).**

- **Inbound Port (Driving Port)**: 외부가 Core를 사용하는 방법. UseCase 인터페이스.
- **Outbound Port (Driven Port)**: Core가 외부를 사용하는 방법. Repository, 외부 API 인터페이스.

```kotlin
// Inbound Port — "배송을 생성하는 방법"
interface CreateShipmentUseCase {
    fun execute(command: CreateShipmentCommand): ShipmentId
}

// Outbound Port — "배송 데이터를 저장하는 방법" (DDD의 Repository 인터페이스)
interface ShipmentRepository {
    fun save(shipment: Shipment)
    fun findById(id: ShipmentId): Shipment?
}

// Outbound Port — "택배사에 배송을 요청하는 방법"
interface CarrierApi {
    fun requestPickup(shipment: Shipment): TrackingNumber
}
```

### 2.2 Adapter (어댑터)

**Port의 실제 구현체. 기술적 세부사항을 담당한다.**

- **Driving Adapter**: Inbound Port를 호출하는 쪽 (Controller, Consumer 등)
- **Driven Adapter**: Outbound Port를 구현하는 쪽 (JPA Repository, 외부 API 클라이언트 등)

### 2.3 Application Core

Port 안쪽의 모든 것:

| 계층 | 역할 | 예시 |
|------|------|------|
| **Domain** | 비즈니스 규칙, Entity, VO | `Shipment`, `Address`, `Money` |
| **Application (UseCase)** | 유즈케이스 오케스트레이션 | `CreateShipmentService` |

> 💡 **DDD와의 연결**: Domain 계층은 [DDD 가이드의 Tactical Design](./ddd-learning-guide.md#4-tactical-design-전술적-설계)에서 설계한 Entity, VO, Domain Service가 위치하는 곳이다.

### 2.4 의존성 역전 원칙 (DIP) — 가장 중요한 규칙

```
    Adapter → Port → Core
    
    ✅ Adapter는 Port를 안다
    ✅ UseCase는 Outbound Port를 안다
    ✅ Domain은 아무것도 모른다 (순수)
    
    ❌ Core가 Adapter를 아는 것은 절대 안 됨
    ❌ Domain이 Spring, JPA 등 프레임워크를 아는 것도 안 됨
```

#### 왜 "역전"인가? — DIP를 신입 눈높이로 풀어보기

보통 코드를 짜면 **비즈니스 로직이 DB를 호출**한다:

```
일반적인 의존성 방향:
Service → JpaRepository → Database

"Service가 JpaRepository를 직접 알고 있다"
→ DB를 바꾸면 Service도 바꿔야 한다
```

DIP는 이 방향을 **뒤집는다(역전시키는)** 것이다:

```
의존성 역전 후:
Service → (의존) → Repository 인터페이스(Port) ← (구현) ← JpaRepositoryAdapter

"Service는 인터페이스만 알고, 구현은 모른다"
→ DB를 바꿔도 새 Adapter만 만들면 된다
→ Service 코드는 한 줄도 안 바뀐다!
```

구체적으로 보면:

```kotlin
// ❌ DIP 적용 전 — Service가 JPA를 직접 안다
class CreateShipmentService(
    private val jpaRepo: ShipmentJpaRepository,  // JPA에 직접 의존!
)

// ✅ DIP 적용 후 — Service는 인터페이스만 안다
class CreateShipmentService(
    private val repo: ShipmentRepository,  // 인터페이스(Port)에 의존
)

// 인터페이스
interface ShipmentRepository {           // Core 안에 정의
    fun save(shipment: Shipment)
}

// 구현 — Adapter 영역
class ShipmentJpaAdapter : ShipmentRepository {  // Adapter가 Port를 구현
    override fun save(shipment: Shipment) { /* JPA 코드 */ }
}
```

> 💡 **신입 팁**: "import문을 보라." Domain 클래스 파일에 `import org.springframework` 이나 `import javax.persistence`가 있으면 뭔가 잘못된 것이다.

---

## 3. 전통적 Layered Architecture와 비교

### 구조 비교

```
Layered Architecture          Hexagonal Architecture

┌──────────────┐             ┌──────────────┐
│ Presentation │             │   Driving    │
│  (Controller)│             │  Adapters    │
├──────────────┤             ├──────────────┤
│   Service    │             │ Inbound Port │
│  (Business)  │             ├──────────────┤
├──────────────┤             │    Core      │
│     DAO      │             │ (Domain+App) │
│ (Data Access)│             ├──────────────┤
├──────────────┤             │ Outbound Port│
│   Database   │             ├──────────────┤
└──────────────┘             │   Driven     │
                             │  Adapters    │
  의존성: 위 → 아래           └──────────────┘
  (Service가 DAO를 직접 의존)
                               의존성: 바깥 → 안쪽
                              (Adapter가 Port를 의존)
```

### 장단점 비교표

| 기준 | Layered Architecture | Hexagonal Architecture |
|------|---------------------|----------------------|
| **학습 난이도** | 쉬움 | 중간 |
| **초기 코드량** | 적음 | 많음 (인터페이스 + 구현) |
| **DB 변경 용이성** | 어려움 (Service가 DAO에 직접 의존) | 쉬움 (Adapter만 교체) |
| **테스트 용이성** | Service 테스트 시 DB 필요 | Domain 테스트에 외부 의존 없음 |
| **비즈니스 로직 위치** | Service에 분산 | Domain에 집중 |
| **프레임워크 의존도** | 전 계층에 걸쳐 Spring 의존 | Core는 프레임워크 무관 |
| **적합한 프로젝트** | 단순 CRUD, 빠른 프로토타입 | 복잡한 비즈니스 로직, 장기 운영 |

> 💡 **실전 판단 기준**: "이 시스템이 3년 뒤에도 운영되고, 비즈니스 로직이 복잡하다면" → Hexagonal. "빠르게 만들고 검증만 하면 된다면" → Layered도 OK.

---

## 4. 물류 도메인 전체 구현 예시 (Kotlin + Spring Boot)

> 이 장에서 만드는 코드는 [DDD 가이드의 배송/주문 모델](./ddd-learning-guide.md#4-tactical-design-전술적-설계)을 Hexagonal Architecture에 배치한 것이다.

### 4.1 패키지 구조

```
com.example.shipping/
├── domain/                          # 🔵 Domain Layer (순수 Kotlin)
│   ├── model/
│   │   ├── Shipment.kt              # Aggregate Root
│   │   ├── ShipmentItem.kt          # Entity
│   │   ├── ShipmentStatus.kt        # Enum
│   │   ├── TrackingNumber.kt        # Value Object
│   │   ├── Recipient.kt             # Value Object
│   │   └── Address.kt               # Value Object
│   ├── event/
│   │   ├── DomainEvent.kt
│   │   └── ShipmentCreated.kt
│   └── service/
│       └── ShippingFeeCalculator.kt  # Domain Service
│
├── application/                     # 🟢 Application Layer (UseCase)
│   ├── port/
│   │   ├── inbound/                 # Inbound Ports
│   │   │   ├── CreateShipmentUseCase.kt
│   │   │   └── TrackShipmentUseCase.kt
│   │   └── outbound/                # Outbound Ports
│   │       ├── ShipmentRepository.kt
│   │       ├── CarrierApi.kt
│   │       └── DomainEventPublisher.kt
│   ├── dto/
│   │   ├── CreateShipmentCommand.kt
│   │   └── ShipmentInfo.kt
│   └── service/
│       ├── CreateShipmentService.kt  # Inbound Port 구현
│       └── TrackShipmentService.kt
│
├── adapter/                         # 🟠 Adapter Layer (기술 구현)
│   ├── inbound/                     # Driving Adapters
│   │   ├── web/
│   │   │   ├── ShipmentController.kt
│   │   │   ├── ShipmentRequest.kt
│   │   │   └── ShipmentResponse.kt
│   │   └── consumer/
│   │       └── OrderEventConsumer.kt
│   ├── outbound/                    # Driven Adapters
│   │   ├── persistence/
│   │   │   ├── ShipmentJpaRepository.kt
│   │   │   ├── ShipmentJpaEntity.kt
│   │   │   └── ShipmentRepositoryAdapter.kt
│   │   └── carrier/
│   │       ├── CjCarrierApiAdapter.kt
│   │       └── CjCarrierResponse.kt
│   └── config/
│       └── BeanConfig.kt            # 의존성 주입 설정
│
└── ShippingApplication.kt
```

> 💡 **ArchUnit으로 의존성 규칙 강제하기**: 패키지 규칙은 사람의 의지만으로는 지켜지지 않는다. ArchUnit 테스트로 자동화하면 CI에서 잡아준다. (8장 테스트 전략에서 예시 확인)

### 4.2 Domain Layer

> 이 절에서 만드는 것: **Shipment Aggregate Root**, Value Objects (Recipient, Address, TrackingNumber), ShipmentItem

```kotlin
// ===== domain/model/Shipment.kt =====
// 순수 Kotlin. Spring, JPA 등 프레임워크 import 없음!

package com.example.shipping.domain.model

import com.example.shipping.domain.event.DomainEvent
import com.example.shipping.domain.event.ShipmentCreated
import java.time.Instant

class Shipment private constructor(
    val id: ShipmentId,
    val orderId: OrderId,
    val recipient: Recipient,
    private val _items: MutableList<ShipmentItem>,
    private var status: ShipmentStatus = ShipmentStatus.CREATED,
    private var trackingNumber: TrackingNumber? = null,
    val createdAt: Instant = Instant.now(),
) {
    val items: List<ShipmentItem> get() = _items.toList()

    private val _events = mutableListOf<DomainEvent>()
    val domainEvents: List<DomainEvent> get() = _events.toList()

    companion object {
        // 새로 생성할 때 — 이벤트 발행
        fun create(
            id: ShipmentId,
            orderId: OrderId,
            recipient: Recipient,
            items: List<ShipmentItem>,
        ): Shipment {
            require(items.isNotEmpty()) { "배송 상품이 비어있습니다" }

            val shipment = Shipment(
                id = id,
                orderId = orderId,
                recipient = recipient,
                _items = items.toMutableList(),
            )
            shipment._events.add(
                ShipmentCreated(shipment.id, orderId, Instant.now())
            )
            return shipment
        }

        // DB에서 복원할 때 — 이벤트를 발행하지 않는다!
        // (Repository Adapter에서 사용. DDD 가이드의 reconstitute 패턴)
        fun reconstitute(
            id: ShipmentId,
            orderId: OrderId,
            recipient: Recipient,
            items: List<ShipmentItem>,
            status: ShipmentStatus,
            trackingNumber: TrackingNumber?,
            createdAt: Instant,
        ): Shipment {
            return Shipment(
                id = id,
                orderId = orderId,
                recipient = recipient,
                _items = items.toMutableList(),
                status = status,
                trackingNumber = trackingNumber,
                createdAt = createdAt,
            )
        }
    }

    fun dispatch(trackingNumber: TrackingNumber) {
        require(status == ShipmentStatus.CREATED) {
            "CREATED 상태에서만 출고 가능합니다. 현재: $status"
        }
        this.status = ShipmentStatus.DISPATCHED
        this.trackingNumber = trackingNumber
    }

    fun complete() {
        require(status == ShipmentStatus.DISPATCHED) {
            "DISPATCHED 상태에서만 배송 완료 처리 가능합니다. 현재: $status"
        }
        this.status = ShipmentStatus.DELIVERED
    }

    fun currentStatus(): ShipmentStatus = status
    fun currentTrackingNumber(): TrackingNumber? = trackingNumber

    fun clearEvents() = _events.clear()
}

// Kotlin @JvmInline value class:
// 런타임에는 String이지만, 컴파일 타임에 타입을 구분해준다.
// ShipmentId와 OrderId를 헷갈려서 넣는 실수를 방지한다.
@JvmInline value class ShipmentId(val value: String)
@JvmInline value class OrderId(val value: String)

// ===== domain/model/Recipient.kt =====
data class Recipient(
    val name: String,
    val phone: String,
    val address: Address,
) {
    init {
        require(name.isNotBlank()) { "수령인 이름은 필수입니다" }
        require(phone.matches(Regex("\\d{10,11}"))) { "올바른 전화번호 형식이 아닙니다" }
    }
}

// ===== domain/model/Address.kt =====
data class Address(
    val zipCode: String,
    val city: String,
    val street: String,
    val detail: String,
) {
    init {
        require(zipCode.matches(Regex("\\d{5}"))) { "우편번호는 5자리 숫자" }
    }

    fun isRemoteArea(): Boolean =
        REMOTE_PREFIXES.any { zipCode.startsWith(it) }

    companion object {
        private val REMOTE_PREFIXES = listOf("63", "69")
    }
}

// ===== domain/model/ShipmentItem.kt =====
data class ShipmentItem(
    val productId: String,
    val productName: String,
    val quantity: Int,
    val weightGrams: Int,
) {
    init {
        require(quantity > 0) { "수량은 1 이상" }
        require(weightGrams > 0) { "무게는 0보다 커야 합니다" }
    }
}

// ===== domain/model/TrackingNumber.kt =====
@JvmInline
value class TrackingNumber(val value: String) {
    init { require(value.isNotBlank()) { "운송장 번호는 필수" } }
}

// ===== domain/model/ShipmentStatus.kt =====
enum class ShipmentStatus {
    CREATED,      // 배송 생성됨
    DISPATCHED,   // 출고됨
    IN_TRANSIT,   // 배송중
    DELIVERED,    // 배송 완료
    RETURNED,     // 반송됨
}
```

### 4.3 Port — Inbound (UseCase 인터페이스)

> 이 절에서 만드는 것: **CreateShipmentUseCase**, **TrackShipmentUseCase** (Inbound Ports)

```kotlin
// ===== application/port/inbound/CreateShipmentUseCase.kt =====
package com.example.shipping.application.port.inbound

import com.example.shipping.application.dto.CreateShipmentCommand
import com.example.shipping.domain.model.ShipmentId

interface CreateShipmentUseCase {
    fun execute(command: CreateShipmentCommand): ShipmentId
}

// ===== application/dto/CreateShipmentCommand.kt =====
data class CreateShipmentCommand(
    val orderId: String,
    val recipientName: String,
    val recipientPhone: String,
    val zipCode: String,
    val city: String,
    val street: String,
    val addressDetail: String,
    val items: List<ShipmentItemCommand>,
)

data class ShipmentItemCommand(
    val productId: String,
    val productName: String,
    val quantity: Int,
    val weightGrams: Int,
)

// ===== application/port/inbound/TrackShipmentUseCase.kt =====
interface TrackShipmentUseCase {
    fun execute(shipmentId: String): ShipmentInfo
}

data class ShipmentInfo(
    val shipmentId: String,
    val orderId: String,
    val status: String,
    val trackingNumber: String?,
    val recipientName: String,
)
```

### 4.4 Port — Outbound (Repository, 외부 API)

> 이 절에서 만드는 것: **ShipmentRepository**, **CarrierApi**, **DomainEventPublisher** (Outbound Ports)

```kotlin
// ===== application/port/outbound/ShipmentRepository.kt =====
package com.example.shipping.application.port.outbound

import com.example.shipping.domain.model.Shipment
import com.example.shipping.domain.model.ShipmentId
import com.example.shipping.domain.model.OrderId

interface ShipmentRepository {
    fun save(shipment: Shipment)
    fun findById(id: ShipmentId): Shipment?
    fun existsByOrderId(orderId: OrderId): Boolean  // 멱등성 체크용
    fun nextId(): ShipmentId
}

// ===== application/port/outbound/CarrierApi.kt =====
import com.example.shipping.domain.model.Shipment
import com.example.shipping.domain.model.TrackingNumber

interface CarrierApi {
    fun requestPickup(shipment: Shipment): TrackingNumber
    fun getTrackingStatus(trackingNumber: TrackingNumber): String
}

// ===== application/port/outbound/DomainEventPublisher.kt =====
import com.example.shipping.domain.event.DomainEvent

interface DomainEventPublisher {
    fun publish(event: DomainEvent)
}
```

### 4.5 Application Service (UseCase 구현)

> 이 절에서 만드는 것: **CreateShipmentService** (Inbound Port 구현체 = DDD의 Application Service)

```kotlin
// ===== application/service/CreateShipmentService.kt =====
package com.example.shipping.application.service

import com.example.shipping.application.dto.CreateShipmentCommand
import com.example.shipping.application.port.inbound.CreateShipmentUseCase
import com.example.shipping.application.port.outbound.DomainEventPublisher
import com.example.shipping.application.port.outbound.ShipmentRepository
import com.example.shipping.domain.model.*

class CreateShipmentService(
    private val shipmentRepository: ShipmentRepository,
    private val eventPublisher: DomainEventPublisher,
) : CreateShipmentUseCase {

    override fun execute(command: CreateShipmentCommand): ShipmentId {
        val shipment = Shipment.create(
            id = shipmentRepository.nextId(),
            orderId = OrderId(command.orderId),
            recipient = Recipient(
                name = command.recipientName,
                phone = command.recipientPhone,
                address = Address(
                    zipCode = command.zipCode,
                    city = command.city,
                    street = command.street,
                    detail = command.addressDetail,
                ),
            ),
            items = command.items.map {
                ShipmentItem(
                    productId = it.productId,
                    productName = it.productName,
                    quantity = it.quantity,
                    weightGrams = it.weightGrams,
                )
            },
        )

        shipmentRepository.save(shipment)

        // Aggregate에서 수집된 이벤트를 저장 성공 후 발행
        // (DDD 가이드 6장의 이벤트 수집-발행 패턴)
        shipment.domainEvents.forEach { eventPublisher.publish(it) }
        shipment.clearEvents()

        return shipment.id
    }
}
```

> 💡 **주목**: `CreateShipmentService`에는 Spring 어노테이션이 없다. `@Service`, `@Transactional` 같은 것은 Config에서 처리한다. ([6장 트랜잭션 가이드](#6-트랜잭션-처리-가이드) 참조)

### 4.6 Adapter — Web (REST Controller)

> 이 절에서 만드는 것: **ShipmentController** (Driving Adapter), Request/Response DTO

```kotlin
// ===== adapter/inbound/web/ShipmentController.kt =====
package com.example.shipping.adapter.inbound.web

import com.example.shipping.application.dto.CreateShipmentCommand
import com.example.shipping.application.dto.ShipmentItemCommand
import com.example.shipping.application.port.inbound.CreateShipmentUseCase
import com.example.shipping.application.port.inbound.TrackShipmentUseCase
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/shipments")
class ShipmentController(
    private val createShipmentUseCase: CreateShipmentUseCase,
    private val trackShipmentUseCase: TrackShipmentUseCase,
) {
    @PostMapping
    fun create(@RequestBody request: CreateShipmentRequest): ResponseEntity<CreateShipmentResponse> {
        val shipmentId = createShipmentUseCase.execute(request.toCommand())
        return ResponseEntity.ok(CreateShipmentResponse(shipmentId.value))
    }

    @GetMapping("/{shipmentId}")
    fun track(@PathVariable shipmentId: String): ResponseEntity<ShipmentResponse> {
        val info = trackShipmentUseCase.execute(shipmentId)
        return ResponseEntity.ok(ShipmentResponse.from(info))
    }
}

// Request/Response는 Adapter에 속함 — Core를 모름
data class CreateShipmentRequest(
    val orderId: String,
    val recipientName: String,
    val recipientPhone: String,
    val zipCode: String,
    val city: String,
    val street: String,
    val addressDetail: String,
    val items: List<ShipmentItemRequest>,
) {
    fun toCommand() = CreateShipmentCommand(
        orderId = orderId,
        recipientName = recipientName,
        recipientPhone = recipientPhone,
        zipCode = zipCode,
        city = city,
        street = street,
        addressDetail = addressDetail,
        items = items.map {
            ShipmentItemCommand(it.productId, it.productName, it.quantity, it.weightGrams)
        },
    )
}

data class ShipmentItemRequest(
    val productId: String,
    val productName: String,
    val quantity: Int,
    val weightGrams: Int,
)

data class CreateShipmentResponse(val shipmentId: String)

data class ShipmentResponse(
    val shipmentId: String,
    val orderId: String,
    val status: String,
    val trackingNumber: String?,
    val recipientName: String,
) {
    companion object {
        fun from(info: ShipmentInfo) = ShipmentResponse(
            shipmentId = info.shipmentId,
            orderId = info.orderId,
            status = info.status,
            trackingNumber = info.trackingNumber,
            recipientName = info.recipientName,
        )
    }
}
```

### 4.7 Adapter — Persistence (JPA)

> 이 절에서 만드는 것: **ShipmentJpaEntity**, **ShipmentRepositoryAdapter** (Driven Adapter)
> 
> **핵심**: Domain Model ↔ JPA Entity를 분리하고, Adapter가 변환을 담당한다. 변환의 번거로움이 왜 필요한지는 [5장 매핑 전략](#5-매핑-전략--domain--jpa-entity-변환)에서 상세히 다룬다.

```kotlin
// ===== adapter/outbound/persistence/ShipmentJpaEntity.kt =====
package com.example.shipping.adapter.outbound.persistence

import jakarta.persistence.*
import java.time.Instant

@Entity
@Table(name = "shipments")
class ShipmentJpaEntity(
    @Id val id: String,
    val orderId: String,
    val recipientName: String,
    val recipientPhone: String,
    val zipCode: String,
    val city: String,
    val street: String,
    val addressDetail: String,
    @Enumerated(EnumType.STRING)
    var status: String,
    var trackingNumber: String?,
    val createdAt: Instant,

    @OneToMany(cascade = [CascadeType.ALL], orphanRemoval = true)
    @JoinColumn(name = "shipment_id")
    val items: MutableList<ShipmentItemJpaEntity> = mutableListOf(),
)

@Entity
@Table(name = "shipment_items")
class ShipmentItemJpaEntity(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,
    val productId: String,
    val productName: String,
    val quantity: Int,
    val weightGrams: Int,
)

// ===== adapter/outbound/persistence/ShipmentJpaRepository.kt =====
import org.springframework.data.jpa.repository.JpaRepository

interface ShipmentJpaRepository : JpaRepository<ShipmentJpaEntity, String> {
    fun existsByOrderId(orderId: String): Boolean
}

// ===== adapter/outbound/persistence/ShipmentRepositoryAdapter.kt =====
// 핵심! Outbound Port를 구현하는 Adapter
import com.example.shipping.application.port.outbound.ShipmentRepository
import com.example.shipping.domain.model.*
import org.springframework.stereotype.Component
import java.util.UUID

@Component
class ShipmentRepositoryAdapter(
    private val jpaRepository: ShipmentJpaRepository,
) : ShipmentRepository {

    override fun save(shipment: Shipment) {
        jpaRepository.save(toJpaEntity(shipment))
    }

    override fun findById(id: ShipmentId): Shipment? {
        return jpaRepository.findById(id.value)
            .map { toDomain(it) }
            .orElse(null)
    }

    override fun existsByOrderId(orderId: OrderId): Boolean {
        return jpaRepository.existsByOrderId(orderId.value)
    }

    override fun nextId(): ShipmentId =
        ShipmentId(UUID.randomUUID().toString())

    // JPA Entity → Domain Model
    // ⚠️ reconstitute()를 사용해서 이벤트 없이 복원한다!
    // create()를 호출하면 이벤트가 중복 생성되는 버그 발생
    private fun toDomain(entity: ShipmentJpaEntity): Shipment {
        return Shipment.reconstitute(
            id = ShipmentId(entity.id),
            orderId = OrderId(entity.orderId),
            recipient = Recipient(
                name = entity.recipientName,
                phone = entity.recipientPhone,
                address = Address(entity.zipCode, entity.city, entity.street, entity.addressDetail),
            ),
            items = entity.items.map {
                ShipmentItem(it.productId, it.productName, it.quantity, it.weightGrams)
            },
            status = ShipmentStatus.valueOf(entity.status),
            trackingNumber = entity.trackingNumber?.let { TrackingNumber(it) },
            createdAt = entity.createdAt,
        )
    }

    // Domain Model → JPA Entity
    private fun toJpaEntity(shipment: Shipment): ShipmentJpaEntity {
        return ShipmentJpaEntity(
            id = shipment.id.value,
            orderId = shipment.orderId.value,
            recipientName = shipment.recipient.name,
            recipientPhone = shipment.recipient.phone,
            zipCode = shipment.recipient.address.zipCode,
            city = shipment.recipient.address.city,
            street = shipment.recipient.address.street,
            addressDetail = shipment.recipient.address.detail,
            status = shipment.currentStatus().name,
            trackingNumber = shipment.currentTrackingNumber()?.value,
            createdAt = shipment.createdAt,
            items = shipment.items.map {
                ShipmentItemJpaEntity(
                    productId = it.productId,
                    productName = it.productName,
                    quantity = it.quantity,
                    weightGrams = it.weightGrams,
                )
            }.toMutableList(),
        )
    }
}
```

### 4.8 Adapter — External (택배사 API)

> 이 절에서 만드는 것: **CjCarrierApiAdapter** (Driven Adapter) — 에러 처리 포함

```kotlin
// ===== adapter/outbound/carrier/CjCarrierApiAdapter.kt =====
package com.example.shipping.adapter.outbound.carrier

import com.example.shipping.application.port.outbound.CarrierApi
import com.example.shipping.domain.model.Shipment
import com.example.shipping.domain.model.TrackingNumber
import org.springframework.stereotype.Component
import org.springframework.web.client.RestTemplate
import org.springframework.web.client.RestClientException

@Component
class CjCarrierApiAdapter(
    private val restTemplate: RestTemplate,
) : CarrierApi {

    override fun requestPickup(shipment: Shipment): TrackingNumber {
        try {
            val response = restTemplate.postForObject(
                "https://api.cjlogistics.com/pickup",
                CjPickupRequest(
                    senderAddress = "우리 물류센터 주소",
                    receiverName = shipment.recipient.name,
                    receiverPhone = shipment.recipient.phone,
                    receiverZipCode = shipment.recipient.address.zipCode,
                    receiverAddress = "${shipment.recipient.address.city} ${shipment.recipient.address.street}",
                ),
                CjPickupResponse::class.java,
            )
            return TrackingNumber(response!!.trackingNumber)
        } catch (e: RestClientException) {
            // 택배사 API 실패 시 → 도메인 예외로 변환
            // Adapter가 기술 예외를 삼키고, Core가 이해할 수 있는 예외로 바꾼다
            throw CarrierApiException("CJ택배 픽업 요청 실패: ${e.message}", e)
        }
    }

    override fun getTrackingStatus(trackingNumber: TrackingNumber): String {
        try {
            val response = restTemplate.getForObject(
                "https://api.cjlogistics.com/tracking/${trackingNumber.value}",
                CjTrackingResponse::class.java,
            )
            return response!!.status
        } catch (e: RestClientException) {
            throw CarrierApiException("CJ택배 배송 추적 실패: ${e.message}", e)
        }
    }
}

// 도메인이 이해할 수 있는 예외 (application 또는 domain 패키지에 정의)
class CarrierApiException(message: String, cause: Throwable? = null) : RuntimeException(message, cause)

// CJ택배 API 전용 DTO — Adapter 안에만 존재
data class CjPickupRequest(
    val senderAddress: String,
    val receiverName: String,
    val receiverPhone: String,
    val receiverZipCode: String,
    val receiverAddress: String,
)

data class CjPickupResponse(val trackingNumber: String)
data class CjTrackingResponse(val status: String, val history: List<String>)
```

> 💡 택배사를 한진으로 변경하려면? `HanjinCarrierApiAdapter`를 만들고, Config에서 교체하면 끝. Domain, UseCase 코드는 한 줄도 안 바뀐다.

### 4.9 Configuration — 의존성 주입

```kotlin
// ===== adapter/config/BeanConfig.kt =====
package com.example.shipping.adapter.config

import com.example.shipping.application.port.outbound.DomainEventPublisher
import com.example.shipping.application.port.outbound.ShipmentRepository
import com.example.shipping.application.service.CreateShipmentService
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class BeanConfig {

    @Bean
    fun createShipmentUseCase(
        shipmentRepository: ShipmentRepository,
        eventPublisher: DomainEventPublisher,
    ) = CreateShipmentService(shipmentRepository, eventPublisher)
    // 여기서 Spring과 연결!
    // CreateShipmentService 자체에는 Spring 어노테이션 없음
}
```

#### 왜 @Service를 안 쓰나? — Config vs Annotation 비교

| 방식 | 장점 | 단점 |
|------|------|------|
| **`@Configuration` + `@Bean`** (위 방식) | Core에 Spring 의존 없음. 테스트 시 순수 Kotlin으로 | Config 파일 관리 필요 |
| **`@Service` on UseCase** | 간편, 자동 스캔 | Core에 Spring이 침투 (`import org.springframework`) |

> 💡 **팀 선택**: 두 방식 다 실무에서 쓰인다. 엄격하게 Core 순수성을 지키려면 `@Bean` 방식, 실용적으로 가려면 `@Service`도 OK. **팀에서 하나를 정하고 일관되게 쓰는 게 중요**하다.

### 4.10 전체 배송 생성 흐름

```
1. Client → POST /api/shipments
2. ShipmentController.create()
   → request.toCommand() (Adapter DTO → Application DTO 변환)
3. CreateShipmentUseCase.execute(command)
   → 실제로는 CreateShipmentService가 실행됨
4. Shipment.create() — Domain 객체 생성, 불변식 검증
5. ShipmentRepository.save() — Port 호출
   → 실제로는 ShipmentRepositoryAdapter가 JPA로 저장
6. DomainEventPublisher.publish(ShipmentCreated)
   → 다른 컨텍스트에 알림
7. ShipmentId 반환 → Controller → 201 Created 응답
```

---

## 5. 매핑 전략 — Domain ↔ JPA Entity 변환

### 왜 Domain Model과 JPA Entity를 분리하는가?

```
"왜 이렇게 귀찮게 두 개를 만들어?" — 모든 신입의 의문
```

**분리하지 않으면 생기는 문제:**

1. **Domain 오염**: Entity에 `@Column`, `@Table` 같은 JPA 어노테이션이 붙으면, Domain이 DB 구조에 종속된다
2. **테스트 어려움**: Domain 테스트에 JPA/DB가 필요해진다
3. **DB 변경 불가**: Domain 구조를 바꾸면 DB 스키마도 바꿔야 하고, 그 반대도 마찬가지

**분리하면 얻는 것:**

- Domain은 순수 비즈니스 로직만 담는다
- DB 스키마와 Domain 구조가 독립적으로 진화할 수 있다
- 테스트가 빨라진다

### 매핑 전략 비교

| 전략 | 설명 | 사용 시점 |
|------|------|----------|
| **완전 매핑 (Full Mapping)** | Domain ↔ JPA Entity 별도, Adapter에서 변환 | **권장 기본값**. 이 가이드의 방식 |
| **매핑 생략 (No Mapping)** | Domain Entity에 JPA 어노테이션 직접 부착 | 극히 단순한 CRUD |
| **단방향 매핑** | 저장 시에만 변환, 조회는 JPA Entity 직접 사용 | 읽기 전용 모델 (CQRS Read Side) |

> 💡 **실전 가이드**: 처음에 "귀찮다"고 느껴도 완전 매핑으로 시작하자. 시스템이 복잡해질수록 분리한 보람을 느끼게 된다. 특히 `reconstitute()` 팩토리로 이벤트 없이 복원하는 패턴은 완전 매핑에서만 깔끔하게 구현된다.

---

## 6. 트랜잭션 처리 가이드

### `@Transactional`을 어디에 붙이나?

Hexagonal Architecture에서 가장 많이 받는 질문:

```
"UseCase(Application Service)에 @Transactional을 붙이면 Spring 의존 아닌가?"
```

### 방법 1: Config에서 트랜잭션 래핑 (엄격한 방식)

```kotlin
// UseCase 구현 — Spring 의존 없음
class CreateShipmentService(
    private val shipmentRepository: ShipmentRepository,
    private val eventPublisher: DomainEventPublisher,
) : CreateShipmentUseCase {
    override fun execute(command: CreateShipmentCommand): ShipmentId { /* ... */ }
}

// Config에서 트랜잭션 적용
@Configuration
class BeanConfig {
    @Bean
    fun createShipmentUseCase(
        shipmentRepository: ShipmentRepository,
        eventPublisher: DomainEventPublisher,
    ): CreateShipmentUseCase {
        val service = CreateShipmentService(shipmentRepository, eventPublisher)
        // Spring의 TransactionTemplate을 사용하거나,
        // Proxy로 트랜잭션을 감싸는 방법도 있다
        return service
    }
}
```

### 방법 2: UseCase에 직접 @Transactional (실용적 방식)

```kotlin
// 실무에서 가장 많이 쓰는 방식
@Service
@Transactional
class CreateShipmentService(
    private val shipmentRepository: ShipmentRepository,
    private val eventPublisher: DomainEventPublisher,
) : CreateShipmentUseCase {
    override fun execute(command: CreateShipmentCommand): ShipmentId { /* ... */ }
}
```

### 팀 룰 예시

> **우리 팀의 트랜잭션 규칙:**
> 1. `@Transactional`은 **Application Service(UseCase 구현체)**에만 붙인다
> 2. Domain 계층 (`domain/` 패키지)에는 절대 Spring 어노테이션 없음
> 3. Adapter 계층에는 `@Transactional` 붙이지 않음 (UseCase가 경계)
> 4. 읽기 전용 UseCase는 `@Transactional(readOnly = true)`
> 5. 이벤트 발행은 `@TransactionalEventListener(phase = AFTER_COMMIT)`으로 커밋 후 발행

```kotlin
// 팀 룰 적용 예시
@Service
@Transactional
class CreateShipmentService(/* ... */) : CreateShipmentUseCase {
    override fun execute(command: CreateShipmentCommand): ShipmentId {
        // 이 메서드 전체가 하나의 트랜잭션
        val shipment = Shipment.create(/* ... */)
        shipmentRepository.save(shipment)
        
        // 이벤트는 TransactionalEventListener가 커밋 후 처리
        shipment.domainEvents.forEach { eventPublisher.publish(it) }
        shipment.clearEvents()
        
        return shipment.id
    }
}

@Service
@Transactional(readOnly = true)
class TrackShipmentService(/* ... */) : TrackShipmentUseCase {
    override fun execute(shipmentId: String): ShipmentInfo {
        // readOnly → DB 성능 최적화
        val shipment = shipmentRepository.findById(ShipmentId(shipmentId))
            ?: throw ShipmentNotFoundException(shipmentId)
        return ShipmentInfo(/* ... */)
    }
}
```

### 외부 API 호출 + 트랜잭션

**원칙: 트랜잭션 안에서 외부 API를 호출하지 마라.**

```kotlin
// ❌ 나쁨: 트랜잭션 안에서 외부 API 호출
@Transactional
fun dispatchShipment(shipmentId: ShipmentId) {
    val shipment = shipmentRepository.findById(shipmentId)!!
    val trackingNumber = carrierApi.requestPickup(shipment) // 🚨 외부 API 호출!
    // → 택배사 API가 느리면 DB 커넥션을 오래 잡고 있음
    // → API 실패 시 트랜잭션 전체 롤백
    shipment.dispatch(trackingNumber)
    shipmentRepository.save(shipment)
}

// ✅ 좋음: 트랜잭션을 분리
fun dispatchShipment(shipmentId: ShipmentId) {
    // Step 1: 외부 API 호출 (트랜잭션 밖)
    val shipment = shipmentRepository.findById(shipmentId)!!
    val trackingNumber = carrierApi.requestPickup(shipment)

    // Step 2: DB 저장 (트랜잭션 안)
    updateShipmentStatus(shipmentId, trackingNumber)
}

@Transactional
fun updateShipmentStatus(shipmentId: ShipmentId, trackingNumber: TrackingNumber) {
    val shipment = shipmentRepository.findById(shipmentId)!!
    shipment.dispatch(trackingNumber)
    shipmentRepository.save(shipment)
}
```

---

## 7. Adapter 에러 처리와 복원력

### 외부 API Adapter 에러 처리 체크리스트

| 항목 | 설명 | 예시 |
|------|------|------|
| **타임아웃** | API 응답 시간 제한 | `connectTimeout=3s, readTimeout=5s` |
| **재시도** | 일시적 장애 대응 | 최대 3회, 지수 백오프 |
| **Circuit Breaker** | 장기 장애 시 빠른 실패 | Resilience4j |
| **예외 변환** | 기술 예외 → 도메인 예외 | `RestClientException → CarrierApiException` |
| **로깅** | 장애 추적 | 요청/응답 전체 로깅 |

### RestTemplate 타임아웃 설정 예시

```kotlin
@Configuration
class RestTemplateConfig {
    @Bean
    fun carrierRestTemplate(): RestTemplate {
        val factory = SimpleClientHttpRequestFactory().apply {
            setConnectTimeout(3_000)  // 연결 타임아웃 3초
            setReadTimeout(5_000)     // 읽기 타임아웃 5초
        }
        return RestTemplate(factory)
    }
}
```

### 이벤트 핸들러의 멱등성 — 중복 처리 방지

비동기 이벤트 환경(Kafka 등)에서는 같은 이벤트가 재전송될 수 있다. 핸들러를 멱등하게 만들어야 한다:

```kotlin
@Component
class OrderPlacedEventHandler(
    private val createShipmentUseCase: CreateShipmentUseCase,
    private val shipmentRepository: ShipmentRepository,
) {
    @EventListener
    fun handle(event: OrderPlaced) {
        // 멱등성 체크: 이미 처리된 주문인지 확인
        if (shipmentRepository.existsByOrderId(event.orderId)) {
            log.info("이미 배송이 생성된 주문: ${event.orderId} — 무시")
            return
        }
        createShipmentUseCase.execute(/* ... */)
    }
}
```

---

## 8. 테스트 전략

### 8.1 Domain 단위 테스트 — 외부 의존성 제로

```kotlin
class ShipmentTest {

    @Test
    fun `배송 생성 시 상태는 CREATED`() {
        val shipment = createTestShipment()
        assertEquals(ShipmentStatus.CREATED, shipment.currentStatus())
    }

    @Test
    fun `CREATED 상태에서 출고 가능`() {
        val shipment = createTestShipment()
        shipment.dispatch(TrackingNumber("CJ1234567890"))

        assertEquals(ShipmentStatus.DISPATCHED, shipment.currentStatus())
        assertEquals("CJ1234567890", shipment.currentTrackingNumber()?.value)
    }

    @Test
    fun `이미 출고된 배송은 다시 출고 불가`() {
        val shipment = createTestShipment()
        shipment.dispatch(TrackingNumber("CJ1234567890"))

        assertThrows<IllegalArgumentException> {
            shipment.dispatch(TrackingNumber("CJ9999999999"))
        }
    }

    @Test
    fun `빈 상품 목록으로 배송 생성 불가`() {
        assertThrows<IllegalArgumentException> {
            Shipment.create(
                id = ShipmentId("test-1"),
                orderId = OrderId("order-1"),
                recipient = createTestRecipient(),
                items = emptyList(),
            )
        }
    }

    @Test
    fun `reconstitute로 복원하면 이벤트가 발생하지 않는다`() {
        val shipment = Shipment.reconstitute(
            id = ShipmentId("ship-1"),
            orderId = OrderId("order-1"),
            recipient = createTestRecipient(),
            items = listOf(ShipmentItem("prod-1", "키보드", 1, 800)),
            status = ShipmentStatus.DISPATCHED,
            trackingNumber = TrackingNumber("CJ123"),
            createdAt = Instant.now(),
        )

        assertTrue(shipment.domainEvents.isEmpty()) // 이벤트 없음!
    }

    // 테스트 헬퍼 — DB도, Spring도, Mock도 필요 없다!
    private fun createTestShipment() = Shipment.create(
        id = ShipmentId("ship-1"),
        orderId = OrderId("order-1"),
        recipient = createTestRecipient(),
        items = listOf(
            ShipmentItem("prod-1", "키보드", 1, 800),
        ),
    )

    private fun createTestRecipient() = Recipient(
        name = "홍길동",
        phone = "01012345678",
        address = Address("06000", "서울", "강남대로 1", "2층"),
    )
}
```

> 💡 **핵심**: Domain 테스트에 `@SpringBootTest`, `@MockBean`, 데이터베이스가 **전혀 필요 없다**. 순수 Kotlin 객체니까. 이것이 Hexagonal Architecture의 가장 큰 장점이다.

### 8.2 Port Mock으로 UseCase 테스트

```kotlin
class CreateShipmentServiceTest {

    // Port를 Fake으로 대체
    private val shipmentRepository = FakeShipmentRepository()
    private val eventPublisher = FakeEventPublisher()
    private val useCase = CreateShipmentService(shipmentRepository, eventPublisher)

    @Test
    fun `배송 생성 성공`() {
        val command = CreateShipmentCommand(
            orderId = "order-123",
            recipientName = "홍길동",
            recipientPhone = "01012345678",
            zipCode = "06000",
            city = "서울",
            street = "강남대로 1",
            addressDetail = "2층",
            items = listOf(
                ShipmentItemCommand("prod-1", "키보드", 1, 800),
            ),
        )

        val shipmentId = useCase.execute(command)

        // 저장 확인
        assertNotNull(shipmentRepository.findById(shipmentId))

        // 이벤트 발행 확인
        assertEquals(1, eventPublisher.publishedEvents.size)
        assertTrue(eventPublisher.publishedEvents[0] is ShipmentCreated)
    }

    // Fake 구현 — Mockito보다 직관적
    class FakeShipmentRepository : ShipmentRepository {
        private val store = mutableMapOf<ShipmentId, Shipment>()

        override fun save(shipment: Shipment) {
            store[shipment.id] = shipment
        }

        override fun findById(id: ShipmentId): Shipment? = store[id]

        override fun existsByOrderId(orderId: OrderId): Boolean =
            store.values.any { it.orderId == orderId }

        override fun nextId(): ShipmentId = ShipmentId(UUID.randomUUID().toString())
    }

    class FakeEventPublisher : DomainEventPublisher {
        val publishedEvents = mutableListOf<DomainEvent>()
        override fun publish(event: DomainEvent) {
            publishedEvents.add(event)
        }
    }
}
```

### 8.3 Adapter 통합 테스트

```kotlin
@SpringBootTest
@AutoConfigureMockMvc
class ShipmentControllerIntegrationTest {

    @Autowired lateinit var mockMvc: MockMvc
    @Autowired lateinit var objectMapper: ObjectMapper

    @Test
    fun `POST 배송 생성 API 테스트`() {
        val request = CreateShipmentRequest(
            orderId = "order-123",
            recipientName = "홍길동",
            recipientPhone = "01012345678",
            zipCode = "06000",
            city = "서울",
            street = "강남대로 1",
            addressDetail = "2층",
            items = listOf(
                ShipmentItemRequest("prod-1", "키보드", 1, 800),
            ),
        )

        mockMvc.perform(
            post("/api/shipments")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))
        )
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.shipmentId").isNotEmpty)
    }
}

// JPA Repository Adapter 테스트 — Testcontainers 추천
@DataJpaTest
class ShipmentRepositoryAdapterTest {

    @Autowired lateinit var jpaRepository: ShipmentJpaRepository

    private lateinit var adapter: ShipmentRepositoryAdapter

    @BeforeEach
    fun setUp() {
        adapter = ShipmentRepositoryAdapter(jpaRepository)
    }

    @Test
    fun `배송 저장 후 조회`() {
        val shipment = Shipment.create(
            id = adapter.nextId(),
            orderId = OrderId("order-1"),
            recipient = Recipient("홍길동", "01012345678", Address("06000", "서울", "강남대로", "1층")),
            items = listOf(ShipmentItem("prod-1", "키보드", 1, 800)),
        )

        adapter.save(shipment)

        val found = adapter.findById(shipment.id)
        assertNotNull(found)
        assertEquals(shipment.id, found!!.id)
        assertEquals(1, found.items.size)
    }
}
```

### 8.4 ArchUnit으로 의존성 규칙 강제

```kotlin
@AnalyzeClasses(packages = ["com.example.shipping"])
class ArchitectureTest {

    @ArchTest
    val `domain은 다른 패키지를 import하면 안 됨` = noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..application..", "..adapter..")

    @ArchTest
    val `domain에 Spring 어노테이션 금지` = noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAPackage("org.springframework..")

    @ArchTest
    val `adapter 간에 서로 import 금지` = noClasses()
        .that().resideInAPackage("..adapter.inbound..")
        .should().dependOnClassesThat()
        .resideInAPackage("..adapter.outbound..")
}
```

> 💡 이 테스트가 CI에서 돌면, 누군가 의존성 규칙을 어겼을 때 빌드가 실패한다. **규칙을 코드로 강제하는 것**이 가장 확실하다.

### 테스트 피라미드 정리

```
         /\
        /  \        통합 테스트 (Adapter)
       / 느림 \       → DB, API, Spring Context 필요
      /────────\     → 적게, 핵심 흐름만
     /          \    → Testcontainers, WireMock 활용
    /  UseCase   \   UseCase 테스트
   /   테스트     \   → Fake/Mock Port
  /   적당히 빠름  \  → 비즈니스 시나리오 중심
 /────────────────\
/  Domain 테스트    \  Domain 단위 테스트
/ 매우 빠름, 많이!   \  → 순수 Kotlin, 외부 의존 0
/────────────────────\
```

### 실전 테스트 도구 추천

| 도구 | 용도 | 테스트 레벨 |
|------|------|-----------|
| **JUnit 5** | 기본 테스트 프레임워크 | 전체 |
| **ArchUnit** | 아키텍처 규칙 검증 | 구조 |
| **Testcontainers** | 실제 DB(PostgreSQL 등)로 Adapter 테스트 | 통합 |
| **WireMock** | 외부 API Mock (택배사 등) | 통합 |
| **Fake 구현** | Port Mock (위 예시처럼) | UseCase |

---

## 9. AI와 Hexagonal Architecture

### 9.1 AI가 Hexagonal Architecture의 고질적 단점을 해소한다

Hexagonal Architecture의 최대 단점은 **초기 코드량**이었다. 하나의 기능을 만들려면:

- Inbound Port (UseCase 인터페이스)
- Application Service (UseCase 구현)
- Outbound Port (Repository 인터페이스)
- Driven Adapter (JPA 구현)
- JPA Entity + 매핑 코드
- Driving Adapter (Controller)
- Request/Response DTO
- Config (의존성 주입)

파일 10~20개가 기본이다. "Controller 하나면 될 것을 왜 이렇게 복잡하게?" — 모든 팀이 겪는 고민이었다.

**AI 시대에 이 단점이 완전히 사라진다:**

| Hexagonal의 전통적 단점 | AI가 해소하는 방식 |
|---|---|
| 초기 파일 대량 생성 (Port, Adapter, DTO...) | AI가 패키지 구조째로 30초 만에 생성 |
| 보일러플레이트 (Domain ↔ JPA 변환, DTO 변환) | AI가 가장 잘하는 영역 — 반복적이고 결정적인 코드 |
| 높은 러닝커브 (DIP, Port/Adapter 개념) | AI가 패턴대로 짜주니 생성된 코드를 보면서 학습 |
| 새 기능 추가 시 파일 수 부담 | 인터페이스만 정의하면 나머지는 AI가 채움 |

**반대로, AI의 코드 생성 능력은 Hexagonal의 명확한 경계가 있어야 극대화된다:**

- AI에게 "배송 기능 만들어줘"라고 하면 → 스파게티 코드
- AI에게 "이 Outbound Port의 JPA Adapter를 만들어줘"라고 하면 → 정확한 코드
- **경계가 명확할수록 AI의 출력 품질이 높아진다** — Hexagonal의 Port/Adapter 구조가 AI에게 최적의 가이드라인

> 💡 **핵심 인사이트**: Hexagonal Architecture와 AI는 **서로의 약점을 보완하는 관계**다. Hexagonal은 AI에게 명확한 인터페이스 경계를 주고, AI는 Hexagonal의 높은 초기 비용을 제거한다. AI 시대에 "코드가 많아서" Hexagonal을 포기할 이유가 사라졌다.

### 9.2 AI Native Engineer — 설계자로서의 엔지니어

과거 엔지니어와 AI Native Engineer의 역할 비중은 근본적으로 다르다:

```
과거 엔지니어:        ████████████████████ 80% 구현  ████ 20% 설계
AI Native Engineer:  ████ 20% 구현 감독  ████████████████████ 80% 도메인/비즈니스
```

Hexagonal Architecture 관점에서, **사람이 해야 하는 것과 AI가 잘하는 것**이 깔끔하게 나뉜다:

| 사람이 해야 하는 것 (설계) | AI가 잘하는 것 (구현) |
|---|---|
| Port 인터페이스 설계 (어떤 계약이 필요한지) | Adapter 구현 (JPA, REST Client, Kafka...) |
| 도메인 모델 경계 설정 | Domain ↔ JPA Entity 매핑 코드 |
| UseCase 흐름 설계 | Controller, Request/Response DTO |
| 아키텍처 규칙 정의 (CLAUDE.md) | ArchUnit 테스트, Config 파일 |
| 비즈니스 규칙 판단 | 보일러플레이트, 테스트 코드 |

AI가 구현을 맡으니, 엔지니어는 **"무엇을 만들 것인가"**에 집중하게 된다. Port를 정의하고, 도메인 경계를 긋고, 비즈니스 규칙을 판단하는 것 — 이것이 Hexagonal Architecture 설계의 본질이며, AI Native Engineer의 핵심 역량이다.

> 💡 **결론**: AI 시대의 엔지니어는 Adapter를 직접 짜는 사람이 아니라, **Port를 설계하는 사람**이다. "인터페이스 먼저, 구현은 AI" — 이것이 Hexagonal Architecture가 AI 시대에 더 빛나는 이유다.

### 9.3 "인터페이스 먼저, 구현은 AI" 패턴

Hexagonal Architecture에서 **Port(인터페이스)는 사람이 설계하고, Adapter(구현)는 AI에게 맡기는 패턴**이 매우 효과적이다.

**왜 잘 맞는가:**

- Port는 비즈니스 의도를 담음 → 사람의 판단 필요
- Adapter는 기술적 구현 → AI가 잘하는 영역
- 인터페이스가 명확하면 AI의 구현 품질이 높아짐

### 9.4 실전 프롬프트: Port 정의 → AI한테 Adapter 구현 시키기

#### 프롬프트 예시 1: Outbound Adapter 구현

```
아래 Outbound Port 인터페이스가 있어.
이것의 JPA 기반 Adapter를 구현해줘.

Port:
```kotlin
interface ShipmentRepository {
    fun save(shipment: Shipment)
    fun findById(id: ShipmentId): Shipment?
    fun findByOrderId(orderId: OrderId): List<Shipment>
    fun nextId(): ShipmentId
}
```

요구사항:
- Spring Data JPA 사용
- JPA Entity ↔ Domain Model 변환 포함
- DB에서 복원 시 reconstitute() 사용 (이벤트 중복 방지)
- UUID 기반 ID 생성
- Kotlin, 패키지: com.example.shipping.adapter.outbound.persistence

Domain 모델:
[Shipment, ShipmentItem, Recipient, Address 클래스 붙여넣기]
```

#### 프롬프트 예시 2: Driving Adapter 구현

```
아래 Inbound Port(UseCase)에 대한 REST Controller를 구현해줘.

UseCase:
```kotlin
interface CreateShipmentUseCase {
    fun execute(command: CreateShipmentCommand): ShipmentId
}
```

요구사항:
- Spring Boot REST Controller
- POST /api/shipments
- Request/Response DTO 별도 정의
- 에러 핸들링 포함
- Kotlin
```

#### 프롬프트 예시 3: DDD 모델 → Hexagonal 배치

```
DDD 가이드에서 설계한 아래 도메인 모델을 Hexagonal Architecture에 배치해줘.

[DDD 가이드의 Order, Shipment 등 코드 붙여넣기]

요구사항:
- Inbound/Outbound Port 정의
- Application Service (UseCase 구현)
- JPA Adapter (reconstitute 포함)
- 패키지 구조 포함
```

### 9.5 CLAUDE.md에 아키텍처 규칙 넣기

프로젝트에 `CLAUDE.md` (또는 AI 코딩 에이전트 설정 파일)를 만들어서 아키텍처 규칙을 명시하면, AI가 일관된 코드를 생성한다.

```markdown
# CLAUDE.md

## Architecture Rules

이 프로젝트는 Hexagonal Architecture를 사용합니다.

### 패키지 구조
- `domain/` — 순수 Kotlin, 프레임워크 import 금지
- `application/port/inbound/` — UseCase 인터페이스
- `application/port/outbound/` — Repository, 외부 API 인터페이스
- `application/service/` — UseCase 구현
- `adapter/inbound/web/` — REST Controller
- `adapter/outbound/persistence/` — JPA 구현
- `adapter/outbound/external/` — 외부 API 클라이언트
- `adapter/config/` — Spring Configuration

### 의존성 규칙
- domain은 다른 패키지를 import하면 안 됨
- application은 domain만 import 가능
- adapter는 application과 domain을 import 가능
- adapter 간에는 서로 import 금지

### 코드 규칙
- Domain Entity에 JPA 어노테이션 금지 (별도 JPA Entity 사용)
- DB 복원 시 reconstitute() 사용 (이벤트 중복 방지)
- UseCase는 인터페이스로 정의, 구현 클래스는 별도
- Adapter에 비즈니스 로직 금지 (변환만)
- 새 외부 연동 추가 시: Outbound Port 먼저 정의 → Adapter 구현
- @Transactional은 Application Service에만

### 테스트
- Domain: 단위 테스트, 외부 의존 없이
- UseCase: Fake Port로 테스트
- Adapter: @SpringBootTest 통합 테스트
- ArchUnit: 의존성 규칙 검증
```

---

## 10. 흔한 실수와 안티패턴

### 10.1 Port 없이 직접 의존

```kotlin
// ❌ UseCase가 JPA Repository에 직접 의존
class CreateShipmentService(
    private val jpaRepository: ShipmentJpaRepository, // JPA 직접 의존!
) {
    fun execute(command: CreateShipmentCommand) {
        val entity = ShipmentJpaEntity(/* ... */)
        jpaRepository.save(entity) // JPA Entity를 직접 다룸
    }
}
// 문제: DB를 MongoDB로 바꾸면 UseCase까지 다 수정해야 함
```

```kotlin
// ✅ Port(인터페이스)를 통해 의존
class CreateShipmentService(
    private val shipmentRepository: ShipmentRepository, // Port(인터페이스)에 의존
) {
    fun execute(command: CreateShipmentCommand) {
        val shipment = Shipment.create(/* ... */)
        shipmentRepository.save(shipment) // Domain 객체만 다룸
    }
}
// DB를 바꿔도 Adapter만 새로 만들면 됨
```

### 10.2 Adapter에 비즈니스 로직

```kotlin
// ❌ Controller에 배송비 계산 로직이 있음
@RestController
class ShipmentController {
    @PostMapping("/api/shipments")
    fun create(@RequestBody request: CreateShipmentRequest): ResponseEntity<*> {
        // 이게 Controller에 있으면 안 됨!
        val shippingFee = if (request.totalPrice >= 50000) 0 else 3000
        val remoteFee = if (request.zipCode.startsWith("63")) 3000 else 0
        val totalFee = shippingFee + remoteFee

        // ...
    }
}
```

```kotlin
// ✅ Controller는 변환만, 로직은 Domain에
@RestController
class ShipmentController(
    private val createShipmentUseCase: CreateShipmentUseCase,
) {
    @PostMapping("/api/shipments")
    fun create(@RequestBody request: CreateShipmentRequest): ResponseEntity<*> {
        val result = createShipmentUseCase.execute(request.toCommand())
        return ResponseEntity.ok(CreateShipmentResponse(result.value))
    }
}
```

### 10.3 DB 복원 시 create() 호출 (이벤트 중복)

```kotlin
// ❌ DB에서 읽어올 때 create() 호출 → 이벤트 중복!
private fun toDomain(entity: ShipmentJpaEntity): Shipment {
    return Shipment.create(/* ... */) // ShipmentCreated 이벤트가 또 생긴다!
}

// ✅ reconstitute()로 이벤트 없이 복원
private fun toDomain(entity: ShipmentJpaEntity): Shipment {
    return Shipment.reconstitute(/* ... */) // 이벤트 없음 ✓
}
```

### 10.4 Aggregate에 setter 방식 상태 변경 메서드

```kotlin
// ❌ 외부에서 상태값을 직접 set — Adapter나 UseCase에서 아무 상태로든 변경 가능
class Shipment {
    fun updateStatus(newStatus: ShipmentStatus) {
        status = newStatus
    }
}
// UseCase에서: shipment.updateStatus(ShipmentStatus.DELIVERED)
// → 비즈니스 의도가 없고, 검증도 빠지기 쉽고, 이벤트도 발행할 수 없다

// ✅ 도메인 행위 메서드 — 상태 검증 + 변경 + 이벤트 발행을 캡슐화
class Shipment {
    fun dispatch(trackingNumber: TrackingNumber) {
        require(status == ShipmentStatus.CREATED) { "CREATED 상태에서만 출고 가능" }
        this.status = ShipmentStatus.DISPATCHED
        this.trackingNumber = trackingNumber
        _events.add(ShipmentDispatched(id, trackingNumber, Instant.now()))
    }
}
```

> 💡 특히 Hexagonal Architecture에서는 UseCase(Application Service)가 Aggregate의 상태 변경을 조율하므로, Aggregate가 **자기 보호를 못하면** UseCase에 비즈니스 로직이 새어나간다. `updateStatus` 대신 행위 메서드를 사용하면 Aggregate가 자신의 불변식을 스스로 지킨다.

### 10.5 과도한 추상화

```kotlin
// ❌ 과도함 — 모든 것에 인터페이스
interface StringFormatter { fun format(s: String): String }
interface DateProvider { fun now(): Instant }
interface IdGenerator { fun generate(): String }
// 바뀔 일 없는 것까지 추상화하면 코드만 복잡해짐
```

**Port로 추상화할 가치가 있는 것:**

| 추상화 할 것 (변경 가능성 높음) | 하지 말 것 (변경 가능성 낮음) |
|------|------|
| DB 접근 (Repository) | 날짜/시간 유틸 |
| 외부 API (택배사, 결제) | 문자열 포맷팅 |
| 메시지 발행 (Kafka, RabbitMQ) | 로깅 |
| 파일 저장 (S3, 로컬) | 내부 헬퍼 함수 |

> 💡 **판별 기준**: "이 기술을 나중에 다른 것으로 교체할 수 있는가?" → Yes면 Port로 추상화. No면 직접 사용.

---

## 11. 참고 자료

### 📚 책

| 책 | 설명 | 난이도 |
|---|------|-------|
| **만들면서 배우는 클린 아키텍처** (톰 홈버그) | Hexagonal + Spring Boot 실전. 얇고 실용적 | ⭐ 입문 |
| **클린 아키텍처** (로버트 마틴) | 아키텍처 원칙 전반. 의존성 규칙 상세 | ⭐⭐ 중급 |
| **도메인 주도 설계 구현** (반 버논) | DDD + Hexagonal 조합 | ⭐⭐ 중급 |

### 🎓 강의 / 영상

- [만들면서 배우는 클린 아키텍처 (인프런)](https://www.inflearn.com/) — Kotlin/Java + Spring Boot
- [Alistair Cockburn - Hexagonal Architecture 원문](https://alistair.cockburn.us/hexagonal-architecture/) — 창시자의 원문
- [Netflix 기술 블로그 - Hexagonal Architecture](https://netflixtechblog.com/) — 대규모 적용 사례

### 🌐 블로그 / 글

- [Herberto Graca - DDD, Hexagonal, Clean Architecture](https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/) — 아키텍처 패턴 종합 비교 (필독!)
- [우아한형제들 기술 블로그](https://techblog.woowahan.com/) — 실전 적용기
- [카카오페이 기술 블로그](https://tech.kakaopay.com/) — Clean Architecture 도입기

### 🛠️ 레퍼런스 프로젝트

- [buckpal (GitHub, 톰 홈버그)](https://github.com/thombergs/buckpal) — 「만들면서 배우는 클린 아키텍처」 예제 코드
- [ArchUnit](https://www.archunit.org/) — 아키텍처 규칙을 코드로 검증

---

> 📌 **함께 읽기**: [DDD 학습 가이드](./ddd-learning-guide.md)에서 도메인 모델링을 먼저 익히고, 이 문서의 아키텍처에 배치하는 순서로 학습하면 효과적이다. 특히 DDD 가이드의 `reconstitute()` 패턴, 이벤트 수집-발행 패턴이 이 문서의 Repository Adapter와 어떻게 연결되는지 확인하자.

# Temporal을 활용한 Saga 패턴 학습 가이드

> 물류 이커머스 도메인 기준 | 신입 AI Native Engineer 대상  
> 언어: Kotlin (Temporal Java SDK)  
> 최종 업데이트: 2026-03-11

---

## 목차

1. [Temporal이란?](#1-temporal이란)
2. [Saga 패턴 심화](#2-saga-패턴-심화)
3. [아키텍처 설계: Modular Monolith + Temporal](#3-아키텍처-설계-modular-monolith--temporal)
4. [물류 도메인 전체 구현 (Kotlin)](#4-물류-도메인-전체-구현-kotlin)
5. [고급 패턴](#5-고급-패턴)
6. [테스트 전략](#6-테스트-전략)
7. [운영 가이드](#7-운영-가이드)
8. [AI와 Temporal](#8-ai와-temporal)
9. [흔한 실수와 안티패턴](#9-흔한-실수와-안티패턴)
10. [참고 자료](#10-참고-자료)

---

## 1. Temporal이란?

### 1.1 정의

Temporal은 **Durable Execution 플랫폼**이다. 분산 시스템에서 장시간 실행되는 비즈니스 프로세스를 **일반 코드처럼** 작성할 수 있게 해준다.

쉽게 말하면: "서버가 중간에 죽어도 코드가 멈춘 지점부터 다시 실행된다."

전통적으로 분산 트랜잭션을 구현하려면 메시지 큐, 상태 머신, 재시도 로직, 보상 트랜잭션 등을 직접 만들어야 했다. Temporal은 이 모든 인프라를 **플랫폼 레벨**에서 제공한다.

### 1.2 왜 필요한지

물류 이커머스를 예로 들자. 주문 하나를 처리하려면:

1. 주문 생성 (Order Service)
2. 재고 차감 (Inventory Service)  
3. 결제 처리 (Payment Service)
4. 배송 생성 (Shipment Service)

이 4단계가 **모두 성공**해야 주문이 완료된다. 3번(결제)에서 실패하면? 2번(재고)을 롤백해야 한다. 4번(배송)에서 실패하면? 3번(결제)과 2번(재고)을 모두 롤백해야 한다.

**Temporal 없이** 이걸 구현하면:
- 각 서비스 간 메시지 큐 연결
- 각 단계의 상태를 DB에 저장
- 실패 시 보상 이벤트 발행
- 재시도 로직 직접 구현
- 타임아웃 처리
- 중복 메시지 처리 (멱등성)
- Dead Letter Queue 모니터링
- ...수백 줄의 인프라 코드

**Temporal로** 구현하면:

```kotlin
// 이게 전부다. 진짜로.
class OrderSagaWorkflowImpl : OrderSagaWorkflow {
    override fun processOrder(order: OrderRequest): OrderResult {
        Saga.addCompensation { orderActivity.cancelOrder(order.orderId) }
        val orderId = orderActivity.createOrder(order)
        
        Saga.addCompensation { inventoryActivity.releaseStock(orderId) }
        inventoryActivity.reserveStock(orderId, order.items)
        
        Saga.addCompensation { paymentActivity.refund(orderId) }
        paymentActivity.charge(orderId, order.totalAmount)
        
        shipmentActivity.createShipment(orderId, order.address)
        
        return OrderResult(orderId, "COMPLETED")
    }
}
```

서버가 죽어도, 네트워크가 끊겨도, Temporal이 알아서 복구한다.

### 1.3 기존 메시지 큐 기반 Saga와 비교

| 비교 항목 | Kafka/RabbitMQ 기반 Saga | Temporal 기반 Saga |
|---|---|---|
| **코드 복잡도** | 높음 (이벤트 핸들러, 상태 머신, 보상 이벤트 등) | 낮음 (순차 코드로 작성) |
| **상태 관리** | 직접 구현 (DB에 saga 상태 저장) | 자동 (Temporal이 관리) |
| **재시도** | 직접 구현 (DLQ, backoff 등) | 선언적 설정 (RetryOptions) |
| **타임아웃** | 직접 구현 (스케줄러 필요) | 내장 (Activity/Workflow 타임아웃) |
| **보상 트랜잭션** | 이벤트로 직접 발행/처리 | Saga.addCompensation()으로 자동 |
| **가시성** | 로그/메트릭으로 추적 | Temporal UI에서 전체 흐름 시각화 |
| **디버깅** | 매우 어려움 (이벤트 추적 필요) | 쉬움 (Event History 제공) |
| **장기 실행** | 매우 어려움 | 기본 지원 (며칠~몇 달도 가능) |
| **테스트** | 통합 테스트 위주 (큐 필요) | 단위 테스트 가능 (TestEnvironment) |
| **학습 곡선** | 낮음 (큐는 잘 알려짐) | 중간 (Temporal 개념 학습 필요) |
| **인프라** | Kafka/RabbitMQ 클러스터 | Temporal Server 클러스터 |
| **벤더 종속** | 낮음 | 중간 (Temporal에 종속) |

**핵심 차이**: Kafka 기반은 "이벤트 중심"으로 생각해야 한다. 각 서비스가 이벤트를 발행하고 구독하며, 전체 흐름은 암묵적이다. Temporal 기반은 "코드 중심"으로 생각한다. 전체 비즈니스 흐름이 하나의 함수에 명시적으로 보인다.

### 1.4 Temporal의 핵심 개념

#### Workflow

**비즈니스 프로세스의 정의**. 주문 처리, 배송 추적 같은 전체 흐름을 코드로 표현한다.

```kotlin
@WorkflowInterface
interface OrderSagaWorkflow {
    @WorkflowMethod
    fun processOrder(request: OrderRequest): OrderResult
}
```

**핵심 제약**: Workflow 코드는 **결정적(deterministic)**이어야 한다.
- ❌ `Random()`, `System.currentTimeMillis()`, `UUID.randomUUID()`
- ❌ 네트워크 호출, DB 접근, 파일 I/O
- ❌ `Thread.sleep()`
- ✅ `Workflow.currentTimeMillis()` (Temporal이 제공하는 결정적 버전)
- ✅ `Workflow.sleep()` (내부적으로 타이머로 관리)
- ✅ Activity 호출 (I/O는 Activity에서)

왜? Temporal은 Workflow를 **재실행(replay)**해서 상태를 복구하기 때문이다. 같은 입력에 같은 결과가 나와야 한다.

#### Activity

**실제 작업을 수행하는 단위**. DB 쿼리, API 호출, 파일 처리 등 부수 효과(side effect)가 있는 모든 작업.

```kotlin
@ActivityInterface
interface PaymentActivity {
    @ActivityMethod
    fun charge(orderId: String, amount: Long): PaymentResult
    
    @ActivityMethod
    fun refund(orderId: String): RefundResult
}
```

Activity는 비결정적이어도 된다. 네트워크 호출, DB 접근 자유롭게 가능. Temporal이 Activity 실행 결과를 **Event History에 기록**하기 때문에, Workflow 재실행 시 Activity를 다시 호출하지 않고 기록된 결과를 사용한다.

#### Worker

**Workflow와 Activity를 실제로 실행하는 프로세스**. Temporal Server에 연결해서 "나 이 Task Queue의 일을 처리할 수 있어"라고 등록한다.

```kotlin
fun main() {
    val client = WorkflowClient.newInstance(
        WorkflowServiceStubs.newLocalServiceStubs()
    )
    val factory = WorkerFactory.newInstance(client)
    val worker = factory.newWorker("order-saga-queue")
    
    worker.registerWorkflowImplementationTypes(OrderSagaWorkflowImpl::class.java)
    worker.registerActivitiesImplementations(
        OrderActivityImpl(orderRepository),
        PaymentActivityImpl(paymentGateway)
    )
    
    factory.start()
}
```

#### Task Queue

Worker와 Workflow/Activity를 연결하는 **이름 기반 라우팅 메커니즘**. 같은 Task Queue에 여러 Worker를 등록하면 자동으로 로드밸런싱된다.

```
[Temporal Server]
    │
    ├── Task Queue: "order-saga-queue"
    │       ├── Worker A (pod-1)
    │       ├── Worker B (pod-2)
    │       └── Worker C (pod-3)
    │
    └── Task Queue: "shipment-tracking-queue"
            ├── Worker D (pod-4)
            └── Worker E (pod-5)
```

### 1.5 Durable Execution이란

Durable Execution은 Temporal의 핵심 가치다. **코드의 실행 상태가 영구적으로 보존된다**는 뜻이다.

#### 작동 원리

1. Workflow가 Activity를 호출하면, Temporal Server에 **이벤트**가 기록된다
2. Activity가 완료되면, 그 **결과**도 이벤트로 기록된다
3. Worker가 죽으면? 다른 Worker가 이벤트 히스토리를 **재실행(replay)**해서 같은 상태에 도달한다
4. 재실행 시 이미 완료된 Activity는 실제로 호출하지 않고, 기록된 결과를 반환한다

```
시간 ──────────────────────────────────────────────────►

Workflow 시작 → createOrder() → reserveStock() → 💥 Worker 죽음
                  결과 기록 ✓     결과 기록 ✓

                              ... 5분 후 다른 Worker가 픽업 ...

Replay: createOrder() → reserveStock() → charge() → createShipment() → 완료!
        (기록에서 복원)  (기록에서 복원)   실제 실행     실제 실행
```

> **💡 Replay의 핵심 이해**: Replay 시 Workflow 코드는 **처음부터 다시 실행**된다. 하지만 이미 완료된 Activity 호출 지점에 도달하면, Temporal이 Event History에서 기록된 결과를 반환하므로 실제 Activity 호출은 일어나지 않는다. 코드가 다시 실행되는데 왜 두 번 호출되지 않는지 — 이것이 Temporal의 마법이자 결정적(deterministic) 코드가 필수인 이유다.

#### 왜 강력한가

1. **장애 투명성**: 개발자는 "서버가 죽으면 어쩌지?"를 생각할 필요가 없다
2. **장기 실행**: 며칠, 몇 주 걸리는 프로세스도 하나의 Workflow로 표현 가능
3. **타이머**: `Workflow.sleep(Duration.ofDays(7))`이 실제로 7일 후에 깨어난다 (서버 리소스 소모 없이)
4. **코드 = 상태 머신**: 별도의 상태 관리 없이 코드 자체가 상태를 표현한다
5. **완전한 감사 추적**: 모든 실행 이벤트가 기록되어 디버깅과 감사가 용이하다

---

## 2. Saga 패턴 심화

> **이 장에서 배우는 것**: Choreography vs Orchestration 비교, 보상 트랜잭션 설계 원칙, 멱등성 구현 전략  
> **선행 지식**: 1장의 Temporal 핵심 개념

### 2.1 Choreography vs Orchestration Saga 비교

#### Choreography Saga (안무 방식)

각 서비스가 이벤트를 발행하고, 다른 서비스가 이벤트를 구독해서 다음 단계를 실행한다. **중앙 조율자가 없다**.

```
Order Service ──publish──► "OrderCreated"
                                │
Inventory Service ◄─subscribe───┘
    │
    ├──publish──► "StockReserved"
    │                   │
Payment Service ◄──subscribe──┘
    │
    ├──publish──► "PaymentCompleted"
    │                   │
Shipment Service ◄─subscribe──┘
    │
    └──publish──► "ShipmentCreated"
```

**실패 시 보상도 이벤트로:**
```
Payment Service ──publish──► "PaymentFailed"
                                  │
Inventory Service ◄──subscribe────┘
    │
    └──publish──► "StockReleased"
```

#### Orchestration Saga (오케스트라 방식)

**중앙 오케스트레이터**가 전체 흐름을 제어한다. 각 단계를 순서대로 호출하고, 실패 시 보상을 역순으로 실행한다.

```
Orchestrator (Temporal Workflow)
    │
    ├─1─► Order Service: createOrder()
    ├─2─► Inventory Service: reserveStock()
    ├─3─► Payment Service: charge()
    └─4─► Shipment Service: createShipment()
    
    ❌ 3번 실패 시:
    ├─보상─► Inventory Service: releaseStock()
    └─보상─► Order Service: cancelOrder()
```

#### 비교표

| 비교 항목 | Choreography | Orchestration |
|---|---|---|
| **흐름 가시성** | 낮음 (이벤트 추적 필요) | 높음 (Workflow 코드에 전부 보임) |
| **결합도** | 낮음 (이벤트로 느슨한 결합) | 중간 (오케스트레이터가 서비스 알아야 함) |
| **복잡도 증가** | 서비스 수 증가 시 기하급수적 | 서비스 수 증가 시 선형적 |
| **디버깅** | 매우 어려움 | 쉬움 |
| **보상 트랜잭션** | 각 서비스가 독립적으로 처리 | 오케스트레이터가 중앙에서 관리 |
| **새 단계 추가** | 모든 관련 서비스 수정 필요 가능 | 오케스트레이터만 수정 |
| **순환 의존성** | 발생 가능 | 발생 불가 (단방향) |
| **단일 장애점** | 없음 | 오케스트레이터 (단, Temporal은 HA 지원) |
| **적합한 규모** | 2-3개 서비스 | 3개 이상 서비스 |
| **테스트** | 통합 테스트 위주 | 단위 테스트 가능 |

### 2.2 왜 Orchestration이 AI 시대에 더 적합한가

**1. 코드가 곧 명세서**

AI (Copilot, Claude 등)에게 "주문 Saga를 구현해줘"라고 하면, Orchestration은 **하나의 함수**로 전체 흐름이 표현된다. AI가 이해하기 쉽고, 생성하기도 쉽다.

Choreography는 여러 서비스에 흩어진 이벤트 핸들러를 생성해야 하므로 AI가 전체 맥락을 유지하기 어렵다.

**2. 구조가 실수를 방지**

Temporal Workflow에서 `Random()`을 쓰면 즉시 에러가 난다. AI가 생성한 코드에서 비결정적 코드가 포함되면 **컴파일은 되지만 런타임에서 바로 잡힌다**. 구조 자체가 안전장치 역할을 한다.

**3. Activity 인터페이스 = AI에게 주는 계약**

Activity 인터페이스만 정의해주면, AI가 WorkflowImpl을 구현할 때 "어떤 도구를 쓸 수 있는지"가 명확하다. 마치 도구(Tool)를 정의해주고 AI에게 계획을 세우라고 하는 것과 같다.

**4. 검증 가능한 출력**

AI가 생성한 Workflow 코드는 `TestWorkflowEnvironment`로 바로 단위 테스트 가능하다. Choreography는 전체 인프라를 띄워야 검증 가능하다.

**5. AI가 보일러플레이트를 해소한다**

Temporal/Saga 패턴의 대표적 단점은 **보일러플레이트 코드가 많다**는 것이다 — ActivityOptions 설정, Saga 보상 등록, RetryOptions 선언, Worker 설정 등. AI 코드 생성이 이 문제를 근본적으로 해소한다. Activity 인터페이스와 도메인 흐름만 설계하면, Workflow 구현체·Worker 설정·테스트 코드를 AI가 자동 생성할 수 있다.

> **AI Native Engineer의 역할**: 도메인 흐름을 설계하고 Activity 인터페이스(계약)를 정의하는 것에 집중한다. Workflow 구현, 보상 로직, 재시도 설정 같은 반복적 코드는 AI가 생성한다. 엔지니어는 **"무엇을 해야 하는가"**를 결정하고, AI는 **"어떻게 구현하는가"**를 담당한다.

### 2.3 보상 트랜잭션 설계 원칙

#### 원칙 1: 보상은 "취소"가 아니라 "되돌리기"

결제 보상 ≠ 결제 취소. 이미 실행된 작업의 **효과를 상쇄**하는 새로운 작업이다.

```kotlin
// ❌ 잘못된 생각: 결제를 "없던 것"으로 만듦
fun cancelPayment(paymentId: String) {
    paymentRepository.delete(paymentId) // 기록이 사라짐!
}

// ✅ 올바른 방법: 환불이라는 새로운 트랜잭션 생성
fun refundPayment(paymentId: String): RefundResult {
    val payment = paymentRepository.findById(paymentId)
    val refund = paymentGateway.refund(payment.transactionId, payment.amount)
    paymentRepository.save(
        PaymentRecord(
            orderId = payment.orderId,
            type = PaymentType.REFUND,
            amount = -payment.amount,
            originalPaymentId = paymentId
        )
    )
    return RefundResult(refund.id)
}
```

#### 원칙 2: 보상은 반드시 성공해야 한다

보상 트랜잭션이 실패하면 시스템이 불일치 상태에 빠진다. 따라서:

- 보상 Activity에도 재시도 정책을 설정한다
- 최종적으로 실패하면 수동 개입 알림을 보낸다
- 보상 자체가 멱등적이어야 한다

#### 원칙 3: 보상 순서는 역순

```kotlin
// 실행: A → B → C
// 보상: C보상 → B보상 → A보상
Saga.addCompensation { orderActivity.cancelOrder(orderId) }    // 3번째 보상
val orderId = orderActivity.createOrder(request)

Saga.addCompensation { inventoryActivity.releaseStock(orderId) } // 2번째 보상
inventoryActivity.reserveStock(orderId, items)

Saga.addCompensation { paymentActivity.refund(orderId) }        // 1번째 보상
paymentActivity.charge(orderId, amount)
```

`Saga.addCompensation`은 **스택처럼** 동작한다. 마지막에 추가된 보상이 먼저 실행된다.

#### 원칙 4: 마지막 단계는 보상이 필요 없을 수 있다

마지막 단계(배송 생성)가 실패하면, 그 단계 자체는 실행되지 않았으므로 보상할 것이 없다. 이전 단계들만 보상하면 된다.

```kotlin
// 배송 생성이 실패하면 → 결제 환불 → 재고 복원 → 주문 취소
// 배송 생성에 대한 보상은 불필요 (실패했으니까)
shipmentActivity.createShipment(orderId, address) // 여기서 예외 발생 시 위의 보상들이 역순 실행
```

### 2.4 멱등성의 중요성

**멱등성(Idempotency)**: 같은 작업을 여러 번 실행해도 결과가 동일한 성질.

Temporal은 Activity를 **재시도**할 수 있다. 네트워크 타임아웃으로 결과를 못 받았지만 실제로는 성공했을 수 있다. 이때 재시도하면 **두 번 실행**된다.

```kotlin
// ❌ 멱등적이지 않음: 재시도 시 이중 결제!
class PaymentActivityImpl : PaymentActivity {
    override fun charge(orderId: String, amount: Long): PaymentResult {
        return paymentGateway.charge(amount) // 매번 새로운 결제
    }
}

// ✅ 멱등적: orderId를 idempotency key로 사용
class PaymentActivityImpl : PaymentActivity {
    override fun charge(orderId: String, amount: Long): PaymentResult {
        // 이미 결제된 건이면 기존 결과 반환
        val existing = paymentRepository.findByOrderId(orderId)
        if (existing != null) return existing.toResult()
        
        val result = paymentGateway.charge(
            amount = amount,
            idempotencyKey = "order-$orderId" // PG사에 멱등키 전달
        )
        paymentRepository.save(result.toRecord(orderId))
        return result
    }
}
```

**멱등성 구현 전략:**
1. **Idempotency Key**: 주문 ID 같은 비즈니스 키를 사용해 중복 실행 방지
2. **상태 확인 후 실행**: 이미 처리된 건인지 먼저 확인
3. **DB Unique Constraint**: 중복 삽입 방지
4. **외부 API Idempotency Key**: PG사, 배송 API 등에 멱등키 전달

> 멱등성은 이 가이드 전체에서 반복적으로 등장한다. 4장의 ActivityImpl 구현과 9장의 안티패턴에서도 다루지만, **모든 Activity는 멱등적으로 구현한다**는 원칙은 여기서 확립하고 간다.

---

*Saga 패턴의 원칙을 이해했으니, 이제 이를 실제 코드 구조로 어떻게 옮기는지 살펴보자. 핵심은 Workflow와 Activity를 물리적으로 분리하는 아키텍처다.*

---

## 3. 아키텍처 설계: Modular Monolith + Temporal

> **이 장에서 배우는 것**: repo 분리 전략, 역할 오염 방지, Task Queue 라우팅 원리  
> **선행 지식**: 1장의 핵심 개념, 2장의 Orchestration Saga

### 3.1 Repo 분리 전략

Temporal을 도입할 때 가장 중요한 아키텍처 결정은 **repo를 어떻게 나누느냐**다.

#### 두 개의 repo

```
📁 saga-orchestrator/          ← Temporal Workflow만
│   ├── shared-activity-api/   ← Activity 인터페이스 (공유 모듈)
│   ├── order-saga/            ← WorkflowImpl
│   └── shipment-tracking/     ← WorkflowImpl
│
📁 modular-monolith/           ← 비즈니스 로직 + ActivityImpl
    ├── shared-activity-api/   ← Activity 인터페이스 (같은 모듈)
    ├── order-module/          ← OrderActivityImpl + 도메인 로직
    ├── inventory-module/      ← InventoryActivityImpl + 도메인 로직
    ├── payment-module/        ← PaymentActivityImpl + 도메인 로직
    └── shipment-module/       ← ShipmentActivityImpl + 도메인 로직
```

`shared-activity-api`는 Activity 인터페이스와 DTO만 담긴 경량 모듈이다. 양쪽 repo에서 **공유 라이브러리(Maven/Gradle artifact)**로 참조한다.

#### 왜 이렇게 나누는가?

**saga-orchestrator repo**에는:
- `@WorkflowInterface`, `@WorkflowMethod` 어노테이션이 붙은 인터페이스
- `WorkflowImpl` 클래스
- Workflow 시작/관리하는 API 코드
- **DB 의존성이 없다** (JPA, JDBC 등 일절 없음)
- **외부 API 클라이언트가 없다** (HTTP client, gRPC client 등 없음)

**modular-monolith repo**에는:
- `@ActivityInterface` 구현 클래스 (ActivityImpl)
- 도메인 모델, 리포지토리, 서비스
- DB 접근, 외부 API 호출 등 모든 I/O
- **다른 모듈의 Activity를 직접 호출하지 않는다**

### 3.2 왜 분리하는가: 역할 오염 방지

#### Workflow에서 DB 접근 불가

Temporal Workflow는 결정적이어야 한다. DB를 직접 조회하면:
- 같은 쿼리라도 시간에 따라 결과가 달라진다 (비결정적)
- Replay 시 DB를 다시 조회하면 다른 결과가 나올 수 있다

**repo 분리로 이를 물리적으로 차단한다.** saga-orchestrator repo에 JPA 의존성 자체가 없으므로, 개발자가 실수로라도 DB를 접근할 수 없다.

#### Activity에서 다른 Activity 호출 불가

Activity는 "원자적 작업 단위"다. Activity A가 Activity B를 호출하면:
- 재시도 범위가 모호해진다 (A를 재시도하면 B도 다시 실행)
- 보상 범위가 불명확해진다
- 타임아웃 관리가 복잡해진다

Activity 간 조율은 반드시 **Workflow에서** 해야 한다. modular-monolith repo에서 ActivityImpl은 자기 모듈의 서비스만 호출한다.

### 3.3 구조가 규칙을 강제하는 설계 철학

> "규칙을 문서에 적는 것보다, 구조로 강제하는 것이 낫다."

코드 리뷰에서 "Workflow에서 DB 접근하지 마세요"라고 매번 지적하는 것보다, **repo에 DB 라이브러리가 없어서 물리적으로 불가능**하게 만드는 것이 효과적이다.

이 원칙은 AI 코딩 시대에 더욱 중요하다. AI가 코드를 생성할 때:
- saga-orchestrator repo에서는 DB 접근 코드를 생성할 수 없다 (import 자체가 불가)
- modular-monolith repo에서는 Workflow를 정의할 수 없다 (Workflow 관련 의존성 없음)

**구조 자체가 AI의 실수를 방지한다.**

> **💡 설계 철학 — 규칙을 문서가 아니라 구조로 강제한다**
> 
> repo 분리는 단순한 코드 정리가 아니다. **역할 오염을 구조적으로 불가능하게 만드는 설계**다. 문서에 "하지 마라"라고 쓰는 것은 사람도 AI도 어길 수 있다. 하지만 의존성 자체가 없으면 물리적으로 어길 수 없다. 이것이 saga repo와 monolith repo를 분리하는 근본적 이유이며, AI 시대에 더욱 빛을 발하는 설계 원칙이다.

### 3.4 배포 구조와 Task Queue Poller 분리 원리

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                        │
│                                                                   │
│  ┌──────────────────┐     ┌──────────────────────────────────┐  │
│  │  Temporal Server  │     │        Application Pods           │  │
│  │  ┌──────────┐    │     │                                    │  │
│  │  │ Frontend │    │     │  ┌─────────────────────────────┐  │  │
│  │  │  (gRPC)  │◄───┼─────┼──┤  saga-orchestrator Worker   │  │  │
│  │  └──────────┘    │     │  │  ┌───────────────────────┐  │  │  │
│  │  ┌──────────┐    │     │  │  │ OrderSagaWorkflowImpl │  │  │  │
│  │  │ History  │    │     │  │  │ ShipmentTrackingImpl   │  │  │  │
│  │  │ Service  │    │     │  │  └───────────────────────┘  │  │  │
│  │  └──────────┘    │     │  │  Task Queue: order-saga     │  │  │
│  │  ┌──────────┐    │     │  └─────────────────────────────┘  │  │
│  │  │ Matching │    │     │                                    │  │
│  │  │ Service  │    │     │  ┌─────────────────────────────┐  │  │
│  │  └──────────┘    │     │  │  modular-monolith Worker    │  │  │
│  │  ┌──────────┐    │     │  │  ┌───────────────────────┐  │  │  │
│  │  │Cassandra/│    │     │  │  │  OrderActivityImpl    │  │  │  │
│  │  │PostgreSQL│    │     │  │  │  InventoryActivityImpl│  │  │  │
│  │  └──────────┘    │     │  │  │  PaymentActivityImpl  │  │  │  │
│  │                  │     │  │  │  ShipmentActivityImpl │  │  │  │
│  │  ┌──────────┐    │     │  │  └───────────────────────┘  │  │  │
│  │  │Temporal  │    │     │  │  Task Queue: order-saga     │  │  │
│  │  │   UI     │    │     │  └──────────────┬──────────────┘  │  │
│  │  └──────────┘    │     │                 │                  │  │
│  └──────────────────┘     │                 ▼                  │  │
│                           │  ┌──────────────────────────────┐  │  │
│                           │  │     PostgreSQL (비즈니스 DB)  │  │  │
│                           │  └──────────────────────────────┘  │  │
│                           └──────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

#### Task Queue Poller 분리: "같은 큐인데 충돌 안 하나요?"

saga-orchestrator Worker와 modular-monolith Worker가 **같은 Task Queue**(예: `order-saga-queue`)를 polling하는데 어떻게 충돌하지 않을까?

> **핵심**: Temporal은 내부적으로 **Workflow Task Poller**와 **Activity Task Poller**를 분리해서 큐잉한다.

하나의 Task Queue 이름 안에 실제로는 두 종류의 큐가 존재한다:
- **Workflow Task Queue**: Workflow의 다음 단계를 결정하는 태스크 (WorkflowImpl이 등록된 Worker만 수신)
- **Activity Task Queue**: 실제 작업을 실행하는 태스크 (ActivityImpl이 등록된 Worker만 수신)

```
Task Queue: "order-saga-queue"
    │
    ├── [Workflow Task Poller] ← saga-orchestrator Worker가 수신
    │       "OrderSagaWorkflow의 다음 단계를 결정해줘"
    │
    └── [Activity Task Poller] ← modular-monolith Worker가 수신
            "PaymentActivity.charge()를 실행해줘"
```

Worker를 생성할 때 `registerWorkflowImplementationTypes()`만 호출하면 Workflow Task만 수신하고, `registerActivitiesImplementations()`만 호출하면 Activity Task만 수신한다. 따라서 양쪽 Worker가 같은 큐 이름을 써도 **절대 충돌하지 않는다**.

또는 Task Queue를 명시적으로 분리할 수도 있다:
- `order-saga-workflow-queue`: Workflow 전용
- `order-saga-activity-queue`: Activity 전용

### 3.5 CLAUDE.md 규칙 예시

#### saga-orchestrator repo의 CLAUDE.md

```markdown
# CLAUDE.md — saga-orchestrator

## 이 repo의 역할
Temporal Workflow 정의만 담는다. 비즈니스 로직의 "흐름(orchestration)"을 표현한다.

## 절대 금지 사항
1. **DB 접근 코드 금지**: JPA, JDBC, R2DBC, 어떤 DB 라이브러리도 사용하지 않는다
2. **HTTP 클라이언트 금지**: RestTemplate, WebClient, Retrofit 등 외부 API 직접 호출 금지
3. **파일 I/O 금지**: 파일 읽기/쓰기 금지
4. **비결정적 코드 금지**:
   - `Random()`, `UUID.randomUUID()` → `Workflow.randomUUID()` 사용
   - `System.currentTimeMillis()` → `Workflow.currentTimeMillis()` 사용
   - `Thread.sleep()` → `Workflow.sleep()` 사용
5. **Activity 구현 금지**: `@ActivityInterface`의 구현 클래스를 만들지 않는다

## 허용 사항
- `@WorkflowInterface`, `@WorkflowMethod` 정의
- `WorkflowImpl` 클래스 구현
- Activity **인터페이스** 사용 (Stub으로 호출)
- `Saga.addCompensation()` 패턴
- Child Workflow 호출
- Signal, Query 핸들러 정의
- Workflow 타이머, 슬립

## 코드 패턴
- Activity 호출 시 반드시 ActivityOptions (타임아웃, 재시도) 설정
- 모든 Saga 단계에 보상 등록
- WorkflowImpl은 상태를 멤버 변수로 관리 (Temporal이 직렬화)

## 테스트
- TestWorkflowEnvironment 사용
- Activity는 Mock으로 대체
```

#### modular-monolith repo의 CLAUDE.md

```markdown
# CLAUDE.md — modular-monolith

## 이 repo의 역할
비즈니스 도메인 로직과 Temporal Activity 구현을 담는다.

## 절대 금지 사항
1. **Workflow 정의 금지**: `@WorkflowInterface`, `WorkflowImpl` 만들지 않는다
2. **다른 모듈의 Activity 직접 호출 금지**: 
   - OrderActivityImpl에서 InventoryActivityImpl을 직접 호출하지 않는다
   - 모듈 간 통신이 필요하면 Workflow에서 조율한다
3. **Activity에서 다른 Activity 호출 금지**: Activity는 원자적 단위
4. **Workflow 관련 import 금지**: `io.temporal.workflow.*` import하지 않는다
   (단, `io.temporal.activity.*`는 허용)

## 허용 사항
- `@ActivityInterface` 구현 (ActivityImpl)
- DB 접근 (JPA, JDBC 등)
- 외부 API 호출 (PG사, 배송 API 등)
- 모듈 내부 서비스/리포지토리 사용

## 코드 패턴
- 모든 Activity는 멱등적으로 구현
- idempotency key로 중복 실행 방지
- Activity 메서드는 JSON 직렬화 가능한 파라미터/반환값만 사용 (Kotlin data class)

## 모듈 구조
- 각 모듈은 독립적인 패키지 (order, inventory, payment, shipment)
- 모듈 간 공유가 필요한 것은 shared-activity-api로
```

---

*아키텍처를 설계했으니 이제 실제 코드를 작성한다. 4장에서는 공유 DTO부터 WorkflowImpl, ActivityImpl, Worker 설정까지 전체 구현을 다룬다.*

---

## 4. 물류 도메인 전체 구현 (Kotlin)

> **이 장에서 배우는 것**: 공유 DTO/인터페이스 설계, WorkflowImpl 전체 구현, ActivityImpl 멱등 구현, Worker 설정  
> **이 장이 긴 이유**: 실무에서 바로 참고할 수 있는 완성된 코드를 제공하기 위함

### 4.1 유즈케이스

**주문 처리 Saga**:

```
고객이 주문 → 주문 생성 → 재고 차감 → 결제 → 배송 생성
                                        │
                                   실패 시 보상:
                                   결제 환불 → 재고 복원 → 주문 취소
```

### 4.2 공유 모듈: Activity 인터페이스 정의

`shared-activity-api` 모듈에 Activity 인터페이스와 DTO를 정의한다. 이 모듈은 양쪽 repo에서 참조한다.

#### DTO 정의

> **💡 직렬화 방식**: Temporal의 기본 DataConverter는 **Jackson JSON 직렬화**를 사용한다. `java.io.Serializable`은 불필요하다. 순수 Kotlin data class로 정의하면 된다.

```kotlin
// shared-activity-api/src/main/kotlin/com/example/saga/api/dto/OrderDtos.kt

package com.example.saga.api.dto

data class OrderRequest(
    val customerId: String,
    val items: List<OrderItem>,
    val shippingAddress: Address,
    val totalAmount: Long // 원 단위
)

data class OrderItem(
    val productId: String,
    val quantity: Int,
    val unitPrice: Long
)

data class Address(
    val zipCode: String,
    val street: String,
    val detail: String,
    val city: String
)

data class OrderResult(
    val orderId: String,
    val status: String,
    val shipmentId: String? = null,
    val failureReason: String? = null
)

data class CreateOrderResult(
    val orderId: String
)

data class ReserveStockResult(
    val reservationId: String
)

data class PaymentResult(
    val paymentId: String,
    val transactionId: String
)

data class RefundResult(
    val refundId: String
)

data class ShipmentResult(
    val shipmentId: String,
    val trackingNumber: String
)
```

#### Activity 인터페이스

```kotlin
// shared-activity-api/src/main/kotlin/com/example/saga/api/activity/OrderActivity.kt

package com.example.saga.api.activity

import com.example.saga.api.dto.*
import io.temporal.activity.ActivityInterface
import io.temporal.activity.ActivityMethod

@ActivityInterface
interface OrderActivity {
    @ActivityMethod
    fun createOrder(request: OrderRequest): CreateOrderResult

    @ActivityMethod
    fun cancelOrder(orderId: String)

    @ActivityMethod
    fun completeOrder(orderId: String)
}
```

```kotlin
// shared-activity-api/src/main/kotlin/com/example/saga/api/activity/InventoryActivity.kt

package com.example.saga.api.activity

import com.example.saga.api.dto.*
import io.temporal.activity.ActivityInterface
import io.temporal.activity.ActivityMethod

@ActivityInterface
interface InventoryActivity {
    @ActivityMethod
    fun reserveStock(orderId: String, items: List<OrderItem>): ReserveStockResult

    @ActivityMethod
    fun releaseStock(orderId: String)

    @ActivityMethod
    fun confirmStockDeduction(orderId: String)
}
```

```kotlin
// shared-activity-api/src/main/kotlin/com/example/saga/api/activity/PaymentActivity.kt

package com.example.saga.api.activity

import com.example.saga.api.dto.*
import io.temporal.activity.ActivityInterface
import io.temporal.activity.ActivityMethod

@ActivityInterface
interface PaymentActivity {
    @ActivityMethod
    fun charge(orderId: String, amount: Long): PaymentResult

    @ActivityMethod
    fun refund(orderId: String): RefundResult
}
```

```kotlin
// shared-activity-api/src/main/kotlin/com/example/saga/api/activity/ShipmentActivity.kt

package com.example.saga.api.activity

import com.example.saga.api.dto.*
import io.temporal.activity.ActivityInterface
import io.temporal.activity.ActivityMethod

@ActivityInterface
interface ShipmentActivity {
    @ActivityMethod
    fun createShipment(orderId: String, address: Address): ShipmentResult

    @ActivityMethod
    fun cancelShipment(shipmentId: String)
}
```

#### Workflow 인터페이스

```kotlin
// shared-activity-api/src/main/kotlin/com/example/saga/api/workflow/OrderSagaWorkflow.kt

package com.example.saga.api.workflow

import com.example.saga.api.dto.OrderRequest
import com.example.saga.api.dto.OrderResult
import io.temporal.workflow.QueryMethod
import io.temporal.workflow.SignalMethod
import io.temporal.workflow.WorkflowInterface
import io.temporal.workflow.WorkflowMethod

@WorkflowInterface
interface OrderSagaWorkflow {
    @WorkflowMethod
    fun processOrder(request: OrderRequest): OrderResult

    @QueryMethod
    fun getStatus(): String

    @SignalMethod
    fun cancel(reason: String)
}
```

### 4.3 WorkflowImpl 전체 코드 (saga-orchestrator repo)

```kotlin
// saga-orchestrator/order-saga/src/main/kotlin/com/example/saga/workflow/OrderSagaWorkflowImpl.kt

package com.example.saga.workflow

import com.example.saga.api.activity.*
import com.example.saga.api.dto.*
import com.example.saga.api.workflow.OrderSagaWorkflow
import io.temporal.activity.ActivityOptions
import io.temporal.common.RetryOptions
import io.temporal.failure.ActivityFailure
import io.temporal.failure.ApplicationFailure
import io.temporal.workflow.Saga
import io.temporal.workflow.Workflow
import org.slf4j.LoggerFactory
import java.time.Duration

class OrderSagaWorkflowImpl : OrderSagaWorkflow {

    private val logger = Workflow.getLogger(OrderSagaWorkflowImpl::class.java)

    // ──────────────────────────────────────────────
    // Activity Stub 설정
    // ──────────────────────────────────────────────

    private val defaultActivityOptions = ActivityOptions.newBuilder()
        .setStartToCloseTimeout(Duration.ofSeconds(30))    // Activity 실행 최대 시간
        .setScheduleToCloseTimeout(Duration.ofMinutes(5))  // 스케줄~완료 최대 시간
        .setRetryOptions(
            RetryOptions.newBuilder()
                .setInitialInterval(Duration.ofSeconds(1))     // 첫 재시도 간격
                .setMaximumInterval(Duration.ofSeconds(30))    // 최대 재시도 간격
                .setBackoffCoefficient(2.0)                     // 지수 백오프
                .setMaximumAttempts(3)                           // 최대 재시도 횟수
                .setDoNotRetry(
                    // 비즈니스 에러는 재시도하지 않음
                    "INSUFFICIENT_STOCK",
                    "PAYMENT_DECLINED",
                    "INVALID_ADDRESS"
                )
                .build()
        )
        .build()

    // 보상 Activity는 더 공격적으로 재시도 (반드시 성공해야 하므로)
    private val compensationActivityOptions = ActivityOptions.newBuilder()
        .setStartToCloseTimeout(Duration.ofMinutes(1))
        .setScheduleToCloseTimeout(Duration.ofMinutes(10))
        .setRetryOptions(
            RetryOptions.newBuilder()
                .setInitialInterval(Duration.ofSeconds(1))
                .setMaximumInterval(Duration.ofMinutes(1))
                .setBackoffCoefficient(2.0)
                .setMaximumAttempts(10)  // 보상은 많이 재시도
                .build()
        )
        .build()

    private val orderActivity = Workflow.newActivityStub(
        OrderActivity::class.java, defaultActivityOptions
    )
    private val inventoryActivity = Workflow.newActivityStub(
        InventoryActivity::class.java, defaultActivityOptions
    )
    private val paymentActivity = Workflow.newActivityStub(
        PaymentActivity::class.java, defaultActivityOptions
    )
    private val shipmentActivity = Workflow.newActivityStub(
        ShipmentActivity::class.java, defaultActivityOptions
    )

    // 보상용 Activity Stub (더 공격적인 재시도)
    private val orderCompensation = Workflow.newActivityStub(
        OrderActivity::class.java, compensationActivityOptions
    )
    private val inventoryCompensation = Workflow.newActivityStub(
        InventoryActivity::class.java, compensationActivityOptions
    )
    private val paymentCompensation = Workflow.newActivityStub(
        PaymentActivity::class.java, compensationActivityOptions
    )

    // ──────────────────────────────────────────────
    // Workflow 상태
    // ──────────────────────────────────────────────

    private var status = "INITIALIZED"
    private var cancelRequested = false
    private var cancelReason: String? = null

    // ──────────────────────────────────────────────
    // Signal / Query 핸들러
    // ──────────────────────────────────────────────

    override fun getStatus(): String = status

    override fun cancel(reason: String) {
        cancelRequested = true
        cancelReason = reason
    }

    // ──────────────────────────────────────────────
    // 메인 Workflow 로직
    // ──────────────────────────────────────────────

    override fun processOrder(request: OrderRequest): OrderResult {
        val sagaOptions = Saga.Options.Builder()
            .setParallelCompensation(false)  // 보상을 순차 실행 (역순)
            .setContinueWithError(true)       // 보상 중 에러 발생해도 나머지 보상 계속 실행
            .build()
        val saga = Saga(sagaOptions)

        try {
            // ──── Step 1: 주문 생성 ────
            status = "CREATING_ORDER"
            logger.info("Step 1: 주문 생성 시작")
            val createOrderResult = orderActivity.createOrder(request)
            val orderId = createOrderResult.orderId
            saga.addCompensation { orderCompensation.cancelOrder(orderId) }
            logger.info("Step 1: 주문 생성 완료 - orderId={}", orderId)

            // 취소 요청 확인
            checkCancellation(saga, orderId)

            // ──── Step 2: 재고 차감 ────
            status = "RESERVING_STOCK"
            logger.info("Step 2: 재고 차감 시작 - orderId={}", orderId)
            // Replay 시: 이 라인은 실행되지만, Activity는 실제 호출 없이 Event History에서 결과를 복원한다
            val reserveResult = inventoryActivity.reserveStock(orderId, request.items)
            saga.addCompensation { inventoryCompensation.releaseStock(orderId) }
            logger.info("Step 2: 재고 차감 완료 - reservationId={}", reserveResult.reservationId)

            // 취소 요청 확인
            checkCancellation(saga, orderId)

            // ──── Step 3: 결제 ────
            status = "PROCESSING_PAYMENT"
            logger.info("Step 3: 결제 시작 - orderId={}, amount={}", orderId, request.totalAmount)
            // Replay 시: charge()도 실제 호출 없이 기록된 PaymentResult를 반환한다
            val paymentResult = paymentActivity.charge(orderId, request.totalAmount)
            saga.addCompensation { paymentCompensation.refund(orderId) }
            logger.info("Step 3: 결제 완료 - paymentId={}", paymentResult.paymentId)

            // 취소 요청 확인
            checkCancellation(saga, orderId)

            // ──── Step 4: 배송 생성 ────
            status = "CREATING_SHIPMENT"
            logger.info("Step 4: 배송 생성 시작 - orderId={}", orderId)
            val shipmentResult = shipmentActivity.createShipment(orderId, request.shippingAddress)
            logger.info("Step 4: 배송 생성 완료 - shipmentId={}", shipmentResult.shipmentId)

            // ──── Step 5: 최종 확정 ────
            status = "FINALIZING"
            // 확정 단계도 Saga 안에서 관리 — 실패 시 재시도로 복구
            inventoryActivity.confirmStockDeduction(orderId) // 예약 → 확정
            orderActivity.completeOrder(orderId)

            status = "COMPLETED"
            logger.info("주문 처리 완료 - orderId={}", orderId)

            return OrderResult(
                orderId = orderId,
                status = "COMPLETED",
                shipmentId = shipmentResult.shipmentId
            )

        } catch (e: ActivityFailure) {
            logger.error("주문 처리 실패, 보상 트랜잭션 시작", e)
            status = "COMPENSATING"
            saga.compensate()
            status = "FAILED"

            val reason = extractFailureReason(e)
            // 실패하더라도 이미 생성된 orderId를 반환해서 운영에서 추적 가능하게 한다
            val failedOrderId = extractOrderIdFromContext()
            return OrderResult(
                orderId = failedOrderId,
                status = "FAILED",
                failureReason = reason
            )
        }
    }

    private var createdOrderId: String? = null

    private fun extractOrderIdFromContext(): String {
        return createdOrderId ?: ""
    }

    private fun checkCancellation(saga: Saga, orderId: String) {
        if (cancelRequested) {
            logger.info("취소 요청 감지 - orderId={}, reason={}", orderId, cancelReason)
            status = "CANCELLING"
            saga.compensate()
            status = "CANCELLED"
            throw ApplicationFailure.newFailure(
                "주문이 취소되었습니다: $cancelReason",
                "ORDER_CANCELLED"
            )
        }
    }

    private fun extractFailureReason(e: ActivityFailure): String {
        val cause = e.cause
        return if (cause is ApplicationFailure) {
            "${cause.type}: ${cause.message}"
        } else {
            e.message ?: "Unknown error"
        }
    }
}
```

#### Child Workflow 활용 예시

배송 추적처럼 장기 실행이 필요한 경우 Child Workflow로 분리한다:

```kotlin
// saga-orchestrator/order-saga/src/main/kotlin/com/example/saga/workflow/OrderSagaWithChildWorkflowImpl.kt

package com.example.saga.workflow

import com.example.saga.api.dto.*
import com.example.saga.api.workflow.OrderSagaWorkflow
import com.example.saga.api.workflow.ShipmentTrackingWorkflow
import io.temporal.workflow.Async
import io.temporal.workflow.ChildWorkflowOptions
import io.temporal.workflow.Workflow
import java.time.Duration

class OrderSagaWithChildWorkflowImpl : OrderSagaWorkflow {
    
    // ... (Activity Stub 설정 동일) ...
    
    private var status = "INITIALIZED"
    private var cancelRequested = false
    private var cancelReason: String? = null
    
    override fun getStatus(): String = status
    override fun cancel(reason: String) {
        cancelRequested = true
        cancelReason = reason
    }

    override fun processOrder(request: OrderRequest): OrderResult {
        // ... (Saga 단계 1~4 동일) ...
        
        // Step 5: 배송 추적 Child Workflow 시작 (비동기)
        val childOptions = ChildWorkflowOptions.newBuilder()
            .setWorkflowId("shipment-tracking-${shipmentResult.shipmentId}")
            .setTaskQueue("shipment-tracking-queue")
            .build()
        
        val trackingWorkflow = Workflow.newChildWorkflowStub(
            ShipmentTrackingWorkflow::class.java, childOptions
        )
        
        // 비동기로 시작 — 배송 추적은 며칠 걸릴 수 있으므로 기다리지 않음
        Async.function { trackingWorkflow.trackShipment(shipmentResult.shipmentId) }
        
        status = "COMPLETED"
        return OrderResult(
            orderId = orderId,
            status = "COMPLETED",
            shipmentId = shipmentResult.shipmentId
        )
    }
}
```

### 4.4 ActivityImpl 전체 코드 (modular-monolith repo)

#### OrderActivityImpl

```kotlin
// modular-monolith/order-module/src/main/kotlin/com/example/order/activity/OrderActivityImpl.kt

package com.example.order.activity

import com.example.order.domain.Order
import com.example.order.domain.OrderStatus
import com.example.order.repository.OrderRepository
import com.example.saga.api.activity.OrderActivity
import com.example.saga.api.dto.CreateOrderResult
import com.example.saga.api.dto.OrderRequest
import io.temporal.failure.ApplicationFailure
import org.springframework.stereotype.Component
import java.util.UUID

@Component
class OrderActivityImpl(
    private val orderRepository: OrderRepository
) : OrderActivity {

    override fun createOrder(request: OrderRequest): CreateOrderResult {
        // 멱등성: 같은 customerId + items 조합으로 이미 생성된 주문이 있으면 반환
        val existingOrder = orderRepository.findByCustomerIdAndItemsHash(
            request.customerId,
            request.items.hashCode().toString()
        )
        if (existingOrder != null && existingOrder.status != OrderStatus.CANCELLED) {
            return CreateOrderResult(existingOrder.id)
        }

        val order = Order(
            id = UUID.randomUUID().toString(),
            customerId = request.customerId,
            items = request.items.map { item ->
                Order.LineItem(
                    productId = item.productId,
                    quantity = item.quantity,
                    unitPrice = item.unitPrice
                )
            },
            totalAmount = request.totalAmount,
            shippingAddress = request.shippingAddress.let { addr ->
                Order.Address(addr.zipCode, addr.street, addr.detail, addr.city)
            },
            status = OrderStatus.CREATED,
            itemsHash = request.items.hashCode().toString()
        )

        orderRepository.save(order)
        return CreateOrderResult(order.id)
    }

    override fun cancelOrder(orderId: String) {
        val order = orderRepository.findById(orderId)
            ?: return // 이미 없으면 멱등하게 무시

        if (order.status == OrderStatus.CANCELLED) {
            return // 이미 취소됨 — 멱등
        }

        order.status = OrderStatus.CANCELLED
        orderRepository.save(order)
    }

    override fun completeOrder(orderId: String) {
        val order = orderRepository.findById(orderId)
            ?: throw ApplicationFailure.newNonRetryableFailure(
                "주문을 찾을 수 없습니다: $orderId",
                "ORDER_NOT_FOUND"
            )

        if (order.status == OrderStatus.COMPLETED) {
            return // 멱등
        }

        order.status = OrderStatus.COMPLETED
        orderRepository.save(order)
    }
}
```

#### InventoryActivityImpl

```kotlin
// modular-monolith/inventory-module/src/main/kotlin/com/example/inventory/activity/InventoryActivityImpl.kt

package com.example.inventory.activity

import com.example.inventory.domain.StockReservation
import com.example.inventory.domain.ReservationStatus
import com.example.inventory.repository.ProductRepository
import com.example.inventory.repository.StockReservationRepository
import com.example.saga.api.activity.InventoryActivity
import com.example.saga.api.dto.OrderItem
import com.example.saga.api.dto.ReserveStockResult
import io.temporal.failure.ApplicationFailure
import org.springframework.stereotype.Component
import org.springframework.transaction.annotation.Transactional
import java.util.UUID

@Component
class InventoryActivityImpl(
    private val productRepository: ProductRepository,
    private val reservationRepository: StockReservationRepository
) : InventoryActivity {

    @Transactional
    override fun reserveStock(orderId: String, items: List<OrderItem>): ReserveStockResult {
        // 멱등성: 이미 예약된 건이 있으면 반환
        val existing = reservationRepository.findByOrderId(orderId)
        if (existing != null && existing.status == ReservationStatus.RESERVED) {
            return ReserveStockResult(existing.id)
        }

        // 재고 확인 및 차감
        for (item in items) {
            val product = productRepository.findByIdWithLock(item.productId)
                ?: throw ApplicationFailure.newNonRetryableFailure(
                    "상품을 찾을 수 없습니다: ${item.productId}",
                    "PRODUCT_NOT_FOUND"
                )

            if (product.availableStock < item.quantity) {
                throw ApplicationFailure.newNonRetryableFailure(
                    "재고 부족: ${item.productId} (요청: ${item.quantity}, 가용: ${product.availableStock})",
                    "INSUFFICIENT_STOCK"
                )
            }

            product.availableStock -= item.quantity
            product.reservedStock += item.quantity
            productRepository.save(product)
        }

        // 예약 기록 저장
        val reservation = StockReservation(
            id = UUID.randomUUID().toString(),
            orderId = orderId,
            items = items.map { StockReservation.Item(it.productId, it.quantity) },
            status = ReservationStatus.RESERVED
        )
        reservationRepository.save(reservation)

        return ReserveStockResult(reservation.id)
    }

    @Transactional
    override fun releaseStock(orderId: String) {
        val reservation = reservationRepository.findByOrderId(orderId)
            ?: return // 예약이 없으면 멱등하게 무시

        if (reservation.status == ReservationStatus.RELEASED) {
            return // 이미 해제됨 — 멱등
        }

        // 재고 복원
        for (item in reservation.items) {
            val product = productRepository.findByIdWithLock(item.productId) ?: continue
            product.availableStock += item.quantity
            product.reservedStock -= item.quantity
            productRepository.save(product)
        }

        reservation.status = ReservationStatus.RELEASED
        reservationRepository.save(reservation)
    }

    @Transactional
    override fun confirmStockDeduction(orderId: String) {
        val reservation = reservationRepository.findByOrderId(orderId)
            ?: throw ApplicationFailure.newNonRetryableFailure(
                "예약을 찾을 수 없습니다: $orderId",
                "RESERVATION_NOT_FOUND"
            )

        if (reservation.status == ReservationStatus.CONFIRMED) {
            return // 멱등
        }

        // 예약된 재고를 확정 차감으로 전환
        for (item in reservation.items) {
            val product = productRepository.findByIdWithLock(item.productId) ?: continue
            product.reservedStock -= item.quantity
            // availableStock은 이미 줄어든 상태
            productRepository.save(product)
        }

        reservation.status = ReservationStatus.CONFIRMED
        reservationRepository.save(reservation)
    }
}
```

#### PaymentActivityImpl

```kotlin
// modular-monolith/payment-module/src/main/kotlin/com/example/payment/activity/PaymentActivityImpl.kt

package com.example.payment.activity

import com.example.payment.domain.PaymentRecord
import com.example.payment.domain.PaymentType
import com.example.payment.gateway.PaymentGateway
import com.example.payment.repository.PaymentRepository
import com.example.saga.api.activity.PaymentActivity
import com.example.saga.api.dto.PaymentResult
import com.example.saga.api.dto.RefundResult
import io.temporal.failure.ApplicationFailure
import org.springframework.stereotype.Component
import java.util.UUID

@Component
class PaymentActivityImpl(
    private val paymentGateway: PaymentGateway,
    private val paymentRepository: PaymentRepository
) : PaymentActivity {

    override fun charge(orderId: String, amount: Long): PaymentResult {
        // 멱등성: 이미 결제된 건 반환
        val existing = paymentRepository.findByOrderIdAndType(orderId, PaymentType.CHARGE)
        if (existing != null) {
            return PaymentResult(existing.id, existing.transactionId)
        }

        // PG사 결제 요청 (idempotency key 포함)
        val pgResult = try {
            paymentGateway.charge(
                amount = amount,
                idempotencyKey = "charge-order-$orderId",
                description = "주문 결제: $orderId"
            )
        } catch (e: PaymentGateway.PaymentDeclinedException) {
            throw ApplicationFailure.newNonRetryableFailure(
                "결제 거절: ${e.message}",
                "PAYMENT_DECLINED"
            )
        } catch (e: PaymentGateway.PaymentGatewayException) {
            // 일시적 오류 → 재시도 가능
            throw ApplicationFailure.newFailure(
                "결제 게이트웨이 오류: ${e.message}",
                "PAYMENT_GATEWAY_ERROR"
            )
        }

        // 결제 기록 저장
        val record = PaymentRecord(
            id = UUID.randomUUID().toString(),
            orderId = orderId,
            type = PaymentType.CHARGE,
            amount = amount,
            transactionId = pgResult.transactionId
        )
        paymentRepository.save(record)

        return PaymentResult(record.id, pgResult.transactionId)
    }

    override fun refund(orderId: String): RefundResult {
        // 원래 결제 건 조회
        val chargeRecord = paymentRepository.findByOrderIdAndType(orderId, PaymentType.CHARGE)
            ?: return RefundResult("no-charge") // 결제 건이 없으면 환불 불필요 — 멱등

        // 이미 환불된 건 확인
        val existingRefund = paymentRepository.findByOrderIdAndType(orderId, PaymentType.REFUND)
        if (existingRefund != null) {
            return RefundResult(existingRefund.id) // 멱등
        }

        // PG사 환불 요청
        val pgResult = paymentGateway.refund(
            transactionId = chargeRecord.transactionId,
            amount = chargeRecord.amount,
            idempotencyKey = "refund-order-$orderId"
        )

        // 환불 기록 저장
        val refundRecord = PaymentRecord(
            id = UUID.randomUUID().toString(),
            orderId = orderId,
            type = PaymentType.REFUND,
            amount = -chargeRecord.amount,
            transactionId = pgResult.transactionId,
            originalPaymentId = chargeRecord.id
        )
        paymentRepository.save(refundRecord)

        return RefundResult(refundRecord.id)
    }
}
```

#### ShipmentActivityImpl

```kotlin
// modular-monolith/shipment-module/src/main/kotlin/com/example/shipment/activity/ShipmentActivityImpl.kt

package com.example.shipment.activity

import com.example.saga.api.activity.ShipmentActivity
import com.example.saga.api.dto.Address
import com.example.saga.api.dto.ShipmentResult
import com.example.shipment.domain.Shipment
import com.example.shipment.domain.ShipmentStatus
import com.example.shipment.gateway.DeliveryGateway
import com.example.shipment.repository.ShipmentRepository
import io.temporal.failure.ApplicationFailure
import org.springframework.stereotype.Component
import java.util.UUID

@Component
class ShipmentActivityImpl(
    private val deliveryGateway: DeliveryGateway,
    private val shipmentRepository: ShipmentRepository
) : ShipmentActivity {

    override fun createShipment(orderId: String, address: Address): ShipmentResult {
        // 멱등성: 이미 생성된 배송 건 반환
        val existing = shipmentRepository.findByOrderId(orderId)
        if (existing != null && existing.status != ShipmentStatus.CANCELLED) {
            return ShipmentResult(existing.id, existing.trackingNumber)
        }

        // 주소 유효성 검증
        val isValid = deliveryGateway.validateAddress(
            zipCode = address.zipCode,
            street = address.street,
            city = address.city
        )
        if (!isValid) {
            throw ApplicationFailure.newNonRetryableFailure(
                "유효하지 않은 배송 주소입니다",
                "INVALID_ADDRESS"
            )
        }

        // 배송 생성 요청
        val deliveryResult = deliveryGateway.createDelivery(
            orderId = orderId,
            address = DeliveryGateway.DeliveryAddress(
                zipCode = address.zipCode,
                street = address.street,
                detail = address.detail,
                city = address.city
            ),
            idempotencyKey = "shipment-order-$orderId"
        )

        // 배송 기록 저장
        val shipment = Shipment(
            id = UUID.randomUUID().toString(),
            orderId = orderId,
            trackingNumber = deliveryResult.trackingNumber,
            status = ShipmentStatus.CREATED,
            deliveryId = deliveryResult.deliveryId
        )
        shipmentRepository.save(shipment)

        return ShipmentResult(shipment.id, deliveryResult.trackingNumber)
    }

    override fun cancelShipment(shipmentId: String) {
        val shipment = shipmentRepository.findById(shipmentId)
            ?: return // 멱등

        if (shipment.status == ShipmentStatus.CANCELLED) {
            return // 멱등
        }

        if (shipment.status == ShipmentStatus.IN_TRANSIT) {
            throw ApplicationFailure.newNonRetryableFailure(
                "이미 배송 중인 건은 취소할 수 없습니다",
                "SHIPMENT_IN_TRANSIT"
            )
        }

        deliveryGateway.cancelDelivery(shipment.deliveryId)
        shipment.status = ShipmentStatus.CANCELLED
        shipmentRepository.save(shipment)
    }
}
```

### 4.5 Worker 설정 코드

#### saga-orchestrator Worker

```kotlin
// saga-orchestrator/src/main/kotlin/com/example/saga/SagaOrchestratorApplication.kt

package com.example.saga

import com.example.saga.workflow.OrderSagaWorkflowImpl
import io.temporal.client.WorkflowClient
import io.temporal.serviceclient.WorkflowServiceStubs
import io.temporal.serviceclient.WorkflowServiceStubsOptions
import io.temporal.worker.WorkerFactory
import io.temporal.worker.WorkerOptions

fun main() {
    // Temporal Server 연결
    val serviceStubs = WorkflowServiceStubs.newServiceStubs(
        WorkflowServiceStubsOptions.newBuilder()
            .setTarget("temporal-server:7233")  // Temporal gRPC 주소
            .build()
    )

    val client = WorkflowClient.newInstance(serviceStubs)
    val factory = WorkerFactory.newInstance(client)

    // Worker 생성 — Workflow만 등록
    val worker = factory.newWorker(
        "order-saga-queue",
        WorkerOptions.newBuilder()
            .setMaxConcurrentWorkflowTaskExecutionSize(100)
            .build()
    )

    // Workflow 구현체 등록 (Activity는 등록하지 않음!)
    worker.registerWorkflowImplementationTypes(
        OrderSagaWorkflowImpl::class.java
    )

    factory.start()
    println("Saga Orchestrator Worker started, listening on 'order-saga-queue'")
}
```

#### modular-monolith Worker (Spring Boot)

```kotlin
// modular-monolith/src/main/kotlin/com/example/monolith/config/TemporalWorkerConfig.kt

package com.example.monolith.config

import com.example.saga.api.activity.*
import io.temporal.client.WorkflowClient
import io.temporal.serviceclient.WorkflowServiceStubs
import io.temporal.serviceclient.WorkflowServiceStubsOptions
import io.temporal.worker.WorkerFactory
import io.temporal.worker.WorkerOptions
import org.springframework.beans.factory.annotation.Value
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import jakarta.annotation.PostConstruct
import jakarta.annotation.PreDestroy

@Configuration
class TemporalWorkerConfig(
    @Value("\${temporal.server.target}") private val temporalTarget: String,
    @Value("\${temporal.worker.task-queue}") private val taskQueue: String,
    private val orderActivity: OrderActivity,
    private val inventoryActivity: InventoryActivity,
    private val paymentActivity: PaymentActivity,
    private val shipmentActivity: ShipmentActivity
) {

    private lateinit var factory: WorkerFactory

    @Bean
    fun workflowServiceStubs(): WorkflowServiceStubs {
        return WorkflowServiceStubs.newServiceStubs(
            WorkflowServiceStubsOptions.newBuilder()
                .setTarget(temporalTarget)
                .build()
        )
    }

    @Bean
    fun workflowClient(stubs: WorkflowServiceStubs): WorkflowClient {
        return WorkflowClient.newInstance(stubs)
    }

    @PostConstruct
    fun startWorker() {
        val client = workflowClient(workflowServiceStubs())
        factory = WorkerFactory.newInstance(client)

        val worker = factory.newWorker(
            taskQueue,
            WorkerOptions.newBuilder()
                .setMaxConcurrentActivityExecutionSize(200)
                .build()
        )

        // Activity 구현체만 등록 (Workflow는 등록하지 않음!)
        worker.registerActivitiesImplementations(
            orderActivity,
            inventoryActivity,
            paymentActivity,
            shipmentActivity
        )

        factory.start()
    }

    @PreDestroy
    fun stopWorker() {
        factory.shutdown()
    }
}
```

#### Workflow 시작 API

```kotlin
// modular-monolith/src/main/kotlin/com/example/monolith/api/OrderController.kt

package com.example.monolith.api

import com.example.saga.api.dto.OrderRequest
import com.example.saga.api.dto.OrderResult
import com.example.saga.api.workflow.OrderSagaWorkflow
import io.temporal.client.WorkflowClient
import io.temporal.client.WorkflowOptions
import org.springframework.web.bind.annotation.*
import java.time.Duration

@RestController
@RequestMapping("/api/orders")
class OrderController(
    private val workflowClient: WorkflowClient
) {

    @PostMapping
    fun createOrder(@RequestBody request: OrderRequest): OrderStartResponse {
        val workflowId = "order-saga-${request.customerId}-${System.currentTimeMillis()}"

        val workflow = workflowClient.newWorkflowStub(
            OrderSagaWorkflow::class.java,
            WorkflowOptions.newBuilder()
                .setWorkflowId(workflowId)
                .setTaskQueue("order-saga-queue")
                .setWorkflowExecutionTimeout(Duration.ofMinutes(30))
                .build()
        )

        // 비동기 시작
        val execution = WorkflowClient.start(workflow::processOrder, request)

        return OrderStartResponse(
            workflowId = workflowId,
            runId = execution.runId
        )
    }

    @GetMapping("/{workflowId}/status")
    fun getOrderStatus(@PathVariable workflowId: String): OrderStatusResponse {
        val workflow = workflowClient.newWorkflowStub(
            OrderSagaWorkflow::class.java,
            workflowId
        )
        return OrderStatusResponse(
            workflowId = workflowId,
            status = workflow.getStatus()
        )
    }

    @PostMapping("/{workflowId}/cancel")
    fun cancelOrder(
        @PathVariable workflowId: String,
        @RequestBody request: CancelRequest
    ) {
        val workflow = workflowClient.newWorkflowStub(
            OrderSagaWorkflow::class.java,
            workflowId
        )
        workflow.cancel(request.reason)
    }

    data class OrderStartResponse(val workflowId: String, val runId: String)
    data class OrderStatusResponse(val workflowId: String, val status: String)
    data class CancelRequest(val reason: String)
}
```

### 4.6 보상 시나리오별 흐름

#### 시나리오 1: 결제 실패 (잔액 부족)

```
1. ✅ createOrder()     → orderId: "ORD-001"
2. ✅ reserveStock()    → reservationId: "RSV-001"  
3. ❌ charge()          → PAYMENT_DECLINED (잔액 부족)
   
   🔄 보상 시작 (역순):
   ├── releaseStock("ORD-001")   ← 재고 복원
   └── cancelOrder("ORD-001")    ← 주문 취소
   
   결과: OrderResult(orderId="ORD-001", status="FAILED", failureReason="PAYMENT_DECLINED: 잔액 부족")
```

#### 시나리오 2: 재고 부족

```
1. ✅ createOrder()     → orderId: "ORD-002"
2. ❌ reserveStock()    → INSUFFICIENT_STOCK (재고 부족)
   
   🔄 보상 시작 (역순):
   └── cancelOrder("ORD-002")    ← 주문 취소
   
   결과: OrderResult(orderId="ORD-002", status="FAILED", failureReason="INSUFFICIENT_STOCK: ...")
```

#### 시나리오 3: 배송 생성 실패 (잘못된 주소)

```
1. ✅ createOrder()     → orderId: "ORD-003"
2. ✅ reserveStock()    → reservationId: "RSV-003"
3. ✅ charge()          → paymentId: "PAY-003"
4. ❌ createShipment()  → INVALID_ADDRESS
   
   🔄 보상 시작 (역순):
   ├── refund("ORD-003")          ← 결제 환불
   ├── releaseStock("ORD-003")    ← 재고 복원
   └── cancelOrder("ORD-003")     ← 주문 취소
   
   결과: OrderResult(orderId="ORD-003", status="FAILED", failureReason="INVALID_ADDRESS: ...")
```

#### 시나리오 4: 중간에 Worker 장애

```
1. ✅ createOrder()      → orderId: "ORD-004" (결과 기록됨)
2. ✅ reserveStock()     → reservationId: "RSV-004" (결과 기록됨)
3.    charge() 실행 중... 💥 Worker 죽음
   
   ... 30초 후 다른 Worker가 Workflow를 픽업 ...
   
   Replay:
   1. createOrder()   → Event History에서 결과 복원 (실제 호출 안 함)
   2. reserveStock()  → Event History에서 결과 복원 (실제 호출 안 함)
   3. charge()        → 결과가 기록 안 됐으므로 실제 호출 (멱등성 필요!)
   4. createShipment() → 실제 호출
   
   결과: 정상 완료 ✅
```

---

## 5. 고급 패턴

> **이 장에서 배우는 것**: 장기 실행 Workflow, Signal/Query, Timer, Versioning, Continue-As-New

### 5.1 긴 실행 Workflow: 배송 추적

배송은 며칠이 걸린다. Temporal Workflow는 이런 장기 프로세스를 자연스럽게 표현한다.

```kotlin
// shared-activity-api
@WorkflowInterface
interface ShipmentTrackingWorkflow {
    @WorkflowMethod
    fun trackShipment(shipmentId: String): ShipmentTrackingResult

    @QueryMethod
    fun getCurrentStatus(): ShipmentTrackingStatus

    @SignalMethod
    fun updateStatus(status: ShipmentStatusUpdate)
}

data class ShipmentTrackingStatus(
    val shipmentId: String,
    val currentStatus: String,
    val location: String?,
    val estimatedDelivery: String?,
    val history: List<StatusHistoryEntry>
)

data class StatusHistoryEntry(
    val status: String,
    val location: String?,
    val timestamp: Long
)

data class ShipmentStatusUpdate(
    val status: String,
    val location: String?
)

data class ShipmentTrackingResult(
    val shipmentId: String,
    val finalStatus: String,
    val deliveredAt: Long?
)
```

```kotlin
// saga-orchestrator repo
class ShipmentTrackingWorkflowImpl : ShipmentTrackingWorkflow {

    private val trackingActivity = Workflow.newActivityStub(
        ShipmentTrackingActivity::class.java,
        ActivityOptions.newBuilder()
            .setStartToCloseTimeout(Duration.ofSeconds(30))
            .build()
    )

    private var currentStatus = ShipmentTrackingStatus(
        shipmentId = "",
        currentStatus = "CREATED",
        location = null,
        estimatedDelivery = null,
        history = emptyList()
    )
    private var isDelivered = false

    override fun getCurrentStatus(): ShipmentTrackingStatus = currentStatus

    override fun updateStatus(status: ShipmentStatusUpdate) {
        // Signal로 외부(배송사 webhook 등)에서 상태 업데이트
        val entry = StatusHistoryEntry(
            status = status.status,
            location = status.location,
            timestamp = Workflow.currentTimeMillis()
        )
        currentStatus = currentStatus.copy(
            currentStatus = status.status,
            location = status.location,
            history = currentStatus.history + entry
        )

        if (status.status == "DELIVERED") {
            isDelivered = true
        }

        // 상태 변경을 DB에도 기록
        trackingActivity.recordStatusChange(currentStatus.shipmentId, status)
    }

    override fun trackShipment(shipmentId: String): ShipmentTrackingResult {
        currentStatus = currentStatus.copy(shipmentId = shipmentId)

        // 배송 완료 또는 7일 타임아웃까지 대기
        val deadline = Workflow.currentTimeMillis() + Duration.ofDays(7).toMillis()

        while (!isDelivered && Workflow.currentTimeMillis() < deadline) {
            // 12시간마다 배송사 API로 상태 폴링
            Workflow.sleep(Duration.ofHours(12))

            if (!isDelivered) {
                val polledStatus = trackingActivity.pollDeliveryStatus(shipmentId)
                updateStatus(ShipmentStatusUpdate(polledStatus.status, polledStatus.location))
            }
        }

        if (!isDelivered) {
            // 7일 경과 — 배송 지연 알림
            trackingActivity.notifyDeliveryDelay(shipmentId)
        }

        return ShipmentTrackingResult(
            shipmentId = shipmentId,
            finalStatus = currentStatus.currentStatus,
            deliveredAt = if (isDelivered) Workflow.currentTimeMillis() else null
        )
    }
}
```

**핵심 포인트**: 이 Workflow는 최대 7일 동안 실행된다. 하지만 `Workflow.sleep()` 중에는 서버 리소스를 전혀 소모하지 않는다. Temporal이 타이머로 관리하다가 시간이 되면 Worker를 깨운다.

### 5.2 Signal/Query 활용

#### Signal: 외부에서 Workflow 상태 변경

배송사 webhook이 호출되면, 해당 Workflow에 Signal을 보낸다:

```kotlin
// Webhook Controller
@RestController
@RequestMapping("/webhook/delivery")
class DeliveryWebhookController(
    private val workflowClient: WorkflowClient
) {
    @PostMapping
    fun handleDeliveryUpdate(@RequestBody event: DeliveryWebhookEvent) {
        val workflowId = "shipment-tracking-${event.shipmentId}"
        
        val workflow = workflowClient.newWorkflowStub(
            ShipmentTrackingWorkflow::class.java,
            workflowId
        )
        
        // Signal 전송 — Workflow가 sleep 중이어도 깨워서 처리
        workflow.updateStatus(
            ShipmentStatusUpdate(
                status = event.status,
                location = event.location
            )
        )
    }
}
```

#### Query: 외부에서 Workflow 상태 조회

```kotlin
// API Controller
@GetMapping("/shipments/{shipmentId}/tracking")
fun getTracking(@PathVariable shipmentId: String): ShipmentTrackingStatus {
    val workflowId = "shipment-tracking-$shipmentId"
    
    val workflow = workflowClient.newWorkflowStub(
        ShipmentTrackingWorkflow::class.java,
        workflowId
    )
    
    // Query — Workflow 실행을 방해하지 않고 현재 상태만 조회
    return workflow.getCurrentStatus()
}
```

**Signal vs Query 차이**:
- **Signal**: Workflow의 상태를 **변경**한다. 비동기로 처리되며, Workflow가 sleep 중이면 깨운다
- **Query**: Workflow의 상태를 **읽기만** 한다. Event History에 기록되지 않는다 (읽기 전용)

### 5.3 Timer 활용: 30분 내 미결제 → 자동 취소

```kotlin
@WorkflowInterface
interface PaymentWaitingWorkflow {
    @WorkflowMethod
    fun waitForPayment(orderId: String, amount: Long): PaymentWaitResult

    @SignalMethod
    fun confirmPayment(paymentInfo: PaymentConfirmation)
}

class PaymentWaitingWorkflowImpl : PaymentWaitingWorkflow {

    private var paymentConfirmed = false
    private var paymentInfo: PaymentConfirmation? = null

    override fun confirmPayment(paymentInfo: PaymentConfirmation) {
        this.paymentConfirmed = true
        this.paymentInfo = paymentInfo
    }

    override fun waitForPayment(orderId: String, amount: Long): PaymentWaitResult {
        // 30분 동안 결제 확인을 기다림
        val paymentReceived = Workflow.await(Duration.ofMinutes(30)) { paymentConfirmed }

        return if (paymentReceived) {
            // 결제 확인됨 → 다음 단계 진행
            PaymentWaitResult(
                orderId = orderId,
                status = "PAID",
                paymentId = paymentInfo!!.paymentId
            )
        } else {
            // 30분 초과 → 자동 취소
            val cancelActivity = Workflow.newActivityStub(
                OrderActivity::class.java,
                ActivityOptions.newBuilder()
                    .setStartToCloseTimeout(Duration.ofSeconds(30))
                    .build()
            )
            cancelActivity.cancelOrder(orderId)

            PaymentWaitResult(
                orderId = orderId,
                status = "CANCELLED_TIMEOUT",
                paymentId = null
            )
        }
    }
}
```

`Workflow.await(timeout) { condition }`:
- `condition`이 `true`가 되면 즉시 반환 (`true`)
- `timeout`이 지나도 `condition`이 `false`면 반환 (`false`)
- 기다리는 동안 서버 리소스 소모 없음

### 5.4 Versioning: Workflow 코드 변경 시 하위 호환

이미 실행 중인 Workflow가 있을 때 코드를 변경하면 문제가 생긴다. Workflow는 Replay로 상태를 복구하는데, 코드가 바뀌면 다른 결과가 나올 수 있기 때문이다.

```kotlin
class OrderSagaWorkflowImpl : OrderSagaWorkflow {
    override fun processOrder(request: OrderRequest): OrderResult {
        // ... Step 1, 2, 3 동일 ...

        // Version 1에서는 배송 생성만 했음
        // Version 2에서는 배송 생성 전에 주소 검증 단계 추가
        val version = Workflow.getVersion(
            "add-address-validation",  // 변경 식별자
            Workflow.DEFAULT_VERSION,   // 최소 지원 버전
            1                            // 최대 (현재) 버전
        )

        if (version >= 1) {
            // 새 코드: 주소 검증 추가
            val validationActivity = Workflow.newActivityStub(
                AddressValidationActivity::class.java,
                defaultActivityOptions
            )
            validationActivity.validateAddress(request.shippingAddress)
        }

        // 배송 생성 (기존 코드)
        val shipmentResult = shipmentActivity.createShipment(orderId, request.shippingAddress)
        
        // ...
    }
}
```

**작동 원리**:
- 이미 실행 중인 Workflow (version에 대한 이벤트가 없음): `DEFAULT_VERSION` 반환 → 주소 검증 건너뜀
- 새로 시작되는 Workflow: version `1` 반환 → 주소 검증 실행
- 이 방식으로 기존 Workflow와 새 Workflow가 **동시에 정상 동작**한다

### 5.5 Continue-As-New: 무한 루프 Workflow

Workflow Event History는 무한히 커질 수 없다 (기본 50,000 이벤트 제한). 무한히 실행되는 Workflow는 주기적으로 "새 실행"으로 교체해야 한다.

```kotlin
@WorkflowInterface
interface InventoryMonitorWorkflow {
    @WorkflowMethod
    fun monitor(config: MonitorConfig)

    @SignalMethod
    fun updateConfig(newConfig: MonitorConfig)
}

class InventoryMonitorWorkflowImpl : InventoryMonitorWorkflow {

    private lateinit var config: MonitorConfig
    private var iterationCount = 0

    override fun updateConfig(newConfig: MonitorConfig) {
        this.config = newConfig
    }

    override fun monitor(config: MonitorConfig) {
        this.config = config

        val monitorActivity = Workflow.newActivityStub(
            InventoryMonitorActivity::class.java,
            ActivityOptions.newBuilder()
                .setStartToCloseTimeout(Duration.ofMinutes(1))
                .build()
        )

        // 100번 반복 후 Continue-As-New
        while (iterationCount < 100) {
            // 5분마다 재고 체크
            Workflow.sleep(Duration.ofMinutes(5))
            
            val report = monitorActivity.checkLowStock(this.config.threshold)
            if (report.lowStockItems.isNotEmpty()) {
                monitorActivity.sendAlert(report)
            }
            
            iterationCount++
        }

        // Event History가 커지기 전에 새 실행으로 교체
        Workflow.continueAsNew(this.config)
        // 이 이후 코드는 실행되지 않음
    }
}
```

**Continue-As-New**는:
- 현재 Workflow를 종료하고 같은 타입의 새 Workflow를 즉시 시작한다
- Event History가 리셋된다 (깨끗한 시작)
- 외부에서 보면 같은 Workflow ID로 계속 실행되는 것처럼 보인다
- 상태는 파라미터로 전달해야 한다 (멤버 변수는 리셋됨)

---

## 6. 테스트 전략

> **이 장에서 배우는 것**: Workflow 단위 테스트, Activity Mock, 보상 로직 검증, Time Skipping, 통합 테스트

### 6.1 Workflow 단위 테스트 (TestWorkflowEnvironment)

```kotlin
// saga-orchestrator/src/test/kotlin/com/example/saga/workflow/OrderSagaWorkflowTest.kt

package com.example.saga.workflow

import com.example.saga.api.activity.*
import com.example.saga.api.dto.*
import com.example.saga.api.workflow.OrderSagaWorkflow
import io.temporal.client.WorkflowOptions
import io.temporal.testing.TestWorkflowEnvironment
import io.temporal.testing.TestWorkflowExtension
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.extension.RegisterExtension
import org.mockito.Mockito.*
import kotlin.test.assertEquals

class OrderSagaWorkflowTest {

    private lateinit var testEnv: TestWorkflowEnvironment
    private lateinit var orderActivity: OrderActivity
    private lateinit var inventoryActivity: InventoryActivity
    private lateinit var paymentActivity: PaymentActivity
    private lateinit var shipmentActivity: ShipmentActivity

    @BeforeEach
    fun setUp() {
        testEnv = TestWorkflowEnvironment.newInstance()

        // Activity Mock 생성
        orderActivity = mock(OrderActivity::class.java)
        inventoryActivity = mock(InventoryActivity::class.java)
        paymentActivity = mock(PaymentActivity::class.java)
        shipmentActivity = mock(ShipmentActivity::class.java)

        // Worker 등록
        val worker = testEnv.newWorker("order-saga-queue")
        worker.registerWorkflowImplementationTypes(OrderSagaWorkflowImpl::class.java)
        worker.registerActivitiesImplementations(
            orderActivity,
            inventoryActivity,
            paymentActivity,
            shipmentActivity
        )

        testEnv.start()
    }

    @AfterEach
    fun tearDown() {
        testEnv.close()
    }

    @Test
    fun `정상 주문 처리 - 모든 단계 성공`() {
        // Given
        val request = createTestOrderRequest()

        `when`(orderActivity.createOrder(request))
            .thenReturn(CreateOrderResult("ORD-001"))
        `when`(inventoryActivity.reserveStock("ORD-001", request.items))
            .thenReturn(ReserveStockResult("RSV-001"))
        `when`(paymentActivity.charge("ORD-001", 50000L))
            .thenReturn(PaymentResult("PAY-001", "TXN-001"))
        `when`(shipmentActivity.createShipment("ORD-001", request.shippingAddress))
            .thenReturn(ShipmentResult("SHP-001", "TRACK-001"))

        // When
        val workflow = testEnv.workflowClient.newWorkflowStub(
            OrderSagaWorkflow::class.java,
            WorkflowOptions.newBuilder()
                .setWorkflowId("test-order-1")
                .setTaskQueue("order-saga-queue")
                .build()
        )
        val result = workflow.processOrder(request)

        // Then
        assertEquals("COMPLETED", result.status)
        assertEquals("ORD-001", result.orderId)
        assertEquals("SHP-001", result.shipmentId)

        // 보상이 호출되지 않았는지 확인
        verify(orderActivity, never()).cancelOrder(anyString())
        verify(inventoryActivity, never()).releaseStock(anyString())
        verify(paymentActivity, never()).refund(anyString())
    }

    @Test
    fun `결제 실패 시 재고 복원 및 주문 취소`() {
        // Given
        val request = createTestOrderRequest()

        `when`(orderActivity.createOrder(request))
            .thenReturn(CreateOrderResult("ORD-002"))
        `when`(inventoryActivity.reserveStock("ORD-002", request.items))
            .thenReturn(ReserveStockResult("RSV-002"))
        `when`(paymentActivity.charge("ORD-002", 50000L))
            .thenThrow(
                io.temporal.failure.ApplicationFailure.newNonRetryableFailure(
                    "잔액 부족", "PAYMENT_DECLINED"
                )
            )

        // When
        val workflow = testEnv.workflowClient.newWorkflowStub(
            OrderSagaWorkflow::class.java,
            WorkflowOptions.newBuilder()
                .setWorkflowId("test-order-2")
                .setTaskQueue("order-saga-queue")
                .build()
        )
        val result = workflow.processOrder(request)

        // Then
        assertEquals("FAILED", result.status)
        assert(result.failureReason!!.contains("PAYMENT_DECLINED"))

        // 보상이 역순으로 호출되었는지 확인
        val inOrder = inOrder(inventoryActivity, orderActivity)
        inOrder.verify(inventoryActivity).releaseStock("ORD-002")
        inOrder.verify(orderActivity).cancelOrder("ORD-002")

        // 결제 환불은 결제 자체가 실패했으므로 호출되지 않아야 함
        verify(paymentActivity, never()).refund(anyString())
    }

    @Test
    fun `재고 부족 시 주문만 취소`() {
        // Given
        val request = createTestOrderRequest()

        `when`(orderActivity.createOrder(request))
            .thenReturn(CreateOrderResult("ORD-003"))
        `when`(inventoryActivity.reserveStock("ORD-003", request.items))
            .thenThrow(
                io.temporal.failure.ApplicationFailure.newNonRetryableFailure(
                    "재고 부족", "INSUFFICIENT_STOCK"
                )
            )

        // When
        val workflow = testEnv.workflowClient.newWorkflowStub(
            OrderSagaWorkflow::class.java,
            WorkflowOptions.newBuilder()
                .setWorkflowId("test-order-3")
                .setTaskQueue("order-saga-queue")
                .build()
        )
        val result = workflow.processOrder(request)

        // Then
        assertEquals("FAILED", result.status)
        verify(orderActivity).cancelOrder("ORD-003")
        verify(inventoryActivity, never()).releaseStock(anyString())
        verify(paymentActivity, never()).charge(anyString(), anyLong())
    }

    private fun createTestOrderRequest() = OrderRequest(
        customerId = "CUST-001",
        items = listOf(
            OrderItem("PROD-001", 2, 15000),
            OrderItem("PROD-002", 1, 20000)
        ),
        shippingAddress = Address("06234", "테헤란로 123", "4층", "서울"),
        totalAmount = 50000
    )
}
```

### 6.2 Time Skipping 테스트 — Temporal의 킬러 테스팅 기능

`Workflow.sleep(Duration.ofDays(7))`을 포함한 Workflow를 테스트한다면? 실제로 7일을 기다릴 수 없다. Temporal의 `TestWorkflowEnvironment`는 **Time Skipping**을 지원한다 — 가상 시간을 사용해서 sleep/timer를 밀리초 단위로 건너뛴다.

```kotlin
class ShipmentTrackingWorkflowTest {

    private lateinit var testEnv: TestWorkflowEnvironment
    private lateinit var trackingActivity: ShipmentTrackingActivity

    @BeforeEach
    fun setUp() {
        testEnv = TestWorkflowEnvironment.newInstance()
        trackingActivity = mock(ShipmentTrackingActivity::class.java)

        val worker = testEnv.newWorker("shipment-tracking-queue")
        worker.registerWorkflowImplementationTypes(ShipmentTrackingWorkflowImpl::class.java)
        worker.registerActivitiesImplementations(trackingActivity)
        testEnv.start()
    }

    @AfterEach
    fun tearDown() {
        testEnv.close()
    }

    @Test
    fun `배송 완료 Signal 수신 시 즉시 종료`() {
        // Given: 12시간 폴링 시 IN_TRANSIT 반환
        `when`(trackingActivity.pollDeliveryStatus("SHP-001"))
            .thenReturn(DeliveryStatusResult("IN_TRANSIT", "서울 허브"))

        val workflow = testEnv.workflowClient.newWorkflowStub(
            ShipmentTrackingWorkflow::class.java,
            WorkflowOptions.newBuilder()
                .setWorkflowId("tracking-test-1")
                .setTaskQueue("shipment-tracking-queue")
                .build()
        )

        // When: 비동기로 Workflow 시작
        val future = WorkflowClient.execute(workflow::trackShipment, "SHP-001")

        // 1일 후 배송 완료 Signal 전송
        // ⭐ testEnv가 자동으로 시간을 건너뜀 — 실제 1일을 기다리지 않는다!
        testEnv.sleep(Duration.ofDays(1))
        workflow.updateStatus(ShipmentStatusUpdate("DELIVERED", "서울 강남구"))

        // Then: Workflow가 즉시 완료됨
        val result = future.get()
        assertEquals("DELIVERED", result.finalStatus)
        assertNotNull(result.deliveredAt)
    }

    @Test
    fun `7일 타임아웃 시 배송 지연 알림`() {
        // Given: 폴링 시 항상 IN_TRANSIT
        `when`(trackingActivity.pollDeliveryStatus("SHP-002"))
            .thenReturn(DeliveryStatusResult("IN_TRANSIT", "부산 허브"))

        val workflow = testEnv.workflowClient.newWorkflowStub(
            ShipmentTrackingWorkflow::class.java,
            WorkflowOptions.newBuilder()
                .setWorkflowId("tracking-test-2")
                .setTaskQueue("shipment-tracking-queue")
                .build()
        )

        val future = WorkflowClient.execute(workflow::trackShipment, "SHP-002")

        // ⭐ 7일을 건너뜀 — 실제 테스트는 몇 초 만에 완료!
        testEnv.sleep(Duration.ofDays(8))

        val result = future.get()
        assertEquals("IN_TRANSIT", result.finalStatus)
        assertNull(result.deliveredAt)

        // 배송 지연 알림이 호출되었는지 확인
        verify(trackingActivity).notifyDeliveryDelay("SHP-002")
    }
}
```

> **💡 왜 중요한가**: 배송 추적(7일), 결제 대기(30분), 재고 모니터링(무한 루프) 같은 장기 실행 Workflow를 **실제 시간 없이 테스트**할 수 있다. Temporal의 `TestWorkflowEnvironment`가 타이머와 sleep을 가상 시간으로 처리하기 때문이다. 이것이 메시지 큐 기반 Saga 대비 Temporal 테스팅의 가장 큰 장점이다.

### 6.3 Activity Mock 테스트

Activity 자체의 단위 테스트 (Temporal 없이):

```kotlin
// modular-monolith/src/test/kotlin/com/example/payment/activity/PaymentActivityImplTest.kt

package com.example.payment.activity

import com.example.payment.domain.PaymentRecord
import com.example.payment.domain.PaymentType
import com.example.payment.gateway.PaymentGateway
import com.example.payment.repository.PaymentRepository
import io.temporal.failure.ApplicationFailure
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.assertThrows
import org.mockito.Mockito.*
import kotlin.test.assertEquals

class PaymentActivityImplTest {

    private val paymentGateway = mock(PaymentGateway::class.java)
    private val paymentRepository = mock(PaymentRepository::class.java)
    private val activity = PaymentActivityImpl(paymentGateway, paymentRepository)

    @Test
    fun `결제 성공`() {
        `when`(paymentRepository.findByOrderIdAndType("ORD-001", PaymentType.CHARGE))
            .thenReturn(null)
        `when`(paymentGateway.charge(50000L, "charge-order-ORD-001", "주문 결제: ORD-001"))
            .thenReturn(PaymentGateway.ChargeResult("TXN-001"))

        val result = activity.charge("ORD-001", 50000L)

        assertEquals("TXN-001", result.transactionId)
        verify(paymentRepository).save(any())
    }

    @Test
    fun `멱등성 - 이미 결제된 건은 기존 결과 반환`() {
        val existingRecord = PaymentRecord(
            id = "PAY-001",
            orderId = "ORD-001",
            type = PaymentType.CHARGE,
            amount = 50000,
            transactionId = "TXN-001"
        )
        `when`(paymentRepository.findByOrderIdAndType("ORD-001", PaymentType.CHARGE))
            .thenReturn(existingRecord)

        val result = activity.charge("ORD-001", 50000L)

        assertEquals("PAY-001", result.paymentId)
        assertEquals("TXN-001", result.transactionId)
        // PG사 호출 안 함
        verify(paymentGateway, never()).charge(anyLong(), anyString(), anyString())
    }

    @Test
    fun `결제 거절 시 NonRetryable 에러`() {
        `when`(paymentRepository.findByOrderIdAndType("ORD-001", PaymentType.CHARGE))
            .thenReturn(null)
        `when`(paymentGateway.charge(anyLong(), anyString(), anyString()))
            .thenThrow(PaymentGateway.PaymentDeclinedException("잔액 부족"))

        val exception = assertThrows<ApplicationFailure> {
            activity.charge("ORD-001", 50000L)
        }
        assertEquals("PAYMENT_DECLINED", exception.type)
    }
}
```

### 6.4 보상 로직 테스트

```kotlin
@Test
fun `배송 생성 실패 시 결제 환불, 재고 복원, 주문 취소 모두 호출`() {
    val request = createTestOrderRequest()

    `when`(orderActivity.createOrder(request))
        .thenReturn(CreateOrderResult("ORD-004"))
    `when`(inventoryActivity.reserveStock("ORD-004", request.items))
        .thenReturn(ReserveStockResult("RSV-004"))
    `when`(paymentActivity.charge("ORD-004", 50000L))
        .thenReturn(PaymentResult("PAY-004", "TXN-004"))
    `when`(shipmentActivity.createShipment(eq("ORD-004"), any()))
        .thenThrow(
            io.temporal.failure.ApplicationFailure.newNonRetryableFailure(
                "유효하지 않은 주소", "INVALID_ADDRESS"
            )
        )

    val workflow = testEnv.workflowClient.newWorkflowStub(
        OrderSagaWorkflow::class.java,
        WorkflowOptions.newBuilder()
            .setWorkflowId("test-order-4")
            .setTaskQueue("order-saga-queue")
            .build()
    )
    val result = workflow.processOrder(request)

    assertEquals("FAILED", result.status)

    // 보상이 역순으로 실행되는지 확인
    val inOrder = inOrder(paymentActivity, inventoryActivity, orderActivity)
    inOrder.verify(paymentActivity).refund("ORD-004")
    inOrder.verify(inventoryActivity).releaseStock("ORD-004")
    inOrder.verify(orderActivity).cancelOrder("ORD-004")
}
```

### 6.5 통합 테스트 (Temporal Test Server)

```kotlin
// Temporal Test Server를 사용한 통합 테스트 (실제 Activity 실행)

package com.example.integration

import com.example.saga.api.workflow.OrderSagaWorkflow
import com.example.saga.workflow.OrderSagaWorkflowImpl
import io.temporal.client.WorkflowOptions
import io.temporal.testing.TestWorkflowEnvironment
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers
import kotlin.test.assertEquals

@SpringBootTest
@Testcontainers
class OrderSagaIntegrationTest {

    companion object {
        @Container
        val postgres = PostgreSQLContainer("postgres:15")
            .withDatabaseName("testdb")

        @DynamicPropertySource
        @JvmStatic
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url") { postgres.jdbcUrl }
            registry.add("spring.datasource.username") { postgres.username }
            registry.add("spring.datasource.password") { postgres.password }
        }
    }

    @Autowired
    private lateinit var orderActivity: OrderActivity

    @Autowired
    private lateinit var inventoryActivity: InventoryActivity

    @Autowired
    private lateinit var paymentActivity: PaymentActivity

    @Autowired
    private lateinit var shipmentActivity: ShipmentActivity

    @Test
    fun `전체 주문 처리 통합 테스트`() {
        val testEnv = TestWorkflowEnvironment.newInstance()
        val worker = testEnv.newWorker("order-saga-queue")

        worker.registerWorkflowImplementationTypes(OrderSagaWorkflowImpl::class.java)
        // 실제 Activity 구현체 사용 (Mock 아님)
        worker.registerActivitiesImplementations(
            orderActivity,
            inventoryActivity,
            paymentActivity,
            shipmentActivity
        )

        testEnv.start()

        val request = OrderRequest(
            customerId = "CUST-INTEG-001",
            items = listOf(OrderItem("PROD-001", 1, 30000)),
            shippingAddress = Address("06234", "테헤란로 123", "4층", "서울"),
            totalAmount = 30000
        )

        val workflow = testEnv.workflowClient.newWorkflowStub(
            OrderSagaWorkflow::class.java,
            WorkflowOptions.newBuilder()
                .setWorkflowId("integ-test-1")
                .setTaskQueue("order-saga-queue")
                .build()
        )

        val result = workflow.processOrder(request)

        assertEquals("COMPLETED", result.status)
        // DB에서 실제 데이터 확인
        // ...

        testEnv.close()
    }
}
```

### 6.6 Signal/Query 테스트

```kotlin
@Test
fun `Signal로 주문 취소`() {
    val request = createTestOrderRequest()

    // createOrder는 즉시 반환하지만, reserveStock은 5초 대기
    `when`(orderActivity.createOrder(request))
        .thenReturn(CreateOrderResult("ORD-005"))
    `when`(inventoryActivity.reserveStock(eq("ORD-005"), any()))
        .thenAnswer {
            Thread.sleep(5000) // 5초 대기 (이 사이에 취소 Signal)
            ReserveStockResult("RSV-005")
        }

    val workflow = testEnv.workflowClient.newWorkflowStub(
        OrderSagaWorkflow::class.java,
        WorkflowOptions.newBuilder()
            .setWorkflowId("test-order-5")
            .setTaskQueue("order-saga-queue")
            .build()
    )

    // 비동기 시작
    WorkflowClient.start(workflow::processOrder, request)

    // 잠시 후 취소 Signal 전송
    Thread.sleep(1000)
    workflow.cancel("고객 변심")

    // Query로 상태 확인
    val status = workflow.getStatus()
    // 상태는 CANCELLING 또는 CANCELLED
    assert(status in listOf("CANCELLING", "CANCELLED", "CREATING_ORDER", "RESERVING_STOCK"))
}
```

---

## 7. 운영 가이드

> **이 장에서 배우는 것**: Temporal Server 배포, UI 모니터링, 디버깅/재시도, 메트릭 알림, 프로덕션 체크리스트

### 7.1 Temporal Server 배포

#### Docker Compose (개발/스테이징)

```yaml
# docker-compose.yml
version: "3.8"
services:
  postgresql:
    image: postgres:15
    environment:
      POSTGRES_USER: temporal
      POSTGRES_PASSWORD: temporal
      POSTGRES_DB: temporal
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  temporal:
    image: temporalio/auto-setup:1.24
    depends_on:
      - postgresql
    environment:
      - DB=postgresql
      - DB_PORT=5432
      - POSTGRES_USER=temporal
      - POSTGRES_PWD=temporal
      - POSTGRES_SEEDS=postgresql
      - DYNAMIC_CONFIG_FILE_PATH=config/dynamicconfig/development-sql.yaml
    ports:
      - "7233:7233"  # gRPC (Worker/Client 연결)

  temporal-ui:
    image: temporalio/ui:2.26
    depends_on:
      - temporal
    environment:
      - TEMPORAL_ADDRESS=temporal:7233
      - TEMPORAL_CORS_ORIGINS=http://localhost:3000
    ports:
      - "8080:8080"  # Web UI

  temporal-admin-tools:
    image: temporalio/admin-tools:1.24
    depends_on:
      - temporal
    environment:
      - TEMPORAL_ADDRESS=temporal:7233
    stdin_open: true
    tty: true

volumes:
  postgres_data:
```

#### Kubernetes (프로덕션)

프로덕션에서는 [Temporal Helm Chart](https://github.com/temporalio/helm-charts)를 사용한다:

```bash
# Helm repo 추가
helm repo add temporal https://temporalio.github.io/helm-charts
helm repo update

# 설치 (기본 설정)
helm install temporal temporal/temporal \
  --namespace temporal \
  --create-namespace \
  --set server.replicaCount=3 \
  --set cassandra.config.cluster_size=3 \
  --set prometheus.enabled=true \
  --set grafana.enabled=true

# 또는 PostgreSQL 백엔드 사용
helm install temporal temporal/temporal \
  --namespace temporal \
  --create-namespace \
  --set server.replicaCount=3 \
  --set cassandra.enabled=false \
  --set mysql.enabled=false \
  --set postgresql.enabled=true \
  --set postgresql.auth.password=<strong-password>
```

### 7.2 Temporal UI로 Workflow 모니터링

Temporal UI (`http://localhost:8080`)에서 확인할 수 있는 것:

1. **Workflow 목록**: 실행 중/완료/실패한 모든 Workflow
2. **Workflow 상세**: 
   - Event History: 모든 이벤트를 시간순으로 표시
   - Input/Output: Workflow 입력과 결과
   - Pending Activities: 현재 실행 중인 Activity
   - Stack Trace: Workflow가 어디서 대기 중인지
3. **검색**: Workflow ID, 상태, 시간 범위로 필터링
4. **네임스페이스**: 환경별 격리 (dev, staging, prod)

### 7.3 실패한 Workflow 디버깅/재시도

#### CLI로 디버깅

```bash
# Workflow 상태 확인
tctl workflow describe --workflow_id "order-saga-ORD-001"

# Event History 조회
tctl workflow show --workflow_id "order-saga-ORD-001"

# 실패한 Workflow 재시도 (Reset)
# 특정 이벤트부터 다시 실행
tctl workflow reset \
  --workflow_id "order-saga-ORD-001" \
  --reason "결제 게이트웨이 복구 후 재시도" \
  --reset_type LastWorkflowTask

# Signal 전송
tctl workflow signal \
  --workflow_id "shipment-tracking-SHP-001" \
  --name "updateStatus" \
  --input '{"status":"DELIVERED","location":"서울 강남구"}'

# Query 실행
tctl workflow query \
  --workflow_id "shipment-tracking-SHP-001" \
  --query_type "getCurrentStatus"
```

#### Reset 타입

| Reset 타입 | 설명 | 사용 시점 |
|---|---|---|
| `FirstWorkflowTask` | 처음부터 다시 실행 | 전체 재실행이 필요할 때 |
| `LastWorkflowTask` | 마지막 Workflow Task부터 | 일시적 오류 후 재시도 |
| `BadBinary` | 특정 바이너리로 실행된 부분부터 | 버그가 있는 배포 롤백 |
| `EventId` | 특정 이벤트 ID부터 | 정밀한 복구 |

#### 운영 시나리오 예시: 결제 PG 장애 대응

```
1. 알림 수신: "Workflow 실패율 10% 초과" (Grafana Alert)
2. Temporal UI에서 실패한 Workflow 확인
   → failureReason: "PAYMENT_GATEWAY_ERROR: Connection timeout"
3. PG사 상태 확인 → 장애 진행 중
4. PG사 복구 확인 후 실패한 Workflow들을 일괄 Reset:
   
   tctl workflow reset-batch \
     --query 'WorkflowType="OrderSagaWorkflow" AND ExecutionStatus="Failed"' \
     --reason "PG사 장애 복구 후 재시도" \
     --reset_type LastWorkflowTask
     
5. Temporal UI에서 재시도된 Workflow들의 진행 상태 확인
6. 정상 완료율 모니터링
```

### 7.4 메트릭/알림 설정

Temporal은 Prometheus 메트릭을 기본 제공한다.

#### 주요 메트릭

```yaml
# Grafana 대시보드 또는 Prometheus Alert Rules

# 1. Workflow 실행 관련
temporal_workflow_completed_total          # 완료된 Workflow 수
temporal_workflow_failed_total             # 실패한 Workflow 수
temporal_workflow_canceled_total           # 취소된 Workflow 수
temporal_workflow_execution_latency_bucket # Workflow 실행 시간 분포

# 2. Activity 관련
temporal_activity_execution_latency_bucket # Activity 실행 시간
temporal_activity_schedule_to_start_latency_bucket # 스케줄~시작 대기 시간 (높으면 Worker 부족)

# 3. Task Queue 관련
temporal_task_queue_backlog               # Task Queue 대기열 크기 (높으면 Worker 확장 필요)

# 4. Worker 관련
temporal_worker_task_slots_available      # Worker 여유 슬롯
```

#### 알림 설정 예시 (Prometheus AlertManager)

```yaml
groups:
  - name: temporal-alerts
    rules:
      - alert: HighWorkflowFailureRate
        expr: |
          rate(temporal_workflow_failed_total[5m]) / 
          rate(temporal_workflow_completed_total[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Workflow 실패율 10% 초과"

      - alert: TaskQueueBacklog
        expr: temporal_task_queue_backlog > 1000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Task Queue 대기열 1000 초과 — Worker 확장 필요"

      - alert: ActivityScheduleLatency
        expr: |
          histogram_quantile(0.95, temporal_activity_schedule_to_start_latency_bucket) > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Activity 스케줄~시작 대기 시간 30초 초과"
```

### 7.5 프로덕션 체크리스트

#### 배포 전

- [ ] Temporal Server HA 구성 (최소 3 replicas)
- [ ] DB 백업 설정 (Temporal의 상태 저장소)
- [ ] 네임스페이스 분리 (prod, staging)
- [ ] TLS 설정 (Worker ↔ Server 간 통신)
- [ ] 네트워크 정책 (Temporal Server 접근 제한)

#### Workflow 코드

- [ ] 모든 Activity에 타임아웃 설정
- [ ] 모든 Activity에 재시도 정책 설정
- [ ] 비즈니스 에러는 `setDoNotRetry`로 재시도 제외
- [ ] 보상 Activity는 더 공격적인 재시도
- [ ] Workflow Execution 타임아웃 설정
- [ ] 모든 Activity는 멱등적으로 구현
- [ ] Versioning 전략 수립

#### 운영

- [ ] Prometheus/Grafana 메트릭 대시보드
- [ ] 알림 설정 (실패율, 대기열, 지연 시간)
- [ ] Temporal UI 접근 설정
- [ ] Worker Auto-scaling 설정
- [ ] Dead Letter Queue 모니터링 (보상 실패 건)
- [ ] 정기적인 Workflow 정리 (오래된 완료 건 아카이빙)
- [ ] 장애 대응 Runbook 작성 (7.3절 시나리오 참고)

---

## 8. AI와 Temporal

> **이 장에서 배우는 것**: AI 코드 생성 프롬프트 패턴, Temporal이 AI에 적합한 이유, CLAUDE.md 작성 가이드

### 8.1 AI에게 Workflow 코드 생성 시키는 프롬프트 패턴

#### 패턴 1: Activity 인터페이스 먼저, Workflow 구현은 AI에게

```
다음 Activity 인터페이스들을 사용해서 OrderSagaWorkflow를 구현해줘.

Activity 인터페이스:
- OrderActivity: createOrder(request) → orderId, cancelOrder(orderId)
- InventoryActivity: reserveStock(orderId, items) → reservationId, releaseStock(orderId)
- PaymentActivity: charge(orderId, amount) → paymentId, refund(orderId)
- ShipmentActivity: createShipment(orderId, address) → shipmentId

요구사항:
1. Saga 패턴으로 구현 (각 단계에 보상 등록)
2. 모든 Activity에 타임아웃 30초, 최대 재시도 3회
3. INSUFFICIENT_STOCK, PAYMENT_DECLINED는 재시도하지 않음
4. 보상용 Activity Stub은 별도로 (재시도 10회)
5. Signal로 취소 가능
6. Query로 현재 상태 조회 가능
```

이 패턴이 효과적인 이유:
- Activity 인터페이스가 AI에게 "사용 가능한 도구"를 명확히 알려준다
- 요구사항이 구조화되어 있어 AI가 빠짐없이 구현한다
- 생성된 코드가 Activity 인터페이스에 맞는지 컴파일로 바로 검증 가능

#### 패턴 2: 시나리오 기반 프롬프트

```
물류 이커머스에서 다음 시나리오를 처리하는 Temporal Workflow를 구현해줘.

시나리오: 주문 후 30분 내 결제 대기
1. 주문 생성
2. 재고 임시 예약 (30분간)
3. 고객에게 결제 링크 발송
4. 30분 내 결제 확인 Signal을 기다림
   - 결제 완료 Signal 수신 → 배송 생성
   - 30분 타임아웃 → 재고 해제, 주문 취소
5. 배송 생성 실패 시 → 결제 환불, 재고 해제, 주문 취소

사용 가능한 Activity: [인터페이스 목록]
```

#### 패턴 3: 안티패턴 명시

```
다음 규칙을 반드시 지켜서 구현해줘:

❌ 금지:
- Workflow 안에서 DB 접근
- Random(), System.currentTimeMillis(), Thread.sleep()
- Activity에서 다른 Activity 호출
- 보상 없는 Saga 단계

✅ 필수:
- Workflow.sleep(), Workflow.currentTimeMillis() 사용
- 모든 단계에 Saga.addCompensation()
- Activity에 타임아웃 설정
- 비즈니스 에러는 NonRetryable
```

### 8.2 Temporal이 AI 코딩에 적합한 이유

**1. 명확한 구조**

Temporal은 코드 구조를 강제한다:
- Workflow = 순수한 조율 로직 (부수 효과 없음)
- Activity = 부수 효과가 있는 실제 작업
- Worker = 실행 환경 설정

AI가 "어디에 뭘 넣어야 하는지" 헷갈릴 여지가 없다.

**2. 비결정적 코드 자동 차단**

AI가 실수로 `Random()`이나 `Thread.sleep()`을 Workflow에 넣으면, **런타임에 즉시 에러**가 발생한다. 코드 리뷰 없이도 잘못된 코드가 걸러진다.

**3. 테스트가 쉬움**

AI가 생성한 Workflow를 `TestWorkflowEnvironment`로 바로 검증할 수 있다. Activity를 Mock하면 외부 의존성 없이 전체 Saga 흐름을 테스트한다.

**4. 인터페이스 = 계약**

Activity 인터페이스는 AI에게 "이 도구들을 사용해서 이 문제를 풀어"라고 말하는 것과 같다. AI Agent의 Tool 정의와 본질적으로 같은 패턴이다.

**5. AI가 보일러플레이트를 해소한다**

Temporal/Saga 패턴의 대표적 비판은 보일러플레이트가 많다는 것이다 — ActivityOptions, RetryOptions, Saga 보상 등록, Worker 설정 등. AI 코드 생성이 이 문제를 근본적으로 해소한다. 도메인 흐름과 Activity 인터페이스만 설계하면, 나머지 구현 코드는 AI가 생성할 수 있다. 2.2절에서 언급한 것처럼, **AI Native Engineer는 "무엇을 해야 하는가"에 집중하고, AI는 "어떻게 구현하는가"를 담당한다.**

### 8.3 CLAUDE.md 작성 가이드

각 repo에 CLAUDE.md를 작성하면 AI 코딩 도구(Claude, Copilot 등)가 프로젝트 규칙을 이해하고 따른다. 3.5절의 예시를 참고하되, 핵심 원칙:

1. **금지 사항을 명확히**: "~하지 마라"를 구체적으로 나열
2. **허용 사항도 명시**: AI가 안전한 범위를 알 수 있게
3. **코드 패턴 예시**: 기대하는 코드 스타일을 짧은 예시로
4. **의존성 규칙**: 어떤 라이브러리를 쓸 수 있고 없는지
5. **테스트 규칙**: 어떤 테스트를 작성해야 하는지

```markdown
# CLAUDE.md 템플릿

## 이 repo는 무엇인가
(한 줄 설명)

## 절대 금지
1. ...
2. ...

## 반드시 지켜야 할 것
1. ...
2. ...

## 코드 패턴
(짧은 코드 예시)

## 테스트
(어떤 테스트를 작성해야 하는지)
```

---

## 9. 흔한 실수와 안티패턴

### 9.1 ❌ Workflow에서 I/O 직접 수행

```kotlin
// ❌ 절대 금지
class OrderSagaWorkflowImpl : OrderSagaWorkflow {
    override fun processOrder(request: OrderRequest): OrderResult {
        // DB 접근 — Replay 시 다른 결과가 나올 수 있음!
        val product = productRepository.findById(request.items[0].productId)
        
        // HTTP 호출 — 비결정적!
        val rate = RestTemplate().getForObject("https://api.exchange.com/rate", Rate::class.java)
        
        // 파일 I/O — 비결정적!
        val config = File("config.json").readText()
    }
}

// ✅ Activity로 분리
class OrderSagaWorkflowImpl : OrderSagaWorkflow {
    override fun processOrder(request: OrderRequest): OrderResult {
        val product = productActivity.getProduct(request.items[0].productId)
        val rate = exchangeActivity.getCurrentRate()
        val config = configActivity.loadConfig()
    }
}
```

### 9.2 ❌ Activity를 너무 크게/작게 만들기

```kotlin
// ❌ 너무 큼: 하나의 Activity에 모든 것을 넣음
@ActivityInterface
interface OrderActivity {
    @ActivityMethod
    fun processEntireOrder(request: OrderRequest): OrderResult
    // 주문 생성 + 재고 차감 + 결제 + 배송을 한 번에?
    // → 보상을 세밀하게 할 수 없음, 재시도 범위가 너무 넓음
}

// ❌ 너무 작음: 모든 DB 쿼리를 개별 Activity로
@ActivityInterface
interface OrderActivity {
    @ActivityMethod
    fun insertOrderHeader(order: OrderHeader)
    
    @ActivityMethod
    fun insertOrderLine(line: OrderLine)
    
    @ActivityMethod
    fun updateOrderStatus(orderId: String, status: String)
    // → Activity 호출 오버헤드가 너무 큼, Event History가 빠르게 커짐
}

// ✅ 적절한 크기: 비즈니스 의미 단위
@ActivityInterface
interface OrderActivity {
    @ActivityMethod
    fun createOrder(request: OrderRequest): CreateOrderResult  // 주문 생성 (헤더+라인 포함)
    
    @ActivityMethod
    fun cancelOrder(orderId: String)  // 주문 취소 (상태 변경)
    
    @ActivityMethod
    fun completeOrder(orderId: String)  // 주문 완료 (상태 변경)
}
```

**원칙**: Activity는 **보상 가능한 비즈니스 단위**로 나눈다.

### 9.3 ❌ 보상 로직 누락

```kotlin
// ❌ 보상 없음 — 실패 시 불일치 상태
class OrderSagaWorkflowImpl : OrderSagaWorkflow {
    override fun processOrder(request: OrderRequest): OrderResult {
        val orderId = orderActivity.createOrder(request)
        inventoryActivity.reserveStock(orderId, request.items)
        paymentActivity.charge(orderId, request.totalAmount) // 여기서 실패하면?
        // → 재고는 차감됐지만 결제는 안 됨, 주문은 CREATED 상태로 방치
    }
}

// ✅ 모든 단계에 보상 등록
class OrderSagaWorkflowImpl : OrderSagaWorkflow {
    override fun processOrder(request: OrderRequest): OrderResult {
        val saga = Saga(Saga.Options.Builder().build())
        
        saga.addCompensation { orderActivity.cancelOrder(orderId) }
        val orderId = orderActivity.createOrder(request)
        
        saga.addCompensation { inventoryActivity.releaseStock(orderId) }
        inventoryActivity.reserveStock(orderId, request.items)
        
        saga.addCompensation { paymentActivity.refund(orderId) }
        paymentActivity.charge(orderId, request.totalAmount)
        // 이제 실패 시 saga.compensate()가 역순으로 보상 실행
    }
}
```

### 9.4 ❌ 타임아웃 미설정

```kotlin
// ❌ 타임아웃 없음 — Activity가 영원히 hang될 수 있음
val activity = Workflow.newActivityStub(
    PaymentActivity::class.java,
    ActivityOptions.newBuilder().build() // 타임아웃 없음!
)

// ✅ 반드시 타임아웃 설정
val activity = Workflow.newActivityStub(
    PaymentActivity::class.java,
    ActivityOptions.newBuilder()
        .setStartToCloseTimeout(Duration.ofSeconds(30))     // 실행 시간 제한
        .setScheduleToCloseTimeout(Duration.ofMinutes(5))   // 전체 시간 제한 (재시도 포함)
        .setHeartbeatTimeout(Duration.ofSeconds(10))         // 장기 Activity의 생존 확인
        .build()
)
```

**타임아웃 종류:**
- `StartToCloseTimeout`: Activity가 시작된 후 완료까지 최대 시간
- `ScheduleToCloseTimeout`: Activity가 스케줄된 후 완료까지 최대 시간 (재시도 포함)
- `ScheduleToStartTimeout`: Activity가 스케줄된 후 Worker가 픽업할 때까지 최대 시간
- `HeartbeatTimeout`: 장기 실행 Activity의 heartbeat 간격 (이 시간 내 heartbeat 없으면 실패 처리)

### 9.5 ❌ Non-deterministic 코드

```kotlin
// ❌ 비결정적 코드들
class BadWorkflowImpl : SomeWorkflow {
    override fun execute() {
        // Random — Replay마다 다른 값
        val id = UUID.randomUUID().toString()
        
        // 현재 시간 — Replay마다 다른 값
        val now = System.currentTimeMillis()
        
        // Thread.sleep — Temporal이 관리할 수 없음
        Thread.sleep(5000)
        
        // 새 스레드 — Temporal이 추적할 수 없음
        Thread { doSomething() }.start()
        
        // 컬렉션 순서 — HashMap은 순서 보장 안 함
        val map = HashMap<String, String>()
        for ((k, v) in map) { /* Replay마다 순서 다를 수 있음 */ }
    }
}

// ✅ 결정적 대안
class GoodWorkflowImpl : SomeWorkflow {
    override fun execute() {
        val id = Workflow.randomUUID().toString()
        val now = Workflow.currentTimeMillis()
        Workflow.sleep(Duration.ofSeconds(5))
        Async.function { doSomething() }  // Temporal이 관리하는 비동기
        val map = LinkedHashMap<String, String>()  // 순서 보장
    }
}
```

### 9.6 ❌ Kotlin + Temporal 직렬화 이슈 (Jackson Kotlin Module)

Temporal의 기본 `DataConverter`는 **Jackson**을 사용해 Workflow/Activity의 입출력을 JSON으로 직렬화한다. Kotlin data class를 사용할 때 흔히 발생하는 문제와 해결법:

#### 문제: 기본 생성자 없는 data class 역직렬화 실패

```kotlin
// ❌ Jackson이 기본 생성자를 찾지 못해 역직렬화 실패할 수 있음
data class OrderRequest(
    val customerId: String,
    val items: List<OrderItem>,
    val totalAmount: Long
)
// Jackson은 기본적으로 no-arg constructor + setter를 기대한다.
// Kotlin data class는 no-arg constructor가 없다.
```

#### 해결: Jackson Kotlin Module 등록

```kotlin
// Worker 생성 시 커스텀 DataConverter 설정
import com.fasterxml.jackson.module.kotlin.KotlinModule
import io.temporal.common.converter.DefaultDataConverter
import io.temporal.common.converter.JacksonJsonPayloadConverter

val objectMapper = ObjectMapper().apply {
    registerModule(KotlinModule.Builder().build())
    // 알 수 없는 프로퍼티 무시 (DTO 버전 호환)
    configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
}

val dataConverter = DefaultDataConverter(
    // 기본 converter 목록에서 Jackson converter만 교체
    JacksonJsonPayloadConverter(objectMapper)
)

// WorkflowClient 생성 시 적용
val client = WorkflowClient.newInstance(
    serviceStubs,
    WorkflowClientOptions.newBuilder()
        .setDataConverter(dataConverter)
        .build()
)

// WorkerFactory에도 동일하게 적용
val factory = WorkerFactory.newInstance(client)
```

> **💡 팁**: `jackson-module-kotlin`을 Gradle/Maven에 추가하고 `KotlinModule`을 등록하면, Kotlin data class의 primary constructor를 자동으로 인식한다. 이 설정 없이는 `MismatchedInputException`이 발생할 수 있다.

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin:2.17.0")
}
```

#### DTO 설계 팁

```kotlin
// ✅ 좋은 DTO: 순수 Kotlin data class, Serializable 불필요
data class OrderRequest(
    val customerId: String,
    val items: List<OrderItem>,
    val totalAmount: Long
)

// ✅ 기본값이 있으면 하위 호환에 유리
data class OrderResult(
    val orderId: String,
    val status: String,
    val shipmentId: String? = null,      // nullable + 기본값
    val failureReason: String? = null
)

// ❌ 피해야 할 패턴: java.io.Serializable은 불필요
data class OrderRequest(
    val customerId: String
) : Serializable  // ← Temporal은 JSON 직렬화를 사용하므로 무의미
```

---

## 10. 참고 자료

### 공식 문서
- [Temporal 공식 문서](https://docs.temporal.io/) — 가장 중요한 레퍼런스
- [Temporal Java SDK 문서](https://www.javadoc.io/doc/io.temporal/temporal-sdk/latest/index.html)
- [Temporal Java SDK GitHub](https://github.com/temporalio/sdk-java)
- [Temporal Samples (Java)](https://github.com/temporalio/samples-java) — 다양한 패턴 예시

### 책
- *"Designing Data-Intensive Applications"* (Martin Kleppmann) — 분산 시스템 기초
- *"Enterprise Integration Patterns"* (Hohpe, Woolf) — 메시징 패턴의 교과서

### 강의/영상
- [Temporal 101 (공식 강좌)](https://learn.temporal.io/courses/temporal_101/) — 무료, 입문
- [Temporal 102](https://learn.temporal.io/courses/temporal_102/) — 무료, 심화
- [Maxim Fateev - Designing a Workflow Engine](https://www.youtube.com/watch?v=t524U9CixZ0) — Temporal 창립자 강연

### 블로그
- [Temporal 공식 블로그](https://temporal.io/blog)
- [Why Temporal (by Swyx)](https://www.swyx.io/why-temporal) — Temporal 도입 이유 정리
- [Saga Pattern with Temporal](https://docs.temporal.io/encyclopedia/detecting-activity-failures#saga-pattern) — 공식 Saga 가이드

### 커뮤니티
- [Temporal Slack](https://temporal.io/slack) — 공식 슬랙 (질문 답변 활발)
- [Temporal Forum](https://community.temporal.io/) — 공식 포럼

---

> **이 가이드는 시작점이다.** 실제로 코드를 작성하고, 실행하고, 실패시켜 보면서 체득해야 한다. Temporal의 가장 좋은 학습 방법은 `docker-compose up`으로 서버를 띄우고, 예제 Workflow를 실행하고, Temporal UI에서 Event History를 직접 확인하는 것이다.

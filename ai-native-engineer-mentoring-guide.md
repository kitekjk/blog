# AI Native Engineer 신입 멘토링 가이드

> 물류 이커머스 도메인 기준 | 작성일: 2026-03-11 | 최종 수정: 2026-03-11

---

## 목차

1. [AI Native Engineer란?](#1-ai-native-engineer란)
2. [온보딩 로드맵 (12주)](#2-온보딩-로드맵-12주)
3. [DDD + Hexagonal Architecture 핵심](#3-ddd--hexagonal-architecture-핵심)
4. [AI 도구 활용 가이드](#4-ai-도구-활용-가이드)
5. [AI 인프라 구축](#5-ai-인프라-구축)
6. [관측성 & 트러블슈팅](#6-관측성--트러블슈팅)
7. [멘토링 원칙](#7-멘토링-원칙)
8. [평가 기준](#8-평가-기준)

---

## 1. AI Native Engineer란?

### 1.1 역할 정의

AI Native Engineer는 **AI를 보조 도구가 아닌 핵심 개발 파트너로 활용**하여 소프트웨어를 설계·구현·운영하는 엔지니어다.

단순히 "AI가 코드를 짜주니까 편하다"가 아니라, **AI가 올바른 코드를 생성하도록 아키텍처를 설계하고, AI 출력을 검증하며, AI 인프라를 구축·운영**하는 역할이다.

### 1.2 기존 엔지니어와의 차이

| 구분 | Traditional Engineer | AI Native Engineer |
|------|---------------------|--------------------|
| **코드 작성** | 직접 타이핑 중심 | AI 생성 + 검증·수정 |
| **설계 방식** | 구현 중심 설계 | 인터페이스 먼저 → AI가 구현 |
| **문서화** | 코드 작성 후 문서화 | 문서(프롬프트)가 곧 설계 |
| **디버깅** | 코드 추적 중심 | AI에게 컨텍스트 전달 → 원인 분석 위임 |
| **학습 방식** | 책·강의 → 실습 | AI와 대화하며 실습 → 원리 역추적 |
| **아키텍처** | 유지보수 편의 중심 | AI 생성 품질 최적화 + 유지보수 |
| **코드 리뷰** | 사람 코드 리뷰 | AI 생성 코드의 정합성·보안·성능 검증 |

### 1.3 핵심 역량

#### ① 프롬프트 엔지니어링
- 의도를 정확히 전달하는 구조화된 프롬프트 작성
- 컨텍스트 관리 (어떤 정보를 줘야 AI가 좋은 코드를 만드는지)
- Few-shot, Chain-of-Thought 등 기법 활용

#### ② AI 도구 활용
- Claude Code, GitHub Copilot, Cursor 등 상황별 최적 도구 선택
- CLAUDE.md, AGENTS.md 등 AI 컨텍스트 파일 관리
- AI 도구 간 워크플로우 구성

#### ③ 코드 검증
- AI가 생성한 코드의 정확성, 보안, 성능 검증
- 테스트 코드 작성으로 AI 출력 자동 검증
- 도메인 로직 정합성 확인 (AI는 비즈니스 컨텍스트를 모른다)

#### ④ AI 인프라 구축
- RAG (Retrieval-Augmented Generation) 파이프라인 설계·운영
- MCP (Model Context Protocol) 서버 구축
- 벡터DB 선택·운영·최적화

---

## 2. 온보딩 로드맵 (12주)

### Phase 1: 기초 세팅 (1~2주)

> **이 단계가 끝나면:** AI 도구를 자유롭게 사용하고, 구조화된 프롬프트로 원하는 코드를 생성하며, AI 출력을 비판적으로 검토할 수 있다.

#### 1주차: AI 도구 세팅 & 첫 경험

**목표:** 개발 환경에 AI 도구를 설치하고, 첫 AI 협업 코딩을 경험한다.

| 일차 | 할 일 | 산출물 |
|------|-------|--------|
| Day 1 | 개발 환경 세팅 (IntelliJ + Kotlin + Git) | 로컬 빌드 성공 |
| Day 2 | Claude Code 설치, API Key 설정, CLAUDE.md 작성 | 첫 AI 대화 로그 |
| Day 3 | GitHub Copilot 설치, Cursor 설치 | 도구별 첫 코드 생성 경험 |
| Day 4 | 팀 코드베이스 탐색 — AI에게 "이 프로젝트 구조를 설명해줘" | 프로젝트 구조 이해 메모 |
| Day 5 | 간단한 버그 수정 또는 유틸 함수 작성 (AI 활용) | 첫 PR |

**과제:**
- 같은 함수를 (1) 직접 작성 (2) AI로 작성 → 비교 회고 작성
- 팀 CLAUDE.md에 자신이 이해한 프로젝트 컨텍스트 추가

#### 2주차: 프롬프트 엔지니어링 기본 & AI 코드 리뷰 습관

**목표:** 좋은 프롬프트의 구조를 이해하고, AI 생성 코드를 비판적으로 검토하는 습관을 기른다.

| 일차 | 할 일 | 산출물 |
|------|-------|--------|
| Day 1 | 프롬프트 구조 학습 (역할/컨텍스트/제약/출력형식) | 정리 노트 |
| Day 2 | 같은 요구사항을 5가지 다른 프롬프트로 → 결과 비교 | 프롬프트 비교표 |
| Day 3 | AI 생성 코드 리뷰 실습 (의도적으로 나쁜 프롬프트 → 문제 찾기) | 리뷰 체크리스트 초안 |
| Day 4 | 팀 코드 리뷰 참여 (AI 생성 코드 포함) | 리뷰 코멘트 3개 이상 |
| Day 5 | 1~2주 회고 & 멘토 1:1 | 회고 문서 |

---

### Phase 2: 아키텍처 기초 (3~4주)

> **이 단계가 끝나면:** DDD와 Hexagonal Architecture의 핵심 개념을 이해하고, AI에게 "Port 인터페이스의 Adapter를 구현해줘"처럼 경계가 명확한 프롬프트를 작성할 수 있다.

> **💡 멘토 참고:** 이 단계의 목표는 DDD/Hexagonal을 완벽히 체화하는 것이 **아니다**. 신입이 2주 만에 마스터할 수 있는 주제가 아니다. 핵심은 **"AI에게 명확한 질문을 던지기 위한 격리 수단"**으로서 개념을 이해하는 것이다. 평가 기준도 '완벽한 설계'가 아니라 'Port와 Adapter를 분리해서 AI에게 지시할 수 있는지'에 맞춘다.

#### 3주차: DDD 핵심 개념 — 단계적 접근

**목표:** Domain-Driven Design의 핵심 개념을 **물류 실무 사례**와 함께 이해한다.

**[Step 1] 먼저 "왜 필요한지" 이해하기**

AI에게 "배송비 계산 기능 만들어줘"라고 하면 모든 로직이 하나의 클래스에 뭉친다. 하지만 "이 `ShippingPort` 인터페이스의 CJ대한통운 Adapter를 구현해줘"라고 하면 AI가 정확한 범위의 코드를 생성한다. DDD는 이런 **경계를 정의하는 방법론**이다.

**[Step 2] 개념을 물류 사례로 연결하기**

| 개념 | 정의 (한 줄) | 물류 도메인 예시 |
|------|-------------|-----------------|
| **Entity** | 고유 ID가 있고 생명주기가 있는 객체 | `Order` (주문번호로 식별), `Shipment` (배송번호로 식별) |
| **Value Object** | ID 없이 값 자체가 의미인 불변 객체 | `Money(3000, "KRW")`, `Address(...)`, `TrackingNumber(...)` |
| **Aggregate** | 함께 변경되어야 하는 객체 묶음 | Order + OrderLine + ShippingInfo = Order Aggregate |
| **Bounded Context** | 도메인 모델의 적용 경계 | 주문 Context ↔ 배송 Context ↔ 정산 Context |

> **💡 신입을 위한 팁:** "주문번호(OrderId)가 왜 Value Object인가?"를 생각해보자. OrderId 자체는 식별자가 아니라 **값**이다. `OrderId("ORD-001")`과 `OrderId("ORD-001")`은 같다. 하지만 `Order`는 Entity다 — 같은 주문이라도 시간이 지나면 상태가 변한다.

| 일차 | 할 일 | 산출물 |
|------|-------|--------|
| Day 1 | DDD 핵심 개념 학습 (Entity, VO, Aggregate) — 위 표와 함께 | 개념 정리 노트 |
| Day 2 | 팀 도메인 모델 분석 — AI에게 "이 코드의 도메인 모델을 분석해줘" | 도메인 모델 다이어그램 |
| Day 3 | 직접 Entity/VO 작성 실습 (Order, Address, Money 등) | 코드 PR |
| Day 4 | Bounded Context 학습 — 팀 코드에서 Context 경계 찾아보기 | 간단한 Context Map 스케치 |
| Day 5 | 코드 리뷰 & 피드백 반영 | 수정된 PR |

**멘토가 물어볼 핵심 질문:**
- "주문번호(OrderId)가 왜 Value Object인가?"
- "Shipment Aggregate의 경계는 어디까지인가?"
- "Order와 Shipment은 같은 Bounded Context인가?"

#### 4주차: Hexagonal Architecture — "AI를 위한 경계 설계"

**목표:** Port/Adapter 패턴을 이해하고, **"인터페이스를 설계하면 AI가 구현한다"**는 워크플로우를 체감한다.

**[Step 1] 핵심 개념 3줄 요약**

```
Port   = 도메인이 외부와 소통하는 인터페이스 (계약서)
Adapter = Port의 구현체 (CJ대한통운 API 호출, DB 저장 등)
UseCase = 비즈니스 로직 (Port만 의존, 구현 몰라도 됨)
```

**[Step 2] 왜 AI 시대에 더 중요한가**

| 상황 | Port/Adapter 없이 | Port/Adapter 있으면 |
|------|-------------------|-------------------|
| AI에게 코드 요청 | "배송 기능 만들어줘" → 범위 모호 | "이 ShippingPort의 한진 Adapter 만들어줘" → 명확 |
| 배송사 변경 | 비즈니스 로직 전체 수정 | Adapter만 교체 |
| 테스트 | 외부 API 의존 → 테스트 어려움 | Port를 Mock → 쉬운 단위 테스트 |

| 일차 | 할 일 | 산출물 |
|------|-------|--------|
| Day 1 | Hexagonal Architecture 학습 (Port, Adapter, UseCase) — 위 표 참고 | 개념 정리 노트 |
| Day 2 | 팀 코드에서 Port/Adapter 패턴 찾기 | 구조 분석 문서 |
| Day 3 | ShippingPort 인터페이스 직접 설계 | 인터페이스 코드 |
| Day 4 | AI에게 Adapter 구현 시키기 (프롬프트만으로) | AI 생성 Adapter + 검증 |
| Day 5 | 3~4주 회고 & 멘토 1:1 | 회고 문서 |

---

### Phase 3: AI 활용 개발 실습 (5~6주)

> **이 단계가 끝나면:** "인터페이스 먼저 설계 → AI가 구현" 워크플로우를 실전에 적용하고, 실제 백로그 태스크를 독립적으로 수행할 수 있다.

#### 5주차: AI와 협업 코딩 — 인터페이스 먼저 패턴

**목표:** "인터페이스를 설계하고 AI가 구현"하는 워크플로우를 체화한다.

**실습 과제: 배송비 계산 기능 구현**

```
1일차: 요구사항 분석 → 도메인 모델 설계 (직접)
2일차: Port 인터페이스 설계 (직접) → AI에게 UseCase 구현 지시
3일차: AI 생성 코드 리뷰 & 테스트 작성 (직접)
4일차: Adapter 구현을 AI에게 지시 → 통합 테스트
5일차: 리팩토링 & 코드 리뷰
```

#### 6주차: 실전 태스크 수행

**목표:** 실제 백로그 태스크를 AI와 함께 처리한다.

- 멘토가 선별한 실전 태스크 2~3개 수행
- 각 태스크에서 AI 활용 비율과 직접 작성 비율 기록
- PR마다 "AI 활용 로그" 코멘트 첨부 (템플릿은 [부록 C](#c-ai-활용-로그-템플릿) 참고)

---

### Phase 4: AI 인프라 (7~8주)

> **이 단계가 끝나면:** RAG 파이프라인의 개념과 구조를 이해하고, 간단한 MCP 서버를 구현하여 AI 도구와 연동할 수 있다.

> **⚠️ 스택 안내:** 이 단계의 RAG/MCP 예시는 Python과 TypeScript를 사용한다. 이는 해당 생태계(LlamaIndex, MCP SDK)가 가장 성숙하고 레퍼런스가 풍부하기 때문이다. 팀 메인 스택은 Kotlin이지만, AI 인프라는 별도 서비스로 운영되므로 최적의 도구를 선택하는 것이 실무적이다. Kotlin 생태계 대안(Spring AI, LangChain4j)도 있으니 관심 있으면 탐색해볼 것.

#### 7주차: RAG 기초

| 일차 | 할 일 | 산출물 |
|------|-------|--------|
| Day 1 | RAG 개념 학습 (Embedding, Vector Search, Chunking) — 아래 개념도 참고 | 개념 정리 |
| Day 2 | pgvector 로컬 설치 & 기본 CRUD | 동작하는 벡터DB |
| Day 3 | LlamaIndex(Python) 기본 파이프라인 구축 | 문서 인덱싱 파이프라인 |
| Day 4 | 팀 위키 문서 → 벡터DB 인덱싱 | 검색 가능한 인덱스 |
| Day 5 | 질의 테스트 & 품질 평가 | 품질 리포트 |

#### 8주차: MCP 서버 구현

| 일차 | 할 일 | 산출물 |
|------|-------|--------|
| Day 1 | MCP 프로토콜 이해 & 기존 MCP 서버 분석 | 개념 정리 |
| Day 2 | 간단한 MCP 서버 구현 (TypeScript, DB 조회 도구) | MCP 서버 코드 |
| Day 3 | Claude Code에서 MCP 서버 연동 테스트 | 동작 확인 |
| Day 4 | 벡터DB 비교 학습 (pgvector vs Qdrant — 개념 수준) | 간단 비교 정리 |
| Day 5 | 7~8주 회고 & 멘토 1:1 | 회고 문서 |

---

### Phase 5: 자율 프로젝트 (9~10주)

> **이 단계가 끝나면:** 설계부터 구현·데모·발표까지 하나의 프로젝트를 독립적으로 완수하고, 팀에 기여할 수 있는 결과물을 만들어낸다.

#### 프로젝트 후보 (택 1)

| 프로젝트 | 난이도 | 설명 |
|----------|--------|------|
| 팀 위키 RAG 봇 | ★★☆ | Confluence 문서를 RAG로 검색하는 Slack 봇 |
| 배송 상태 알림 MCP | ★★★ | 배송 추적 API를 MCP로 래핑하여 AI가 조회 |
| 코드 리뷰 어시스턴트 | ★★★ | PR diff를 분석하여 아키텍처 위반 탐지 |
| CS 자동 답변 초안 | ★★☆ | 고객 문의를 분류하고 답변 초안 생성 |

**프로젝트 진행 방식:**
1. 9주차 Day 1: 프로젝트 선정 & 설계 문서 작성
2. 9주차 Day 2~5: 구현 Sprint 1
3. 10주차 Day 1~3: 구현 Sprint 2
4. 10주차 Day 4: 데모 & 피드백 → **다음 단계(마무리)로 전환하기 위한 체크:**
   - 데모가 동작하는가?
   - 설계 문서가 최신 상태인가?
   - 팀원 피드백을 반영했는가?
5. 10주차 Day 5: 최종 정리 & 팀 발표

---

### Phase 6: 마무리 (11~12주)

> **이 단계가 끝나면:** 코드 리뷰어로서 팀에 기여하고, 독립적인 엔지니어로 활동할 준비가 된다.

#### 11주차: 코드 리뷰어로 성장

- 다른 팀원의 PR 리뷰 3건 이상
- AI 생성 코드 리뷰 가이드라인 문서 작성
- 아키텍처 의사결정 기록(ADR) 1건 작성

#### 12주차: 회고 & 독립

- 12주 전체 회고 문서 작성
- 멘토와 최종 1:1 (독립 판단)
- 팀 내 지식 공유 세션 (30분 발표)
- 다음 분기 개인 성장 목표 수립

---

## 3. DDD + Hexagonal Architecture 핵심

### 3.1 왜 AI 시대에 더 중요한가

AI 코드 생성의 가장 큰 문제는 **"맥락 없이 코드를 만든다"**는 것이다.

```
❌ AI에게: "배송비 계산 기능 만들어줘"
→ 모든 로직이 하나의 Service 클래스에 뭉침
→ 외부 API 호출이 비즈니스 로직에 직접 섞임
→ 테스트 불가능, 교체 불가능

✅ AI에게: "이 ShippingPort 인터페이스의 CJLogisticsAdapter를 구현해줘"
→ 경계가 명확하므로 AI가 정확한 범위의 코드를 생성
→ 인터페이스 계약을 지키므로 품질 검증이 쉬움
→ 다른 Adapter로 교체 가능
```

**핵심 원리:**

| 원리 | AI 시대에 중요한 이유 |
|------|----------------------|
| **인터페이스 분리** | AI에게 명확한 범위를 지정 → 생성 품질 ↑ |
| **의존성 역전** | 외부 시스템 변경 시 Adapter만 AI로 재생성 |
| **Bounded Context** | AI에게 전달할 컨텍스트 범위를 자연스럽게 제한 |
| **Aggregate** | AI가 트랜잭션 경계를 인식하여 일관성 유지 |
| **Ubiquitous Language** | 프롬프트에 도메인 용어를 쓰면 AI 이해도 ↑ |

### 3.2 물류 도메인 예시 코드 (Kotlin)

> 아래는 온보딩 3~5주차에 참고할 예시 코드다. 전체를 한 번에 읽으려 하지 말고, 주차별 실습에 맞춰 필요한 부분만 참고하자.

#### 프로젝트 구조

```
src/main/kotlin/com/example/logistics/
├── domain/         ← 비즈니스 규칙 (외부 의존성 없음)
│   ├── order/      ← Order Aggregate
│   ├── shipping/   ← Shipment Aggregate
│   └── common/     ← 공유 Value Objects (Money, Address)
├── application/    ← 유스케이스 & 포트
│   ├── port/inbound/   ← UseCase 인터페이스
│   ├── port/outbound/  ← 외부 시스템 Port
│   └── service/        ← UseCase 구현
└── adapter/        ← Port 구현체
    ├── inbound/web/    ← REST Controller
    └── outbound/       ← 외부 API, DB
```

#### Domain Layer — Value Objects

```kotlin
// 값 자체가 의미인 불변 객체들

@JvmInline
value class OrderId(val value: String) {
    init { require(value.isNotBlank()) { "OrderId는 비어있을 수 없습니다" } }
}

@JvmInline
value class ShipmentId(val value: String)

@JvmInline
value class TrackingNumber(val value: String) {
    init { require(value.matches(Regex("^[A-Z0-9]{10,20}$"))) { "유효하지 않은 운송장 번호" } }
}

data class Money(
    val amount: BigDecimal,
    val currency: String = "KRW"
) {
    init { require(amount >= BigDecimal.ZERO) { "금액은 0 이상이어야 합니다" } }

    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "통화가 다릅니다" }
        return Money(amount + other.amount, currency)
    }

    companion object {
        fun krw(amount: Long) = Money(amount.toBigDecimal(), "KRW")
        val ZERO = Money(BigDecimal.ZERO, "KRW")
    }
}

data class Address(
    val zipCode: String,
    val city: String,
    val street: String,
    val detail: String,
    val receiverName: String,
    val receiverPhone: String
) {
    /** 권역 판별 (배송비 계산에 사용) */
    fun isRemoteArea(): Boolean = zipCode.startsWith("63") || zipCode.startsWith("69")
}
```

#### Domain Layer — Aggregate Root

> **⚠️ 안티패턴 주의: `updateStatus(newStatus)` 같은 setter 메서드**
>
> 외부에서 상태값을 직접 set하는 메서드(`updateStatus`, `setStatus` 등)는 DDD 안티패턴이다.
> - **문제**: 누구나 아무 상태로든 바꿀 수 있어서, 상태 전이 규칙이 깨진다
> - **문제**: "왜 상태가 바뀌었는지" 비즈니스 의도가 코드에 드러나지 않는다
> - **문제**: 도메인 이벤트를 적절히 발행할 수 없다 (어떤 행위인지 모르니까)
> - **해결**: `pickUp()`, `startTransit()`, `deliver()`, `cancel()` 등 **비즈니스 행위를 이름으로** 표현하는 메서드를 만들고, 메서드 내부에서 상태 검증 → 변경 → 이벤트 발행을 처리한다

```kotlin
// Shipment: 배송 Aggregate Root
// Aggregate 내부 상태 변경은 반드시 Root의 도메인 행위 메서드를 통해서만

enum class ShipmentStatus {
    CREATED, PICKED_UP, IN_TRANSIT, OUT_FOR_DELIVERY, DELIVERED, RETURNED, CANCELLED
}

class Shipment private constructor(
    val id: ShipmentId,
    val orderId: OrderId,
    val address: Address,
    val carrier: CarrierType,
    val shippingFee: ShippingFee,
    private var _status: ShipmentStatus = ShipmentStatus.CREATED,
    private var _trackingNumber: TrackingNumber? = null,
) {
    val status get() = _status
    val trackingNumber get() = _trackingNumber

    // ──────────────────────────────────────────────────────
    // 도메인 행위 메서드: 외부에서 상태값을 직접 set하지 않고,
    // 비즈니스 행위를 통해 내부에서 상태를 변경한다.
    // 각 메서드가 (1) 현재 상태 검증 (2) 상태 변경 (3) 이벤트 발행을 담당한다.
    // ──────────────────────────────────────────────────────

    private val _events = mutableListOf<DomainEvent>()
    val domainEvents: List<DomainEvent> get() = _events.toList()
    fun clearEvents() = _events.clear()

    /** 집하 처리 — 택배사가 물건을 수거했을 때 */
    fun pickUp(trackingNumber: TrackingNumber) {
        require(_status == ShipmentStatus.CREATED) {
            "집하는 CREATED 상태에서만 가능합니다. 현재: $_status"
        }
        check(_trackingNumber == null) { "이미 운송장 번호가 할당됨" }
        _trackingNumber = trackingNumber
        _status = ShipmentStatus.PICKED_UP
        _events.add(ShipmentPickedUp(id, trackingNumber, Instant.now()))
    }

    /** 간선 운송 시작 — 물류 허브 간 이동 */
    fun startTransit() {
        require(_status == ShipmentStatus.PICKED_UP) {
            "간선 운송은 PICKED_UP 상태에서만 가능합니다. 현재: $_status"
        }
        _status = ShipmentStatus.IN_TRANSIT
        _events.add(ShipmentInTransit(id, Instant.now()))
    }

    /** 배송 출발 — 최종 배송 기사가 배달 시작 */
    fun startDelivery() {
        require(_status == ShipmentStatus.IN_TRANSIT) {
            "배송 출발은 IN_TRANSIT 상태에서만 가능합니다. 현재: $_status"
        }
        _status = ShipmentStatus.OUT_FOR_DELIVERY
        _events.add(ShipmentOutForDelivery(id, Instant.now()))
    }

    /** 배송 완료 */
    fun deliver() {
        require(_status == ShipmentStatus.OUT_FOR_DELIVERY) {
            "배송 완료는 OUT_FOR_DELIVERY 상태에서만 가능합니다. 현재: $_status"
        }
        _status = ShipmentStatus.DELIVERED
        _events.add(ShipmentDelivered(id, Instant.now()))
    }

    /** 반송 처리 — 수취 거부, 주소 불명 등 */
    fun returnShipment(reason: String) {
        require(_status in listOf(ShipmentStatus.IN_TRANSIT, ShipmentStatus.OUT_FOR_DELIVERY)) {
            "반송은 IN_TRANSIT 또는 OUT_FOR_DELIVERY 상태에서만 가능합니다. 현재: $_status"
        }
        _status = ShipmentStatus.RETURNED
        _events.add(ShipmentReturned(id, reason, Instant.now()))
    }

    /** 배송 취소 — 아직 간선 운송 전에만 가능 */
    fun cancel(reason: String) {
        require(_status in listOf(ShipmentStatus.CREATED, ShipmentStatus.PICKED_UP)) {
            "현재 상태(${_status})에서는 취소할 수 없습니다"
        }
        _status = ShipmentStatus.CANCELLED
        _events.add(ShipmentCancelled(id, reason, Instant.now()))
    }

    companion object {
        fun create(id: ShipmentId, orderId: OrderId, address: Address,
                   carrier: CarrierType, shippingFee: ShippingFee) =
            Shipment(id, orderId, address, carrier, shippingFee)
    }
}
```

#### Port Layer — 도메인과 외부의 경계

```kotlin
// Outbound Port: 배송사 연동 인터페이스
// 각 배송사별 Adapter가 이 인터페이스를 구현한다

interface ShippingPort {
    fun calculateFee(request: ShippingFeeRequest): ShippingFee
    fun requestPickup(shipment: Shipment): TrackingNumber
    fun getTrackingStatus(trackingNumber: TrackingNumber): TrackingStatus
    fun cancelShipment(trackingNumber: TrackingNumber): Boolean
    fun supportedCarrier(): CarrierType

    data class ShippingFeeRequest(
        val origin: Address,
        val destination: Address,
        val weightGram: Int,
        val volumeCm3: Int,
        val itemCount: Int,
        val totalItemPrice: Money
    )

    data class TrackingStatus(
        val trackingNumber: TrackingNumber,
        val status: ShipmentStatus,
        val lastLocation: String?,
        val estimatedDelivery: LocalDateTime?,
    )
}

// Inbound Port (UseCase): 외부에서 도메인을 호출하는 인터페이스
interface CalculateShippingFeeUseCase {
    fun calculate(command: CalculateShippingFeeCommand): List<ShippingFeeResult>

    data class CalculateShippingFeeCommand(
        val originAddress: Address,
        val destinationAddress: Address,
        val weightGram: Int,
        val volumeCm3: Int,
        val itemCount: Int,
        val totalItemPrice: Money,
        val preferredCarriers: List<CarrierType>? = null
    )

    data class ShippingFeeResult(
        val carrier: CarrierType,
        val fee: ShippingFee,
        val estimatedDays: Int
    )
}

// 영속성 Port
interface ShipmentRepository {
    fun save(shipment: Shipment): Shipment
    fun findById(id: ShipmentId): Shipment?
    fun findByOrderId(orderId: OrderId): List<Shipment>
}
```

#### Application Service — UseCase 구현

```kotlin
@Service
class ShippingFeeService(
    private val shippingPorts: List<ShippingPort>,
) : CalculateShippingFeeUseCase {

    private val logger = LoggerFactory.getLogger(javaClass)

    override fun calculate(
        command: CalculateShippingFeeUseCase.CalculateShippingFeeCommand
    ): List<CalculateShippingFeeUseCase.ShippingFeeResult> {

        val targetCarriers = command.preferredCarriers ?: CarrierType.entries.toList()
        val targetPorts = shippingPorts.filter { it.supportedCarrier() in targetCarriers }

        val request = ShippingPort.ShippingFeeRequest(
            origin = command.originAddress,
            destination = command.destinationAddress,
            weightGram = command.weightGram,
            volumeCm3 = command.volumeCm3,
            itemCount = command.itemCount,
            totalItemPrice = command.totalItemPrice
        )

        return targetPorts.mapNotNull { port ->
            try {
                val fee = port.calculateFee(request)
                CalculateShippingFeeUseCase.ShippingFeeResult(
                    carrier = port.supportedCarrier(),
                    fee = fee,
                    estimatedDays = estimateDeliveryDays(port.supportedCarrier(), command.destinationAddress)
                )
            } catch (e: Exception) {
                logger.warn("배송비 조회 실패 [${port.supportedCarrier()}]: ${e.message}")
                null
            }
        }.sortedBy { it.fee.totalFee.amount }
    }

    private fun estimateDeliveryDays(carrier: CarrierType, destination: Address): Int {
        val base = when (carrier) {
            CarrierType.CJ_LOGISTICS -> 2
            CarrierType.NAVER_FULFILLMENT -> 1
            CarrierType.HANJIN, CarrierType.LOTTE -> 2
        }
        return if (destination.isRemoteArea()) base + 2 else base
    }
}
```

#### Adapter Layer — 외부 시스템 연동

```kotlin
// CJ대한통운 Adapter (ShippingPort 구현)
@Component
class CJLogisticsAdapter(
    private val cjApiClient: CJLogisticsApiClient,
    private val properties: CJLogisticsProperties
) : ShippingPort {

    private val logger = LoggerFactory.getLogger(javaClass)

    override fun supportedCarrier() = CarrierType.CJ_LOGISTICS

    override fun calculateFee(request: ShippingPort.ShippingFeeRequest): ShippingFee {
        val response = cjApiClient.calculateFee(
            CJFeeRequest(
                senderZipCode = request.origin.zipCode,
                receiverZipCode = request.destination.zipCode,
                weight = request.weightGram,
                volume = request.volumeCm3
            )
        )

        val baseFee = Money.krw(response.baseFee)
        val extraFee = if (request.destination.isRemoteArea()) {
            Money.krw(response.remoteAreaSurcharge ?: 3000L)
        } else Money.ZERO

        // 무료배송 조건: 상품 총액 50,000원 이상
        val discount = if (request.totalItemPrice.amount >= BigDecimal(50000)) baseFee else Money.ZERO

        return ShippingFee(baseFee = baseFee, extraFee = extraFee,
            discountAmount = discount, carrier = CarrierType.CJ_LOGISTICS)
    }

    override fun requestPickup(shipment: Shipment): TrackingNumber {
        val response = cjApiClient.requestPickup(/* 요청 매핑 */)
        logger.info("CJ 배송 접수 완료: orderId=${shipment.orderId.value}, 운송장=${response.trackingNumber}")
        return TrackingNumber(response.trackingNumber)
    }

    override fun getTrackingStatus(trackingNumber: TrackingNumber): ShippingPort.TrackingStatus {
        val response = cjApiClient.getTracking(trackingNumber.value)
        return ShippingPort.TrackingStatus(
            trackingNumber = trackingNumber,
            status = mapCJStatus(response.status),
            lastLocation = response.lastLocation,
            estimatedDelivery = response.estimatedDelivery
        )
    }

    override fun cancelShipment(trackingNumber: TrackingNumber): Boolean = try {
        cjApiClient.cancel(trackingNumber.value); true
    } catch (e: Exception) {
        logger.error("CJ 배송 취소 실패: ${trackingNumber.value}", e); false
    }

    private fun mapCJStatus(cjStatus: String): ShipmentStatus = when (cjStatus) {
        "RECEIPT" -> ShipmentStatus.CREATED
        "PICK_UP" -> ShipmentStatus.PICKED_UP
        "IN_TRANSIT" -> ShipmentStatus.IN_TRANSIT
        "DELIVERING" -> ShipmentStatus.OUT_FOR_DELIVERY
        "DELIVERED" -> ShipmentStatus.DELIVERED
        "RETURNED" -> ShipmentStatus.RETURNED
        else -> ShipmentStatus.IN_TRANSIT
    }
}
```

#### 테스트 코드

```kotlin
class ShippingFeeServiceTest {

    private val cjPort = mockk<ShippingPort>()
    private val naverPort = mockk<ShippingPort>()
    private val service = ShippingFeeService(listOf(cjPort, naverPort))

    @Test
    fun `모든 배송사의 배송비를 비교하여 최저가순으로 반환한다`() {
        every { cjPort.supportedCarrier() } returns CarrierType.CJ_LOGISTICS
        every { naverPort.supportedCarrier() } returns CarrierType.NAVER_FULFILLMENT
        every { cjPort.calculateFee(any()) } returns ShippingFee(baseFee = Money.krw(3500), carrier = CarrierType.CJ_LOGISTICS)
        every { naverPort.calculateFee(any()) } returns ShippingFee(baseFee = Money.krw(3000), carrier = CarrierType.NAVER_FULFILLMENT)

        val results = service.calculate(defaultCommand)

        assertThat(results).hasSize(2)
        assertThat(results[0].carrier).isEqualTo(CarrierType.NAVER_FULFILLMENT) // 더 저렴한 것이 먼저
    }

    @Test
    fun `특정 배송사 조회 실패 시 해당 배송사만 결과에서 제외한다`() {
        every { cjPort.supportedCarrier() } returns CarrierType.CJ_LOGISTICS
        every { naverPort.supportedCarrier() } returns CarrierType.NAVER_FULFILLMENT
        every { cjPort.calculateFee(any()) } throws RuntimeException("CJ API 타임아웃")
        every { naverPort.calculateFee(any()) } returns ShippingFee(baseFee = Money.krw(3000), carrier = CarrierType.NAVER_FULFILLMENT)

        val results = service.calculate(defaultCommand)

        assertThat(results).hasSize(1)
        assertThat(results[0].carrier).isEqualTo(CarrierType.NAVER_FULFILLMENT)
    }
}

class ShipmentTest {

    @Test
    fun `CREATED에서 바로 배송 완료하면 예외 발생`() {
        val shipment = Shipment.create(/* ... */)
        // deliver()는 OUT_FOR_DELIVERY 상태에서만 가능
        assertThatThrownBy { shipment.deliver() }
            .isInstanceOf(IllegalArgumentException::class.java)
            .hasMessageContaining("OUT_FOR_DELIVERY 상태에서만")
    }

    @Test
    fun `정상적인 배송 흐름 — pickUp → startTransit → startDelivery → deliver`() {
        val shipment = Shipment.create(/* ... */)
        shipment.pickUp(TrackingNumber("CJ1234567890"))
        shipment.startTransit()
        shipment.startDelivery()
        shipment.deliver()
        // 각 단계마다 도메인 이벤트가 발행됨
        assertThat(shipment.domainEvents).hasSize(4)
    }

    @Test
    fun `배송 중(IN_TRANSIT) 상태에서는 취소할 수 없다`() {
        val shipment = Shipment.create(/* ... */)
        shipment.pickUp(TrackingNumber("CJ1234567890"))
        shipment.startTransit()
        assertThatThrownBy { shipment.cancel("고객 변심") }
            .isInstanceOf(IllegalArgumentException::class.java)
            .hasMessageContaining("취소할 수 없습니다")
    }
}
```

### 3.3 AI에게 구현 시키는 패턴: 인터페이스 먼저

**워크플로우:**

```
1. 사람이 Port(인터페이스) 설계 ← 핵심!
2. AI에게 Adapter 구현 지시 (인터페이스 + 외부 API 스펙 제공)
3. 사람이 UseCase 설계
4. AI에게 Service 구현 지시 (UseCase + Port 인터페이스 제공)
5. 사람이 테스트 시나리오 설계 → AI에게 테스트 코드 작성 지시
6. 사람이 테스트 실행 & 검증
```

**예시 프롬프트:**

```markdown
## 역할
너는 Kotlin + Spring Boot 백엔드 개발자야.

## 컨텍스트
물류 이커머스 시스템에서 한진택배 연동 Adapter를 구현해야 해.

## 인터페이스 (반드시 이 인터페이스를 구현)
[ShippingPort 인터페이스 코드 붙여넣기]

## 한진 API 스펙
- 배송비 조회: POST /api/v1/fee/calculate
  - Request: { senderZip, receiverZip, weight, volume }
  - Response: { fee, surcharge, estimatedDays }

## 제약 조건
- CJLogisticsAdapter를 참고하되 복사하지 말 것
- 한진 API의 에러 코드 처리 포함
- 로깅 포함 (SLF4J)
- 무료배송 기준: 상품 총액 40,000원 이상

## 출력
HanjinAdapter.kt 전체 코드
```

---

## 4. AI 도구 활용 가이드

### 4.1 🔒 AI 사용 보안 수칙

> **이것은 선택이 아니라 필수 규칙이다.** 위반 시 보안 사고로 이어질 수 있다.

#### 절대 금지 사항

| ❌ 금지 | 이유 | ✅ 대신 이렇게 |
|---------|------|---------------|
| 실제 고객 개인정보(PII) 입력 | 외부 AI에 고객 데이터 유출 | 가상 데이터로 대체 (`홍길동`, `010-0000-0000`) |
| API Key, Secret, 토큰 입력 | 시크릿 탈취 위험 | `<API_KEY>` 플레이스홀더 사용 |
| DB 접속 정보 입력 | 인프라 접근 정보 노출 | 접속 정보 대신 스키마만 공유 |
| 실제 운영 로그 전체 복붙 | 고객 정보, 내부 IP 노출 | 민감 정보 마스킹 후 필요한 부분만 발췌 |
| 사내 전용 비즈니스 로직 전체 공유 | 영업 비밀 유출 | 인터페이스/시그니처만 공유, 핵심 로직은 설명으로 |

#### AI 생성 코드 보안 검토 체크리스트

PR 머지 전 반드시 확인:

- [ ] **하드코딩된 시크릿 없는가?** — AI가 예시 값을 실제 코드에 넣는 경우가 많다
- [ ] **SQL Injection 가능성** — 문자열 연결로 쿼리 구성하지 않았는가?
- [ ] **입력값 검증** — 외부 입력을 신뢰하지 않고 validate 하는가?
- [ ] **권한 체크** — 인증/인가 로직이 빠지지 않았는가?
- [ ] **로그에 민감 정보 미포함** — 고객 전화번호, 주소 등이 로그에 찍히지 않는가?
- [ ] **의존성 보안** — AI가 추가한 라이브러리에 알려진 취약점이 없는가?

### 4.2 도구 비교

| 기능 | Claude Code | GitHub Copilot | Cursor |
|------|-------------|----------------|--------|
| **형태** | CLI / 대화형 | IDE 플러그인 (자동완성) | AI-native IDE |
| **강점** | 복잡한 설계 대화, 리팩토링, 코드 분석 | 라인 단위 자동완성, 빠른 코드 생성 | 파일 간 컨텍스트 인식, Composer |
| **적합한 상황** | 아키텍처 설계, 코드 리뷰, 복잡한 구현 | 일상적 코딩, 보일러플레이트 | 중간 규모 기능 개발, 빠른 프로토타이핑 |
| **비용** | API 사용량 기반 | $10/월 (Individual) | $20/월 (Pro) |

**추천 조합:** 설계·리뷰·분석은 Claude Code / 일상 코딩은 Copilot / 빠른 프로토타이핑은 Cursor

### 4.3 프롬프트 비용 최적화

AI API는 토큰 단위로 과금된다. 불필요한 비용을 줄이려면:

| 원칙 | 설명 | 예시 |
|------|------|------|
| **필요한 컨텍스트만 제공** | 전체 코드베이스 복붙 금지 | 인터페이스 + 관련 VO만 첨부 |
| **프롬프트 재사용** | 반복 패턴은 템플릿화 | CLAUDE.md에 공통 규칙 정의 → 매번 안 써도 됨 |
| **단계별 요청** | 한 번에 전부 시키지 않기 | 1) 설계 → 2) 구현 → 3) 테스트 순서로 |
| **모델 선택** | 단순 작업에 고성능 모델 낭비 금지 | 보일러플레이트는 작은 모델, 설계는 큰 모델 |
| **캐싱 활용** | CLAUDE.md/시스템 프롬프트 활용 | 프로젝트 컨텍스트를 매번 반복하지 않음 |

> **💡 팁:** Claude Code의 CLAUDE.md에 아키텍처 규칙을 잘 정리해두면, 매 프롬프트마다 반복 설명하지 않아도 되므로 토큰과 시간 모두 절약된다.

### 4.4 좋은 프롬프트 vs 나쁜 프롬프트

#### 나쁜 프롬프트 ❌

```
배송 기능 만들어줘                    → 범위 불명확
이 에러 고쳐줘: NPE at line 45       → 코드도 상황도 없음
주문 API 만들어줘. Spring Boot로.     → 아키텍처 제약 없음
```

#### 좋은 프롬프트 ✅

```markdown
## 역할
너는 우리 팀의 시니어 Kotlin 개발자야.
프로젝트 컨벤션: Hexagonal Architecture, kotest + mockk.

## 요청
아래 ShippingPort 인터페이스를 구현하는 HanjinAdapter를 작성해줘.

## 인터페이스
[코드]

## 제약 조건
- Spring @Component로 등록
- WebClient 사용, 타임아웃 3초, 재시도 최대 2회

## 출력
HanjinAdapter.kt 전체 코드
```

**좋은 프롬프트의 5요소:** 역할 / 컨텍스트(인터페이스, 스펙) / 제약 조건 / 출력 형식 / 예시(필요 시)

### 4.5 CLAUDE.md에 아키텍처 규칙 넣는 법

```markdown
# CLAUDE.md

## 프로젝트 개요
물류 이커머스 백엔드. Kotlin + Spring Boot 3.x + JDK 21.

## 아키텍처
- Hexagonal Architecture (Port & Adapter)
- Domain 레이어: 외부 의존성 금지 (Spring 어노테이션도 금지)
- UseCase는 반드시 Port 인터페이스를 통해 외부와 소통

## 코딩 컨벤션
- VO는 data class 또는 value class
- Entity 상태 변경은 도메인 행위 메서드로 (updateStatus/setStatus 금지 → dispatch/deliver/cancel 등 행위 이름 사용)
- 각 행위 메서드 내부에서: 상태 검증(require/check) → 상태 변경 → 도메인 이벤트 발행
- 한국어 에러 메시지

## 테스트
- kotest + mockk / Domain: 순수 단위 테스트 / Service: Port mockk

## 🔒 보안
- 프롬프트에 실제 PII, API Key 입력 금지
- AI 생성 코드에 하드코딩된 시크릿 반드시 제거

## 금지 사항
- ❌ Domain에서 Infrastructure import
- ❌ Adapter 간 직접 호출
- ❌ catch(e: Exception) 후 무시
```

### 4.6 AI 코드 리뷰 체크리스트

#### 🔴 Critical (반드시 확인)

- [ ] **아키텍처 위반:** Domain이 Infrastructure를 import하고 있지 않은가?
- [ ] **보안:** SQL Injection, XSS, 하드코딩된 시크릿 없는가?
- [ ] **널 안전성:** Kotlin인데 `!!` 남용하고 있지 않은가?
- [ ] **트랜잭션 경계:** @Transactional이 올바른 위치에 있는가?
- [ ] **에러 처리:** 예외를 삼키고 있지 않은가?

#### 🟡 Important (검토 필요)

- [ ] **비즈니스 로직 정합성:** 도메인 규칙과 일치하는가? (AI는 비즈니스를 모른다)
- [ ] **엣지 케이스:** 빈 리스트, null, 경계값, 동시성 등
- [ ] **성능:** N+1 쿼리, 불필요한 전체 조회
- [ ] **테스트:** 생성된 테스트가 실제로 의미 있는 검증을 하는가?

#### 🟢 Nice to have

- [ ] 불필요하게 복잡한 코드, 기존 코드와 중복, AI가 넣은 불필요한 주석

---

## 5. AI 인프라 구축

> **⚠️ 이 장의 기술 스택에 대해:** 팀 메인 스택은 Kotlin/Spring Boot이지만, RAG/MCP 생태계는 Python(LlamaIndex, LangChain)과 TypeScript(MCP SDK)가 가장 성숙하다. AI 인프라는 백엔드 서비스와 별도로 운영되는 도구이므로, 최적의 생태계를 선택하는 것이 실무적이다. Kotlin 기반 대안으로는 **Spring AI**, **LangChain4j**가 있으며, 팀 사정에 따라 선택하면 된다.

### 5.1 RAG 개념과 구축

#### RAG란?

LLM이 모르는 내부 정보(사내 문서, 위키)를 **검색하여** 답변에 활용하는 기법.

```
사용자 질문 → [Embedding] 벡터 변환 → [Vector Search] 유사 문서 검색
                                           ↓
                              [LLM] 질문 + 검색 결과 → 답변 생성
```

#### 핵심 개념 (신입이 알아야 할 것)

| 개념 | 한 줄 설명 | 비유 |
|------|-----------|------|
| **Embedding** | 텍스트를 숫자 벡터로 변환 | 단어를 좌표로 바꾸기 |
| **Vector Search** | 벡터 간 유사도로 관련 문서 검색 | 좌표가 가까운 문서 찾기 |
| **Chunking** | 긴 문서를 적절한 크기로 분할 | 책을 페이지로 나누기 |
| **pgvector** | PostgreSQL에 벡터 검색 추가하는 확장 | 기존 DB에 검색 엔진 부착 |

#### 구축 예시: LlamaIndex + pgvector (Python)

```python
# 왜 Python? LlamaIndex 생태계가 Python 중심으로 가장 성숙함
# Kotlin 대안: Spring AI (spring-ai-pgvector) 또는 LangChain4j

from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.core.node_parser import SentenceSplitter
from llama_index.vector_stores.postgres import PGVectorStore

# 1. 벡터 스토어 설정
vector_store = PGVectorStore.from_params(
    database="logistics_rag",
    host="localhost",
    port=5432,
    table_name="document_embeddings",
    embed_dim=1536,
)

# 2. 문서 로드 & 청킹
documents = SimpleDirectoryReader("./confluence_docs").load_data()
parser = SentenceSplitter(chunk_size=512, chunk_overlap=50)
nodes = parser.get_nodes_from_documents(documents)

# 3. 인덱스 생성 (임베딩 + 벡터DB 저장)
index = VectorStoreIndex(nodes, vector_store=vector_store)

# 4. 질의
query_engine = index.as_query_engine(similarity_top_k=5)
response = query_engine.query("배송비 무료 기준이 뭐야?")
```

#### pgvector 기본 설정

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE document_embeddings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    metadata JSONB,
    embedding vector(1536),
    created_at TIMESTAMP DEFAULT NOW()
);

-- 벡터 검색 인덱스 (시작할 때는 IVFFlat으로 충분)
CREATE INDEX ON document_embeddings
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

#### RAG 품질이 안 나올 때

| 증상 | 원인 | 해결 |
|------|------|------|
| 관련 없는 문서가 검색됨 | 청크가 너무 큼 | chunk_size 줄이기 (512→256) |
| 최신 문서가 안 나옴 | 인덱싱이 일회성 | 증분 인덱싱 Cron 구축 |
| 중복 결과 | 유사한 문서 반복 | MMR (Maximum Marginal Relevance) 적용 |

### 5.2 MCP Server 연동

#### MCP란?

AI 모델이 **외부 도구(API, DB)에 접근**할 수 있게 해주는 프로토콜. Claude Code에서 MCP 서버를 연동하면 AI가 직접 배송 상태를 조회하는 등의 작업이 가능해진다.

#### MCP 서버 예시: 배송 조회 (TypeScript)

```typescript
// 왜 TypeScript? MCP 공식 SDK가 TypeScript 기반
// Kotlin 대안: Spring AI MCP Server (spring-ai-mcp-server-webflux)

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server({
  name: "shipping-tracker",
  version: "1.0.0",
}, { capabilities: { tools: {} } });

// 도구 정의
server.setRequestHandler("tools/list", async () => ({
  tools: [{
    name: "track_shipment",
    description: "운송장 번호로 배송 상태를 조회합니다",
    inputSchema: {
      type: "object",
      properties: {
        trackingNumber: { type: "string", description: "운송장 번호" }
      },
      required: ["trackingNumber"]
    }
  }]
}));

// 도구 실행
server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;
  if (name === "track_shipment") {
    const result = await fetch(
      `https://api.internal.example.com/shipments/tracking/${args.trackingNumber}`
    );
    return { content: [{ type: "text", text: JSON.stringify(await result.json(), null, 2) }] };
  }
  throw new Error(`Unknown tool: ${name}`);
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

**Claude Code 연동 설정:**

```json
// .claude/mcp.json
{
  "mcpServers": {
    "shipping-tracker": {
      "command": "node",
      "args": ["./mcp-shipping-server/dist/index.js"],
      "env": { "API_BASE_URL": "https://api.internal.example.com" }
    }
  }
}
```

### 5.3 벡터DB 선택 가이드

| 항목 | pgvector | Qdrant |
|------|----------|--------|
| **유형** | PostgreSQL 확장 | 전용 벡터DB |
| **장점** | 기존 PG에 추가만 하면 됨, 운영 부담 최소 | 고성능, 필터링·멀티벡터 지원 |
| **추천 상황** | PG 이미 사용 중 + 문서 100만 건 이하 | 대규모 + 고성능 필요 |

**결론:** 시작할 때는 pgvector로 충분하다. 스케일업이 필요해지면 Qdrant를 검토한다.

---

## 6. 관측성 & 트러블슈팅

### 6.1 AI가 만든 코드도 모니터링이 필요하다

AI가 생성한 코드라고 해서 버그가 없는 건 아니다. 오히려 AI 코드는 **엣지 케이스 누락, 잘못된 에러 처리, 성능 문제**가 숨어있을 가능성이 높다. 배포 후 반드시 관측할 것.

### 6.2 배포 후 확인 체크리스트

AI 생성 코드가 포함된 PR 배포 후:

- [ ] **에러 로그 확인** — 배포 직후 30분간 새로운 Exception 패턴 발생 여부
- [ ] **응답 시간 확인** — 변경된 API의 p95 latency가 이전과 유사한가?
- [ ] **에러율 확인** — 해당 엔드포인트의 5xx 비율이 증가하지 않았는가?
- [ ] **비즈니스 메트릭** — 주문 수, 배송 접수 수 등이 정상 범위인가?

### 6.3 AI에게 트러블슈팅 요청하는 프롬프트

장애 상황에서 AI를 효과적으로 활용하려면, 구조화된 정보를 제공해야 한다:

```markdown
## 장애 상황
ShipmentService.track()에서 NullPointerException 발생.
배포 직후부터 5xx 에러율 3% → 15% 증가.

## 에러 로그 (민감 정보 마스킹 완료)
java.lang.NullPointerException
  at ShipmentService.track(ShipmentService.kt:45)
  at ShippingController.getTracking(ShippingController.kt:23)
발생 조건: trackingNumber가 null인 Shipment에 대해 조회 시

## 관련 코드
[해당 Service, Controller 코드 붙여넣기]

## 요청
1. 근본 원인 분석
2. 방어 코드 추가
3. 이 케이스를 잡는 단위 테스트 작성
```

> **⚠️ 주의:** 에러 로그를 AI에 전달할 때는 고객 정보, 내부 IP, DB 접속 정보 등을 반드시 마스킹한다. (4.1 보안 수칙 참고)

### 6.4 기본 로깅 패턴

AI에게 코드를 생성시킬 때, 다음 로깅 규칙을 CLAUDE.md에 포함시켜두면 운영 시 트러블슈팅이 쉬워진다:

```kotlin
// 좋은 로깅 예시
logger.info("배송비 조회 완료 [carrier={}, orderId={}, fee={}]",
    carrier.name, orderId.value, fee.totalFee.amount)

logger.warn("배송비 조회 실패 [carrier={}, orderId={}]: {}",
    carrier.name, orderId.value, e.message)

// ❌ 나쁜 로깅 — 민감 정보 포함
logger.info("배송 접수: 수신자=${address.receiverName}, 전화=${address.receiverPhone}")
```

---

## 7. 멘토링 원칙

### 7.1 AI는 도구지 실력이 아니다

> "ChatGPT가 코드를 짜줬으니까 나는 잘하는 거다" — ❌

AI가 생성한 코드를 **이해하지 못하면** 그건 실력이 아니다.

**멘토가 할 질문:**
- "이 코드가 왜 이렇게 동작하는지 설명해봐"
- "AI가 생성한 이 부분, 다른 방법으로 구현하면?"
- "이 쿼리가 느려지면 어떻게 최적화해?"

### 7.2 기본기를 건너뛰지 마라

AI 시대에도 변하지 않는 기본기:

| 분야 | AI가 못 대신해주는 것 |
|------|---------------------|
| **자료구조/알고리즘** | "이 코드가 O(n²)인데 괜찮은가?" 판단 |
| **네트워크** | 타임아웃, 재시도, 서킷브레이커 설계 판단 |
| **데이터베이스** | 인덱스 전략, 락 이슈 판단 |
| **설계 원칙** | "이 설계가 확장 가능한가?" 판단 |

**실습:** 매주 1회 "기본기 퀴즈" — 멘토가 기본 CS 질문을 던지고, AI 없이 답변

### 7.3 실패 경험의 가치

AI가 만들어준 코드가 **스테이징에서 터지는 경험**이 최고의 학습이다.

**의도적으로 실패를 경험시키는 방법:**
1. AI 생성 코드를 검증 없이 스테이징에 배포 → 문제 발견하게 함
2. 엣지 케이스가 빠진 AI 코드를 리뷰 없이 머지 → 버그 리포트 대응
3. AI가 만든 쿼리의 실행 계획을 안 보고 배포 → 슬로우 쿼리 발견

**멘토의 역할:** 안전한 환경(스테이징)에서 실패하게 하고, 그 경험을 학습으로 전환

### 7.4 테스트가 검증 장치

> AI가 코드를 만들면, 사람은 테스트를 만든다.

**테스트 작성 원칙:**
1. **테스트 시나리오는 사람이 설계** — AI에게 위임 가능하지만 최종 판단은 사람
2. **AI 생성 테스트는 의미 있는지 확인** — 항상 통과하는 무의미한 테스트 주의
3. **실패 케이스를 먼저 설계** — AI는 "통과하는 테스트"를 만들려 하므로

---

## 8. 평가 기준

### 8.1 주차별 체크리스트

#### 2주차 평가 (기초)

| 항목 | 기준 | Pass |
|------|------|------|
| AI 도구 세팅 | Claude Code + Copilot + Cursor 모두 설치·동작 | ☐ |
| 프롬프트 기본 | 구조화된 프롬프트로 원하는 코드 생성 가능 | ☐ |
| 코드 리뷰 | AI 생성 코드에서 문제점 1개 이상 발견 | ☐ |
| PR 제출 | 최소 2건의 PR 머지 | ☐ |
| 회고 | 1~2주 회고 문서 작성 완료 | ☐ |

#### 4주차 평가 (아키텍처)

| 항목 | 기준 | Pass |
|------|------|------|
| DDD 이해 | Entity/VO/Aggregate 차이를 자기 말로 설명 가능 | ☐ |
| Hexagonal 이해 | Port/Adapter가 왜 필요한지 설명 가능 | ☐ |
| AI 협업 | Port 인터페이스를 설계하고, AI에게 Adapter 구현 지시 가능 | ☐ |
| 회고 | 3~4주 회고 문서 작성 완료 | ☐ |

> **💡 참고:** "완벽한 DDD 설계"가 아니라 "AI에게 명확한 경계를 주는 프롬프트를 작성할 수 있는가"가 핵심 기준이다.

#### 6주차 평가 (AI 활용 개발)

| 항목 | 기준 | Pass |
|------|------|------|
| 실전 태스크 | 백로그 태스크 2건 이상 완료 | ☐ |
| AI 활용 로그 | 각 PR에 AI 활용 내역 기록 | ☐ |
| 코드 품질 | AI 생성 코드 리뷰 통과율 70% 이상 | ☐ |
| 테스트 | 직접 설계한 테스트 시나리오 포함 | ☐ |
| 보안 확인 | AI 생성 코드 보안 체크리스트 적용 | ☐ |

> **리뷰 통과율 측정 방법:** PR 리뷰에서 아키텍처 위반, 보안 이슈, 비즈니스 로직 오류로 reject된 비율을 추적한다. 10건 중 7건 이상 수정 없이(또는 경미한 수정만으로) 통과하면 70%.

#### 8주차 평가 (AI 인프라)

| 항목 | 기준 | Pass |
|------|------|------|
| RAG 이해 | Embedding → Vector Search → LLM 파이프라인 설명 가능 | ☐ |
| pgvector | 로컬에서 벡터 검색 동작 | ☐ |
| MCP | 간단한 MCP 서버 1개 구현 & Claude Code 연동 | ☐ |

#### 10주차 평가 (자율 프로젝트)

| 항목 | 기준 | Pass |
|------|------|------|
| 프로젝트 완성 | 데모 가능한 수준의 결과물 | ☐ |
| 설계 문서 | 아키텍처 설계 문서 작성 | ☐ |
| AI 활용 | 프로젝트 전 과정에서 AI 효과적 활용 | ☐ |
| 발표 | 팀 내 발표 완료 | ☐ |

#### 12주차 최종 평가

| 항목 | 기준 | Pass |
|------|------|------|
| 코드 리뷰어 | 타인 PR 리뷰 3건 이상 (건설적 피드백) | ☐ |
| ADR | 아키텍처 의사결정 기록 1건 작성 | ☐ |
| 회고 | 12주 전체 회고 문서 작성 | ☐ |
| 지식 공유 | 30분 팀 발표 완료 | ☐ |

### 8.2 독립 가능 판단 기준

아래 **5개 항목 모두 충족** 시 독립(멘토링 종료) 판단:

1. **자율적 태스크 수행** — 멘토 도움 없이 분석 → 설계 → 구현 → PR 가능
2. **아키텍처 준수** — Hexagonal 패턴 위반 없이 올바른 레이어에 코드 배치
3. **AI 출력 비판적 검증** — "AI가 이렇게 했으니까 맞겠지"가 아닌, 근거 기반 판단
4. **기본기** — CS 기초 질문 답변 가능, SQL 실행 계획 읽기 가능
5. **팀 기여** — 코드 리뷰 피드백 제공, 팀 문서/위키 기여

---

## 부록

### A. 추천 학습 자료

| 분야 | 자료 | 형태 |
|------|------|------|
| DDD | 「도메인 주도 설계 핵심」 (반 버논) | 도서 |
| Hexagonal | Alistair Cockburn 원문 + Netflix Hexagonal Architecture 블로그 | 아티클 |
| Kotlin | 「코틀린 인 액션」 2판 | 도서 |
| AI 활용 | Anthropic Prompt Engineering Guide | 공식 문서 |
| RAG | LlamaIndex 공식 문서 + 튜토리얼 | 공식 문서 |
| MCP | Model Context Protocol 공식 스펙 | 공식 문서 |
| 테스트 | 「단위 테스트」 (블라디미르 코리코프) | 도서 |
| Spring AI | Spring AI 공식 문서 (Kotlin 기반 RAG 대안) | 공식 문서 |

### B. 용어 사전

| 용어 | 설명 |
|------|------|
| Aggregate | 일관성 경계를 가진 도메인 객체 클러스터 |
| Bounded Context | 도메인 모델의 적용 경계 |
| Port | 도메인과 외부의 소통 인터페이스 |
| Adapter | Port의 구체적 구현체 |
| RAG | Retrieval-Augmented Generation, 검색 증강 생성 |
| MCP | Model Context Protocol, AI 도구 연동 프로토콜 |
| Embedding | 텍스트를 고차원 벡터로 변환하는 것 |
| Vector DB | 벡터 유사도 검색에 특화된 데이터베이스 |
| Ubiquitous Language | 개발자와 도메인 전문가가 공유하는 공통 언어 |
| PII | Personally Identifiable Information, 개인 식별 정보 |

### C. AI 활용 로그 템플릿

PR에 아래 형식으로 코멘트를 첨부한다:

```markdown
## AI 활용 로그

| 항목 | 내용 |
|------|------|
| 사용 도구 | Claude Code / Copilot / Cursor |
| AI 작성 비율 | 약 60% |
| 직접 작성 | Port 인터페이스, 테스트 시나리오 설계 |
| AI 작성 | Adapter 구현, 보일러플레이트, DTO |
| AI 수정 사항 | 에러 처리 로직 보강, 불필요한 주석 제거 |
| 보안 체크 | ✅ 시크릿 미포함 확인, 입력값 검증 확인 |
```

### D. 멘토-멘티 주간 1:1 체크리스트

```markdown
## 주간 1:1 (n주차)

### 지난주 리뷰
- [ ] 지난주 목표 달성 여부
- [ ] PR 머지 현황
- [ ] 어려웠던 점

### 이번 주 계획
- [ ] 이번 주 학습/태스크 목표
- [ ] 필요한 도움

### AI 활용 리뷰
- [ ] AI를 효과적으로 사용한 사례
- [ ] AI 출력을 수정한 사례 (왜 수정했는지)
- [ ] AI 없이 해본 경험 (있다면)

### 기본기 퀴즈 (1문제)
- 질문:
- 답변:
```

---

> **이 문서는 살아있는 문서입니다.** 멘토링을 진행하면서 팀 상황에 맞게 지속적으로 업데이트하세요.
>
> 최종 수정: 2026-03-11

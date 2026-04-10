# 02. 서비스 분해 원칙 — Bounded Context와 단일 책임

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Bounded Context란 무엇이며, 왜 서비스 경계의 기준이 되는가?
- "기능 단위"로 서비스를 나누면 왜 분산 모놀리스가 되는가?
- 단일 책임 원칙(SRP)을 서비스 수준에서 어떻게 적용하는가?
- Business Capability 기반 분해와 Bounded Context 기반 분해의 차이는 무엇인가?
- 서비스가 너무 작은지, 너무 큰지 판단하는 기준은 무엇인가?

---

## 🔍 왜 이 개념이 MSA에서 중요한가

서비스 경계를 어디서 긋느냐가 MSA 성공의 80%를 결정한다. 경계를 잘못 그으면 서비스 간 호출이 폭발적으로 늘어나고, 데이터 일관성 유지가 불가능해지며, 결국 서비스를 다시 합쳐야 하는 상황이 온다. 반대로 경계가 너무 크면 모놀리스와 차이가 없다. Bounded Context는 DDD(도메인 주도 설계)에서 가져온 개념으로, 언어와 모델이 일관되게 유지되는 경계를 정의한다.

> 📌 선행 학습: [ddd-deep-dive](../README.md#-선행-학습--연결-레포) — Bounded Context의 심화 개념

---

## 😱 흔한 실수 (Before — 기능 단위로 잘게 쪼갤 때)

```
잘못된 분해 방식: 기능(Function) 단위 분해

OrderCreateService   → 주문 생성만 담당
OrderReadService     → 주문 조회만 담당
OrderUpdateService   → 주문 상태 변경만 담당
OrderDeleteService   → 주문 취소만 담당

문제점:
  - 주문 생성이 주문 조회를 HTTP로 호출 → 네트워크 의존
  - 주문 상태 변경이 주문 데이터를 읽으려면 OrderReadService 호출
  - 하나의 비즈니스 로직이 4개 서비스에 분산됨
  - 트랜잭션 경계가 네트워크를 가로지름 → 데이터 불일치

결과: CRUD를 서비스로 쪼갠 것일 뿐, Bounded Context 분리가 아님
```

---

## ✨ 올바른 접근 (After — Bounded Context 기반 분해)

```
올바른 분해 방식: Bounded Context 기반

주문 컨텍스트 (Order Context)
  → 주문 생성, 조회, 수정, 취소 모두 포함
  → "주문"에 관한 모든 언어와 모델이 일관됨
  → order_db 단독 소유

결제 컨텍스트 (Payment Context)
  → 결제 처리, 환불, 결제 내역
  → "결제"라는 언어 맥락에서의 모델
  → payment_db 단독 소유

배송 컨텍스트 (Delivery Context)
  → 배송 생성, 배송 추적, 배송 완료
  → "배송"이라는 언어 맥락에서의 모델
  → delivery_db 단독 소유

경계 확인 기준:
  "이 서비스의 코드베이스를 팀 하나가 독립적으로 배포할 수 있는가?"
  "이 서비스의 DB 스키마 변경이 다른 서비스 코드를 수정하게 만드는가?"
```

---

## 🔬 내부 동작 원리 — 분해 기준의 3가지 접근법

### 1. Bounded Context 기반 분해 (DDD 연결)

DDD에서 Bounded Context는 "동일한 용어가 동일한 의미를 가지는 경계"다. 예를 들어 "Product(상품)"는 컨텍스트마다 의미가 다르다.

```
상품 컨텍스트에서 Product:
  - name, description, price, stock, category, images
  - 상품 등록, 수정, 삭제, 검색

주문 컨텍스트에서 Product:
  - productId, productName, orderedPrice (주문 당시 가격)
  - 주문 시점의 가격 스냅샷만 필요 (현재 가격 불필요)

배송 컨텍스트에서 Product:
  - productId, weight, size
  - 물리적 배송 정보만 필요

→ 같은 "상품"이지만 컨텍스트마다 필요한 속성이 다름
→ 하나의 Product 서비스가 이 모든 것을 담으면 책임 과부하
→ 각 컨텍스트가 자신에게 필요한 Product 속성만 복사하여 소유
```

**Bounded Context 경계를 찾는 질문들:**

```
1. 이 팀의 언어(Ubiquitous Language)가 다른 팀과 다른가?
   "주문팀은 '고객'을 OrderCustomer로, 회원팀은 Member로 부른다"
   → 언어가 다르면 컨텍스트가 다른 것

2. 이 도메인의 규칙이 다른 도메인과 독립적인가?
   "결제 정책 변경이 주문 로직을 바꾸지 않는가?"
   → 규칙이 독립적이면 분리 가능

3. 이 데이터의 생명주기가 다른 데이터와 다른가?
   "주문이 취소돼도 결제 기록은 보존되어야 하는가?"
   → 생명주기가 다르면 별도 컨텍스트
```

### 2. Business Capability 기반 분해

비즈니스 능력(Business Capability)은 조직이 비즈니스 가치를 창출하기 위해 수행하는 기능 단위다. 기술이 아닌 비즈니스 관점에서 분해한다.

```
전자상거래의 Business Capability:

주문 관리 (Order Management)
  → 주문 생성, 주문 취소, 주문 이력 조회

결제 처리 (Payment Processing)
  → 결제 승인, 환불 처리, 결제 수단 관리

재고 관리 (Inventory Management)
  → 재고 확인, 재고 차감, 재고 복원

배송 관리 (Shipping Management)
  → 배송 요청, 배송 추적, 배송 완료 처리

고객 관리 (Customer Management)
  → 회원 가입, 로그인, 프로필 관리

상품 카탈로그 (Product Catalog)
  → 상품 등록, 상품 검색, 카테고리 관리

→ 각 Capability가 하나의 서비스 후보
→ Capability가 커지면 하위 Capability로 분해
```

### 3. 단일 책임 원칙(SRP) — 서비스 수준 적용

클래스 수준의 SRP("하나의 클래스는 하나의 이유로만 변경돼야 한다")를 서비스 수준으로 확장한다.

```
SRP 위반 사례:
  "주문 서비스"가 아래를 모두 담당할 때
  - 주문 생성/조회/취소
  - 결제 처리 (결제 API 호출)
  - 재고 차감 (재고 DB 직접 업데이트)
  - 배송 요청 (배송사 API 호출)
  - 알림 발송 (이메일, SMS)

  → 결제 정책 변경 시 주문 서비스 수정 필요
  → 배송사 API 변경 시 주문 서비스 수정 필요
  → 알림 전략 변경 시 주문 서비스 수정 필요
  → 이유가 5가지 → SRP 위반

SRP 준수:
  주문 서비스: 주문의 생명주기만 관리
  → 변경 이유: 주문 비즈니스 로직 변경 시에만

  결제 서비스: 결제 처리만 담당
  → 변경 이유: 결제 정책, PG사 변경 시에만

  배송 서비스: 배송 요청/추적만 담당
  → 변경 이유: 배송사 API 변경 시에만

  알림 서비스: 알림 발송만 담당
  → 변경 이유: 알림 채널(이메일/SMS/푸시) 변경 시에만
```

---

## 💻 실전 구현 — 서비스 경계 설계 예시

### 주문 서비스의 올바른 경계

```java
// ✅ 올바른 주문 서비스 — 주문의 생명주기만 관리
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    public Order placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(
            command.getCustomerId(),
            command.getOrderItems()
        );
        orderRepository.save(order);

        // 직접 호출 ❌ → 이벤트 발행 ✅
        eventPublisher.publishEvent(new OrderPlacedEvent(order.getId(), order.getItems()));

        return order;
    }

    public void confirmOrder(OrderId orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        order.confirm();
        orderRepository.save(order);
    }

    public void cancelOrder(OrderId orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        order.cancel();
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderCancelledEvent(order.getId()));
    }
}
```

```java
// ❌ 잘못된 주문 서비스 — 책임 과부하
@Service
public class OrderService {

    private final PaymentClient paymentClient;     // 결제 서비스 직접 호출
    private final InventoryClient inventoryClient; // 재고 서비스 직접 호출
    private final ShippingClient shippingClient;   // 배송 서비스 직접 호출
    private final NotificationClient notifClient;  // 알림 서비스 직접 호출

    public Order placeOrder(PlaceOrderCommand command) {
        inventoryClient.checkStock(command.getItems()); // 실패 시 주문 전체 실패
        paymentClient.processPayment(command.getPayment());
        Order order = orderRepository.save(Order.from(command));
        shippingClient.requestShipping(order);
        notifClient.sendOrderConfirmation(order);
        return order;
        // → 이 메서드 하나가 4개 서비스 장애에 모두 노출됨
    }
}
```

### 서비스 크기 기준

```
너무 작은 서비스 신호:
  - 기능 하나를 구현하려면 항상 다른 서비스를 수정해야 함
  - 서비스 간 호출이 같은 팀 내에서 발생
  - 한 번의 사용자 요청이 10개 이상의 서비스를 직렬 호출

너무 큰 서비스 신호:
  - 두 팀 이상이 같은 서비스를 수정
  - 배포 시 여전히 다른 팀의 승인이 필요
  - 서비스의 기능 목록이 5개 이상의 무관한 도메인을 포함

균형점:
  "팀이 독립적으로 배포할 수 있는 가장 큰 단위"
  = Bounded Context 하나 = 서비스 하나 (원칙)
```

---

## 📊 패턴 비교 — 분해 전략 비교

| 기준 | 기능(Function) 분해 | Business Capability | Bounded Context |
|------|-------------------|--------------------|--------------| 
| 분해 기준 | CRUD 단위 | 비즈니스 가치 | 언어/모델 일관성 |
| 경계 결정자 | 개발자 | 비즈니스 팀 | 도메인 전문가 |
| 데이터 소유 | 불명확 | 서비스별 소유 | 컨텍스트별 소유 |
| 장점 | 구현 단순 | 비즈니스 정렬 | DDD와 자연 연결 |
| 단점 | 분산 모놀리스 위험 | 경계 재조정 빈번 | 학습 비용 |
| 권장 상황 | 비권장 | 도메인 복잡도 낮을 때 | 복잡 도메인 |

---

## ⚖️ 트레이드오프

```
서비스를 더 잘게 쪼갤수록:
  ✅ 독립 배포 범위 작아짐
  ✅ 변경 영향 범위 제한
  ❌ 서비스 간 네트워크 호출 증가
  ❌ 분산 트랜잭션 범위 확대
  ❌ 운영 복잡도 증가

서비스를 더 크게 묶을수록:
  ✅ 네트워크 호출 감소
  ✅ 트랜잭션 경계 단순
  ✅ 운영 대상 서비스 수 감소
  ❌ 배포 결합 증가
  ❌ 팀 결합도 증가
```

---

## 📌 핵심 정리

```
✅ Bounded Context
   - 동일한 언어와 모델이 일관되게 사용되는 경계
   - 컨텍스트 내부에서 자유롭게 변경, 외부에는 계약으로만 통신
   - DDD의 핵심 개념 — ddd-deep-dive 참조

✅ 서비스 분해 3가지 기준
   1. Bounded Context: 언어/모델 일관성 경계
   2. Business Capability: 비즈니스 가치 단위
   3. SRP: 하나의 이유로만 변경되는 단위

✅ 경계 검증 질문
   - "팀이 이 서비스만 독립 배포할 수 있는가?"
   - "DB 스키마 변경이 다른 서비스 코드를 수정하게 만드는가?"
   - "서비스의 변경 이유가 하나인가?"
```

---

## 🤔 생각해볼 문제

**Q1.** 상품(Product) 정보가 주문 서비스, 배송 서비스, 검색 서비스에서 모두 필요하다. 어떻게 설계해야 하는가?

<details>
<summary>해설 보기</summary>

각 서비스가 자신에게 필요한 상품 속성만 복사하여 소유한다(데이터 복제 허용). 주문 서비스는 `orderedProductName`, `orderedPrice`(주문 당시 스냅샷), 배송 서비스는 `weight`, `size`, 검색 서비스는 `name`, `description`, `category`를 자체 DB에 가진다. 상품 서비스에서 이벤트를 발행하면 각 서비스가 구독하여 자신의 데이터를 갱신한다.

</details>

**Q2.** 결제 서비스와 정산 서비스를 같은 Bounded Context로 볼 수 있는가?

<details>
<summary>해설 보기</summary>

아니다. 결제(Payment)와 정산(Settlement)은 언어와 생명주기가 다르다. 결제는 실시간 거래 처리(초 단위), 정산은 일/월 단위 회계 처리다. "결제"에서 `amount`는 결제 금액이지만, "정산"에서 `amount`는 수수료 공제 후 정산 금액이다. 담당 팀도 다르다(결제팀 vs 재무팀). Bounded Context가 다른 두 도메인이므로 서비스를 분리하고, 결제 이벤트를 정산 서비스가 구독하는 방식으로 연결한다.

</details>

**Q3.** 알림 기능(이메일, SMS, 푸시)을 별도 서비스로 분리해야 하는가?

<details>
<summary>해설 보기</summary>

분리해야 한다. 알림은 독립적인 Business Capability이며 변경 이유가 다르다. 주문 서비스는 `OrderConfirmedEvent`를 발행하고, 알림 서비스가 이를 구독하여 어떤 채널로 보낼지 결정한다. 이로써 알림 채널 추가·변경이 주문 서비스에 영향을 주지 않는다. 알림이 실패해도 주문 처리에 영향이 없어야 하므로 이벤트 기반 비동기 통신이 적합하다.

</details>

---

<div align="center">

**[⬅️ 이전: MSA가 해결하는 문제](./01-msa-problem-and-cost.md)** | **[홈으로 🏠](../README.md)** | **[다음: 분산 모놀리스 안티패턴 ➡️](./03-distributed-monolith-antipatterns.md)**

</div>

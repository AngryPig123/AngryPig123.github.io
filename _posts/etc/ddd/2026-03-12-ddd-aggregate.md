---
title: DDD - Aggregate란 무엇인가?
description: Entity와 Value Object를 묶어 일관성을 유지하는 단위인 Aggregate 개념 정리
date: 2026-03-12T15:30:00+09:00
categories: [Architecture, DDD]
tags: [DDD, Domain Driven Design, Aggregate, AggregateRoot, Hexagonal Architecture, 설계, 아키텍처]
---

<h2> 0. 알아보게 된 이유 </h2>

도메인 모델을 설계할 때 객체를 단순히 클래스 단위로 나누는 것이 아니라 의미 있는 기준으로 나누어야 한다는 것을 알 수 있다.

하지만 실제로 설계를 하다보면 객체 하나만 존재하는 경우는 거의 없고 여러 개의 객체가 서로 관계를 가지면서 하나의 기능을 구성하게 된다.

이때 객체를 각각 따로 관리하면

- 일관성이 깨지고
- 상태 관리가 어려워지고
- 트랜잭션 범위가 커지고
- 코드가 복잡해짐

위와 같은 문제가 발생한다.

`DDD`에서는 이런 문제를 해결하기 위해 객체들을 하나의 단위로 묶어서 관리하는 개념을 사용하는데 그 개념이 바로 `Aggregate`다.

이번 글에서는

- `Aggregate`란 무엇인가
- 왜 필요한가
- `Aggregate Root`는 무엇인가
- 어떤 기준으로 묶어야 하는가

를 중심으로 정리해 보려고 한다.

<h2> 1. Aggregate 란? </h2>

`Aggregate`는 관련된 `Entity`와 `Value Object`를 하나의 단위로 묶은 것이다.

이 단위에서는 데이터의 일관성이 반드니 유지되어야 한다. 즉,

```
일관성을 유지해야 하는 객체들의 집합
```

이라고 볼 수 있다.

`DDD`에서는

- 객체를 개별로 관리하지 않고.
- `Aggregate` 단위로 관리한다.

그리고 외부에서는 `Aggregate` 내부 객체에 직접 접근하지 않는다.

<h2> 2. 왜 Aggregate가 필요한가 </h2>

객체를 자유롭게 참조하게 설계를 하면 문제가 발생한다.
모든 객체가 서로 참조하게 되면

- 변경 영향 범위가 커지고
- 트랜잭션이 커지고
- 버그 발생 가능성이 높아진다.

또한 상태 일관성을 유지하기 어려워진다.
예시로 주문 상태는 `cancel`이지만 결제 상태는 `success`인 상태로 존재하는 상태 일관성이 달라지는 문제가 발생할 수 있다.

이를 막기 위해 

```
일관성을 유지해야 하는 범위를 하나로 묶는다.
```

이것이 `Aggregate`다.

![order-service-domain-logic](/assets/img/ddd/order-service-domain-logic.png)

<h2> 3. Aggregate Root </h2>

`Aggregate`에는 반드시 `Root`가 존재한다. `Root`는 `Aggregate`의 대표 객체이다.
외부에서는 `Root`를 통해서만 접근 할 수 있다.

- 예시
  - `Order(Aggregate Root)`
  - `OrderItem`
  - `Product`
  - `Money`

위와 같이 구성되어 있을때 외부에서 접근할때에는 `Order`객체를 통해서만 소통을 해야한다.즉, 직접적으로 `OrderItem` 객체나 `Product`객체에 접근해서 동작을 하면 안된다.

위와 같은 구조에서는 외부에서 `Aggregate` 내부 요소에 직접 접근하지 않고,
반드시 Order를 통해서만 상호작용해야 한다.

즉, `OrderItem`이나 `Product`에 직접 접근하여 상태를 변경하거나 비즈니스 로직을 수행하는 방식은 지양해야 한다.
아래 예시 코드와 같이 Order가 `Aggregate Root`로서 내부 객체를 관리하고,
외부 요청 역시 `Order`를 통해 처리되어야 한다.

```java
public class Order extends AggregateRoot<OrderId> {

    private final CustomerId customerId;
    private final RestaurantId restaurantId;
    private final StreetAddress deliveryAddress;
    private final Money price;
    private final List<OrderItem> items;

    private TrackingId trackingId;
    private OrderStatus orderStatus;
    private List<String> failureMessages;

    public void initializeOrder() {
        setId(new OrderId(UUID.randomUUID()));
        trackingId = new TrackingId(UUID.randomUUID());
        orderStatus = OrderStatus.PENDING;
        initializeOrderItems();
    }

    public void validateOrder() {
        validateInitialOrder();
        validateTotalPrice();
        validateItemsPrice();
    }

    public void pay() {
        if (orderStatus != OrderStatus.PENDING) {
            throw new OrderDomainException("Order is not in correct state for pay operation");
        }
        orderStatus = OrderStatus.PAID;
    }

    public void approve() {
        if (orderStatus != OrderStatus.PAID) {
            throw new OrderDomainException("Order is not in correct state for approve operation");
        }
        orderStatus = OrderStatus.APPROVED;
    }

    public void initCancel(List<String> failureMessages) {
        if (orderStatus != OrderStatus.PAID) {
            throw new OrderDomainException("Order is not in correct state for initCancel operation");
        }
        orderStatus = OrderStatus.CANCELLING;
        updateFailureMessages(failureMessages);
    }

    public void cancel(List<String> failureMessages) {
        if (!(orderStatus == OrderStatus.CANCELLING || orderStatus == OrderStatus.PENDING)) {
            throw new OrderDomainException("Order is not in correct state for cancel operation");
        }
        orderStatus = OrderStatus.CANCELLED;
        updateFailureMessages(failureMessages);
    }

    private void updateFailureMessages(List<String> failureMessages) {
        if (this.failureMessages != null && failureMessages != null) {
            this.failureMessages.addAll(
                    failureMessages.stream()
                            .filter(message -> !message.isEmpty())
                            .toList()
            );
        }
        if (this.failureMessages == null) {
            this.failureMessages = failureMessages;
        }
    }

    private void validateInitialOrder() {
        if (orderStatus != null || getId() != null) {
            throw new OrderDomainException("Order is not in correct state for initialization!");
        }
    }

    private void validateTotalPrice() {
        if (price == null || price.isGreaterThanZero()) {
            throw new OrderDomainException("Total price must be greater than zero!");
        }
    }

    private void validateItemsPrice() {
        Money orderItemsTotal = items.stream().map(orderItem -> {
            validateItemPrice(orderItem);
            return orderItem.getSubTotal();
        }).reduce(Money.ZERO, Money::add);
        if (!price.equals(orderItemsTotal)) {
            throw new OrderDomainException(
                    "Total price: " +
                            price.getAmount() +
                            "is not equal to Order items total: " +
                            orderItemsTotal.getAmount() +
                            "!"
            );
        }
    }

    private void validateItemPrice(OrderItem orderItem) {
        if (!orderItem.isPriceValid()) {
            throw new OrderDomainException(
                    "Order item price: " +
                            orderItem.getPrice().getAmount() +
                            "is not valid for product " +
                            orderItem.getProduct().getId().getValue()
            );
        }
    }

    private void initializeOrderItems() {
        long itemId = 1;
        for (OrderItem orderItem : items) {
            orderItem.initializeOrderItem(super.getId(), new OrderItemId(itemId++));
        }
    }
}
```

위와 같이 내부 객체를 조작하는 메소드를 작성하여 `Aggregate Root`를 통해서만 내부 `Aggregate`를 변경해야한다.

<h2> 4. Aggregate 설계 규칙 </h2>

`DDD`에서 자주 말하는 규칙은 아래와 같다.

- `Aggregate Root`만 외부에 공개
- 내부 객체 직접 수정 금지
- `Aggregate` 단위로 저장
- `Aggregate` 단위로 트랜잭션
- `Aggregate`간 직접 참조 최소화

즉, 좋은 설계 기준은
```
한번에 일관성을 유지해야 하는 범위를 Aggregate로 지정
```

이다.

<h2> 5. 정리 </h2>

`Aggregate`는 관련된 `Entity`와 `Value Object`를 묶어서 일관성을 유지하는 단위이고
`DDD`에서는 객체를 개별로 관리하지 않고 `Aggregate` 단위로 관리한다. 

`Aggregate`단위를 잘 나누면

- 변경 영향 범위가 줄고
- 트랜잭션이 단순해지고
- 코드 이해가 쉬워지고
- 유지 보수가 쉬워진다

즉, 좋은 도메인 설계는 

```
어디까지를 하나로 묶어야 하는가
```

를 결정하는 것에서 시작된다.

<h2>6. 다음에 알아볼 내용</h2>

`Aggregate`를 기준으로 모델을 나누었다면,  
이제 다음으로 고민해야 할 것은 비즈니스 로직을 어디에 위치시킬 것인지이다.

실무에서는 하나의 `Aggregate` 내부에서만 처리할 수 없는 로직이 자주 등장한다.  
여러 개의 `Aggregate`가 함께 동작해야 하는 경우도 있고,  
도메인 규칙이지만 특정 객체에 귀속시키기 어려운 로직도 존재한다.

`DDD`에서는 이러한 로직을 다음과 같이 구분해서 배치한다.

- `Entity` 내부에 둘 것인지
- `Domain Service`에 둘 것인지
- `Application Service`에 둘 것인지

이 구분을 명확하게 하지 않으면  
로직이 한 곳에 몰리거나, 반대로 여러 곳에 흩어져서  
모델의 의미가 흐려지고 유지보수가 어려워진다.

다음 글에서는  
`Domain Service`와 `Application Service`의 차이를 중심으로  
각 계층에 어떤 책임을 두어야 하는지 정리해보려고 한다.
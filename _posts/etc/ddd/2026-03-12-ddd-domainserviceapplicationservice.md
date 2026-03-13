---
title: DDD - Domain Service와 Application Service의 차이
description: 비즈니스 로직을 어디에 두어야 하는지, Domain Service와 Application Service의 역할 정리
date: 2026-03-12T16:40:00+09:00
categories: [Architecture, DDD]
tags: [DDD, Domain Driven Design, Domain Service, Application Service, Hexagonal Architecture, 설계, 아키텍처]
---

<h2> 0. 알아보게 된 이유 </h2>

`Aggregate`를 나누고 나면 다음으로 고민하게 되는 것은 비즈니스 로직을 어디에 둘 것인가? 이다. 현실 세계에서는 `Aggregate`만으로 해결되지 않는 요구사항들이 다수 존재하기 때문이다.

처음에는 모든 로직을 `Entity`에 넣으려고 하거나, 반대로 `Service`로 퉁쳐서 넣어버리는 경우가 많다. 하지만 `DDD`에서는 로직의 성격에 따라

- `Entity`에 둘것인지
- `Domain Service`에 둘것인지
- `Application Service`에 둘것인지

를 구분해야 한다고 말한다.
1
이번 글에서는

- `Domain Service`란 무엇인가
- `Application Service`란 무엇인가
- 둘의 차이는 무엇인가
- 언제 어떤 것을 사용해야 하는가

를 중심으로 정리해보려고 한다.

<h2> 1. Domain Service 란? </h2>

`Domain Service`는 특정 `Entity`에 속하지 않는 도메인 로직을 담당하는 객체이다.
도메인 규칙이지만 어느 하나의 `Entity`안에 넣기 애매한 경우에 사용한다. 즉,

```
도메인 로직이지만 특정 객체의 책임이 아닌 경우
```

`Domain Service`를 만든다.

예시

- 주문 금액 계산
- 할인 정책 계산
- 포인트 사용 규칙
- 배송비 계산

이런 로직은 `Order`안에 넣기도 `Product`안에 넣기도 애매하다 그래서 `Domain Service`를 만들어 관리하게 된다.

<h2> 2. Domain Service 예시 </h2>

![참고 이미지](/assets/img/ddd/order-service-domain-logic.png)

주문을 생성하는 과정에서는 `Restaurant` 정보를 확인해야 한다.
하지만 `Restaurant`는 `Order`와 다른 `Aggregate`에 속하며,
주문 생성 과정에서 두 `Aggregate`의 정보를 함께 사용해야 하는 상황이 발생한다.

이 로직을 Order 내부에 넣게 되면
`Order`가 `Restaurant`의 상태와 정책까지 알게 되어
`Aggregate` 경계가 흐려질 수 있다.

이처럼 하나의 `Aggregate` 책임으로 보기 어려운 도메인 규칙은
`Domain Service`에서 처리하는 것이 자연스럽다.


```java
package com.food.ordering.system.order.service.domain;

import com.food.ordering.system.order.service.domain.entity.Order;
import com.food.ordering.system.order.service.domain.entity.Product;
import com.food.ordering.system.order.service.domain.entity.Restaurant;
import com.food.ordering.system.order.service.domain.event.OrderCancelledEvent;
import com.food.ordering.system.order.service.domain.event.OrderCreatedEvent;
import com.food.ordering.system.order.service.domain.event.OrderPaidEvent;
import com.food.ordering.system.order.service.domain.exception.OrderDomainException;
import lombok.extern.slf4j.Slf4j;

import java.time.ZoneId;
import java.time.ZonedDateTime;
import java.util.List;

@Slf4j
public class OrderDomainServiceImpl implements OrderDomainService {

    private static final String UTC = "UTC";

    @Override
    public OrderCreatedEvent validateAndInitiateOrder(Order order, Restaurant restaurant) {
        validateRestaurant(restaurant);
        setOrderProductInformation(order, restaurant);
        order.validateOrder();
        order.initializeOrder();
        log.info("Order with id: {} is initiated", order.getId().getValue());
        return new OrderCreatedEvent(order, ZonedDateTime.now(ZoneId.of(UTC)));
    }

    private void validateRestaurant(Restaurant restaurant) {
        if (!restaurant.isActive()) {
            throw new OrderDomainException(
                    "Restaurant with id " +
                            restaurant.getId().getValue() +
                            " is currently not active!"
            );
        }
    }

    //  todo : n^2 -> n
    private void setOrderProductInformation(Order order, Restaurant restaurant) {
        order.getItems().forEach(orderItem ->

                restaurant.getProducts().forEach(restaurantProduct ->
                        {
                            Product currentProduct = orderItem.getProduct();
                            if (currentProduct.equals(restaurantProduct)) {
                                currentProduct.updateWithConfirmedNameAndPrice(restaurantProduct.getName(), restaurantProduct.getPrice());
                            }
                        }
                )

        );
    }

}

```

<h2> 3. Application Service 란? </h2>

`Application Service`는 사용자의 요청을 받아서 도메인 객체를 사용하는 역할을 한다.
도메인 로직을 `직접 처리하는 것이 아니라`

- 트랜잭션 관리
- 흐름 제어
- `Repository` 호출
- `Domain` 호출

을 담당한다. 즉,

```
유스 케이스를 실행하는 역할
```

이다.

`Application Service`는

- `Controller`와 `Domain` 사이
- `UseCase` 실행 담당

이라고 보면 된다

<h2> 4. Application Service 예시 </h2>


- OrderApplicationService


```java
package com.food.ordering.system.order.service.domain;

import com.food.ordering.system.order.service.domain.dto.create.CreateOrderCommand;
import com.food.ordering.system.order.service.domain.dto.create.CreateOrderResponse;
import com.food.ordering.system.order.service.domain.dto.track.TrackOrderQuery;
import com.food.ordering.system.order.service.domain.dto.track.TrackOrderResponse;
import com.food.ordering.system.order.service.domain.ports.input.service.OrderApplicationService;
import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.validation.annotation.Validated;

@Slf4j
@Service
@Validated
@AllArgsConstructor
class OrderApplicationServiceImpl implements OrderApplicationService {

    private final OrderCreateCommandHandler createCommandHandler;
    private final OrderTrackCommandHandler orderTrackCommandHandler;

    @Override
    public CreateOrderResponse createOrder(CreateOrderCommand createOrderCommand) {
        return createCommandHandler.createOrder(createOrderCommand);
    }

}
```


- OrderCreateCommandHandler

```java
package com.food.ordering.system.order.service.domain;

import com.food.ordering.system.order.service.domain.dto.create.CreateOrderCommand;
import com.food.ordering.system.order.service.domain.dto.create.CreateOrderResponse;
import com.food.ordering.system.order.service.domain.entity.Customer;
import com.food.ordering.system.order.service.domain.entity.Order;
import com.food.ordering.system.order.service.domain.entity.Restaurant;
import com.food.ordering.system.order.service.domain.event.OrderCreatedEvent;
import com.food.ordering.system.order.service.domain.exception.OrderDomainException;
import com.food.ordering.system.order.service.domain.mapper.OrderDataMapper;
import com.food.ordering.system.order.service.domain.ports.output.repository.CustomerRepository;
import com.food.ordering.system.order.service.domain.ports.output.repository.OrderRepository;
import com.food.ordering.system.order.service.domain.ports.output.repository.RestaurantRepository;
import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import javax.validation.constraints.NotNull;
import java.util.Objects;
import java.util.Optional;
import java.util.UUID;

@Slf4j
@Component
@AllArgsConstructor
public class OrderCreateCommandHandler {

    private final OrderDomainService orderDomainService;
    private final OrderRepository orderRepository;
    private final CustomerRepository customerRepository;
    private final RestaurantRepository restaurantRepository;
    private final OrderDataMapper orderDataMapper;
    private final ApplicationDomainEventsPublisher applicationDomainEventsPublisher;

    @Transactional
    public CreateOrderResponse createOrder(CreateOrderCommand createOrderCommand) {

        checkCustomer(createOrderCommand.getCustomerId());
        Restaurant restaurant = checkRestaurant(createOrderCommand);
        Order order = orderDataMapper.createOrderCommandToOrder(createOrderCommand);
        OrderCreatedEvent orderCreatedEvent = orderDomainService.validateAndInitiateOrder(order, restaurant);
        Order orderResult = saveOrder(order);
        log.info("Order is created with id: {}", orderResult.getId().getValue());
        applicationDomainEventsPublisher.publish(orderCreatedEvent);
        return orderDataMapper.orderToCreateOrderResponse(orderResult, "Order Created success");

    }

    private void checkCustomer(@NotNull UUID customerId) {
        Optional<Customer> customer = customerRepository.findCustomer(customerId);
        if (customer.isEmpty()) {
            log.warn("Could not find customer with customer id: {}", customerId);
            throw new OrderDomainException("Could not find customer with customer id: " + customerId);
        }

    }

    private Restaurant checkRestaurant(CreateOrderCommand createOrderCommand) {
        Restaurant restaurant = orderDataMapper.createOrderCommandToRestaurant(createOrderCommand);
        Optional<Restaurant> optionalRestaurant = restaurantRepository.findRestaurantInformation(restaurant);
        if (optionalRestaurant.isEmpty()) {
            log.warn("Could not find restaurant with restaurant id: {}", createOrderCommand.getRestaurantId());
            throw new OrderDomainException("Could not find restaurant with restaurant id: " + createOrderCommand.getRestaurantId());
        }
        return optionalRestaurant.get();
    }

    private Order saveOrder(Order order) {
        Order orderResult = orderRepository.save(order);
        if (Objects.isNull(orderResult)) {
            log.error("could not save order!");
            throw new OrderDomainException("could not save order!");
        }
        log.info("Order is saved with id: {}", orderResult.getId().getValue());
        return orderResult;

    }

}

```


<h2> 5. 차이점 정리 </h2>

| 구분            | Domain Service      | Application Service |
| ------------- | ------------------- | ------------------- |
| 역할            | 도메인 규칙              | 유스케이스 실행            |
| 위치            | Domain Layer        | Application Layer   |
| 로직            | 비즈니스 규칙             | 흐름 제어               |
| Repository 사용 | 필요한 경우 가능 (도메인 추상화) | 일반적으로 사용            |
| Entity 사용     | 가능                  | 가능                  |
| 트랜잭션          | 직접 관리하지 않음          | 보통 트랜잭션 경계 담당       |


```
Application Service는 보통 트랜잭션 경계를 시작하는 위치이며,
Domain Service는 트랜잭션 내부에서 실행되는 도메인 로직을 담당한다.

도메인 규칙 -> Domain Service
처리 흐름 -> Application Service
```

<h2> 6. 정리 </h2>

`Domain Service`는 도메인 규칙을 담당하는 객체이다.
`Application Service`는 처리 흐름을 담당하는 객체이다.

`DDD`에서는 로직을 한곳에몰아넣지 않고 책임에 따라 나누는것이 중요하다.
이 구분이 명확해야

- 구조가 단순해지고
- 테스트가 쉬워지고
- 변경에 강해진다

좋은 설계는 `Domain Service`와 `Application Service`를 구분하는 것에서 나뉜다.

<h2>7. 다음에 알아볼 내용</h2>

`DDD`의 주요 개념을 정리했다면
이제 실제 구조에서 어떻게 적용되는지 보는 것이 중요하다.

`DDD`는 보통
- `Hexagonal Architecture`
- `Clean Architecture`

와 함께 사용된다.

다음으로는 `Hexagonal Architecture`를 예시로 들어 왜 같이 등장하게 되었는지를 정리하고 `DDD`에 대한 정리를 마치려고 한다.
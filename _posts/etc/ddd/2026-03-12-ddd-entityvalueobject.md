---
title: DDD - Entity와 Value Object의 차이
description: Bounded Context 내부에서 모델을 구성하는 Entity와 Value Object의 개념과 차이 정리
date: 2026-03-12T13:30:00+09:00
categories: [Architecture, DDD]
tags: [DDD, Domain Driven Design, Entity, ValueObject, Hexagonal Architecture, 설계, 아키텍처]
---

<h2> 0. 알아보게 된 이유 </h2>

이전 글에서 `Bounded Context`를 통해 도메인을 여러개의 경계로 나누는 이유에 대해 정리했다.

`Context`를 나누었다면 이제는 `Context`내부에서 어떤 기준으로 모델을 설계할 것인지가 중요해진다.

`DDD`에서는 도메인 모델을 구성할 떄 모든 객체를 동일하게 취급하지 않는다.
객체의 성격에 따라

- 식별자가 중요한 객체
- 값 자체가 중요한 객체

위와같이 나누어 설계해야 하며, 이를 위해 사용하는 개념이

- `Entity`
- `Value Object`

이다.

이번 글에서는 각각의 개념이 무엇인지, 그 차이점과 나누는 방법을 중심으로 정리해 보려고 한다.

<h2> 1. Entity 란? </h2>

`Entity는 식별자를 기준으로 구별되는 객체이다. 값이 변경되더라도 같은 식별자를 가지면 같은 객체로 본다. 즉,

```
 같은 ID를 가지면 같은 객체이다.
```

예시,

```java
class Member {
    MemberId id;
    String name;
    String email;
    Address address;
}
```

- 회원
  - 이름 변경 가능
  - 이메일 변경 가능
  - 주소 변경 가능

하지만 회원 ID가 같으면 같은 회원이다.
여기서 중요한것은 name, email, address가 아니라 `id`이다.

- `Entity`의 특징
  - 고유 식별자가 존재한다
  - 상태가 변경된다
  - 생명주기가 존재한다
  - 동일성은 ID로 판단한다

<h2> 2. Value Object 란? </h2>

`Value Object`는 값 자체로 의미를 가지는 객체이다.
식별자가 없고 값이 같으면 같은 객체로 본다. 즉,

```
값이 같으면 같은 객체이다
```

예시
- 주소
  - 서울 강남구
  - 서울 강남구

위와 같이 값이 같으면 같은 값이다.


```java
class Address {
    String city;
    String street;
}
```

`Value Object`는 `Entity`와 다르게 식별자가 없다.

- `Value Object`의 특징
  - 식별자가 없다
  - 값으로 비교한다
  - 불변 객체로 만드는 것이 일반적이다

<h2> 3. 왜 나누어야 하는가 </h2>

모든 객체를 `Entity`로 만들때 문제점

- ID가 불필요 하게 많아진다
- 비교가 어려워진다
- 객체 관리가 복잡해진다

모든 객체를 `Value Object`로 만들때 문제점

- 변경 추적이 어렵다
- 동일성 판단이 불가능하다
- 생명주기 관리가 안된다

그래서 `DDD`에서는

```
식별이 필요하면 Entity
값 자체가 의미를 가지면 Value Object
```

로 나눈다.

`Entity`와 `Value Object`의 차이

| 구분   | Entity        | Value Object   |
| ---- | ------------- | -------------- |
| 기준   | 식별자           | 값              |
| 동일성  | ID            | 값              |
| 변경   | 가능            | 불변 권장          |
| 생명주기 | 있음            | 없음             |
| 예시   | Member, Order | Address, Money |

이런 기준으로 나누면 설계가 단순해진다.

<h2> 4. 예시 코드 </h2>

음식 주문 시스템 프로젝트에서 주문 도메인 코어에 구현된 예시를 통해
`Entity`와 `Value Object`의 차이를 살펴본다.

개념 이해에 집중하기 위해,
`Entity`와 `Value Object`를 구분하는 데 필요한 필드만 남기고
설명에 불필요한 필드와 메서드는 제외하였다.

---

<h3> 4.1 Entity </h3>

- `Order`

```java
public class Order extends AggregateRoot<OrderId> {
    private final CustomerId customerId;
    private final RestaurantId restaurantId;
    private final StreetAddress deliveryAddress;
    private final Money price;
    private final List<OrderItem> items;
}
```

- `OrderItem`

```java
public class OrderItem extends BaseEntity<OrderItemId> {
    private OrderId orderId;
    private final Product product;
    private final Integer quantity;
    private final Money price;
    private final Money subTotal;
}
```

- `Product`

```java
public class Product extends BaseEntity<ProductId> {

    private String name;
    private Money price;
}
```


- `Restaurant`

```java
public class Restaurant extends AggregateRoot<RestaurantId> {
    private final List<Product> products;
    private boolean active;
}
```

---

<h3> 4.2 Value Object </h3>

- `BaseId<T>`

```java
public abstract class BaseId<T> {

    private T value;

    protected BaseId(T value) {
        this.value = value;
    }

    public T getValue() {
        return value;
    }

    @Override
    public boolean equals(Object o) {
        if (o == null || getClass() != o.getClass()) return false;
        BaseId<?> baseId = (BaseId<?>) o;
        return Objects.equals(value, baseId.value);
    }

    @Override
    public int hashCode() {
        return Objects.hashCode(value);
    }

}
```

- `OrderItemId`

```java
public class OrderItemId extends BaseId<Long> {
    public OrderItemId(Long value) {
        super(value);
    }
}

```

- `StreetAddress`

```java
public class StreetAddress {

    private final UUID id;
    private final String street;
    private final String postalCode;
    private final String city;

    public StreetAddress(UUID id, String street, String postalCode, String city) {
        this.id = id;
        this.street = street;
        this.postalCode = postalCode;
        this.city = city;
    }

    public UUID getId() {
        return id;
    }

    public String getStreet() {
        return street;
    }

    public String getPostalCode() {
        return postalCode;
    }

    public String getCity() {
        return city;
    }

    @Override
    public boolean equals(Object o) {
        if (o == null || getClass() != o.getClass()) return false;
        StreetAddress that = (StreetAddress) o;
        return Objects.equals(id, that.id) && Objects.equals(street, that.street) && Objects.equals(postalCode, that.postalCode) && Objects.equals(city, that.city);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, street, postalCode, city);
    }

}
```


<h2> 5. 정리 </h2>

`Entity`는 식별자를 기준으로 존재하는 객체이고
`Value Object`는 값 자체로 의미를 가지는 객체이다.

`DDD`에서는 모든 객체를 동일하게 만들지 않고 도메인의 의미에 따라 역할을 나누어 설계한다.

이 구분을 정확하게 하지 않으면

- 모델이 복잡해지고
- 변경이 어려워지고
- 코드가 도메인을 제대로 표현하지 못한다.

좋은 도메인 모델을 만들기 위해서는

```
이 객체는 식별이 중요한가?
값 자체가 중요한가?
```

를 먼저 판단하고 설계를 해야한다.

<h2>8. 다음에 알아볼 내용</h2>

`Entity`와 `Value Object`를 나누었다면
그 다음으로 중요한 개념은 `Aggregate`이다.

`DDD`에서는 객체를 개별로 관리하지 않고
일관성을 유지해야 하는 단위로 묶어서 관리하는데
이 개념을 `Aggregate`라고 한다.

다음 글에서는

- `Aggregate`란 무엇인가
- 왜 필요한가
- `AggregateRoot`가 필요한 이유
- 트랜잭션 경계와 관계

를 정리해보려고 한다.


[참고 저장소 링크](https://github.com/AngryPig123/food-ordering-system/tree/master/order-service/order-domain/order-domain-core/src/main/java/com/food/ordering/system/order/service/domain)
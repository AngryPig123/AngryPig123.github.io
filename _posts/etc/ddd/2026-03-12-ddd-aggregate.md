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

![이미지](https://raw.githubusercontent.com/AngryPig123/blog_img/main/order-service-domain-logic.png)
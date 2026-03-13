---
title: DDD와 Hexagonal Architecture는 왜 같이 사용될까?
description: DDD와 Hexagonal Architecture가 함께 등장하는 이유와 역할 관계 정리
date: 2026-03-13T09:30:00+09:00
categories: [Architecture, DDD]
tags: [DDD, Hexagonal Architecture, Ports and Adapters, Clean Architecture, 설계, 아키텍처]
---

<h2> 0. 알아보게 된 이유 </h2>

대부분의 자료에서 `DDD`를 설명할 때 함께 등장하는 구조가 있는데

- `Hexagonal Architecture`
- `Clean Architecture`
- `Layered Architecture`

같은 아키텍처 스타일이 있다.
특히 ``Hexagonal Architecture`는 `DDD`와 같이 설명되는 경우가 많았고, 도메인을 중심으로 설계한다는 점에서 `DDD`와 매우 잘 맞는 구조라고 한다.

이번 글 에서는

- `Hexagonal Architecture`란 무엇인가
- 왜 `DDD`와 같이 사용되는가
- 어떤 문제를 해결하기 위한 구조인가

를 중심으로 정리해 보려고 한다.

<h2> 1. DDD만으로는 구조가 완성되지 않는다. </h2>

`DDD`자체는 도메인을 어떻게 설계할것인지 방법에 대한 제시를 할뿐 구조적인 문제를 어떻게 해결할 것인지에 대해서는 설명하지 않는다. 예를 들어

- `Entity`를 어떻게 나눌지
- `Aggregate`를 어떻게 묶을지
- `Domain Service`를 언제 만들지

와 같은 기준을 제공하고

- `Controller`를 어디에 둘지
- `Repository`를 어디에 둘지
- `DB`접근을 어떻게 할지
- 외부`API`를 어떻게 연결할지

와 같은 구조적인 문제는 해결되지 않는다. 즉,

```
DDD = 모델링 기법
Architecture = 구조 설계 방법
```

이다.

그래서 `DDD`를 사용할 때는 도메인을 중심으로 구조를 잡을 수 있는 아키텍쳐가 필요하다.
그때 많이 사용하는 것이

```
Hexagonal Architecture
```

이다.

<h2> 2. Hexagonal Architecture 란? </h2>

이전에 설명한 글이 있음으로 링크로 대체한다.

링크 : [Hexagonal Architecture 란?](https://angrypig123.github.io/posts/hexagonal-architecture/)

<h2> 3. 왜 DDD와 잘 맞는가 </h2>

`DDD`목표

```
도메인을 중심으로 설계
```

`Hexagonal Architecture` 목표

```
도메인을 외부로 부터 보호
```

위와 같이 둘다 `Domain`에 중점을 둔 설계와 구조이다.

<h2> 4. 예시 구조 </h2>

`MSA + DDD + Hexagonal Architecture` 기반으로 구성된 강의 프로젝트의 실제 구조를 참고하여 정리하였다.  
해당 구조는 `Domain / Application / Adapter / Container` 를 명확하게 분리한 형태이며,  
`Ports & Adapters` 패턴을 기준으로 의존성 방향이 `Domain` 내부를 향하도록 설계되어 있다.

아래는 Order 서비스의 모듈 구조 예시이다.

```text
├─order-application
│  └─src
│      └─main
│          └─java
│              └─com
│                  └─food
│                      └─ordering
│                          └─system
│                              └─order
│                                  └─service
│                                      └─application
│                                          ├─exception
│                                          │  └─handler
│                                          └─rest
├─order-container
│  └─src
│      ├─main
│      │  ├─java
│      │  │  └─com
│      │  │      └─food
│      │  │          └─ordering
│      │  │              └─system
│      │  │                  └─order
│      │  │                      └─service
│      │  │                          └─domain
│      │  └─resources
│      └─test
│          ├─java
│          │  └─com
│          │      └─food
│          │          └─ordering
│          │              └─system
│          │                  └─order
│          │                      └─service
│          │                          └─domain
│          └─resources
│              └─sql
├─order-dataaccess
│  └─src
│      └─main
│          └─java
│              └─com
│                  └─food
│                      └─ordering
│                          └─system
│                              └─order
│                                  └─service
│                                      └─dataaccess
│                                          ├─customer
│                                          │  ├─adapter
│                                          │  ├─entity
│                                          │  ├─mapper
│                                          │  └─repository
│                                          ├─order
│                                          │  ├─adapter
│                                          │  ├─entity
│                                          │  ├─mapper
│                                          │  └─repository
│                                          ├─outbox
│                                          │  ├─payment
│                                          │  │  ├─adapter
│                                          │  │  ├─entity
│                                          │  │  ├─exception
│                                          │  │  ├─mapper
│                                          │  │  └─repository
│                                          │  └─restaurantapproval
│                                          │      ├─adapter
│                                          │      ├─entity
│                                          │      ├─exception
│                                          │      ├─mapper
│                                          │      └─repository
│                                          └─restaurant
│                                              ├─adapter
│                                              └─mapper
├─order-domain
│  ├─order-application-service
│  │  ├─src
│  │  │  ├─main
│  │  │  │  └─java
│  │  │  │      └─com
│  │  │  │          └─food
│  │  │  │              └─ordering
│  │  │  │                  └─system
│  │  │  │                      └─order
│  │  │  │                          └─service
│  │  │  │                              └─domain
│  │  │  │                                  ├─config
│  │  │  │                                  ├─dto
│  │  │  │                                  │  ├─create
│  │  │  │                                  │  ├─message
│  │  │  │                                  │  └─track
│  │  │  │                                  ├─mapper
│  │  │  │                                  ├─outbox
│  │  │  │                                  │  ├─model
│  │  │  │                                  │  │  ├─approval
│  │  │  │                                  │  │  └─payment
│  │  │  │                                  │  └─scheduler
│  │  │  │                                  │      ├─approval
│  │  │  │                                  │      └─payment
│  │  │  │                                  └─ports
│  │  │  │                                      ├─input
│  │  │  │                                      │  ├─message
│  │  │  │                                      │  │  └─listener
│  │  │  │                                      │  │      ├─customer
│  │  │  │                                      │  │      ├─payment
│  │  │  │                                      │  │      └─restaurantapproval
│  │  │  │                                      │  └─service
│  │  │  │                                      └─output
│  │  │  │                                          ├─message
│  │  │  │                                          │  └─publisher
│  │  │  │                                          │      ├─payment
│  │  │  │                                          │      └─restaurantapproval
│  │  │  │                                          └─repository
```

<h2> 5. 정리 </h2>

`DDD`는 도메인을 어떻게 설계할지 알려준다. `Hexagonal Architecture`는 도메인을 중심으로 구조를 만드는 방법을 알려준다.

`DDD`만 사용하면 구조가 흔들릴 수 있고 `Hexagonal Architecture`만 사용한다면 도메인이 약해질 수 있다. 그래서 실무에서는 둘을 같이 사용하는 경우가 많다. 

결과적으로 좋은 설계의 기준은

```
도메인은 외부에 의존하지 않는다.
```

이다.


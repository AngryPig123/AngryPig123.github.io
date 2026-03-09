---
title: Hexagonal Architecture란?
description: 헥사고날 아키텍처의 개념, 목적, 구성 요소와 레이어드 아키텍처와의 차이를 정리한다.
date: 2026-03-09T10:00:00+09:00
categories: [Architecture]
tags: [Architecture, Hexagonal Architecture, Ports and Adapters, DDD]
---

<h2>0. 학습 이유</h2>

2024년쯤 회사 스터디를 통해 DDD와 Hexagonal Architecture를 잠깐 공부한 적이 있었다.  
하지만 당시에는 개념을 깊게 이해하지 못한 채 넘어가게 되어 아쉬움이 남아 있었다.

이번에 재직 중 국비 교육으로 Multi AI Agent 프로젝트 과정을 수강하게 되었고,  
프로젝트를 진행하면서 Hexagonal Architecture 구조를 사용하게 되었다.

예전에 공부했던 내용을 다시 정리하고,  
이번에는 실제 프로젝트에 적용하면서 구조를 제대로 이해해보려고 한다.

이 글은 Hexagonal Architecture를 학습하면서 정리한 내용이다.

---

<h2>1. Hexagonal Architecture란?</h2>

Hexagonal Architecture는 애플리케이션의 핵심 로직을 외부 의존성으로부터 분리하기 위한 아키텍처이다.

Ports and Adapters 아키텍처라고도 불리며,
도메인 중심 설계를 가능하게 한다. 아래 이미지는 개인적으로 Hexagonal Architecture를 가장 쉽게 설명해주는 이미지로 생각되어 첨부.

![Hexagonal Architecture 구조](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdna%2FJVxxG%2FbtsJDffVkAI%2FAAAAAAAAAAAAAAAAAAAAAFYOpWzt_5RRtPxg-VLNLKJeHMS3hbl9HtxOf43zCOQm%2Fimg.png%3Fcredential%3DyqXZFxpELC7KVnFOS48ylbz2pIh7yKj8%26expires%3D1774969199%26allow_ip%3D%26allow_referer%3D%26signature%3DP9CUr0f7fWHtlmgwaUcz%252BjY3L5I%253D)

---

<h2>2. 왜 Hexagonal Architecture가 필요한가</h2>

기존 레이어드 아키텍처에서는 다음과 같은 문제가 발생한다.

- `Domain`이 `Infrastructure`에 의존
- 테스트가 어려움
- DB 변경이 어려움
- 외부 API 변경 영향 큼

이를 해결하기 위해 등장한 구조가 `Hexagonal Architecture`이다.

---

<h2>3. 기존 Layered Architecture의 문제점</h2>

Controller → Service → Repository → DB

문제

- 의존성이 한 방향이지만
- 실제로는 아래쪽에 강하게 묶임
- Domain이 순수하지 않음

---

<h2>4. Hexagonal Architecture의 핵심 개념</h2>

핵심 원칙

- Domain은 외부를 모른다
- 외부가 Domain을 호출한다
- 의존성은 항상 안쪽으로 향한다

---

<h2>5. Ports and Adapters 구조</h2>

`Hexagonal Architecture`는

- `Port`
- `Adapter`

구조로 이루어진다.

`Port` = 인터페이스  
`Adapter` = 구현체

---

<h2>6. 구성 요소 설명</h2>

<h3>6.1 Domain</h3>

비즈니스 규칙

<h3>6.2 Application</h3>

유스케이스

<h3>6.3 Port</h3>

인터페이스

<h3>6.4 Adapter</h3>

외부 연결

<h3>6.5 Infrastructure</h3>

DB / API / 메시지 / 파일

---

<h2>7. 의존성 방향 (Dependency Rule)</h2>

항상

Adapter → Port → Application → Domain

외부 → 내부

---

<h2>8. 요청 흐름 예시</h2>

Controller → Adapter → Port → Application → Domain

---

<h2>9. 장점과 단점</h2>

장점

- 테스트 쉬움
- 유지보수 쉬움
- DB 변경 쉬움
- 외부 API 변경 대응 쉬움

단점

- 구조가 복잡
- 초기 비용 큼
- 작은 프로젝트엔 과함

---

<h2>10. Clean Architecture / DDD와의 관계</h2>

`Hexagonal` = 구조

`DDD` = 설계 방법

`Clean` = 구조 패턴

함께 사용 가능

---

<h2>11. 언제 사용하면 좋은가</h2>

- 도메인이 복잡할 때
- `MSA`
- `DDD`
- 장기 프로젝트
- 테스트 중요할 때

---

<h2>12. 정리</h2>

`Hexagonal Architecture`는

도메인을 중심에 두고
외부 의존성을 분리하는
애플리케이션 아키텍처이다.

---

<h2>13. 어려웠던점</h2>

이번에 `Hexagonal Architecture`를 다시 정리하면서 가장 어려웠던 점은
기존에 익숙해져 있던 `Layered Architecture`의 관점으로 구조를 이해하려 했다는 점이었다.

특히 의존성 방향이라는 개념이 직관적으로 잘 와닿지 않았고,  
아키텍처 그림을 보면서 이해하려다 보니  
호출 흐름과 의존 관계를 같은 것으로 생각하게 되어 혼란이 생겼다.

`Layered Architecture`에서는  
호출 방향과 의존 방향이 거의 동일하기 때문에 큰 문제를 느끼지 못했지만,  
`Hexagonal Architecture`에서는  
의존성은 항상 안쪽으로 향해야 한다는 규칙이 있고  
호출 흐름과 의존 방향이 반드시 같지 않을 수 있다는 점을 이해하는 데 시간이 걸렸다.

이번 정리를 통해 구조의 모양보다  
의존성이 어디를 향하는지가 더 중요한 기준이라는 것을 다시 느끼게 되었다.

또 하나 도움이 되었던 점은  
AI를 활용해서 간단한 예제 프로젝트를 직접 만들어 보면서 구조를 따라가 본 것이었다.

아키텍처 문서나 다이어그램만으로 이해하려고 할 때는  
호출 흐름과 의존 방향이 머릿속에서 잘 정리되지 않았는데,  
직접 코드를 작성하면서 `Domain`, `Application`, `Port`, `Adapter`를 나누어 보니  
의존성이 안쪽을 향해야 한다는 의미를 훨씬 명확하게 이해할 수 있었다.

특히 AI를 통해 기본 구조를 빠르게 만들어 보고,  
그 위에서 직접 수정하면서 동작을 확인해 보는 방식이  
`Hexagonal Architecture`를 이해하는 데 큰 도움이 되었다.(아직 완전히 이해한건 아니지만.)

---

<h2>14.  참고 자료 </h2>

- [Hexagonal Architecture, 진짜 하실 건가요?](https://tech.kakaopay.com/post/home-hexagonal-architecture/)
- [코드 사례로 보는 Domain-Driven 헥사고날 아키텍처](https://devblog.kakaostyle.com/ko/2025-03-21-1-domain-driven-hexagonal-architecture-by-example/)
- [Microservices : 클린 아키텍처, DDD, SAGA, Outbox & Kafka](https://www.udemy.com/share/10cvXd3@AOnrT4s_x28U8qZvtTVV5iegBuN6HgzzOb-_WHSPxvr2VyNSboLLvc_F3Ud9UBTgTw==/)
- [fastapi + hexagonal architecture 예제 프로젝트](https://github.com/AngryPig123/hexagonal_architecture_practice)
  - [Hexagonal Architecture 호출 흐름 예시](https://github.com/AngryPig123/hexagonal_architecture_practice/blob/master/mermaid-diagram.png)

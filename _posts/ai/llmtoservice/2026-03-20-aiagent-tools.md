---
title: Tool은 쉬웠지만 연결은 어려웠다 — AI Agent 구조 리팩토링 기록
description: 단일 Tool 구현에서 Tool 간 연결로 확장하며 겪은 문제와, Application Layer에서의 Tool 위치·Prompt 관리·Service 책임을 재정립한 과정
date: 2026-03-20T14:00:00+09:00
updated: 2026-03-20T14:00:00+09:00
categories: [AI, AI Agent, Architecture]
tags: [AI, AI Agent, Tool, LLM, Hexagonal Architecture, Orchestration, Prompt Engineering]
---

<h2> 1. Tools </h2>

`search_blog_post_tool` 하나만 만들 때는 크게 막히는 부분이 없었다.
단일 책임도 명확했고, 구현 속도도 빠르게 나왔다.

하지만 두 번째 `summarize_context_tool을` 추가하고, 서로 연결하려는 순간부터 상황이 달라졌다.

```
Tool 간 연결은 어떻게 하지?
이 방식이 맞는 구조일까?
```

이 질문이 자연스럽게 나오기 시작했다.

물론 책임이나 확장성을 크게 고려하지 않는다면
단순하게 이어붙이는 것도 가능했다.

하지만 앞으로 추가될 `Tool`들이 많다는 것을 생각하면

```
지금 구조를 정리하지 않으면
나중에 더 큰 비용으로 돌아올 것
```

이라는 판단이 들었다.

이 시점부터는 단순 구현이 아니라
구조를 의식한 설계 단계로 넘어가게 되었다.

<h2> 2. Tool의 위치에 대한 고민 ( adapter → application ) </h2>

헥사고날 구조를 적용하면서 개발을 진행하다 보니
클래스 하나를 만들 때도 어디에 위치해야 하는가를 계속 고민하게 되었다.

그럼에도 불구하고, 실제로 개발을 진행하다 보면
초기 판단이 틀리는 경우가 자주 발생했다.

이번에도 마찬가지였다.

---

왜 `inbound`로 잘못 생각했는가?

초기에는 `inbound adapter` 가 없는 상태에서
서비스를 직접 실행시키는 구조였다.

그래서 자연스럽게 흐름을 이렇게 인식했다.

> Tool → LLM

이 구조만 보면 `Tool`이 외부 요청을 받는 진입점처럼 보였고,
그 결과 `adapter.inbound`에 두는 판단을 하게 되었다.

---

다시 보니 무엇이 달랐는가?

Tool의 실제 역할을 다시 정의해보면 다음과 같다.

```
- 사용자 요청을 직접 받지 않는다
- 특정 기능(검색, 요약, 생성)을 수행한다
- 내부적으로 Service나 Port를 호출한다
```

즉 Tool은

```
외부 요청을 받는 계층이 아니라
애플리케이션 내부에서 사용하는 실행 단위
```

에 더 가깝다.

---

정리

이 기준으로 보면 `Tool`의 위치는 명확해진다.

```
adapter.inbound X
application.tool O
```

헥사고날 구조에 대한 이해가 조금 더 있었더라면
처음부터 올바르게 배치할 수 있었겠지만,

이 과정을 통해 오히려 기준이 더 명확해졌다


<h2> 3. Prompt 관리에 대한 혼란 </h2>

두 번째로 크게 막혔던 부분은
프롬프트를 어디에 둘 것인가였다.

---

기존 방식

초기 구조에서는 `Service`에서 `prompt builder`를 생성하고
LLM을 직접 호출하고 있었다.

> Service → prompt 생성 → LLM 호출

이 구조는 `Tool`이 하나일 때는 크게 어색하지 않았다.

---

문제가 드러난 시점

`summarize_context_tool`이 추가되면서
상황이 달라졌다.

```
요약용 프롬프트
답변 생성 프롬프트
```

이 둘은 완전히 다른 책임을 가지는데

이걸 `Service`에서 모두 관리하려고 하니
책임이 섞이기 시작했다

---

여기서 생긴 판단
`Service`는 `prompt`를 알면 안 된다

이 기준을 세우면서 구조를 다시 보게 되었다.

해결 방법

프롬프트의 소유권을 `Tool`로 이동시켰다.

요약 프롬프트 → `SummarizeContextTool`

답변 프롬프트 → `AnswerDraftTool`

그리고 별도의 공통 `prompt builder`를 두기보다는

각 `Tool` 내부에 `static method` 형태로 포함
시키는 방식으로 단순화했다.

---

결과

구조가 다음과 같이 정리되었다.

```
Service → Tool 실행
Tool → prompt 생성 + LLM 호출
```

이로써 `Service`는
`prompt` 생성
`LLM` 호출
두 가지 책임에서 완전히 분리되었다.


<h2> 4. 깨달은점  </h2>

이번 과정을 통해 가장 크게 느낌점은

<h3> 4.1 Tool 자체는 어렵지 않다 </h3>

각 `Tool`을 만드는 것 자체는 비교적 단순했다.

- 검색
- 요약
- 답변 생성

각각은 독립적으로 구현 가능하다.

<h3> 4.2 진짜 어려운건 연결이다. </h3>

문제는 `Tool`이 늘어나는 순간부터 시작된다.

```
- 입력/출력은 어떻게 맞출 것인가
- 어떤 순서로 실행할 것인가
- 실패하면 어떻게 처리할 것인가
```

즉,

> Tool 구현 <<<<<<<<<< Tool 간 연결과 흐름 설계


<h3> 4.3 이 과정에서 정리된 기준 </h3>

- `Tool`
  - 하나의 명확한 기능 수행
  - 프롬프트 포함
  - `LLM` 호출 포함

- `Service`
  - `Tool` 실행 흐름 담당
  - 비즈니스 시나리오 구성

- `Prompt`
  - `Tool` 내부에 위치
    - 필요시 템플릿으로 분리

<h3> 4.4 그리고 다음 단계 </h3>

현재 구조는 정해진 순서대로 실행하는 구조를 가진다.

> 정해진 순서로 Tool을 실행하는 구조

이 지점에서 자연스럽게 다음 고민이 생긴다.

> 어떤 Tool을 사용할지 동적으로 결정을 하려면 어떻게 해야하지?

이 고민은 결국

- `Tool Registry`
- `Tool Router`
- `Agent` 구조

로 이어지게 되며 위 항목들을 구현해 보면서 확장시킬 예정이다.


<h2> 5. 마무리 </h2>

`Tool`을 하나 만들 때는 보이지 않던 문제들이
`Tool`을 연결하는 순간 드러나기 시작했다.

그리고 이 과정에서 얻은 가장 중요한 인사이트는 이것이다.

```
문제는 Tool 자체가 아니라
Tool 사이의 관계였다
```

이 기준을 정리하면서
구조를 한 단계 더 발전시킬 수 있었고,

다음 단계인 `Agent` 구조로 자연스럽게 넘어갈 준비가 되었다.


- 리펙토링 전 : [https://github.com/AngryPig123/ai-agent/tree/tool/summary-answerdraft-before](https://github.com/AngryPig123/ai-agent/tree/tool/summary-answerdraft-before)

구조

```
\---app
    |   main.py
    |   test_main.http
    |   
    +---adapter
    |   |   
    |   +---inbound
    |   |   |   base_tool.py
    |   |   |   search_blog_tool.py
    |   |   |   
    |   |           
    |   +---outbound
    |   |   |   
    |   |   +---api
    |   |   |       
    |   |   +---embed
    |   |   |   |   ollama_embed_adapter.py
    |   |   |   |   
    |   |   |           
    |   |   +---llm
    |   |   |   |   ollama_llm_adapter.py
    |   |   |   |   
    |   |   |           
    |   |   +---rag
    |   |   |       
    |   |           
    |           
    +---application
    |   |   
    |   +---port
    |   |   |   
    |   |   +---inbound
    |   |   |   |   sync_blog_answer_usecase.py
    |   |   |   |   
    |   |   |           
    |   |   +---outbound
    |   |   |   |   blog_post_chunk_query_port.py
    |   |   |   |   blog_post_query_port.py
    |   |   |   |   embed_port.py
    |   |   |   |   llm_port.py
    |   |   |   |   
    |   |   |           
    |   |           
    |   +---service
    |   |   |   blog_answer_service.py
    |   |   |   
    |   |   +---prompt
    |   |   |   |   blog_answer_prompt_builder.py
    |   |   |   |   prompt_builder.py
    |   |   |   |   
    |   |   |           
    |   |           
    |   +---usecase
    |   |   |   
    |   |           
    |           
    +---domain
    |   |   
    |   +---model
    |   |   |   blog_post.py
    |   |   |   blog_post_chunk.py
    |   |   |   
    |   |           
    |           
    +---infrastructure
    |   |   db.py
    |   |   
    |   +---config
    |   |   |   settings.py
    |   |   |   
    |   |           
    |   +---persistence
    |   |   |   
    |   |   +---entity
    |   |   |   |   blog_post_chunk_entity.py
    |   |   |   |   blog_post_entity.py
    |   |   |   |   
    |   |   |           
    |   |   +---repository
    |   |   |   |   blog_post_chunk_repository.py
    |   |   |   |   blog_post_repository.py
```

- 리펙토링 후 : [https://github.com/AngryPig123/ai-agent/tree/tool/summary-answerdraft-after](https://github.com/AngryPig123/ai-agent/tree/tool/summary-answerdraft-after)

구조

```
\---app
    |   main.py
    |   test_main.http
    |   __init__.py
    |   
    +---adapter
    |   |   __init__.py
    |   |   
    |   +---inbound
    |   |   |   __init__.py
    |   |   |   
    |   |           
    |   +---outbound
    |   |   |   __init__.py
    |   |   |   
    |   |   +---api
    |   |   |       __init__.py
    |   |   |       
    |   |   +---embed
    |   |   |   |   ollama_embed_adapter.py
    |   |   |   |   __init__.py
    |   |   |   |   
    |   |   |           
    |   |   +---llm
    |   |   |   |   ollama_llm_adapter.py
    |   |   |   |   __init__.py
    |   |   |   |   
    |   |   |           
    |   |   +---rag
    |   |   |       __init__.py
    |   |   |       
    |   |           
    |           
    +---application
    |   |   __init__.py
    |   |   
    |   +---port
    |   |   |   __init__.py
    |   |   |   
    |   |   +---inbound
    |   |   |   |   sync_blog_answer_usecase.py
    |   |   |   |   __init__.py
    |   |   |   |   
    |   |   |           
    |   |   +---outbound
    |   |   |   |   blog_post_chunk_query_port.py
    |   |   |   |   blog_post_query_port.py
    |   |   |   |   embed_port.py
    |   |   |   |   llm_port.py
    |   |   |   |   __init__.py
    |   |   |   |   
    |   |   |           
    |   |           
    |   +---prompt
    |   |   |   __init__.py
    |   |   |   
    |   |           
    |   +---service
    |   |   |   blog_answer_service.py
    |   |   |   __init__.py
    |   |   |   
    |   |           
    |   +---tool
    |   |   |   answer_draft_tool.py
    |   |   |   base_tool.py
    |   |   |   search_blog_tool.py
    |   |   |   summarize_context_tool.py
    |   |   |   __init__.py
    |   |   |   
    |   |           
    |   +---usecase
    |   |   |   __init__.py
    |   |   |   
    |   |           
    |           
    +---domain
    |   |   __init__.py
    |   |   
    |   +---model
    |   |   |   blog_post.py
    |   |   |   blog_post_chunk.py
    |   |   |   __init__.py
    |   |   |   
    |   |           
    |           
    +---infrastructure
    |   |   db.py
    |   |   __init__.py
    |   |   
    |   +---config
    |   |   |   settings.py
    |   |   |   __init__.py
    |   |   |   
    |   |           
    |   +---persistence
    |   |   |   __init__.py
    |   |   |   
    |   |   +---entity
    |   |   |   |   blog_post_chunk_entity.py
    |   |   |   |   blog_post_entity.py
    |   |   |   |   __init__.py
    |   |   |   |   
    |   |   |           
    |   |   +---repository
    |   |   |   |   blog_post_chunk_repository.py
    |   |   |   |   blog_post_repository.py
    |   |   |   |   __init__.py
```
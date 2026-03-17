---
title: AI Agent Tool 구조를 실제로 구현하며 헷갈렸던 지점들
description: 블로그 질의응답 기능을 Tool, Service, Port, Adapter 구조로 분리하며 겪었던 고민과 정리
date: 2026-03-18T03:00:00+09:00
updated: 2026-03-18T03:00:00+09:00
categories: [AI Agent, Architecture, Tool]
tags: [AI Agent, Tool, Python, DDD, Hexagonal Architecture, Prompt Engineering]
---

<h2> 1. 정리하게 된 이유 </h2>

AI Agent의 Tool 구조는 글로 읽을 때와 실제 코드로 구현할 때의 간극이 생각보다 컸다.  
특히 `Tool`, `Service`, `UseCase`, `Port`, `Adapter`는 머릿속으로는 어느 정도 구분이 된다고 생각했지만, 실제 코드로 옮기기 시작하니 각 역할의 경계가 흐려졌다.  
무엇을 `Tool`로 보고, 무엇을 `Service`로 두어야 하는지, 또 어떤 코드를 어느 계층에 배치해야 하는지가 예상보다 훨씬 어려웠다.

이번 글에서는 기존 `BlogAnswerService`에 몰려 있던 블로그 글 검색과 답변 생성 기능을 분리하면서, 구조를 다시 정리하는 과정에서 어떤 점이 특히 헷갈렸는지 정리해보려고 한다.

<h2> 2. 기존 코드의 문제점 </h2>

기존 코드는 기능을 빠르게 구현하는 데는 문제가 없었지만, 구조적으로는 몇 가지 어색한 점이 있었다.  
크게 보면 다음 두 가지가 가장 큰 문제였다.

<h2> 2.1 구조 문제 </h2>

현재의 구조에서 가장 먼저 느낀 문제는 `infrastructure` 아래에 `adapter`가 들어가 있다는 점이었다.  
처음에는 블로그 글을 수집하고, 임베딩하고, 답변을 생성하는 기능 자체를 구현하는 데 집중하느라 이 구조의 어색함을 크게 의식하지 못했다.  
하지만 나중에 다시 보니 `adapter`와 `infrastructure`의 역할 구분이 애매했고, 결과적으로 패키지 구조가 전체적으로 읽기 어렵게 되어 있었다.

```text
├── infrastructure
│   ├── adapter
│   │   ├── api
│   │   ├── embed
│   │   ├── llm
│   │   ├── rag
│   │   └── tool
│   ├── config
│   │   └── settings.py
│   ├── db.py
│   └── persistence
│       ├── entity
│       └── repository
```

구조를 다시 생각해보니, `adapter`는 애플리케이션 경계에서 외부와 연결되는 역할이고, `infrastructure`는 설정이나 DB 연결처럼 기술적인 기반을 제공하는 역할에 더 가깝다.
즉 처음 구조는 동작은 했지만, 책임 기준으로 보면 다소 섞여 있었다.

<h2> 2.2 BlogAnswerService의 역할과 책임 문제 </h2>

두 번째 문제는 `BlogAnswerService` 하나에 너무 많은 책임이 몰려 있었다는 점이다.
이 서비스는 단순히 답변만 생성하는 것이 아니라, 질문 임베딩, 유사 문서 검색, 게시글 조회, 프롬프트 생성, LLM 호출까지 한 번에 처리하고 있었다.

```python
class BlogAnswerService(UserAnswerUseCase):

    def __init__(
            self,
            blog_post_chunk_query_port: BlogPostChunkQueryPort,
            blog_post_query_port: BlogPostQueryPort,
            llm: LLMPort,
            embed: EmbedPort
    ):
        self.blog_post_chunk_query_port = blog_post_chunk_query_port
        self.blog_post_query_port = blog_post_query_port
        self.llm = llm
        self.embed = embed

    def execute(self, text):
        query_vector = self.embed.embed(text=text)
        blog_chunk_list = self.blog_post_chunk_query_port.search_similar(query_vector=query_vector, limit=3)

        blog_ids = []
        for blog_chunk in blog_chunk_list:
            blog_ids.append(blog_chunk.blog_post_id)

        blog_posts = self.blog_post_query_port.find_by_blog_post_ids(blog_ids)

        result = []
        for post in blog_posts:
            a = {
                "title": post.title,
                "description": post.description,
                "source_path": post.source_path,
                "tags_json": post.tags_json
            }
            result.append(a)

        prompt = f"""
                """

        return self.llm.generate(prompt)
```

처음에는 기능이 동작하는 것이 중요했기 때문에 이런 구조로 빠르게 구현했지만, 나중에 보니 이 서비스는 조회와 생성이라는 서로 다른 책임을 동시에 갖고 있었다.
이 지점에서 “어디까지를 Tool로 보고, 어디까지를 Service의 책임으로 둘 것인가”라는 고민이 시작됐다.

즉, 기존 코드의 문제는 단순히 클래스가 길다는 수준이 아니라, 조회, 프롬프트 생성, `LLM` 호출이라는 서로 다른 성격의 작업이 하나의 서비스 안에 모여 있었다는 점이었다.
그래서 이후에는 이 기능들을 `Tool`과 `Service`로 나누고, 각자의 책임을 다시 정리하는 방향으로 구조를 바꾸게 되었다.

이번 글에서는 이런 문제를 어떤 기준으로 분리했는지, 그리고 그 과정에서 어떤 부분이 가장 헷갈렸는지를 이어서 정리해보려고 한다.

<h2> 3. Tool과 Service의 경계</h2>

기본 `BlogAnswerService`를 정리하면서 가장 먼저 부딪힌 문제는, 어디까지를 `Tool`로 보고 어디까지를 `Service`로 볼 것 인가 였다.
처음에는 블로그 글을 조회하고, 프롬프트를 만들고, LLM으로 답변을 생성하는 전체 흐름을 하나의 서비스에 넣고 있었기 때문에 막상 `Tool`을 따로 분리하려니 경계가 흐려지게 되었다.

이론적으로는 `Tool`은 기능 단위, `Service`는 흐름 단위로 구분하면 된다고 생각했다.
하지만 실제로 코드를 작성하기 시작하니, 그 경계를 어디에서 끊어야 하는지가 생각보다 훨씬 어려웠다.

예를 들어 블로그 게시글 조회는 입력을 받아 유사한 게시글 목록을 찾아오는 독립적인 기능으로 볼 수 있기 때문에 Tool에 적합했다.
반면 조회된 결과를 가지고 프롬프트를 만들고 `LLM`을 호출해 최종 답변을 하는 것은 `Service` 즉 `UseCase`에 가까웠다.

이 기준을 적용하면서 블로그 게시글 조회 책임은 `SearchBlogTool` 로 분리하고, 최종 답변 생성 흐름은 `BlogAnswerService`에 남겨 두는 방향으로 리팩터링했다.


<h2> 4.  패키지 구조를 다시 보게 된 이유 </h2>

`Tool`과 `Service`를 분리하는 과정에서 자연스럽게 패키지 구조도 다시 보게 되었다.  
처음에는 기능 구현에 집중하느라 `adapter`, `application`, `infrastructure`의 위치가 조금씩 섞여 있었는데, `Tool`을 어디에 두어야 하는지 고민하는 과정에서 이 구조의 어색함이 더 분명하게 보였다.

특히 `BaseTool` 같은 공통 `Tool` 실행 객체를 `port/in`에 둘지, `adapter/in`에 둘지, 혹은 별도 공통 모듈로 둘지를 계속 고민했다.
정리하고 보니 `UseCase`는 애플리케이션이 외부에 제공하는 계약이고, Tool은 외부 요청을 받아 서비스를 호출하는 진입점에 더 가까웠다.
그래서 현재는 `Tool`을 외부 요청을 받아 애플리케이션 서비스를 호출하는 진입점, 즉 `inbound adapter`에 가까운 성격으로 보고 구조를 다시 정리하는 쪽이 더 자연스럽다고 느꼈다.

이 과정은 단순히 폴더를 옮기는 문제가 아니라, 각 컴포넌트가 어떤 책임을 가지는지를 다시 확인하는 과정이기도 했다.

<h2> 5. 마무리하며 </h2>

이번 리팩터링에서 가장 어려웠던 점은 처음부터 완벽한 구조를 한 번에 만드는 일이었다.  
글로 읽을 때는 `Tool`, `Service`, `UseCase`, `Port`, `Adapter`가 어느 정도 구분되는 것처럼 보였지만, 실제로 코드를 옮기기 시작하면 그 경계는 생각보다 쉽게 흐려졌다.  
특히 기능이 하나의 서비스 안에 몰려 있는 상태에서는 어디서부터 분리해야 하는지 판단하는 것 자체가 쉽지 않았다.

그래도 이번에 구조를 다시 나누면서 하나의 기준은 분명하게 잡을 수 있었다.  
블로그 게시글을 조회하는 일은 입력을 받아 관련 데이터를 찾아오는 독립적인 기능이기 때문에 `Tool`로 분리하는 것이 자연스러웠다.  
반면 조회된 결과를 바탕으로 프롬프트를 만들고, `LLM`을 호출해 최종 답변을 생성하는 일은 하나의 흐름을 조합하는 책임이기 때문에 `Service`에 두는 편이 더 적절했다.  
즉 이번 정리에서 핵심 기준은 조회는 `Tool`, 답변 생성 흐름은 `Service`였다.

결국 이번에 느낀 것은 중요한 것이 처음부터 정답처럼 보이는 구조를 만드는 일이 아니라는 점이었다.  
오히려 각 컴포넌트가 왜 존재하는지, 어떤 책임을 가지고 있는지, 그리고 왜 이 위치에 있어야 하는지를 스스로 설명할 수 있는 상태를 만드는 것이 더 중요했다.  
지금 구조도 앞으로 다시 바뀔 수는 있겠지만, 최소한 현재의 나는 `SearchBlogTool`은 왜 `Tool`인지, `BlogAnswerService`는 왜 `Service`인지 이전보다 훨씬 더 분명하게 설명할 수 있게 되었다.  
이번 리팩터링은 구조를 완성한 작업이라기보다, 내가 이해할 수 있는 책임의 경계를 찾아가는 과정에 더 가까웠다.

- 리펙토링 전 : [https://github.com/AngryPig123/ai-agent/tree/tool/search_blog_tool_before](https://github.com/AngryPig123/ai-agent/tree/tool/search_blog_tool_before)

구조

```
└── app
    ├── __init__.py
    ├── application
    │   ├── __init__.py
    │   ├── port
    │   │   ├── __init__.py
    │   │   ├── in
    │   │   │   └── __init__.py
    │   │   └── out
    │   │       ├── __init__.py
    │   │       ├── blog_post_chunk_query_port.py
    │   │       ├── blog_post_query_port.py
    │   │       ├── embed_port.py
    │   │       └── llm_port.py
    │   ├── service
    │   │   ├── __init__.py
    │   │   └── blog_answer_service.py
    │   └── usecase
    │       ├── __init__.py
    │       └── sync_blog_answer_usecase.py
    ├── domain
    │   ├── __init__.py
    │   └── model
    │       ├── __init__.py
    │       ├── blog_post.py
    │       └── blog_post_chunk.py
    ├── infrastructure
    │   ├── __init__.py
    │   ├── adapter
    │   │   ├── __init__.py
    │   │   ├── api
    │   │   │   └── __init__.py
    │   │   ├── embed
    │   │   │   ├── __init__.py
    │   │   │   └── ollama_embed_adapter.py
    │   │   ├── llm
    │   │   │   ├── __init__.py
    │   │   │   └── ollama_llm_adapter.py
    │   │   ├── rag
    │   │   │   └── __init__.py
    │   │   └── tool
    │   │       └── __init__.py
    │   ├── config
    │   │   ├── __init__.py
    │   │   └── settings.py
    │   ├── db.py
    │   └── persistence
    │       ├── __init__.py
    │       ├── entity
    │       │   ├── __init__.py
    │       │   ├── blog_post_chunk_entity.py
    │       │   └── blog_post_entity.py
    │       └── repository
    │           ├── __init__.py
    │           ├── blog_post_chunk_repository.py
    │           └── blog_post_repository.py
    ├── main.py
    └── test_main.http
```

- 리펙토링 후 : [https://github.com/AngryPig123/ai-agent/tree/tool/search_blog_tool_after](https://github.com/AngryPig123/ai-agent/tree/tool/search_blog_tool_after)

구조

```
└── app
    ├── __init__.py
    ├── adapter
    │   ├── __init__.py
    │   ├── inbound
    │   │   ├── __init__.py
    │   │   ├── base_tool.py
    │   │   └── search_blog_tool.py
    │   └── outbound
    │       ├── __init__.py
    │       ├── api
    │       │   └── __init__.py
    │       ├── embed
    │       │   ├── __init__.py
    │       │   └── ollama_embed_adapter.py
    │       ├── llm
    │       │   ├── __init__.py
    │       │   └── ollama_llm_adapter.py
    │       └── rag
    │           └── __init__.py
    ├── application
    │   ├── __init__.py
    │   ├── port
    │   │   ├── __init__.py
    │   │   ├── inbound
    │   │   │   ├── __init__.py
    │   │   │   └── sync_blog_answer_usecase.py
    │   │   └── outbound
    │   │       ├── __init__.py
    │   │       ├── blog_post_chunk_query_port.py
    │   │       ├── blog_post_query_port.py
    │   │       ├── embed_port.py
    │   │       └── llm_port.py
    │   ├── service
    │   │   ├── __init__.py
    │   │   ├── blog_answer_service.py
    │   │   └── prompt
    │   │       ├── __init__.py
    │   │       ├── blog_answer_prompt_builder.py
    │   │       └── prompt_builder.py
    │   └── usecase
    │       └── __init__.py
    ├── domain
    │   ├── __init__.py
    │   └── model
    │       ├── __init__.py
    │       ├── blog_post.py
    │       └── blog_post_chunk.py
    ├── infrastructure
    │   ├── __init__.py
    │   ├── config
    │   │   ├── __init__.py
    │   │   └── settings.py
    │   ├── db.py
    │   └── persistence
    │       ├── __init__.py
    │       ├── entity
    │       │   ├── __init__.py
    │       │   ├── blog_post_chunk_entity.py
    │       │   └── blog_post_entity.py
    │       └── repository
    │           ├── __init__.py
    │           ├── blog_post_chunk_repository.py
    │           └── blog_post_repository.py
    ├── main.py
    └── test_main.http
```
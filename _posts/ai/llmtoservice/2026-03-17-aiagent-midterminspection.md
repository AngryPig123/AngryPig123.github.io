---
title: 블로그 QA AI Agent 자료 수집 및 전체 기능 구현 중간 점검
description: 블로그 QA AI Agent의 자료 수집 구조와 전체 기능 구현 현황을 점검하고, 현재까지의 설계·구현 내용과 이후 보완할 과제를 정리한다.
date: 2026-03-17T14:30:00+09:00
updated: 2026-03-17T14:30:00+09:00
categories: [AI, AI Agent]
tags: [AI, AI Agent, Spring Batch, QA 시스템, RAG, 자료 수집, 블로그, 중간 점검]
---

<h2> 0. 작성하게 된 이유 </h2>

이전에 블로그 `QA AI Agent`를 구현하는 과정에서 아키텍처와 `AI Agent` 구조를 학습하며,
자료 수집을 위한 배치 서비스를 만들고 기능을 하나씩 테스트해 나갔다.

이 과정에서 현재 어떤 방식으로 구현이 진행되었는지,
어디까지 기능이 완성되었는지,
그리고 앞으로 어떤 부분을 추가로 학습하고 구현해야 하는지에 대한 정리가 필요하다고 느꼈다.

그래서 지금까지의 구현 흐름과 구조를 중간 점검하는 기록을 남기려고 한다.


<h2> 1. 처음에 생각했던 개발 플로우 </h2>

초기에는 `LLM` 연결 이후 `Tool` 구조와 `RAG` 적용을 바로 진행할 예정이었다.

하지만 실제로 구현을 진행하는 과정에서  
`RAG`를 적용하려면 먼저 블로그 데이터를 수집하는 과정이 필요하다는 것을 알게 되었고,  
자료 수집을 구현하는 과정에서 추가로 필요한 기술들이 계속 생기기 시작했다.

또한 기능을 하나씩 구현해 나가다 보니  
처음에 생각했던 개발 순서와 실제 구현 흐름이 점점 달라졌고,  
현재까지 어디까지 구현되었는지, 앞으로 어떤 방향으로 진행해야 하는지  
한 번 전체 구조를 점검할 필요가 있다고 느꼈다.

그래서 지금 시점에서 개발 흐름을 다시 정리하고  
현재 상태를 기준으로 이후 구현 방향을 다시 잡아보려고 한다.


<h2> 2. 구현한 기능 </h2>

현재까지는 `inbound adapter` 부분을 제외한 상태에서,
사용자의 질문이 들어왔을 때 LLM의 기본 지식에 더해
질문과 유사한 블로그 글을 검색하여 답변 하단에 함께 제공하는 기능까지 구현하였다.

현재 동작 흐름은 다음과 같다.

- ai-agent-batch
  - 데이터 수집
  - 데이터 저장
  - 임베딩 생성

- ai-agent
  - 질의 임베딩 생성
  - 유사도 검색
  - 답변 프롬프트 생성
  - LLM 답변 생성
  - 관련 블로그 글 첨부

즉, 기본적인 `RAG` 기반 `QA` 흐름까지는 구현된 상태이다.

> 답변 서비스 코드

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
                너는 블로그 글을 기반으로 질문에 답변하는 AI 도우미다.
        
                아래 제공된 블로그 글 정보를 참고해서 질문에 답변해라.
                제공된 내용에 없는 정보는 추측하지 말고 모른다고 답해라.
        
                [참고 블로그 데이터 목록]
                {result}
        
                [사용자 질문]
                {text}
        
                [답변 규칙]
                - 참고 데이터(result)에 (tags_json)와 유사한 블로그글도 첨부한다.
                - 먼저 질문에 대한 답변을 작성한다.
                - 답변은 한국어로 작성한다.
                - 마지막에 참고한 블로그 글 목록을 아래 형식으로 추가한다.
                - 블로그 글 목록이 첨부된 이후에는 사용자가 다음으로 어떤 내용을 알아 보면 좋을지 첨언한다.
        
                [참고한 글]
                - 제목 (source_path)
                - 관련 태그 (tags_json)
        
                답변:
                """

        return self.llm.generate(prompt)
```

![ai-agent-noinbound-answer](/assets/img/aiagent/ai-agent-noinbound-answer.png)

<h3> 2.1 데이터 수집 및 검증 과정 </h3>

데이터 수집과 검증 과정은 별도의 서비스로 분리하여 구현하였다.

- ai-agent  
  - 블로그 글 : [https://angrypig123.github.io/posts/aiagent-aiagentservicesetting/](https://angrypig123.github.io/posts/aiagent-aiagentservicesetting/)
  - 저장소 : [https://github.com/AngryPig123/ai-agent](https://github.com/AngryPig123/ai-agent)

- ai-agent-batch  
  - 블로그 글 : [https://angrypig123.github.io/posts/aiagent-blogpostbatch/](https://angrypig123.github.io/posts/aiagent-blogpostbatch/)
  - 저장소 : [https://github.com/AngryPig123/ai-agent-batch](https://github.com/AngryPig123/ai-agent-batch)


<h2> 3. 다음으로 개발해야 할 것 </h2>

`RAG`는 별도로 뒤늦게 추가한 기능이라기보다,
데이터 수집 및 검증 과정을 구현하는 과정에서 자연스럽게 함께 적용되었다.

그래서 앞으로는 처음에 생각했던 전체 개발 순서를 그대로 따라가기보다는,
현재까지 구현된 기능과 구조를 기준으로
다음 단계에서 필요한 항목들을 다시 정리해 확장해 나가려고 한다.

우선 다음 단계에서는 `Agent` 구조를 중심으로
역할과 책임을 나누는 작업을 진행할 예정이다.

앞으로 추가할 목차는 다음과 같다.

- `Agent` 구조 만들기
  - `Tool` 분리
  - `Prompt` 분리
  - `Agent` 상태 분리


<h2> 4. 프로젝트 방향 변경 </h2>

또한 프로젝트의 주제도 기존에 생각했던 블로그 `QA Agent`에서,
`LLM`의 답변에 내 블로그 글 내용을 참고하거나 함께 첨부해주는 형태로 방향을 변경하기로 했다.

처음에는 블로그 글을 기반으로 `QA` 전용 `Agent`를 만드는 것을 목표로 했지만,
현재 작성된 블로그 글의 양이 많지 않아 `QA` 시스템을 구성하기에는 데이터가 부족했고,
구조를 확장하는 데에도 제약이 있다고 느꼈다.

그래서 특정 질문에 대해 정답을 찾는 `QA` 형태보다는,
사용자의 질문과 유사한 내용을 가진 블로그 글을 검색한 뒤,
`LLM`의 답변 생성 과정에서 해당 내용을 참고하거나 함께 첨부하는 방식으로 방향을 바꾸려고 한다.

이 방식이 현재 프로젝트 규모에서도 적용하기 쉽고,
`RAG`, `Agent`, `Workflow`, `Multi Agent` 구조로 확장하기에도 더 적합하다고 판단했다.

기존에 생각했던 방향과 달라지면서,
앞으로 구현해야 할 `Tool`의 구성도 다시 정리할 필요가 생겼다.

우선은 답변 생성의 기본 흐름을 구성하는 도구부터 구현하고,
이후에 전체 흐름을 제어하는 도구를 추가하는 순서로 진행하려고 한다.

구현 순서는 다음과 같이 정리했다.

- `search_blog_tool`
  - 사용자 질문과 유사한 블로그 글 또는 관련 문맥을 검색한다.
  - 답변에 활용할 근거 데이터를 찾는 가장 기본적인 단계이다.

- `summarize_context_tool`
  - 검색된 블로그 글이나 문맥을 답변에 활용하기 좋은 형태로 요약한다.
  - 검색 결과가 길거나 불필요한 내용이 많을 때 핵심만 추려내는 역할을 한다.

- `answer_draft_tool`
  - 사용자 질문과 검색된 문맥을 바탕으로 답변 초안을 생성한다.
  - 검색과 답변 생성을 분리해 두면 이후에 Writer, Reviewer 구조로 확장하기 쉽다.

- `attach_reference_tool`
  - 최종 답변 하단에 참고한 블로그 글 정보나 링크를 첨부한다.
  - 답변의 근거를 함께 보여주기 위한 도구이다.

- `planner_tool`
  - 질문이 들어왔을 때 어떤 흐름으로 처리할지 결정한다.
  - 예를 들어 블로그 검색이 필요한지, 요약이 필요한지, 바로 답변을 생성해도 되는지를 판단하는 역할을 맡는다.

현재 기준으로는 먼저 검색, 요약, 초안 생성, 참고자료 첨부까지의 기본 흐름을 만든 뒤,
그 이후에 `planner_tool`을 추가하여
각 단계를 상황에 따라 선택적으로 호출할 수 있도록 확장해 나갈 계획이다.

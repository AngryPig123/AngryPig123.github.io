---
title: Tool Registry / Router / Orchestrator로 상태 기반 Tool 실행 구조 만들기
description: Tool 실행 순서를 하드코딩하지 않고 state 기반으로 동적으로 결정하도록 리팩터링한 과정 정리
date: 2026-03-23T15:00:00+09:00
updated: 2026-03-23T15:00:00+09:00
categories: [AI, AI Agent, Architecture]
tags: [AI Agent, Tool, Registry, Router, Orchestrator, Python, Hexagonal]
---

<h2> 0. 이전 글에서 남았던 문제 </h2>

이전 글에서 `Service` 의 책임을 분리하고 `Tool`을 연결하는 구조까지 만들었다.
하지만 `Tool` 개수가 늘어나면서 문제가 생겼다.

- `Tool`의 실행 순서를 `Service`가 알고 있어야 한다.
- `Tool`이 추가 될 때 마다 `Service`코드를 변경 해야 한다.
- `Tool` 사이의 연결 규칙이 코드에 하드 코딩된다.

예를 들어 기존 구조는 아래와 같은 식이었다

> search -> summarize -> answer

위와 같이 서비스에서 직접 하드코딩된 순서로 제어되고 있다.
이 구조가 `Tool`의 개수가 적을 때는 괜찮겠지만 갯수가 늘어나면서부터 유지가 어려워진다.

그래서 이번에는 툴을 `Registry`, `Router`, `Orchestrator`를 이용해 제어하는 구조로 바꾸는 리펙토링을 진행하려 한다.


<h2> 1. 목표 </h2>

- `Tool` 실행 순서를 하드코딩 하지 않는다.
- `Tool`은 자신이 필요한 데이터만 선언한다.
- `Router`가 실행 순서를 결졍한다.
- `Orchestrator`가 실행 루프를 돌린다.

즉,

> Service -> Tool 호출

에서

> Service -> Orchestrator -> Router -> Tool

로 바꾸려는 것이다.

이번 리펙토링에서 하게될것은

- `Tool`메타 데이터 기반 실행
- `state` 기반 파이프 라인
- `Registry`, `Router`, `Orchestrator` 분리

이고 이건 결국 `Multi Agent`로 확장하기 위한 과정중 하나이다.

<h2> 2. BaseTool을 state 기반으로 변경 </h2>

기존 `Tool`은 입력 -> 실행 -> 결과 구조였다.
이번 변경에서는 `Tool`이 `State`를 기준으로 실행되도록 변경하였다.

추가된 핵심 속성 및 메소드

```
#   tool이 실행되기 위한 상태 키 목록
requires: tuple[str, ...] = ()

#   tool이 성공하면 state에 채워줄 키
provides: tuple[str, ...] = ()

#   tool 결과가 최종 산출물인지 여부
is_terminal: bool = False

#   현재 Tool이 실행 가능한 생태인지 판단
def can_handle(self, state: dict[str, Any]) -> bool:


#   공용 state에서 이 Tool이 실행에 필요한 입력만 추출
def build_input(self, state: dict[str, Any]) -> dict[str, Any]:
        
#   Tool 실행 결과를 공용 state에 반영
def update_state(self,state: dict[str, Any],result: ToolResult)
```

코드

```python
class BaseTool(ABC):
    #   registry에서 식별할 고유 이름
    name: str = ""

    #   사람이 읽는 설명
    description = ""

    #   tool이 실행되기 위한 상태 키 목록
    requires: tuple[str, ...] = ()

    #   tool이 성공하면 state에 채워줄 키
    provides: tuple[str, ...] = ()

    #   tool 결과가 최종 산출물인지 여부
    is_terminal: bool = False

    def execute(self, state: dict[str, Any], context: ToolContext) -> tuple[ToolResult, dict[str, Any]]:
        if not self.can_handle(state):
            return ToolResult(success=False, error="cannot handle"), state

        input_data = self.build_input(state)
        error = self.validate(input_data)

        if error:
            return ToolResult(success=False, error=error), state

        result = self.run(input_data, context)
        if not result.success:
            return result, state

        new_state = self.update_state(state, result)
        return result, new_state

    def validate(self, input_data: dict[str, Any]) -> str | None:
        pass

    def run(self, input_data: dict[str, Any], context: ToolContext) -> ToolResult:
        pass

    #   현재 Tool이 실행 가능한 생태인지 판단
    def can_handle(self, state: dict[str, Any]) -> bool:
        return all(
            key in state and state[key] is not None
            for key in self.requires
        )

    #   공용 state에서 이 Tool이 실행에 필요한 입력만 추출
    def build_input(self, state: dict[str, Any]) -> dict[str, Any]:
        return {
            key: state[key]
            for key in self.requires
            if key in state
        }

    #   Tool 실행 결과를 공용 state에 반영
    def update_state(
            self,
            state: dict[str, Any],
            result: ToolResult
    ):
        new_state = dict(state)
        for key in self.provides:
            if key in result.data:
                new_state[key] = result.data[key]
        return new_state
```

위 구조로 바뀌면서

- `Tool`이 무엇을 필요로 하는지
- `Tool`이 무엇을 만드는지

를 코드가 아니라 메타 데이터로 알 수 있게 되었다.


<h2> 3. Tool Registry 추가 </h2>

`Tool`을 직접 생성해서 쓰는 대신 `Registry`에 등록 하도록 변경하였다.

```python
class ToolRegistry:

    def __init__(self) -> None:
        self._tools: dict[str, BaseTool] = {}

    def register(self, tool: BaseTool) -> None:
        if not tool.name:
            raise ValueError("tool name은 비어 있을 수 없습니다.")
        if tool.name in self._tools:
            raise ValueError(f"이미 등록된 tool입니다: {tool.name}")
        self._tools[tool.name] = tool

    def get(self, name: str) -> BaseTool:
        return self._tools[name]

    def list_tools(self) -> list[BaseTool]:
        return list(self._tools.values())
```

위 구조를 추가 하면서

- `Tool` 추가시 `Service` 수정 없음.
- `Tool` 목록을 동적으로 관리 가능
- `Router`가 `Tool` 목록 조회 가능

이 단계에서 `Tool` 실행 구조를 `Service`코드에서 분리할 준비가 됐다.

<h2> 4. Router 도입 - 어떤 Tool을 실행할지 결정 </h2>

`Router`의 역할

> 다음에 실행할 Tool을 선택

지금 프로젝트에서는 `LLM Router` 까지는 과하다 판단하여 `Rule` 기반으로 작성후 추후에 변경하는 과정을 거칠것이다.

코드

```python
class RuleBasedToolRouter(BaseToolRouter):

    def __init__(self, tool_registry: ToolRegistry):
        self.tool_registry = tool_registry

    def next_tool(self, state: dict[str, Any], executed_tools: set[str]) -> str | None:
        candidates = [
            tool
            for tool in self.tool_registry.list_tools()
            if tool.name not in executed_tools and tool.can_handle(state)
        ]

        if not candidates:
            return None

        for tool in candidates:
            if _provides_missing_state(tool, state):
                return tool.name

        terminal_candidates = [tool for tool in candidates if tool.is_terminal]
        if terminal_candidates:
            return terminal_candidates[0].name

        return candidates[0].name
```

`Tool` 선택 기준

- 아직 없는 `state`를 채울 `Tool`
- `terminal Tool`
- 실행 가능한 `Tool`

해당 로직 덕분에 `Tool`의 순서를 코드가 아니라 `state`로 결정할 수 있게 되었다.

<h2> 5. Orchestrator 도입 - 실행 루프 통합 </h2>

`Service`가 `Tool`을 호출하지 않도록 `Orchestrator`를 만들었다.

코드

```python
class ToolFlowOrchestrator:

    def __init__(
            self,
            tool_router: BaseToolRouter,
            tool_registry: ToolRegistry
    ):
        self.tool_router = tool_router
        self.tool_registry = tool_registry

    def run(
            self,
            initial_state: dict[str, Any],
            agent_id: str,
            user_id: str,
            session_id: str,
    ) -> dict[str, Any]:

        state = dict(initial_state)
        import uuid
        context = ToolContext(
            agent_id=agent_id,
            user_id=user_id,
            session_id=session_id,
            trace_id=str(uuid.uuid4()),
        )

        executed_tools: set[str] = set()

        while True:
            tool_name = self.tool_router.next_tool(
                state=state,
                executed_tools=executed_tools,
            )
            if tool_name is None:
                break

            tool = self.tool_registry.get(tool_name)

            result, state = tool.execute(state=state, context=context)
            if not result.success:
                raise RuntimeError(result.error)

            executed_tools.add(tool.name)

            if tool.is_terminal:
                break

        return state
```

위 구조의 장점

- 실행 흐름이 한 곳에 모인다.
- `Service`가 단순해진다
- `Tool` 추가 시 변경이 없다.

<h2> 6. Tool을 state 계약에 맞게 수정 </h2>

각 `Tool`이 명확하게 선언되도록 수정했다.


> search_blog_tool
 
```python
class SearchBlogTool(BaseTool):
    name = "search_blog"
    description = "블로그 게시글 조회"
    requires = ("user_question",)
    provides = ("blog_posts",)
    is_terminal = False

    def __init__(
            self,
            blog_post_chunk_query_port: BlogPostChunkQueryPort,
            blog_post_query_port: BlogPostQueryPort,
            embed: EmbedPort,
    ):
        self.blog_post_chunk_query_port = blog_post_chunk_query_port
        self.blog_post_query_port = blog_post_query_port
        self.embed = embed

    def build_input(self, state: dict[str, Any]) -> dict[str, Any]:
        return {
            "user_question": state["user_question"]
        }

    def validate(self, input_data: dict[str, Any]) -> str | None:
        user_question = input_data.get("user_question")

        if user_question is None:
            return "user_question는 필수입니다."
        if not isinstance(user_question, str):
            return "user_question는 문자열이어야 합니다."
        if not user_question.strip():
            return "user_question는 비어 있을 수 없습니다."
        return None

    def run(self, input_data: dict[str, Any], context: ToolContext) -> ToolResult:

        query = input_data["user_question"].strip()

        query_vector = self.embed.embed(text=query)

        chunks = self.blog_post_chunk_query_port.search_similar(
            query_vector=query_vector,
            limit=3,
        )

        blog_ids = []
        for chunk in chunks:
            if chunk.blog_post_id not in blog_ids:
                blog_ids.append(chunk.blog_post_id)

        posts = self.blog_post_query_port.find_by_blog_post_ids(blog_ids)

        blog_posts = [
            {
                "title": post.title,
                "description": post.description,
                "source_path": post.source_path,
                "tags_json": post.tags_json,
                "content": post.content
            }
            for post in posts
        ]

        return ToolResult(
            success=True,
            data={"blog_posts": blog_posts}
        )
```

> summarize_context_tool

```python
def prompt_builder(contents: list[Any]) -> str:
    joined_contents = "\n\n".join(
        f"[문서 {idx + 1}]\n{str(content)}"
        for idx, content in enumerate(contents)
    )
    return f"""
            ...
            """.strip()


class SummarizeContextTool(BaseTool):
    name = "summarize_context"
    description = "블로그 게시글 조회"
    requires = ("blog_posts",)
    provides = ("summary",)
    is_terminal = False

    def __init__(self, llm: LLMPort):
        self.llm = llm

    def build_input(self, state: dict[str, Any]) -> dict[str, Any]:
        return {
            "blog_posts": state["blog_posts"]
        }

    def validate(self, input_data: dict[str, Any]) -> str | None:
        blog_posts = input_data.get("blog_posts")
        if blog_posts is None:
            return "blog_posts은 필수입니다."
        if not isinstance(blog_posts, list):
            return "blog_posts은 리스트여야 합니다."
        if len(blog_posts) == 0:
            return "blog_posts은 비어 있을 수 없습니다."
        return None

    def run(self, input_data: dict[str, Any], context: ToolContext) -> ToolResult:
        blog_posts = input_data.get("blog_posts")
        prompt = prompt_builder([post['content'] for post in blog_posts])
        summary = self.llm.generate(prompt)
        return ToolResult(success=True, data={"summary": summary})
```

> answer_draft_tool

```python
def prompt_builder(question: str, references: list[dict], summary: str) -> str:
    formatted_refs = "\n".join([
        f"""
        [글 {i + 1}]
        제목: {ref.get("title")}
        내용: {ref.get("content")[:500]}
        태그: {ref.get("tags_json")}
        경로: {ref.get("source_path")}
        """
        for i, ref in enumerate(references)
    ])

    return f"""
            ...
            """


class AnswerDraftTool(BaseTool):
    name = "answer_draft"
    description = "최종 답변 생성 툴"
    requires = ("user_question", "blog_posts", "summary",)
    provides = ("answer",)
    is_terminal = True

    def __init__(self, llm: LLMPort):
        self.llm = llm

    def build_input(self, state: dict[str, Any]) -> dict[str, Any]:
        return {
            "user_question": state["user_question"],
            "blog_posts": state["blog_posts"],
            "summary": state["summary"],
        }

    def validate(self, input_data: dict[str, Any]) -> str | None:
        user_question = input_data["user_question"]
        blog_posts = input_data["blog_posts"]
        summary = input_data["summary"]
        if summary is None:
            return "요약 본문은 필수입니다."
        if blog_posts is None:
            return "요약 본문은 필수입니다."
        if user_question is None:
            return "요약 본문은 필수입니다."
        return None

    def run(self, input_data: dict[str, Any], context: ToolContext) -> ToolResult:

        user_question = input_data["user_question"]
        blog_posts = input_data["blog_posts"]
        summary = input_data["summary"]

        prompt = prompt_builder(user_question, blog_posts, summary)
        answer = self.llm.generate(prompt)

        return ToolResult(success=True, data={"answer": answer})

```

이렇게 바뀌면 `Router`가 자동으로 순서를 제어할 수 있다.


<h2> 7. Service는 Orchestrator만 호출 </h2>

코드

```python
class BlogAnswerService(UserAnswerUseCase):

    def __init__(
            self,
            tool_flow_orchestrator: ToolFlowOrchestrator
    ):
        self.tool_flow_orchestrator = tool_flow_orchestrator

    def execute(self, state: dict[str, Any]) -> str:
        agent_id = "blog-answer-agent"
        session_id = "blog-answer-session"
        user_id = "anonymous"

        state = self.tool_flow_orchestrator.run(
            initial_state=state,
            agent_id=agent_id,
            user_id=user_id,
            session_id=session_id
        )

        return state["answer"]
```

위 구조로 서비스는

- 흐름을 알지 못한다.
- 결과만 사용한다.


<h2> 8. 이번 리펙토링의 핵심 변화 </h2>

정리하면

| 변경             | 목적          |
| -------------- | ----------- |
| BaseTool 메타데이터 | 실행 규칙 명시    |
| Registry       | Tool 관리     |
| Router         | 실행 순서 결정    |
| Orchestrator   | 실행 루프       |
| state 기반 실행    | Tool 연결 일반화 |

동적 Tool 선택을 위한 기반을 마련하기 위한 리펙토링 이었다.


<h2> 9. 동적 Tool 실행 구조를 만들면서 느낀 점 </h2>

이번 리팩터링의 목적은 단순히 코드를 정리하는 것이 아니라  
`Tool`을 상황에 따라 동적으로 선택하고 실행할 수 있는 구조를 만드는 것이었다.

처음 구조는 매우 단순했다.  
`Service`에서 필요한 `Tool`을 순서대로 호출하는 방식이었다.

> search -> summarize -> answer


이 구조는 `Tool`이 몇 개 없을 때는 문제가 없었다.  
하지만 `Tool`의 개수가 늘어나기 시작하면서 점점 한계가 보이기 시작했다.

- 실행 순서를 `Service`가 직접 알고 있어야 한다.
- `Tool`이 추가될 때마다 `Service`를 수정해야 한다.
- `Tool` 사이의 연결 규칙이 코드에 하드코딩된다.
- 실행 흐름을 재사용할 수 없다.

결국 문제의 핵심은 하나였다.

> 실행 순서를 코드가 아니라 상태(state)로 결정할 수 있어야 한다.

이 결론에 도달하면서 자연스럽게 지금 구조가 나오게 되었다.



<h3> requires / provides 가 필요했던 이유 </h3>

동적으로 `Tool`을 선택하려면  
각 `Tool`이 어떤 데이터를 필요로 하고, 어떤 데이터를 만들어내는지를 알아야 한다.

그래야

- 지금 실행 가능한 `Tool`이 무엇인지
- 어떤 `Tool`을 먼저 실행해야 하는지
- 어떤 `Tool`이 마지막인지

를 판단할 수 있다.

그래서 `BaseTool`에 아래 메타데이터를 추가하게 되었다.

```
requires
provides
is_terminal
```

이 정보가 있어야 `Router`가 `Tool`을 선택할 수 있다.

<h3> ToolRegistry가 필요했던 이유 </h3>

`Tool`을 동적으로 선택하려면  
현재 사용할 수 있는 `Tool` 목록을 관리할 수 있어야 한다.

`Service`가 `Tool`을 직접 생성하는 구조에서는

- `Tool` 목록을 조회할 수 없고
- `Tool`을 교체하기 어렵고
- `Router`가 `Tool`을 선택할 수도 없다.

그래서 `Tool`을 등록하고 조회할 수 있는 `ToolRegistry` 가 필요해졌다.


<h3> Router가 필요했던 이유 </h3>

Tool을 여러 개 등록해 놓으면  
그 다음에 생기는 문제는 이것이다.

> 다음에 어떤 Tool을 실행해야 할까?

이 로직을 `Service`에 넣으면  
결국 다시 하드코딩으로 돌아가게 된다.

그래서 실행 순서를 결정하는 역할을 분리했고  
그 역할을 `Router`가 맡도록 했다.

> Router → 다음 Tool 선택


현재는 `Rule` 기반으로 구현했지만

- LLM Router
- Planner
- Agent Router

로 확장할 수 있는 구조가 되었다.


<h3> Orchestrator가 필요했던 이유 </h3>

`Router`가 `Tool`을 선택한다고 해서 실행이 끝나는 것은 아니다.

`Tool`을 실행하고  
`state`를 업데이트하고  
다시 `Router`에게 다음 `Tool`을 물어보고  
이 과정을 반복해야 한다.

이 실행 루프를 `Service`에 넣으면  
`Service`가 다시 너무 많은 책임을 가지게 된다.

그래서 실행 흐름 자체를 담당하는 계층이 필요했고  
그 역할을 `Orchestrator`가 맡게 되었다.

```
Orchestrator → 실행 루프 관리
Router → Tool 선택
Tool → 실제 작업 수행
```

이렇게 역할을 나누면서 구조가 훨씬 명확해졌다.

<h3> state 기반 구조로 바뀌면서 가능해진 것 </h3>

이번 리팩터링 이후로 가능해진 것들이 있다.

- `Tool`을 추가해도 `Service`를 수정하지 않는다.
- 실행 순서를 자동으로 결정할 수 있다.
- `Tool`을 자유롭게 조합할 수 있다.
- 실행 흐름을 재사용할 수 있다.
- `Multi Tool` 구조로 확장할 수 있다.
- `Multi Agent` 구조로 확장할 수 있다.

이 시점부터는 단순한 `Tool` 호출 구조가 아니라  
`Agent` 구조로 확장할 수 있는 기반이 만들어졌다고 생각한다.



<h3> 앞으로 더 필요할 것 </h3>

지금 구조는 기본적인 `Tool` 실행 파이프라인까지 만든 상태다.
하지만 실제 `Agent` 구조로 가려면 아직 부족한 것들이 많다.
앞으로 필요할 것

- `Planner`
- `Retry` 정책
- `Error` 처리 정책
- `Memory`
- `Multi Agent Router`
- `LLM Router`

특히 다음으로 고민해야 할 문제는 이것이다.

> Tool을 선택하는 역할을 Router가 해야 할까  
> 아니면 Planner가 해야 할까

- 다음 글에서는 Planner는 어디에 있어야 하는가
- Router와 Planner의 역할은 어떻게 나눠야 하는가

위의 내용을 Planner를 구현해 보면서 정리해 보려고 한다

<h2> 10. 코드 </h2>

- 리펙토링 전 : [https://github.com/AngryPig123/ai-agent/pull/new/tools/registry-router-orchestrator-before](https://github.com/AngryPig123/ai-agent/pull/new/tools/registry-router-orchestrator-before)

- 리펙토링 후 : [https://github.com/AngryPig123/ai-agent/pull/new/tools/registry-router-orchestrator-after](https://github.com/AngryPig123/ai-agent/pull/new/tools/registry-router-orchestrator-after)

---
title: Ollama LLM 연결하기 — Hexagonal Architecture로 LLM Adapter 구현
description: Hexagonal Architecture 구조를 유지하면서 Ollama LLM을 Port / Adapter 구조로 연결한다.
date: 2026-03-13T13:30:00+09:00
updated: 2026-03-13T13:30:00+09:00
categories: [AI, AI Agent]
tags: [AI, LLM, AI Agent, Hexagonal Architecture, Ollama, Adapter]
---

<h2> 1. LLM 연결하기 </h2>

진행하는 프로젝트에서는 `AI Agent`를 확장 가능한 구조로 만들고 싶었기에 `Hexagonal Architecture`를 기반으로 구현하려고 한다.

그런 관점에서 볼때 `LLM`은 외부 시스템에 해당하기 때문에 도메인 로직에서 직접 호출하지 않고 `Port / Adapter` 구조로 분리하여 구현되어야 한다.

이렇게 구조를 나누는 이유는

- `OpenLLM` -> `ClosedLLM` 추가 및 변경 가능
- 모델 추가 및 변경 가능
- `API`방식 변경 가능
- 테스트 가능
- 의존성 분리

즉,

```
Domain -> Port -> Adapter -> LLM
```

구조로 연결하도록 하기 위함이다.

이번 글에서는 그 첫번째로 `Ollama`를 통해서 `Qwen2.5:7b` 모델을 `Adapter`로 연결해 본다.

---

<h2> 2. LLM Port 정의 </h2>

`LLM`은 외부 시스템이므로 `Port`를 먼저 정의한다.

경로 `domain/port/llm_port.py`

```python
from abc import ABC, abstractmethod


class LLMPort(ABC):

    @abstractmethod
    def generate(self, prompt: str) -> str:
        pass

```

`LLM`을 사용하는 쪽에서는 이 인터페이스만 알고 있으면 된다. 이렇게 되면 `Adapter`를 변경해도 도메인 코드에 추가적은 추정은 필요 없어진다.


<h2> 3. Ollama Adapter 구현 </h2>

`Adapter`는 외부 시스템과 연결되는 구현체이므로 `adapter`레이어에 작성한다.
경로 `adapter/llm/ollama_adapter.py`

```python
import requests

from domain.port.llm_port import LLMPort


class OllamaLLMAdapter(LLMPort):

    def __init__(self, model: str):
        self.model = model
        self.url = "http://localhost:11434/api/generate"

    def generate(self, prompt: str) -> str:
        response = requests.post(
            self.url,
            json={
                "model": self.model,
                "prompt": prompt,
                "stream": False
            },
        )
        response.raise_for_status()
        data = response.json()
        return data["response"]
```

해당 `Adapter`는 

- `Ollama API 호출`
- 모델 실행
- 결과반환

의 역할만 수행한다. 도메인 로직은 `Ollama`에 대해 알 필요가 없다.

<h2> 4. Application Service에서 사용 </h2>

`Agent` 실행 로직은 `application` 레ㅇ이어에 위치 시킨다.
경로 `application/service/agent_service.py`

```python
from domain.port.llm_port import LLMPort


class AgentService:
    def __init__(self, llm: LLMPort):
        self.llm = llm

    def ask(self, text: str) -> str:
        return self.llm.generate(text)

```

`Application`은

- `Port`만 의존
- `Adapter` 모름

이게 `Hexagonal Architecture`의 핵심이다.

<h2> 5. 실행 코드 작성 </h2>

`main`에서 `Adapter`를 생성하고 `Service`에 주입한다.

```python
from adapter.llm.ollama_adapter import OllamaLLMAdapter
from application.service.agent_service import AgentService


def main():
    llm = OllamaLLMAdapter(
        model="qwen2.5:7b"
    )

    service = AgentService(llm)

    result = service.ask(
        "DDD가 무엇인지 설명해줘"
    )

    print(result)


if __name__ == "__main__":
    main()

```

위 코드를 실행하면 `Ollama`를 통해 `qwen2.5:7b` `LLM` 모델이 호출되어 응답을 받을 수 있게 된다.

![호출 이미지](/assets/img/aiagent/llm-connect-llm-response.png)


<h2> 6. 현재 구조 </h2>

현재 구조와 의존성 방향으로 봤을떄 의도한대로 `Domain`은 외부 의존성을 아무것도 모르게 된다.

이런 구조를 유지하면서 기능을 계속 추가할 예정이다.

<h2> 7. 다음 단계 </h2>

지금은 `LLM`만 연결한 상태이기 때문에 `Agent`라고 하기에는 부족한점이 많다.

`AI Agent`가 되려면 외부 기능을 사용할 수 있어야 한다.
다음 글에서는 `Tool`에 대해 알아보고 추가하여 외부 기능을 호출할 수 있도록 기능을 확장해 본다.
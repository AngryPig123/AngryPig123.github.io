---
title: AI Agent를 직접 만들어보려고 한 이유와 프로젝트 설정
description: LLM, Tool Calling, RAG, Agent, Workflow, State, Memory 구조를 이해하기 위해 블로그 QA Agent를 직접 구현해보려고 한다.
date: 2026-03-13T12:00:00+09:00
updated: 2026-03-13T12:00:00+09:00
categories: [AI, AI Agent]
tags: [AI, LLM, AI Agent, RAG, Tool Calling, Workflow]
---

<h2> 1. 왜 직접 만들어보려고 했는가 </h2>

- [AI Agent 란?](https://angrypig123.github.io/posts/aiagent/)

새로운 기술을 이해할 때, 개념을 오래 붙잡고 있기보다는 
전체 구조를 빠르게 훑어보고 직접 구현하면서 익히는 방식이 더 잘 맞는 편이다.

그래서 이번에는 `AI Agent`라는 주제를 정하고, 단순한 개념 정리가 아니라  
실제로 동작하는 `Agent`를 단계별로 만들어 보면서 구조를 이해해 보려고 한다.

이 시리즈의 목표는 `LLM`, `Tool Calling`, `RAG`, `Agent`, `Workflow`, `State`, `Memory` 같은 개념들을 각각 따로 공부하는 것이 아니라 하나의 시스템 안에서 확장해 가며 이해하는 것이다.

이를 위해 주제로 선택한 것은  `내 기술 블로그를 이해하고 분석할 수 있는 AI Agent를 직접 만들어 보는 것`이다.

이 프로젝트는 단순한 예제가 아니라 실제로 코드 `GitHub` 코드 QA와 같은 확장 가능한 구조로 만들어 볼 예정이며,
아래와 같은 순서로 기능을 하나씩 추가해 가면서 구현하려고 한다.

- 프로젝트 설정
- `LLM` 연결
- `Tool` 구조 만들기
- `RAG` 적용
- `Agent` 구현
- `Workflow` 구성
- `State` 관리
- `Memory` 추가
- `LangChain` 적용
- `LangGraph` 적용
- `Service Architecture` 구성
- `Eval` 및 테스트

각 단계마다 필요한 개념을 정리하고, 실제 코드로 구현하면서 전체 구조를 이해하는 것이 목표이다.

이 글에서는 먼저 프로젝트를 생성하고 기본 구조를 설정하는 것부터 시작한다.


<h2> 2. 프로젝트 설정 </h2>

AI Agent를 구현하는 방법은 여러 가지가 있지만,  
이번 프로젝트에서는 다음 기준으로 기술을 선택했다.

- 빠르게 실험할 수 있을 것
- `LLM` 연동이 쉬울 것
- `RAG` / `Agent` / `Workflow` 구조 확장이 가능할 것
- 로컬에서도 실행할 수 있을 것

이 기준을 바탕으로 다음과 같은 환경을 사용하기로 했다.

- `Python`
- `Ollama` (로컬 `LLM` 실행) : [Ollama 사용법 정리](https://angrypig123.github.io/posts/ollama/)
- `requests` / `http client`
- 이후 단계에서 `Vector DB` / `LangChain` / `LangGraph` 추가 예정

프로젝트는 `Python/FastAPI`를 사용하며 하나의 `Agent`를 계속 확장해 가는 방식으로 진행할 예정이기 때문에 처음부터 패키지 구조를 나누고 시작한다.


---


<h3> 2.1 프로젝트 생성 </h3>

이번 프로젝트는 `Python` 기반으로 구현하며 `IDE`는 `Pycharm`을 사용한다.

`AI Agent`를 단순 스크립트가 아니라 확장 가능한 구조로 만들기 위해  
`FastAPI` 프로젝트를 생성하고 서버 형태로 개발을 진행할 예정이다.

LLM은 로컬 환경에서 실행할 수 있도록 `Ollama`를 사용하고,  
모델은 비교적 가볍고 성능이 안정적인 `qwen2.5:7b` 모델을 선택했다.

- 프로젝트 초기화 : ![프로젝트 초기 설정](/assets/img/aiagent/ai-agent-project-init.png)

- `OpenLLM` 모델 :
```
PS D:\gitblog\AngryPig123.github.io> Ollama ps
NAME          ID              SIZE      PROCESSOR    CONTEXT    UNTIL
qwen2.5:7b    845dbda0ea48    4.6 GB    100% CPU     4096       4 minutes from now
```


---

<h3> 2.2 폴더 구조 </h3>

이번 프로젝트는 단순한 예제가 아니라  
`LLM`, `RAG`, `Agent`, `Workflow`, `Memory`, `State` 같은 기능을 계속 확장해 나갈 예정이기 때문에 처음부터 구조를 나누고 시작하는 것이 좋다고 판단했다.

프로젝트 구조는 이전에 정리했던 것처럼 `Hexagonal Architecture` 기반으로 구성하기로 했다.

- [Hexagonal Architecture 란?](https://angrypig123.github.io/posts/hexagonal-architecture/)

`AI Agent`는 외부 시스템(`LLM`, `Vector DB`, `API`, 파일 등)과의 연결이 많고  
구현 방식이 계속 변경될 가능성이 있기 때문에  
도메인 로직과 외부 의존성을 분리하는 구조가 적합하다.

따라서 다음과 같은 구조로 프로젝트 폴더를 구성한다.

```
ai-agent/
 ├ domain/
 │   ├ agent/
 │   ├ model/
 │   └ port/
 │
 ├ application/
 │   ├ service/
 │   └ workflow/
 │
 ├ adapter/
 │   ├ llm/
 │   ├ rag/
 │   ├ tool/
 │   └ api/
 │
 ├ infrastructure/
 │   ├ config/
 │   └ persistence/
 │
 ├ main.py
```

각 디렉토리의 역할은 다음과 같다.

- `domain`  
  `Agent`의 핵심 로직과 모델을 정의한다.  
  외부 라이브러리에 의존하지 않도록 구성한다.

- `application`  
  `Agent`의 실행 흐름과 유스케이스를 담당한다.  
  `Workflow`, `Service` 등이 위치한다.

- `adapter`  
  외부 시스템과 연결되는 구현체가 들어간다.  
  `LLM`, `RAG`, `Tool`, `API` 같은 어댑터가 위치한다.

- `infrastructure`  
  설정, DB, 환경 구성 등 기술적인 요소를 담당한다.

이 구조를 사용하면 이후에 `LLM`을 변경하거나  
`Vector DB`를 교체하거나  
`Agent` 실행 구조를 바꾸더라도  
도메인 로직을 수정하지 않고 확장할 수 있다.

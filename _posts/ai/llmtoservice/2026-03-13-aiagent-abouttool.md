---
title: Tool 구조 만들기 — AI Agent에서 외부 기능을 사용하는 방법
description: AI Agent에서 Tool이 필요한 이유와 구조를 정리하고 블로그 QA Agent에 필요한 Tool을 정의한다.
date: 2026-03-13T14:00:00+09:00
categories: [AI, AI Agent]
tags: [AI, AI Agent, Tool, LLM, Hexagonal Architecture]
---

<h2> 1. AI Agent에서 Tool이 필요한 이유 </h2>

이전 글에서는 `LLM`을 `Port / Adapter` 구조로 연결했다.

현재 구조에서는 `LLM`문자열을 입력 받고 문자열을 생성하는 것만 가능하다.
하지만 우리가 `AI Agent`로 하려는 것들은 단순한 챗봇이 아닌 외부 기능을 사용할 수 있는 `AI Agent`이다.

예를 들어 내가 구현하려는 서비스는 다음과 같은 동작이 가능해야 한다.

- 블로그 글 검색
- 특정 글 읽기
- 글 목록 조회
- 글 요약
- 글 비교
- 키워드 찾기

이런 기능은 `LLM`이 직접 할 수 없다.
`LLM`자체는 머리만 있고 기능을 수행할 팔과 다리는 없는 상태이기 때문이다.

그래서 `Agent`에서는 팔과 다리의 기능을할 `Tool`이 필요하다. 즉,

```
LLM -> Tool 선택 -> Tool 실행 -> 결과 반환 -> LLM
```

이런 구조가 필요하다.

이때 외부 기능을 `Tool`이라고 한다.

<h2> 2. Tool 이란 무엇인가? </h2>

`Tool`은 `LLM`이 사용할 수 있는 외부 기능이다. 예를 들어

- 검색 함수
- `DB`조회 함수
- 파일 읽기 함수
- `API` 호출 함수
- 계산 함수

와 같은 것들이 `Tool` 이다.

`Agent`구조에서는 보통 다음 흐름으로 동작한다.

```
사용자 질문
↓
LLM 판단
↓
Tool 선택
↓
Tool 실행
↓
결과 반환
↓
LLM 최종 답변 생성
```

즉, `Tool`은 `LLM`이 직접 못 하는 일을 대신 해주는 기능 이다.

<h2> 3. Hexagonal Architecture에서 Tool의 위치 </h2>

`Tool`도 결국 외부 시스템이기 때문에 도메인 로직을 보호하기 위하여 `Port / Adapter` 구조로 구현해야 한다.

<h2> 4. Tool 구조 설계 </h2>

`Tool`은 다음 구성으로 나눌 수 있다.

```
Tool Port
Tool Adapter
Tool Registry
Tool Excutor
```

역할

| 구성       | 역할      |
| -------- | ------- |
| Port     | 인터페이스   |
| Adapter  | 실제 구현   |
| Registry | Tool 목록 |
| Executor | 실행 담당   |

이번 글에서는 `Tool Port`와 서비스에 필요한 `Tool`목록 정리까지만 하고 구현은 다음 글에서 한다.


<h2> 5. 블로그 QA Agent에서 필요한 Tool </h2>

이번 프로젝트의 1차 목표는 `내 기술 블로그를 이해하는 AI Agent 서비스` 개발이다.
때문에, 다음 기능이 필요하다.

<h2> 5. 블로그 QA Agent에서 필요한 Tool </h2>

이번 프로젝트의 1차 목표는 `내 기술 블로그를 이해하는 AI Agent 서비스` 개발이다.
때문에, 다음 기능이 필요하다.

- 블로그 글 목록 조회
  - 글 전체 목록 가져오기 : 예 ) `DDD`에 관한 글 뭐 있어?
- 블로그 글 읽기
  - 특정 글 가져오기 : `DDD`에 관한 글 설명해줘
- 블로그 검색
  - 키워드 기반 검색 : `Tool Calling` 관련 글 찾아줘.
- 글 요약
  - 글 내용 요약 : 이 글 요약해줘
- 글 비교
  - 두 글 비교 : `DDD`랑 `Hexagonal Architecture` 차이 설명해줘
- 키워드 추출
  - 글에서 핵심 키워드 찾기 : 이 글 핵심 키워드 뭐야?


<h2> 6. Tool 목록 정리 </h2>

이번 프로젝트에서 사용할 `Tool`아래의 `Tool`들을 `Port / Adapter` 구조로 구현하고 `AI Agent`에서 사용할 수 있도록 만들 예정이다.

```
list_posts
read_post
search_posts
summarize_post
compare_posts
extract_keywords
```
<h2> 7. 다음 단계 </h2>

이제 `Tool`의 종류를 정의했지만, 현재는 실제로 조회할 데이터가 없는 상태이다.
매 호출마다 블로그의 글들을 서칭해서 동작할 순 있겠지만 `OpenLLM` 특성상 안그래도 늦은 응답 시간에 더해져 사용하기 불편할수도 있기 때문이다. 

그렇기 때문에 먼저 블로그 글을 수집하고, `AI Agent`가 사용할 수 있도록 저장하는 과정이 필요하다 생각했다.

따라서 다음 단계에서는

- 블로그 글 수집
- 글 메타데이터 파싱
- 요약 / 키워드 생성
- 저장 구조 설계

를 먼저 구현해보고,
그 이후에 `Tool Port`와 `Adapter`를 만들어
Agent가 해당 데이터를 조회할 수 있도록 구성해볼 예정이다.
---
title: RAG? Embedding? VectorDB?
description: AI Agent 서비스를 구현하기 위한 필수 요소인 RAG, Embedding, VectorDB에 대해 알아본다.
date: 2026-03-16T16:11:30+09:00
date: 2026-03-16T16:11:30+09:00
categories: [AI, Embedding]
tags: [AI, AI Agent, RAG, Embedding, VectorDB]
---

<h2> 0. 알아보게 된 이유 </h2>

블로그 글을 기반으로 질의응답을 하는 `AI Agent`를 만들려고 하다 보니  
단순히 `LLM`만 사용하는 방식으로는 해결되지 않는다는 것을 알게 되었다.

이런 구조를 만들기 위해서는 `RAG(Retrieval Augmented Generation)`라는 개념이 필요하고,  
문서를 검색하기 위한 `Embedding`, `Vector DB`, `Chunking` 같은 기술들도 같이 사용된다.

이전에 개념을 들어본 적은 있지만  
왜 필요한지, 어떤 구조로 동작하는지, 실제로 어떻게 설계해야 하는지는 깊게 이해하지 못한 상태였다.

그래서 이번에 블로그 글을 수집하고 `AI Agent`가 사용할 데이터를 만드는 과정을 진행하면서  
`RAG` 구조와 필요한 기술들을 하나씩 정리해보려고 한다.


<h2> 1. 왜 RAG를 이해해야 하는가? </h2>

블로그 글을 기반으로 질의응답을 하는 `AI Agent`를 만드려면 `LLM`만으로는 해결되지 않는다.

`LLM`은 기본적으로 학습된 지식만 사용할 수 있기 때문에 직접 파인튜닝을 하지 않는이상 내 블로그, 내 문서, 내 코드 같은 외부 데이터를 직접 알지 못한다.

그래서 필요한 구조가 `RAG(Retrieval Augmented Generation)`이다.


> 질문 -> 관련 문서 검색 -> LLM에게 같이 전달 -> 답변 생성


이 구조를 만들기 위해서 알아야 되는 개념으로는 아래와 같은 내용들이 있고

- `Embedding`
- `VectorDB`
- `Chunking`
- `Retrieval`
- `Ingestion`

이 글에서는 그중에서도 `Embedding`이 뭔지 `VectorDB`가 뭔지 `RAG`에서 왜 `Embedding`이 중요한지를 정리한다.


<h2> 2. Embedding 란? </h2>

`Embedding`은 텍스트를 숫자 백터로 변환하는 것이다.

예를 들어 아래와 같이 언어가 달라도 의미가 비슷한 문장은 백터도 비슷해진다.

```
"헥사고날 아키텍처"
→ [0.12, -0.55, 0.91, ...]

"hexagonal architecture"
→ [0.11, -0.52, 0.89, ...]
```

즉 `Embedding` 모델은 문장의 의미를 숫자로 표현하는 모델 이라고 볼 수 있다.

`Embedding` 모델의 역할은 단순하게 생각하면 `text -> vector`로 바꿔주는 모델이다.

대표적인 모델로는 아래와 같은 모델들이 있고 이 모델들은 답변이 아닌 오직 벡터만을 만든다.

- text-embedding-3-large
- bge-large
- e5-large
- nomic-embed-text-v2-moe
- jina-embeddings


<h2> 3. 왜 텍스트를 벡터로 변환해야 하는가? </h2>

문자열 검색은 의미를 이해하지 못한다.

예를 들어 아래와 같이 문자열은 다르지만 의미가 같은 케이스를 예로 들 수 있다.

```
질문 : 주문 취소 어떻게 해?
문서 : order cancel API 사용 방법
```

이 문제를 해결하기 위해 텍스트를 벡터로 변환하고 벡터간의 거리를 계산해서 의미 유사도를 계산하는 방식을 사용한다. 이걸 `sementic search`라고 한다.


<h2> 4. Vector DB란 ? </h2>

`Embedding`을 만들면 아래와 같은 데이터가 생긴다.

```
vector = [0.12, 0.33, ...]
text = "Port는 외부와의 경계이다."
```

해당 벡터를 저장하고 가장 비슷한 벡터를 빠르게 찾아야한다.

일반 데이터 베이스는 이러한 검색에 최적화 되어있지 않다.

그래서 사용하는게 `VectorDB`이다.

`VectorDB`의 역할은 아래와 같다.

- 벡터 저장
- 벡터 유사도 검색
- `nearest search`
- `metadata filter`

대표적인 제품

- pgvector
- qdrant
- pinecone
- milvus
- weaviate
- chroma

저장되는 데이터 형태는 아래와 같이 벡터만 저장하지 않고 메타 데이터도 같이 저장하는게 중요하다.

```
id: chunk_10 
vector: [0.12, 0.33, ...] 
payload: 
    postId: 10 
    title: hexagonal 
    url: /posts/hexagonal 
    chunkIndex: 2 

text: 
    Port는 외부와의 경계이다
```


<h2> 5. RAG란 ? </h2>

`RAG`는 `Retrieval Augmented Generation`의 약자이다.

구조는 `검색 + LLM`의 구조로 단순하다.

`LLM`은 기본적으로 외부 데이터를 몰라서 `RAG`를 통해 외부 정보를 전달해 주어야 한다.
흐름으로는 

```
질문 -> 관련 문서 검색 -> LLM에게 전달 -> 답변 생성
```

여기서 관련 문서 검색을 담당하는것이 `Embedding` + `VectorDB`이다.

<h2> 6. RAG 전체 흐름 </h2>

- 저장 단계 (Ingestion)
  - 문서 수집
  - `chunk` 분할
  - `embedding` 생성
  - `vectorDB` 저장

- 질문 단계 (Query)
  - 질문
  - `embedding` 생성
  - `vectorDB` 검색
  - 관련 `chunk` 가져오기
  - `LLM` 전달
  - 답변 생성

위와 같은 구조가 기본적인 `RAG` 구조이다.

<h2> 7. 왜 Chunk로 나누는가? </h2>

문서 하나를 그대로 임베딩하면 문제가 생긴다.
문서가 너무 길어지는 경우 의미가 흐려지고 검색이 부정확, 토큰이 제한된다.

그래서 문서를 분할해서 `Embedding`를 한다.

예, `chunk1, chunk2, chunk3, chunk4` 이런식으로 각 `chunk`마다 `embedding`을 만든다.

<h2> 7. 왜 Embedding이 중요한가. </h2>

`RAG`의 성능은 대부분 여기서 결정된다.

- `chunk`
- `embedding`
- `vector search`
- `prompt`
- `rerank`

특히 embedding이 좋지 않으면

> 검색 실패 -> 문서 실패 -> 답변 실패  

로 이어진다.


그래서 RAG에서는

> 임베딩 모델 선택과 chunk 설계가 핵심

이라고 한다.
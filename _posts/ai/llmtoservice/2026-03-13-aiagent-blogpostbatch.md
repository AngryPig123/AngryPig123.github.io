---
title: 블로그 글 수집 배치 만들기 — Spring Batch로 AI Agent용 데이터 준비하기 (1)
description: 블로그 QA Agent가 사용할 데이터를 만들기 위해 Spring Batch 프로젝트를 생성하고 글 수집 구조를 설계한다.
date: 2026-03-13T16:00:00+09:00
updated: 2026-03-16T22:00:00+09:00
categories: [AI, AI Agent]
tags: [AI, AI Agent, Spring Batch, Batch, Ingestion, RAG, Embedding, Chunk]
---

<h2> 1. 구성하게 된 이유 </h2>

`AI Agent`를 구현하면서 `Tool` 목록을 정리하다 보니,  
먼저 `Agent`가 조회하고 활용할 수 있는 데이터가 준비되어 있어야 한다고 판단했다.

이를 처리하는 방법은 크게 두 가지였다.

- 요청 시점마다 블로그 글을 직접 탐색하는 방식
- 배치 작업으로 미리 데이터를 수집하고 적재해 두는 방식

이번 프로젝트는 실제 서비스 운영까지 고려하고 있고,  
로컬 LLM 기반으로 동작시킬 예정이기 때문에  
응답 시간을 최대한 줄이는 것이 중요한 과제라고 생각했다.

요청마다 블로그 원문을 다시 읽고 필요한 결과를 생성하는 방식은  
구조는 단순할 수 있지만 응답 속도와 확장성 측면에서 한계가 있다.  
그래서 공수가 더 들더라도,  
블로그 글을 미리 수집하고 가공할 수 있는 별도의 프로젝트를 구성하기로 했다.

이전에 `Spring Batch`를 학습한 경험이 있었기 때문에  
이번에는 복습도 겸해서 `Spring Batch`를 이용해  
블로그 원문을 수집하고, 요약/키워드 생성 및 임베딩 처리 후  
`Vector DB`에 적재하는 배치 서비스를 만들어 보려고 한다.

이 과정에서 필요한 LLM 및 임베딩 작업은 `Ollama API`를 통해 처리할 예정이므로,  
별도의 전용 라이브러리에 강하게 의존하지 않고도 현재 구조에서 충분히 구현 가능하다고 판단했다.


<h2> 2. 데이터 베이스 선택 </h2>


[데이터 베이스 선택 및 설치](https://angrypig123.github.io/posts/dockerpostgresqlpgvector/)


<h2> 3. 블로그 데이터 수집 및 임베딩 전략 </h2>

데이터 수집은 `Jekyll + Chirpy` 기반 깃블로그에서 자동 생성되는 `/my-sitemap.xml`을 시작점으로 한다.
먼저 사이트맵에서 게시글 목록과 기본 정보를 수집한 뒤, 각 게시글 페이지에 직접 접근하여 `Jsoup`으로 본문과 메타데이터를 파싱한다.
파싱한 원문 데이터는 `RDB`에 저장하고, 이후 원문을 청크 단위로 분할한 뒤 임베딩하여 생성한 벡터 데이터는 `Vector DB`에 저장한다.
이렇게 저장된 데이터는 이후 검색 기능이나 `RAG` 파이프라인에서 활용할 수 있다.

- 수집 전략을 설계할 때는 다음 사항을 함께 고려해야 한다.
  - 이미 수집된 게시글은 중복 수집되지 않아야 한다.
  - 수정된 게시글은 원문 데이터를 다시 갱신해야 한다.
  - 수정된 게시글은 변경된 내용을 기준으로 다시 청크를 나누고, 재임베딩해야 한다.

- 고려한 사항
  - 게시글 수정 여부는 어떻게 판단할 것인가?
  - `Jekyll Chirpy` 프레임워크의 공통 설정을 활용해, `front matter`의 커스텀 데이터를 `meta` 정보에 포함시킨다.
  - 이를 통해 게시글의 `updated` 값을 수집하고, 기존 데이터와 비교하여 재수집 및 재임베딩 여부를 판단한다.
  - 블로그 글은 어떻게 수집할 것인가?
  - `Jekyll Chirpy`에서 자동 생성되는 `/my-sitemap.xml`에서 게시글 `URL` 목록을 추출한다.
  - 추출한 `URL`에 순차적으로 접근하여, 배치 프로그램에서 아래 항목을 파싱한 뒤 원문 데이터 `RDB`에 저장한다.
    - `title`
    - `description`
    - `date`
    - `updated`
    - `categories`
    - `tags`
  - 청크 분할은 어떻게 할 것인가?
  - `Jekyll Chirpy`에서는 `Markdown`의 `##` 구문이 `HTML` 변환 시 `h2` 태그로 변환된다.
  - 이를 기준으로 먼저 1차 분할을 수행하고, 각 구간의 길이가 지나치게 길 경우에는 휴리스틱 기반으로 추가 분할한다.
  - 임베딩 데이터는 어떻게 생성할 것인가?
  - 분할된 청크 데이터를 `Ollama Embedding API`에 전달하여 벡터화된 임베딩 값을 생성한다.
  - 벡터 데이터는 어떻게 저장할 것인가?
  - 생성된 벡터 데이터를 기반으로 `JPA Entity`를 구성하고, `pgvector`를 사용하는 벡터 테이블에 저장한다.


- 이후 추가로 고민할 사항
  - 코드 블록 처리
  - 코드 블록은 일반적으로 길이가 길고 구조가 뚜렷하기 때문에, 현재의 휴리스틱 기반 분할 방식에서는 문맥이 부자연스럽게 끊길 가능성이 크다.
  - 따라서 코드 블록은 일반 본문과 다르게 취급할 필요가 있으며, 분할 전략을 별도로 설계할지 추후 검토가 필요하다.


<h3> 4. Spring batch 프로젝트 준비 </h3>


- 의존성 정보

```

JDK : Amazon Correto 17.0.18
SpringBoot : 3.5.11
Postgresql & pgvector : 16

dependencies {

    //  spring boot
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'

    //  util
    implementation 'org.jsoup:jsoup:1.21.2'

    //  db
    runtimeOnly 'org.postgresql:postgresql'
    implementation 'org.hibernate.orm:hibernate-vector:6.6.4.Final'

    //  lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    //  test
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.batch:spring-batch-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'

}
```


<h2> 5. 프로젝트 구조 </h2>

설계 및 구조 : `DDD + Hexagonal Architecture`


```
├── src
│   ├── main
│   │   ├── generated
│   │   ├── java
│   │   │   └── com
│   │   │       └── ai
│   │   │           └── agent
│   │   │               └── batch
│   │   │                   ├── AiAgentBatchApplication.java
│   │   │                   ├── application
│   │   │                   │   ├── dto
│   │   │                   │   │   ├── BlogChunk.java
│   │   │                   │   │   ├── BlogPostSnapshot.java
│   │   │                   │   │   ├── EmbeddedChunk.java
│   │   │                   │   │   ├── H2Section.java
│   │   │                   │   │   └── VectorBlogPostSnapshot.java
│   │   │                   │   ├── mapper
│   │   │                   │   │   └── BlogPostApplicationMapper.java
│   │   │                   │   ├── port
│   │   │                   │   │   └── out
│   │   │                   │   │       ├── BlogPostChunker.java
│   │   │                   │   │       ├── BlogPostParser.java
│   │   │                   │   │       ├── BlogPostRepository.java
│   │   │                   │   │       ├── BlogSourceClient.java
│   │   │                   │   │       ├── EmbeddingPort.java
│   │   │                   │   │       └── VectorDBBlogPostRepository.java
│   │   │                   │   ├── service
│   │   │                   │   │   ├── BlogCatalogSyncService.java
│   │   │                   │   │   └── BlogEmbeddingSyncService.java
│   │   │                   │   └── usecase
│   │   │                   │       ├── SyncBlogCatalogUseCase.java
│   │   │                   │       └── SyncBlogEmbeddingUseCase.java
│   │   │                   ├── common
│   │   │                   │   └── domain
│   │   │                   │       └── model
│   │   │                   │           ├── AggregateRoot.java
│   │   │                   │           ├── BaseEntity.java
│   │   │                   │           ├── BaseId.java
│   │   │                   │           └── DomainException.java
│   │   │                   ├── domain
│   │   │                   │   ├── exception
│   │   │                   │   │   └── BlogPostBatchDomainException.java
│   │   │                   │   └── model
│   │   │                   │       ├── BlogPost.java
│   │   │                   │       ├── BlogPostChunk.java
│   │   │                   │       ├── BlogPostChunkId.java
│   │   │                   │       └── BlogPostId.java
│   │   │                   ├── infrastructure
│   │   │                   │   ├── chunk
│   │   │                   │   │   └── JekyllBlogPostChunker.java
│   │   │                   │   ├── client
│   │   │                   │   │   └── JekyllSitemapClient.java
│   │   │                   │   ├── config
│   │   │                   │   │   ├── AsyncConfig.java
│   │   │                   │   │   └── CommonConfig.java
│   │   │                   │   ├── embed
│   │   │                   │   │   └── NomicEmbeddingAdapter.java
│   │   │                   │   ├── parser
│   │   │                   │   │   └── JsoupBlogPostParser.java
│   │   │                   │   └── persistence
│   │   │                   │       ├── VectorConverter.java
│   │   │                   │       ├── entity
│   │   │                   │       │   ├── BlogPostChunkJpaEntity.java
│   │   │                   │       │   └── BlogPostJpaEntity.java
│   │   │                   │       ├── jpa
│   │   │                   │       │   ├── BlogPostChunkJpaRepository.java
│   │   │                   │       │   └── BlogPostJpaRepository.java
│   │   │                   │       ├── mapper
│   │   │                   │       │   ├── BlogPostChunkPersistenceMapper.java
│   │   │                   │       │   └── BlogPostPersistenceMapper.java
│   │   │                   │       └── repository
│   │   │                   │           ├── JpaBlogPostChunkRepository.java
│   │   │                   │           └── JpaBlogPostRepository.java
│   │   │                   └── job
│   │   │                       └── config
│   │   │                           └── BlogCatalogSyncJobConfig.java
│   │   └── resources
│   │       ├── application.yaml
│   │       └── db
│   │           └── sql
│   │               └── init.sql
```

<h2> 6. 레포지토리 </h2>

링크 : [https://github.com/AngryPig123/ai-agent-batch](https://github.com/AngryPig123/ai-agent-batch)

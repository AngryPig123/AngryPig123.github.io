---
title: 블로그 글 수집 배치 만들기 — Spring Batch로 AI Agent용 수집 임베딩 구조 구현 및 회고
description: 블로그 QA Agent가 사용할 데이터를 만들기 위해 Spring Batch 프로젝트를 생성하고 글 수집 구조를 설계한다.
date: 2026-03-13T16:00:00+09:00
updated: 2026-03-16T22:00:00+09:00
categories: [AI, AI Agent]
tags: [AI, AI Agent, Spring Batch, Batch, Ingestion, RAG, Embedding, Chunk]
---

<h2> 1. 작성하게 된 이유 </h2>

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
링크 : [https://github.com/AngryPig123/ai-agent-batch](https://github.com/AngryPig123/ai-agent-batch)


<h2> 6. 구현된 부분과 힘들었던 점, 앞으로 추가할 작업 </h2>

현재까지 구현된 기능은 다음과 같다.

- 페이지 단위 블로그 글 링크 수집
- 게시된 글 원문 저장
- 원문 데이터에 대한 `chunk` 전략 수립
- `chunk` 데이터 임베딩 및 `Vector DB` 저장

현재는 서버를 항상 실행해 두고 주기적으로 배치를 돌리기에는 아직 이른 단계라고 판단하여,
데이터 수집과 임베딩은 수동으로 실행하는 방식으로 진행하고 있다.

서비스 규모가 작고,
블로그 글의 변경도 자주 발생하지 않는 상태이기 때문에
지금 단계에서 스케줄 기반 배치를 구성하는 것은 과하다고 느꼈다.

추후 `Agent` 서비스의 규모가 커지고,
데이터가 지속적으로 추가되는 구조가 되면
그때 자연스럽게 스케줄 기반 배치로 전환할 예정이다.

구현 과정에서는 예상보다 고민할 부분이 많았고,
몇 가지 문제를 해결하면서 구조를 조금씩 수정하게 되었다.


<h3> 6.1 이해도 문제 </h3>

이번 프로젝트는 말 그대로 거의 맨땅에서 직접 `AI Agent`를 만들어보는 과정에 가까웠다.
그래서 큰 구조나 전체 흐름에 대한 감은 어느 정도 있었지만,
실제로 구현 단계에서 어떤 세부 사항까지 고려해야 하는지는 충분히 알지 못한 상태였다.

예를 들어 `chunk` 전략을 어떻게 가져갈지,
`Vector DB`를 어떤 방식으로 적용할지 같은 부분은
서비스를 만들면서 여러 자료를 찾아보고 이해하는 데 꽤 많은 시간이 필요했다.

개인적으로는 어느 정도 납득할 수 있는 수준까지 이해가 되어야
다음 구현으로 넘어갈 수 있는 성향이 있어서,
이 과정이 더 오래 걸렸던 것 같다.

다만 이런 시행착오 역시 학습 과정에서는 자연스러운 부분이라고 생각한다.
그래서 지금 서비스 구조를 만드는 데 꼭 필요한 내용이 아니라면
우선 기록만 남겨두고,
다음 프로젝트에서 다시 꺼내어 더 깊게 적용해보는 방향으로 정리하고 있다.


<h3> 6.2 블로그 글 변경 여부 판단 문제 </h3>

블로그 글을 수집하는 것 자체는 어렵지 않았지만,
`Jekyll Chirpy` 블로그 특성상 게시글의 내용이 변경되었는지를 판단하는 부분에서 고민이 있었다.

처음에는 `content hash`를 이용해서 변경 여부를 판단하는 방법을 생각했지만,
본문 전체를 기준으로 해시를 만들 경우 단순 오탈자 수정이나 공백 변경 같은 사소한 수정에도
모든 글이 변경된 것으로 처리된다는 문제가 있었다.

그래서 자동으로 변경을 감지하는 방식보다는,
작성자가 직접 업데이트 시점을 명시하도록 하는 방식이 더 적절하다고 판단했다.

이 과정에서 게시글의 헤더 메타 영역에
업데이트 시점을 저장할 수 있도록 필드를 추가하게 되었다.


<h3> 6.3 배치 전략에 대한 고민 </h3>

데이터 수집, 저장, 임베딩까지 필요한 배치 구성은 모두 완료된 상태이지만,
실제로 어떤 방식으로 배치를 운영할지에 대한 전략은 아직 정하지 못했다.

현재는 수동 실행으로 충분하지만,
앞으로는 다음과 같은 기준이 필요할 것 같다.

- 전체 재수집을 할 것인지
- 변경된 글만 다시 임베딩할 것인지
- 일정 주기로 자동 실행할 것인지
- 수동 실행을 유지할 것인지

서비스 규모가 커질수록 배치 전략도 함께 정리해야 할 부분이라고 생각한다.


<h3> 6.4 Vector DB 연동 시 JPA 설정 문제 </h3>

`Vector DB`와 연동되는 필드를 `JPA` 엔티티에 추가하는 과정에서,
추가 의존성이 필요하다는 사실을 몰라 예상보다 많은 시간을 사용했다.

특히 벡터 타입 필드를 사용하려면
기본 `JPA` 설정만으로는 동작하지 않았고,
`Vector DB`와 연동되는 라이브러리를 추가해야 정상적으로 매핑할 수 있었다.

단순한 설정 문제였지만,
처음 사용하는 구조이다 보니 원인을 찾는 데 시간이 오래 걸렸다.

---
title: 블로그 글 수집 배치 만들기 — Spring Batch로 AI Agent용 데이터 준비하기 작성중...
description: 블로그 QA Agent가 사용할 데이터를 만들기 위해 Spring Batch 프로젝트를 생성하고 글 수집 구조를 설계한다.
date: 2026-03-13T16:00:00+09:00
last_modified_at: 2026-03-13 17:57:00 +0900
categories: [AI, AI Agent]
tags: [AI, AI Agent, Spring Batch, Batch, Ingestion, RAG]
---

<meta name="last-modified-at" content="{{ page.last_modified_at | date_to_xmlschema }}">

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

<h3> 3. Spring batch 프로젝트 준비 </h3>

- 의존성 정보

```
JDK : Amazon Correto 17.0.18

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.5'

    // Source: https://mvnrepository.com/artifact/org.jsoup/jsoup
    implementation("org.jsoup:jsoup:1.21.2")

    implementation 'com.fasterxml.jackson.core:jackson-annotations:2.19.4'
    implementation 'com.fasterxml.jackson.core:jackson-core:2.19.4'
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.19.4'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jdk8:2.19.4'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310:2.19.4'
    implementation 'com.fasterxml.jackson.module:jackson-module-parameter-names:2.19.4'

    compileOnly 'org.projectlombok:lombok'
    runtimeOnly 'org.postgresql:postgresql'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter-test:3.0.5'
    testImplementation 'org.springframework.batch:spring-batch-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
```

- 패키지 구조

```
...
```


<h2>4. 수집 전략</h2>

수집 전략은 깃블로그 사이트(jekyll, chirpy 사용)에서 자동으로 생성되는 `/my-sitemap.xml` 파일에서 게시글 목록과 기본 정보를 먼저 수집하는 것이다.
이후 각 게시글 페이지에 접근하여 `Jsoup`으로 본문과 메타데이터를 파싱하고, 파싱한 원문은 `DB`에 저장한다.
추가로 원문을 임베딩하여 생성한 벡터 데이터는 `Vector DB`에 저장해, 이후 검색이나 `RAG`에 활용할 수 있도록 한다.

고려해야 할 사항. 이미 수집된 블로그 글은 수집되지 않아야함.
수정된 글은 다시 임배딩을 해야함.
수정된 글은 원문 테이블에 다시 update를 해야함.

`BlogPost` 엔티티 구조

```java
public class BlogPost extends BaseEntity<BlogPostId> {
    private String title;
    private String description;
    private String sourcePath;
    private String content;
    private String tagsJson;
    private String contentHash;
    private final Instant createdAt;
    private final Instant updatedAt;
}
```


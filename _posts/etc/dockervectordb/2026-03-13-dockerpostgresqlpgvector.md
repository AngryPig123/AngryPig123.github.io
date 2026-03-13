---
title: PostgreSQL + pgvector 환경 구성 — AI Agent를 위한 Vector DB 준비
description: AI Agent에서 사용할 임베딩 저장을 위해 PostgreSQL과 pgvector를 Docker로 구성한다.
date: 2026-03-13T15:40:00+09:00
categories: [AI, AI Agent]
tags: [AI, AI Agent, PostgreSQL, pgvector, VectorDB, Docker]
---


<h2> 1. PostgreSQL + pgvector 환경 구성 </h2>

`블로그 QA AI Agent 서비스` 에서는 블로그 원문, 메타데이터, 요약 정보는 `RDB`에 저장하고 임베딩 벡터는 `Vector DB`에 저장하는 구조로 구성하려고 한다.

초기 단계에서는 별도의 `Vector DB`를 두기보다는  
`PostgreSQL + pgvector` 확장을 사용해서  
하나의 DB에서 원문 데이터와 임베딩을 함께 관리하는 방식으로 진행한다.

`pgvector`는 `PostgreSQL`에서 벡터 타입을 사용할 수 있도록 해주는 확장으로  
임베딩 저장 및 유사도 검색을 수행할 수 있다.

또한 `Docker` 환경에서 `PostgreSQL`과 `pgvector`를 함께 구성해두면  
`Spring Batch`와 `Agent` 서비스에서 동일한 DB를 쉽게 사용할 수 있기 때문에  
`Docker` 기반으로 `DB` 환경을 먼저 구성한다.


- 구조

```
postgres-pgvector/
├─ docker-compose.yml
├─ Dockerfile
└─ initdb/
   └─ 01-init.sql
```

- `DockerFile`

```docker
FROM postgres:16

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       build-essential \
       git \
       ca-certificates \
       postgresql-server-dev-16 \
    && git clone --branch v0.8.2 https://github.com/pgvector/pgvector.git /tmp/pgvector \
    && cd /tmp/pgvector \
    && make \
    && make install \
    && rm -rf /tmp/pgvector \
    && apt-get remove -y git build-essential postgresql-server-dev-16 \
    && apt-get autoremove -y \
    && rm -rf /var/lib/apt/lists/*
```


- `docker-compose.yml`

```docker
services:
  postgres:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: postgres-pgvector
    restart: always
    environment:
      POSTGRES_USER: {POSTGRES_USER}
      POSTGRES_PASSWORD: {POSTGRES_PASSWORD}
      POSTGRES_DB: {POSTGRES_DB}
    ports:
      - "5432:5432"
    volumes:
      - {postgres-local-volume}:/var/lib/postgresql/data
      - {initdb-local-volume}:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

- `initdb/01-init.sql`

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE IF NOT EXISTS document_embedding (
    id BIGSERIAL PRIMARY KEY,
    source_id VARCHAR(100) NOT NULL,
    content TEXT NOT NULL,
    embedding VECTOR(768) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

- 설정 순서, 차례대로 실행하여 설치를 진행한다.
```docker
docker compose up -d --build
docker exec -it postgres-pgvector psql -U {USER_ID} -d {DATABASE}

appdb=# CREATE EXTENSION IF NOT EXISTS vector;

appdb=# SELECT * FROM pg_extension WHERE extname = 'vector';
  oid  | extname | extowner | extnamespace | extrelocatable | extversion | extconfig | extcondition
-------+---------+----------+--------------+----------------+------------+-----------+--------------
 16389 | vector  |       10 |         2200 | t              | 0.8.2      |           |
(1 row)

appdb=# CREATE TABLE IF NOT EXISTS document_embedding (
    id BIGSERIAL PRIMARY KEY,
    source_id VARCHAR(100) NOT NULL,
    content TEXT NOT NULL,
    embedding VECTOR(768) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

- 참고 사항 : 위에서 만든 `document_embedding`테이블에서 `embedding VECTOR(768)` 에서 `VECTOR`값은 사용하게 될 임베딩 모델에 따라 달라질 수 있다.

- bge-m3 → 1024
- nomic → 768
- mxbai → 1024
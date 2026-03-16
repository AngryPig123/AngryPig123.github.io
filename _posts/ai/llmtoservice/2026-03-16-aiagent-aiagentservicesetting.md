---
title: AI Agent 프로젝트에 데이터 베이스 연결하기 작성중 ....
description: 배치 프로그램에서 수집한 데이터에 접근할 수 있게 설정한다.
date: 2026-03-16T22:30:00+09:00
date: 2026-03-16T22:30:00+09:00
categories: [AI, AI Agent]
tags: [AI, LLM, AI Agent, Python DB 연결, Hexagonal Architecture, DDD]
---

<h2> 1. 배치 서비스에서 수집한 데이터에 접근하기 위해 AI Agent 프로젝트에 환경 설정을 추가한다. </h2>

`AI Agent` 프로젝트는 배치 서비스가 수집하고 적재한 원문 데이터와 임베딩 벡터를 조회할 수 있어야 한다.
이를 위해 `AI Agent`애플리케이션에는 배치 서브스와 동일한 디비 연결 설정이 필요하다.

원문과 메타데이터는 일반적인 관계형 처럼 조회하면 되지만, 임베딩 기반 검색을 수행하려면 벡터 연산이 가능한 환경도 갖추어야 한다.
이번 프로젝트에서는 따로 `VectorDB`를 구성하지 않고 `PostgreSQL`에 `pgvector` 익스텐션을 추가하여 간단히 하였다.
이로써 하나의 데이터 베이스로 겁색 기반 `RAG`구조를 구성할 수 있다.


<h2> 2. FastAPI 프로젝트에 필요 라이브러리 설치 </h2>


실무에서 사용할 수 있는 구성으로 조합을 구성해야 함으로 가장 사용 빈도가 높은 아래 조합을 추천 받아서 적용해본다.

- `FastAPI`
- `SQLAlchemy 2.x`
- `psycopg 3`
- `pgvector-python`

패키지 설치

```
pip install fastapi uvicorn sqlalchemy "psycopg[binary]" pgvector python-dotenv
```


<h2> 3. 환경 변수 추가 및 DB 연결 (1) </h2>

루트 프로젝트에 아래의 파일과 내용 추가.
.env

```
DATABASE_URL=postgresql+psycopg://postgres:postgres@localhost:5432/ai_agent
```

app/infrastructure/config/setting.py

```python
BASE_DIR = Path(__file__).resolve().parents[3]
ENV_FILE = BASE_DIR / ".env"

class Settings(BaseSettings):
    database_url: str

    model_config = SettingsConfigDict(
        env_file=BASE_DIR / ".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )


settings = Settings()
```


<h2> 4. 환경 변수 추가 및 DB 연결 (2) </h2>

app/infrastructure/db.py

```python
class Base(DeclarativeBase):
    pass

engine = create_engine(
    settings.database_url,
    pool_pre_ping=True
)

@event.listens_for(engine, "connect")
def connect(dbapi_connection, connection_record):
    register_vector(dbapi_connection)

SessionLocal = sessionmaker(
    bind=engine,
    autocommit=False,
    autoflush=False,
)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

```

<h2> 5. 도메인 포트 및 도메인 모델 정의 </h2>

`Agent`가 데이터 베이스를 모르도록 `out port`를 만든다.

app/domain/port/blog_chunk_query_port.py

```python
class BlogChunkQueryPort(ABC):
    @abstractmethod
    def search_similar(self, query_vector: list[float], limit: int = 5) -> list[BlogChunk]:
        raise NotImplementedError
```

도메인 모델

app/domain/model/blog_chunk.py

```python
@dataclass
class BlogPostChunk:
    id: int | None
    blog_post_id: int
    title: str
    chunk_index: int
    score: float | None = None
```


<h2> 6. ORM 엔티티 </h2>

app/infrastructure/persistence/entity/blog_chunk_entity.py

```python
class BlogChunkEntity(Base):
    __tablename__ = "blog_post_chunk"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    blog_post_id: Mapped[int] = mapped_column(Integer, nullable=False, index=True)
    title: Mapped[str] = mapped_column(String(255), nullable=False)
    chunk_index: Mapped[int] = mapped_column(Integer, nullable=False)
    embedding: Mapped[list[float]] = mapped_column(VECTOR(768), nullable=False)
```

<h2> 7. 영속성 구현체 </h2>


```python
class BlogChunkRepository(BlogChunkQueryPort):

    def __init__(self, db: Session):
        self.db = db

    def search_similar(self, query_vector: list[float], limit: int = 5) -> list[BlogPostChunk]:
        stmt = (
            select(BlogChunkEntity)
            .order_by(BlogChunkEntity.embedding.cosine_distance(query_vector))
            .limit(limit)
        )
        rows = self.db.scalars(stmt).all()
        return [
            BlogPostChunk(
                id=row.id,
                blog_post_id=row.blog_post_id,
                title=row.title,
                chunk_index=row.chunk_index,
            )
            for row in rows
        ]

```

<h2> 8. 저장소 </h2>

링크 : [https://github.com/AngryPig123/ai-agent](https://github.com/AngryPig123/ai-agent)
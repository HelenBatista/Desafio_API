# Desafio_API
## Desenvolvendo sua Primeira API com FastAPI, Python e Docker

## DESAFIO

### Estrutura do Projeto:

        ├── Dockerfile
        ├── docker-compose.yml
        ├── app
        │   ├── __init__.py
        │   ├── main.py
        │   ├── models.py
        │   ├── schemas.py
        │   ├── crud.py
        │   ├── database.py
        │   ├── exceptions.py
        │   └── pagination.py

### Dockerfile
Crie um arquivo Dockerfile com o seguinte conteúdo:
        FROM python:3.10

        WORKDIR /app

        COPY requirements.txt requirements.txt
        RUN pip install --no-cache-dir -r requirements.txt

        COPY . .

        CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

### docker-compose.yml
Crie um arquivo docker-compose.yml com o seguinte conteúdo:
        version: '3.8'

        services:
          web:
            build: .
            ports:
              - "8000:8000"
            volumes:
              - .:/app
            depends_on:
              - db

          db:
            image: postgres:13
            environment:
              POSTGRES_USER: user
              POSTGRES_PASSWORD: password
              POSTGRES_DB: fastapi_db
            ports:
              - "5432:5432"
### Requisitos
Crie um arquivo requirements.txt com o seguinte conteúdo:

        fastapi
        uvicorn
        sqlalchemy
        asyncpg
        alembic
        pydantic
        fastapi-pagination

### Banco de Dados e Modelos
Crie app/database.py para configurar a conexão com o banco de dados:

        from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
        from sqlalchemy.orm import sessionmaker, declarative_base

        DATABASE_URL = "postgresql+asyncpg://user:password@db/fastapi_db"

        engine = create_async_engine(DATABASE_URL, echo=True)
        SessionLocal = sessionmaker(bind=engine, class_=AsyncSession, expire_on_commit=False)
        Base = declarative_base()

        async def get_db():
            async with SessionLocal() as session:
                yield session
                
Crie app/models.py para definir os modelos de banco de dados:

    python
        from sqlalchemy import Column, Integer, String
        from sqlalchemy.orm import relationship
        from .database import Base

        class Atleta(Base):
            __tablename__ = "atletas"

            id = Column(Integer, primary_key=True, index=True)
            nome = Column(String, index=True)
            cpf = Column(String, unique=True, index=True)
            centro_treinamento = Column(String, index=True)
            categoria = Column(String, index=True)
            
Crie app/schemas.py para definir os schemas de dados:

        from pydantic import BaseModel

        class AtletaBase(BaseModel):
            nome: str
            cpf: str

        class AtletaCreate(AtletaBase):
            centro_treinamento: str
            categoria: str

        class Atleta(AtletaBase):
            id: int
            centro_treinamento: str
            categoria: str

            class Config:
                orm_mode = True
### Operações CRUD
Crie app/crud.py para definir as operações de CRUD:

        from sqlalchemy.ext.asyncio import AsyncSession
        from sqlalchemy.future import select
        from sqlalchemy.exc import IntegrityError
        from .models import Atleta
        from .schemas import AtletaCreate

        async def get_atletas(db: AsyncSession, skip: int = 0, limit: int = 10):
            result = await db.execute(select(Atleta).offset(skip).limit(limit))
            return result.scalars().all()

        async def get_atleta_by_id(db: AsyncSession, atleta_id: int):
            result = await db.execute(select(Atleta).where(Atleta.id == atleta_id))
            return result.scalar()

        async def get_atleta_by_cpf(db: AsyncSession, cpf: str):
            result = await db.execute(select(Atleta).where(Atleta.cpf == cpf))
            return result.scalar()

        async def create_atleta(db: AsyncSession, atleta: AtletaCreate):
            db_atleta = Atleta(**atleta.dict())
            db.add(db_atleta)
            try:
                await db.commit()
                await db.refresh(db_atleta)
            except IntegrityError:
                await db.rollback()
                raise IntegrityError("Já existe um atleta cadastrado com o cpf: {0}".format(atleta.cpf), None, None)
            return db_atleta

### Manipulação de Exceções
Crie app/exceptions.py para manipular as exceções:

python

        from fastapi import HTTPException, Request
        from fastapi.responses import JSONResponse
        from sqlalchemy.exc import IntegrityError

        def integrity_error_handler(request: Request, exc: IntegrityError):
            return JSONResponse(
                status_code=303,
                content={"detail": str(exc)},
            )

### Paginação
Crie app/pagination.py para configurar a paginação:

        from fastapi_pagination import Page, add_pagination, paginate
        from fastapi_pagination.ext.async_sqlalchemy import paginate as paginate_sqlalchemy
        from sqlalchemy.ext.asyncio import AsyncSession
        from .models import Atleta

        async def get_paginated_atletas(db: AsyncSession):
            return await paginate_sqlalchemy(db.execute(select(Atleta)))
### Aplicação Principal
Crie app/main.py para a aplicação principal:

        from fastapi import FastAPI, Depends, HTTPException
        from sqlalchemy.ext.asyncio import AsyncSession
        from sqlalchemy.exc import IntegrityError
        from fastapi_pagination import Page, add_pagination, paginate
        from . import crud, models, schemas, database, exceptions, pagination

        app = FastAPI()

        app.add_exception_handler(IntegrityError, exceptions.integrity_error_handler)

        @app.post("/atletas/", response_model=schemas.Atleta)
        async def create_atleta(atleta: schemas.AtletaCreate, db: AsyncSession = Depends(database.get_db)):
            return await crud.create_atleta(db, atleta)

        @app.get("/atletas/", response_model=Page[schemas.Atleta])
        async def read_atletas(skip: int = 0, limit: int = 10, db: AsyncSession = Depends(database.get_db)):
            return await pagination.get_paginated_atletas(db)

        @app.get("/atletas/{atleta_id}", response_model=schemas.Atleta)
        async def read_atleta(atleta_id: int, db: AsyncSession = Depends(database.get_db)):
            db_atleta = await crud.get_atleta_by_id(db, atleta_id)
            if db_atleta is None:
                raise HTTPException(status_code=404, detail="Atleta not found")
            return db_atleta

        add_pagination(app)

### Inicializar Banco de Dados
Você precisará inicializar o banco de dados. Crie um arquivo app/alembic.ini e configure o Alembic para gerenciar as migrações do banco de dados.

Para inicializar o banco de dados, use:

        docker-compose run web alembic init app/alembic

Configure as revisões e execute as migrações com:

        docker-compose run web alembic revision --autogenerate -m "Initial migration"
    docker-compose run web alembic upgrade head
    
### Iniciar o Serviço
Finalmente, inicie os serviços com Docker Compose:

        docker-compose up
### Teste os Endpoints
Agora você pode testar os endpoints usando uma ferramenta como Postman ou curl.

Criação de Atleta:

        curl -X POST "http://localhost:8000/atletas/" -H "Content-Type: application/json" -d '{"nome": "João", "cpf": "12345678900", "centro_treinamento": "Centro A", "categoria":             "Profissional"}'
        
### Listagem de Atletas com Paginação:

        curl -X GET "http://localhost:8000/atletas/?skip=0&limit=10"
        
### Leitura de Atleta por ID:

        curl -X GET "http://localhost:8000/atletas/1",
        
Isso conclui a configuração de uma API assíncrona com FastAPI, integração de banco de dados com SQLAlchemy, paginação com fastapi-pagination e manipulação de exceções.

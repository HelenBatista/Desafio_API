# Desafio_API
## Desenvolvendo sua Primeira API com FastAPI, Python e Docker

## DESAFIO

### Instalação das Dependências

    pip install fastapi sqlalchemy fastapi-pagination uvicorn 

# Criação do Modelo de Dados

    from sqlalchemy import Column, Integer, String, ForeignKey
    from sqlalchemy.orm import relationship
    from database import Base

    class Atleta(Base):
        __tablename__ = "atletas"

        id = Column(Integer, primary_key=True, autoincrement=True)
        nome = Column(String(255), nullable=False)
        cpf = Column(String(11), unique=True, nullable=False)
        centro_treinamento = Column(String(255))
        categoria = Column(String(255))

        # Relacionamento com tabela de treinos (opcional)
        # treinos = relationship("Treino", backref="atleta")
### Configuração do Banco de Dados:

    from` sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker
    from database import Base

    DATABASE_URL = "postgresql://user:password@host:port/database"

    engine = create_engine(DATABASE_URL)
    Base.metadata.create_all(engine)

    SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
### Implementação da API Assíncrona

    ``
    from fastapi import FastAPI, Depends, HTTPException
    from fastapi_pagination import Page, add_pagination
    from sqlalchemy.orm import Session
    from models import Atleta
    from database import SessionLocal

    app = FastAPI()

    @app.on_event("startup")
    async def startup_event():
        global session
        session = SessionLocal()

    @app.on_event("shutdown")
    async def shutdown_event():
        global session
        session.close()

    def get_db():
        db = SessionLocal()
        try:
            yield db
        finally:
            db.close()
### Endpoint para buscar todos os atletas
    @app.get("/atletas", response_model=Page[Atleta])
    async def get_all_atletas(db: Session = Depends(get_db), limit: int | None = None, offset: int | None = None):
        atletas = db.query(Atleta).order_by(Atleta.id).limit(limit).offset(offset).all()
        return Page(atletas, len(atletas))
### Endpoint para buscar atletas por nome e CPF

    @app.get("/atletas/search", response_model=List[Atleta])
    async def get_atleta_by_nome_cpf(nome: str | None = None, cpf: str | None = None, db: Session = Depends(get_db)):
        query = db.query(Atleta)

        if nome:
            query = query.filter(Atleta.nome.ilike(f"%{nome}%"))

        if cpf:
            query = query.filter(Atleta.cpf == cpf)

        atletas = query.all()

        if not atletas:
            raise HTTPException(status_code=404, detail="Atleta não encontrado")

        return atletas
### Endpoint para cadastrar um novo atleta
    @app.post("/atletas")
    async def create_atleta(atleta: Atleta, db: Session = Depends(get_db)):
        db.add(atleta)
        db.commit()
        db.refresh(atleta)
        return atleta

    add_pagination(app)
### Obter sessão do banco de dados

    def get_db(session: Session = Depends(SessionLocal)):
    return session
    
### Endpoint para buscar todos os atletas

    @app.get("/atletas", response_model=Page[Atleta])
    async def get_all_atletas(db: Session = Depends(get_db), limit: int | None = None, offset: int | None = None):
    atletas = db.query(Atleta).order_by(Atleta.id).limit(limit).offset(offset).all()
    return Page(atletas, len(atletas))
### Endpoint para buscar atletas por nome e CPF

    @app.get("/atletas", response_model=List[Atleta])
    async def get_atleta_by_nome_cpf(nome: str | None = None, cpf: str | None = None, db: Session = Depends(get_db)):
    query = db.query(Atleta)

    if nome:
        query = query.filter(Atleta.nome.ilike(f"%{nome}%"))

    if cpf:
        query = query.filter(Atleta.cpf == cpf)

    atletas = query.all()

    if not atletas:
        raise HTTPException(status_code=404, detail="Atleta não encontrado")

    return atletas

### Endpoint para cadastrar um novo atleta

    @app.post("/atletas")
    async def

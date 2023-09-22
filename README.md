# Dojo Python/FastAPI 

## Configurando ambiente

Verifique se o Python está instalado corretamente em seu computador:

```
python --version
```

Caso o comando acima não funcione:

```
python3 --version
```


Crie e ative uma venv:

- No Windows:
```
python -m venv venv
venv/Scripts/activate
```

- No Linux/macOS:
```
python3 -m venv venv
source .venv/bin/activate
```

Instale o FastAPI:

```
pip install fastapi uvicorn[standard] sqlalchemy
```

## Criando o primeiro Olá Mundo!

Crie um arquivo `main.py` e cole o seguinte código:

```
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
async def ola_mundo():
    return {"message": "Ola Mundo"}
```

Para rodar, use o comando:

```
uvicorn main:app --reload
```

No comando acima temos as seguintes partes:

- `main`: se refere ao arquivo `main.py`.
- `app`: se refere ao objeto criando no arquivo `main.py` com a linha `app = FastAPI()`.
- `--reload`: faz o servidor reiniciar depois de alterações no código. Usado apenas em ambiente de desenvolvimento.

## Criando nossa API

Nossa API será uma versão simplificada de um CRUD de Pokemon.

### Configurando conexão com o banco de dados

Crie um arquivo `config.py` e use o seguinte código:

```
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

SQLALCHEMY_DATABASE_URL = "sqlite:///db.sqlite3"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False}
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

Esse código usa o SQLAlchemy para configurar a conexão com um banco de dados sqlite.

### Criando uma camada de modelo

A camada de modelos da aplicação contém as classes que representam as tabelas no banco de dados. 

Em um arquivo chamado `model.py` criamos o nosso modelo:

```
from sqlalchemy import Column, Integer, String
from config import Base

class Pokemon(Base):
    __tablename__ = 'pokemon'
    id: int = Column(Integer, primary_key=True, index=True)
    nome: str = Column(String(50), nullable=False)
    tipo_1: str = Column(String(20), nullable=False)
    tipo_2: str = Column(String(20), nullable=True)
```

### Criando uma camada de repositório

Existe um padrão de projeto chamado Repository Pattern, onde o acesso aos dados é isolado em uma única camada. 

Em um arquivo chamado `repository.py` coloque o seguinte código:

```
from sqlalchemy.orm import Session
from model import Pokemon

class PokemonRepository:
    @staticmethod
    def find_all(db: Session) -> list[Pokemon]:
        return db.query(Pokemon).all()
    
    @staticmethod
    def save(db: Session, pokemon: Pokemon) -> Pokemon:
        if pokemon.id:
            db.merge(pokemon)
        else:
            db.add(pokemon)
        db.commit()
        return pokemon
    
    @staticmethod
    def find_by_id(db: Session, id:int) -> Pokemon:
        return db.query(Pokemon).filter(Pokemon.id == id).first()
    
    @staticmethod
    def exists_by_id(db: Session, id: int) -> bool:
        return db.query(Pokemon).filter(Pokemon.id == id).first() is not None
    
    @staticmethod
    def delete_by_id(db: Session, id: int) -> None:
        pokemon = db.query(Pokemon).filter(Pokemon.id == id).first()
        if pokemon is not None:
            db.delete(pokemon)
            db.commit()
```

### Criando uma camada de Schema

Essa camada contém as classes responsáveis por representar os dados que serão recebidos e retornados no corpo das requisições e respostas HTTP.

Para isso, é usada a biblioteca [Pydantic](https://docs.pydantic.dev/latest/). Essa biblioteca é bastante usada em FastAPI no processo de validação de dados, e tipagem dos dados que são recebidos e retornados por requisições e respostas HTTP.

Em um arquivo `schema.py` coloque o seguinte código:

```
from pydantic import BaseModel

class PokemonBase(BaseModel):
    nome: str
    tipo_1: str
    tipo_2: str

class PokemonRequest(PokemonBase):
    ...

class PokemonResponse(PokemonBase):
    id:  int

    class Config:
        from_attributes = True
```

### Criando Operações de CRUD

Em um arquivo `controller.py` coloque o seguinte código:

```
from fastapi import APIRouter, HTTPException, Response, status, Depends

from schema import PokemonRequest, PokemonResponse
from model import Pokemon
from repository import PokemonRepository
from config import get_db
from sqlalchemy.orm import Session

pokemon = APIRouter(
    prefix='/pokemon',
)
```

Não se esqueça de adicionar o seguinte trecho de código em `main.py`, para incluir as rotas de pokemon da nossa API:

```
app.include_router(pokemon)
```

#### Endpoint de Criação de um pokemon

Método HTTP do tipo POST. Endpoint usado para criar um pokemon, adicione no arquivo `controller.py`:

```
@pokemon.post('/', 
              status_code= status.HTTP_201_CREATED, 
              response_model=PokemonResponse)
def create(request: PokemonRequest, db: Session = Depends(get_db)):
    '''Cria e salva um pokemon'''
    pokemon = PokemonRepository.save(db, Pokemon(**request.model_dump()))
    return PokemonResponse.model_validate(pokemon)
``` 

#### Endpoint de Listar todos os Pokemons

Método HTTP do tipo GET. Endpoint usado para trazer uma listagem de todos os pokemons, adicione no arquivo `controller.py`:

```
@pokemon.get('/',
              status_code = status.HTTP_200_OK,
              response_model=list[PokemonResponse])
def find_all(db: Session = Depends(get_db)):
    '''Lista todos os pokemons'''
    pokemons = PokemonRepository.find_all(db)
    return [PokemonResponse.model_validate(pokemon) for pokemon in pokemons]
```

#### Endpoint para consultar um pokemon por id

Método HTTP do tipo GET. Endpoint usado para ler um único pokemon pelo seu id, adicione no arquivo `controller.py`:

```
@pokemon.get('/{id}', 
             status_code= status.HTTP_200_OK,
             response_model= PokemonResponse)
def find_by_id(id: int, db: Session = Depends(get_db)):
    '''Lista um pokemon por id'''
    pokemon = PokemonRepository.find_by_id(db, id)
    return PokemonResponse.model_validate(pokemon)
```

#### Endpoint para editar um pokemon por id

Método HTTP do tipo PUT. Endpoint usado para editar um único pokemon pelo seu id, adicione no arquivo `controller.py`:

```
@pokemon.put('/{id}',
             status_code= status.HTTP_200_OK,
             response_model= PokemonResponse)
def update(id: int, request: PokemonRequest, db: Session = Depends(get_db)):
    '''Atualiza um pokemon'''
    if not PokemonRepository.exists_by_id(db, id):
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, detail="Pokemon não encontrado"
        )
    pokemon = PokemonRepository.save(db, Pokemon(id=id, **request.model_dump()))
    return PokemonResponse.model_validate(pokemon)
```

#### Endpoint para excluir um pokemon por id

Método HTTP do tipo DELETE. Endpoint usado para excluir um único pokemon pelo seu id, adicione no arquivo `controller.py`:

```
@pokemon.delete('/{id}',
                status_code= status.HTTP_200_OK,
                response_model= PokemonResponse)
def delete(id: int, db: Session = Depends(get_db)):
    if not PokemonRepository.exists_by_id(db, id):
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND, detail="Pokemon não encontrado"
        )
    PokemonRepository.delete_by_id(db, id)
    return Response(status_code=status.HTTP_200_OK)
```

E voilá! Nosso CRUD está pronto.

### Testando a API

Utilize algum programa como *Postman* ou *Insomnia*. Você pode usar os seguintes JSONs:

```
{
    "nome": "Bulbasaur",
    "tipo_1": "Grass",
    "tipo_2": "Poison"
}
```

```
{
    "nome": "Charmander",
    "tipo_1": "Fire",
    "tipo_2": ""
}
```

```
{
    "nome": "Squirtle",
    "tipo_1": "Water",
    "tipo_2": ""
}
```

# Bitácora de Requerimientos y Auditoría IA — Sesión 22
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Modelado Relacional (1:1, 1:N, N:M) con ORMs  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda Palomo  
**Matrícula:** [23531112]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Definición de tablas relacionales en código  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Sin ORM, una relación 1:N obliga a escribir el `CREATE TABLE` con `FOREIGN KEY`, hacer el `INSERT` del libro con el `autor_id` correcto y armar el `JOIN` a mano para traer al autor con sus libros. Después hay que convertir las filas planas en el JSON anidado (`autor` → `libros`). También hay que comprobar manualmente que el autor exista antes de insertar el libro, porque si no queda un libro huérfano.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+), SQLAlchemy 2.0 con SQLite (`biblioteca.db`), Pydantic v2, ejecución en Colab |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | Modelos `Autor` (id, nombre) y `Libro` (id, titulo, anio, autor_id con `ForeignKey("autores.id")`); relación 1:N con `relationship` y `back_populates`; `POST /api/autores` → 201; `POST /api/autores/{id}/libros` → 201 (404 si el autor no existe); `GET /api/autores/{id}` → 200 con libros anidados; `GET /api/libros` → 200 con `{"libros": [...]}` |
| **Output (O)** | Solo código modular en `app.py`, esquemas Pydantic y códigos HTTP semánticos |

**Diagrama entidad-relación (entregable):**
```mermaid
erDiagram
    AUTORES ||--o{ LIBROS : "escribe"
    AUTORES {
        int id PK
        string nombre
    }
    LIBROS {
        int id PK
        string titulo
        int anio
        int autor_id FK
    }
```

### 2.2 Prompt Estructurado
```text
CONTEXTO: API REST con FastAPI 0.110+ (Python 3.10+), SQLAlchemy 2.0 y SQLite (archivo biblioteca.db). Pydantic v2 para los esquemas.

ROL: Actúa como arquitecto de software senior, especialista en APIs RESTful y seguridad web.

ESPECIFICACIÓN:
1. Modelo Autor: id (PK), nombre (no nulo). Modelo Libro: id (PK), titulo, anio (no nulos) y autor_id (ForeignKey a autores.id, no nulo).
2. Relación 1:N declarada con relationship() y back_populates en ambos lados.
3. Crea las tablas al iniciar con Base.metadata.create_all.
4. POST /api/autores recibe {"nombre": str} y responde 201 con {"id", "nombre"}.
5. POST /api/autores/{id}/libros recibe {"titulo": str, "anio": int} y responde 201 con {"id", "titulo", "anio", "autor_id"}; si el autor no existe, responde 404.
6. GET /api/autores/{id} responde 200 con el autor y sus libros anidados [{"id", "titulo"}]; 404 si no existe.
7. GET /api/libros responde 200 con {"libros": [...]}.
8. Usa una dependencia get_db que abra y cierre la sesión siempre.

OUTPUT: Solo código en un único archivo app.py, con esquemas Pydantic y comentarios breves.
```

---

## 3. Código Devuelto por el Modelo
```python
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from typing import List
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey
from sqlalchemy.orm import declarative_base, sessionmaker, relationship, Session

# --- Configuración de la Base de Datos SQLite ---
DATABASE_URL = "sqlite:///./biblioteca.db"
engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# --- Modelos SQLAlchemy ---
class Autor(Base):
    __tablename__ = "autores"

    id = Column(Integer, primary_key=True, index=True)
    nombre = Column(String, nullable=False)

    # Relación uno a muchos (1:N)
    libros = relationship("Libro", back_populates="autor", cascade="all, delete-orphan")

class Libro(Base):
    __tablename__ = "libros"

    id = Column(Integer, primary_key=True, index=True)
    titulo = Column(String, nullable=False)
    anio = Column(Integer, nullable=False)
    autor_id = Column(Integer, ForeignKey("autores.id"), nullable=False)

    # Relación inversa
    autor = relationship("Autor", back_populates="libros")

# Crear las tablas en la base de datos
Base.metadata.create_all(bind=engine)

# --- Esquemas de Pydantic ---
class LibroBase(BaseModel):
    titulo: str
    anio: int

class LibroCreate(LibroBase):
    pass

class LibroResponse(LibroBase):
    id: int
    autor_id: int

    class Config:
        from_attributes = True

class AutorBase(BaseModel):
    nombre: str

class AutorCreate(AutorBase):
    pass

# Esquema para la respuesta de un libro simplificado dentro de Autor
class LibroEnAutor(BaseModel):
    id: int
    titulo: str

    class Config:
        from_attributes = True

class AutorResponse(AutorBase):
    id: int
    libros: List[LibroEnAutor] = []

    class Config:
        from_attributes = True

# --- Dependencia de Sesión de la Base de Datos ---
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# --- Aplicación FastAPI ---
app = FastAPI(title="Relación autor -> libros con SQLAlchemy")

@app.post("/api/autores", response_model=AutorResponse, status_code=201)
def crear_autor(autor: AutorCreate, db: Session = Depends(get_db)):
    db_autor = Autor(nombre=autor.nombre)
    db.add(db_autor)
    db.commit()
    db.refresh(db_autor)
    return db_autor

@app.post("/api/autores/{autor_id}/libros", response_model=LibroResponse, status_code=201)
def crear_libro_autor(autor_id: int, libro: LibroCreate, db: Session = Depends(get_db)):
    autor_existente = db.query(Autor).filter(Autor.id == autor_id).first()
    if not autor_existente:
        raise HTTPException(status_code=404, detail="Autor no encontrado")

    db_libro = Libro(**libro.model_dump(), autor_id=autor_id)
    db.add(db_libro)
    db.commit()
    db.refresh(db_libro)
    return db_libro

@app.get("/api/autores/{autor_id}", response_model=AutorResponse)
def obtener_autor(autor_id: int, db: Session = Depends(get_db)):
    db_autor = db.query(Autor).filter(Autor.id == autor_id).first()
    if not db_autor:
        raise HTTPException(status_code=404, detail="Autor no encontrado")
    return db_autor

@app.get("/api/libros")
def listar_libros(db: Session = Depends(get_db)):
    libros = db.query(Libro).all()
    # El validador espera una estructura {"libros": [...]}
    return {"libros": [LibroResponse.model_validate(l).model_dump() for l in libros]}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Verificación HTTP del contrato (4 casos) y revisión de que la relación ORM esté declarada. Resultado: **4/4 casos correctos** y **1/1 comprobación correcta**. Las pruebas de integridad referencial y cascadas se revisaron solo leyendo el código, no se ejecutaron.

| Caso | Resultado |
| :--- | :--- |
| POST `/api/autores` → 201 con `id: 1` | ✅ |
| POST `/api/autores/1/libros` → 201 con `autor_id: 1` | ✅ |
| GET `/api/autores/1` → 200 con libros anidados | ✅ |
| GET `/api/libros` → 200 con `{"libros": [...]}` | ✅ |
| `relationship(` / `ForeignKey(` presentes en `app.py` | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`). *El código contiene `404` para autor inexistente, pero ningún caso lo ejercitó; `400` y `422` tampoco se probaron.*
* [ ] No existen respuestas HTTP `500 Internal Server Error` no capturadas. *No se probó: no hay manejo de errores de base de datos.*
* [ ] Encabezados `Content-Type: application/json` correctamente configurados. *FastAPI lo envía por defecto, pero no lo verifiqué de forma explícita.*

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> El verificador solo cubre el camino feliz, y el código pasa en verde aunque tiene omisiones:
>
> 1. **Llaves foráneas no forzadas por SQLite.** SQLite ignora las `FOREIGN KEY` si no se activa `PRAGMA foreign_keys=ON` en cada conexión. Hoy la integridad la protege solo la consulta manual del autor en el endpoint; a nivel de base de datos nada impide un `autor_id` huérfano.
> 2. **Cascada sin probar.** `cascade="all, delete-orphan"` solo actúa a nivel ORM y no existe un endpoint `DELETE`, así que el borrado en cascada nunca se ejercita.
> 3. **Sin manejo de errores de la base.** Un fallo en `commit()` (`SQLAlchemyError`) sube sin capturar y el cliente recibe un `500`. Lo correcto sería un `503` o `409` con mensaje controlado.
> 4. **Validación débil.** `nombre` y `titulo` aceptan cadenas vacías y `anio` acepta cualquier entero (negativo o absurdo).
> 5. **Sintaxis obsoleta.** `class Config` está deprecada en Pydantic v2; debe usarse `model_config = ConfigDict(from_attributes=True)`.
> 6. **Alcance parcial del tema.** Solo se implementó 1:N; las relaciones 1:1 y N:M del tema no se modelaron.
> 7. **Lo que sí está bien:** el `404` para autor inexistente está implementado y las consultas pasan por el ORM, que parametriza el SQL.
>
> Mejora propuesta (no ejecutada):
> ```python
> from sqlalchemy import event
> from pydantic import BaseModel, ConfigDict, Field
>
> @event.listens_for(engine, "connect")
> def activar_fk(dbapi_con, _):
>     dbapi_con.execute("PRAGMA foreign_keys=ON")
>
> class AutorBase(BaseModel):
>     nombre: str = Field(min_length=1, max_length=200)
>
> class LibroBase(BaseModel):
>     titulo: str = Field(min_length=1, max_length=200)
>     anio: int = Field(ge=0, le=2100)
>
> class LibroResponse(LibroBase):
>     model_config = ConfigDict(from_attributes=True)
>     id: int
>     autor_id: int
> ```

---
**Firma digital:** FernandaPalomo

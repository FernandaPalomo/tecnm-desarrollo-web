# Bitácora de Requerimientos y Auditoría IA — Sesión 23
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Operaciones CRUD Transaccionales y Async Sessions  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda Palomo  
**Matrícula:** [23531112]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Operaciones de inserción y consulta manual  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Hacer un CRUD completo a mano implica repetir en cada endpoint el mismo ciclo: abrir la sesión, consultar, verificar si el registro existe (para devolver 404), modificar, hacer `commit` y cerrar. Con async hay que recordar el `await` en cada operación (`execute`, `commit`, `refresh`, `delete`); olvidar uno devuelve una corrutina en lugar de datos. También hay que cuidar el `expire_on_commit=False`, porque con sesiones asíncronas el objeto expira tras el `commit` y acceder a sus atributos sin `refresh` provoca errores. Por último, DELETE debe responder 204 sin cuerpo, y eso cambia cómo se escribe el `return`.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+), SQLAlchemy 2.0 asíncrono con `sqlite+aiosqlite`, archivo `biblioteca.db`, Pydantic v2, ejecución en Colab |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | Repositorio asíncrono con `AsyncSession` y `commit`; `POST /api/libros` → 201; `GET /api/libros` → 200 con `{"libros": [...]}`; `GET /api/libros/{id}` → 200; `PUT /api/libros/{id}` → 200; `DELETE /api/libros/{id}` → 204; id inexistente → 404; tabla creada al iniciar la aplicación |
| **Output (O)** | Solo código modular en `app.py`, esquemas Pydantic y códigos HTTP semánticos |

**Entregable: diferencia entre `def` y `async def`**

| | `def` + sesión síncrona | `async def` + `AsyncSession` |
| :--- | :--- | :--- |
| **Dónde se ejecuta** | FastAPI lo manda a un pool de hilos (threadpool) | Directamente en el event loop |
| **Mientras espera la base** | El hilo queda ocupado y bloqueado | `await` cede el control y el loop atiende otras peticiones |
| **Límite de concurrencia** | El tamaño del pool de hilos | Muchas peticiones en espera con un solo hilo |
| **Riesgo principal** | Agotar el pool de hilos bajo carga | Una llamada bloqueante dentro de `async def` congela todo el servidor |

Regla del curso: motor async, sesión async, endpoint async. Mezclarlos (por ejemplo, una sesión síncrona dentro de `async def`) bloquea el event loop.

### 2.2 Prompt Estructurado
```text
CONTEXTO: API REST con FastAPI 0.110+ (Python 3.10+), SQLAlchemy 2.0 asíncrono y SQLite mediante aiosqlite (archivo biblioteca.db). Pydantic v2 para los esquemas.

ROL: Actúa como arquitecto de software senior, especialista en APIs RESTful y seguridad web.

ESPECIFICACIÓN:
1. Usa create_async_engine con sqlite+aiosqlite y async_sessionmaker con AsyncSession (expire_on_commit=False).
2. Modelo Libro: id (PK autoincremental), titulo (no nulo), anio (no nulo).
3. La tabla se crea al iniciar la aplicación, sin pasos manuales.
4. Todos los endpoints son async def y usan await con la sesión.
5. POST /api/libros recibe {"titulo": str, "anio": int} y responde 201 con {"id", "titulo", "anio"}.
6. GET /api/libros responde 200 con {"libros": [...]}.
7. GET /api/libros/{id} responde 200 o 404.
8. PUT /api/libros/{id} actualiza el libro y responde 200 o 404.
9. DELETE /api/libros/{id} responde 204 sin cuerpo o 404.
10. Dependencia get_db que abra y cierre la sesión siempre.

OUTPUT: Solo código en un único archivo app.py, con esquemas Pydantic y comentarios breves.
```

---

## 3. Código Devuelto por el Modelo
```python
import asyncio
import os
from fastapi import FastAPI, HTTPException, status, Depends
from pydantic import BaseModel
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import select

DATABASE_URL = "sqlite+aiosqlite:///./biblioteca.db"

engine = create_async_engine(
    DATABASE_URL,
    echo=False,
    connect_args={"check_same_thread": False}
)
async_session = async_sessionmaker(engine, expire_on_commit=False, class_=AsyncSession)

class Base(DeclarativeBase):
    pass

class LibroDB(Base):
    __tablename__ = "libros"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    titulo: Mapped[str] = mapped_column(nullable=False)
    anio: Mapped[int] = mapped_column(nullable=False)

class LibroCreate(BaseModel):
    titulo: str
    anio: int

class LibroResponse(BaseModel):
    id: int
    titulo: str
    anio: int

    class Config:
        from_attributes = True

class ListaLibrosResponse(BaseModel):
    libros: list[LibroResponse]

app = FastAPI(title="CRUD Asíncrono de Libros")

# Esta función garantiza la creación dinámica de la tabla si no existe al recibir cada petición
async def asegurar_tablas():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

async def get_db():
    await asegurar_tablas()
    async with async_session() as session:
        try:
            yield session
        finally:
            await session.close()

@app.post("/api/libros", response_model=LibroResponse, status_code=status.HTTP_201_CREATED)
async def crear_libro(libro: LibroCreate, db: AsyncSession = Depends(get_db)):
    nuevo_libro = LibroDB(titulo=libro.titulo, anio=libro.anio)
    db.add(nuevo_libro)
    await db.commit()
    await db.refresh(nuevo_libro)
    return nuevo_libro

@app.get("/api/libros", response_model=ListaLibrosResponse)
async def listar_libros(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(LibroDB))
    libros = result.scalars().all()
    return {"libros": list(libros)}

@app.get("/api/libros/{libro_id}", response_model=LibroResponse)
async def obtener_libro(libro_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(LibroDB).where(LibroDB.id == libro_id))
    libro = result.scalar_one_or_none()
    if libro is None:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Libro no encontrado")
    return libro

@app.put("/api/libros/{libro_id}", response_model=LibroResponse)
async def actualizar_libro(libro_id: int, libro_data: LibroCreate, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(LibroDB).where(LibroDB.id == libro_id))
    libro = result.scalar_one_or_none()
    if libro is None:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Libro no encontrado")
    libro.titulo = libro_data.titulo
    libro.anio = libro_data.anio
    await db.commit()
    await db.refresh(libro)
    return libro

@app.delete("/api/libros/{libro_id}", status_code=status.HTTP_204_NO_CONTENT)
async def eliminar_libro(libro_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(LibroDB).where(LibroDB.id == libro_id))
    libro = result.scalar_one_or_none()
    if libro is None:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Libro no encontrado")
    await db.delete(libro)
    await db.commit()
    return
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Verificación HTTP del contrato (5 casos) y revisión de que la persistencia sea asíncrona. Resultado: **5/5 casos correctos** y **1/1 comprobación correcta**. Las pruebas de inserción doble y llaves únicas no se ejecutaron; solo se analizó el código.

| Caso | Resultado |
| :--- | :--- |
| POST `/api/libros` → 201 con `id: 1` | ✅ |
| GET `/api/libros` → 200 con el libro listado | ✅ |
| PUT `/api/libros/1` → 200 con título actualizado | ✅ |
| DELETE `/api/libros/1` → 204 | ✅ |
| GET `/api/libros/1` tras eliminar → 404 | ✅ |
| `AsyncSession` / `async_sessionmaker` / `aiosqlite` presentes en `app.py` | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `204 No Content`, `404 Not Found`). *`400` y `422` no se probaron.*
* [ ] No existen respuestas HTTP `500 Internal Server Error` no capturadas. *No se probó: no hay manejo de errores de la base de datos.*
* [ ] Encabezados `Content-Type: application/json` correctamente configurados. *FastAPI lo envía por defecto en las respuestas con cuerpo, pero no lo verifiqué de forma explícita.*

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> El verificador solo cubre el camino feliz, y el código pasa en verde aunque tiene omisiones:
>
> 1. **La tabla no se crea al iniciar, sino en cada petición.** `get_db` llama a `asegurar_tablas()` antes de cada request, así que se ejecuta un `create_all` por petición. El contrato pide crearla al iniciar la aplicación; lo correcto es hacerlo una vez en el `lifespan`.
> 2. **El ejemplo del contrato no coincide con el código.** El ejemplo del cuaderno muestra `PUT /api/libros/1 {"titulo": "Rayuela (edición)"}` sin `anio`, pero el esquema exige `titulo` y `anio`, así que esa petición recibiría `422`. El verificador no lo detectó porque su caso envía ambos campos.
> 3. **Sin manejo de errores de la base.** Un fallo en `commit()` (`SQLAlchemyError`, base bloqueada) sube sin capturar y el cliente recibe un `500`. Lo correcto sería un `503` con mensaje controlado.
> 4. **Inserción doble y llaves únicas sin cubrir.** No hay restricción `UNIQUE`; enviar dos veces el mismo POST crea dos filas y ambas responden `201`. Si se quisiera impedirlo, haría falta `unique=True` y capturar `IntegrityError` para devolver `409 Conflict`.
> 5. **Validación débil.** `titulo` acepta cadenas vacías y `anio` acepta cualquier entero (negativo o absurdo).
> 6. **Sintaxis obsoleta y código sobrante.** `class Config` está deprecada en Pydantic v2 (usar `model_config = ConfigDict(from_attributes=True)`); `asyncio` y `os` se importan sin usarse, y el `finally: await session.close()` es redundante dentro de `async with`.
> 7. **Lo que sí está bien:** todo el flujo es `async def` con `await`, el `404` está implementado en GET, PUT y DELETE, `expire_on_commit=False` evita errores tras el `commit`, y las consultas pasan por el ORM, que parametriza el SQL.
>
> Mejora propuesta (no ejecutada):
> ```python
> from contextlib import asynccontextmanager
> from pydantic import BaseModel, ConfigDict, Field
> from sqlalchemy.exc import SQLAlchemyError
> from fastapi.responses import JSONResponse
> from fastapi import Request
>
> @asynccontextmanager
> async def lifespan(app: FastAPI):
>     async with engine.begin() as conn:
>         await conn.run_sync(Base.metadata.create_all)
>     yield
>     await engine.dispose()
>
> app = FastAPI(title="CRUD Asíncrono de Libros", lifespan=lifespan)
>
> async def get_db():
>     async with async_session() as session:
>         yield session
>
> class LibroCreate(BaseModel):
>     titulo: str = Field(min_length=1, max_length=200)
>     anio: int = Field(ge=0, le=2100)
>
> class LibroResponse(LibroCreate):
>     model_config = ConfigDict(from_attributes=True)
>     id: int
>
> @app.exception_handler(SQLAlchemyError)
> async def error_bd(request: Request, exc: SQLAlchemyError):
>     return JSONResponse(status_code=503, content={"detail": "Base de datos no disponible"})
> ```

---
**Firma digital:** FernandaPalomo 

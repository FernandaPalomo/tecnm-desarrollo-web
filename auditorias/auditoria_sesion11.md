# Bitácora de Requerimientos y Auditoría IA — Sesión 11
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Arquitectura en Capas: Router, Servicio y Capa de Datos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Separación manual de controladores y lógica  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Separé a mano el código en «controladores» y «servicio» usando funciones sueltas en el mismo archivo. El problema fue que mi servicio seguía lanzando `HTTPException`, así que la lógica de negocio dependía de FastAPI: no podía probar la regla de «libro duplicado» sin levantar el servidor. Tampoco tenía una capa de datos real, solo un diccionario global al que accedían todas las funciones. Me di cuenta de que separar en funciones no es lo mismo que desacoplar las capas.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app_ia.py`. Datos en memoria, sin base de datos. |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | Tres capas: **Router** (`APIRouter` `/api/libros`, solo traduce HTTP ↔ servicio), **Servicio** (`ServicioLibros`, reglas: título no vacío, sin duplicados sin importar mayúsculas; lanza excepciones de dominio, nunca `HTTPException`) y **Repositorio** (`RepositorioMemoria`, que cumple un `Protocol`). Un único *handler* traduce `LibroNoEncontrado` → 404, `LibroDuplicado` → 409, `DatoInvalido` → 422 con sobre `{error: {codigo, mensaje}}`. `POST` 201, `GET` 200. |
| **Output (O)** | Solo código modular en `app_ia.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: FastAPI 0.110+ y Python 3.10+ en Google Colab con uvicorn. Un solo archivo app_ia.py, datos en memoria.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Refactoriza la API de libros con el patrón Service Layer en tres capas.
1. Repositorio: RepositorioMemoria con agregar, obtener, buscar_por_titulo y listar, definido contra un Protocol.
2. Servicio: ServicioLibros recibe el repositorio en el constructor y aplica las reglas: título no vacío (tras quitar espacios) y sin títulos duplicados (sin distinguir mayúsculas). Debe lanzar excepciones de dominio (LibroNoEncontrado, LibroDuplicado, DatoInvalido) y NO debe importar nada de FastAPI.
3. Router: APIRouter con prefijo /api/libros: POST (201), GET (200 con {"libros": [...]}) y GET /{id} (200).
4. Un solo exception_handler traduce las excepciones de dominio a 404, 409 y 422 con {"error": {"codigo", "mensaje"}}.
5. DTOs con extra="forbid", titulo 1-120 caracteres y anio entre 1450 y 2100.

Output: Solo el código de app_ia.py, modular y con códigos HTTP semánticos, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
*Versión final de `app_ia.py`, ya con las correcciones de la auditoría (sección 4).*
```python
"""Arquitectura en capas: Router -> Servicio -> Repositorio.

El servicio no conoce HTTP: lanza excepciones de dominio y el router
(o los exception handlers) las traduce a códigos HTTP.
"""
from typing import Protocol

from fastapi import APIRouter, FastAPI, Request, status
from fastapi.responses import JSONResponse
from pydantic import BaseModel, ConfigDict, Field


# ===================== Esquemas (DTOs) =====================
class LibroIn(BaseModel):
    model_config = ConfigDict(extra="forbid")
    titulo: str = Field(..., min_length=1, max_length=120)
    anio: int = Field(..., ge=1450, le=2100)


class LibroOut(BaseModel):
    id: int
    titulo: str
    anio: int


class ListaLibros(BaseModel):
    libros: list[LibroOut]


# ===================== Errores de dominio =====================
class ErrorDominio(Exception):
    codigo = "ERROR_DOMINIO"
    http_status = status.HTTP_400_BAD_REQUEST

    def __init__(self, mensaje: str):
        super().__init__(mensaje)
        self.mensaje = mensaje


class LibroNoEncontrado(ErrorDominio):
    codigo = "NO_ENCONTRADO"
    http_status = status.HTTP_404_NOT_FOUND


class LibroDuplicado(ErrorDominio):
    codigo = "DUPLICADO"
    http_status = status.HTTP_409_CONFLICT


class DatoInvalido(ErrorDominio):
    codigo = "VALIDACION"
    http_status = 422


# ===================== Capa de datos (Repositorio) =====================
class RepositorioLibros(Protocol):
    def agregar(self, titulo: str, anio: int) -> dict: ...
    def obtener(self, libro_id: int) -> dict | None: ...
    def buscar_por_titulo(self, titulo: str) -> dict | None: ...
    def listar(self) -> list[dict]: ...


class RepositorioMemoria:
    def __init__(self) -> None:
        self._datos: dict[int, dict] = {}
        self._siguiente_id = 1

    def agregar(self, titulo: str, anio: int) -> dict:
        libro = {"id": self._siguiente_id, "titulo": titulo, "anio": anio}
        self._datos[self._siguiente_id] = libro
        self._siguiente_id += 1
        return dict(libro)

    def obtener(self, libro_id: int) -> dict | None:
        libro = self._datos.get(libro_id)
        return dict(libro) if libro else None

    def buscar_por_titulo(self, titulo: str) -> dict | None:
        for libro in self._datos.values():
            if libro["titulo"].casefold() == titulo.casefold():
                return dict(libro)
        return None

    def listar(self) -> list[dict]:
        return [dict(l) for l in self._datos.values()]


# ===================== Capa de servicio (reglas de negocio) =====================
class ServicioLibros:
    def __init__(self, repo: RepositorioLibros) -> None:
        self.repo = repo

    def crear(self, titulo: str, anio: int) -> dict:
        titulo = titulo.strip()
        if not titulo:
            raise DatoInvalido("El título no puede estar vacío")
        if self.repo.buscar_por_titulo(titulo):
            raise LibroDuplicado(f"Ya existe un libro con el título '{titulo}'")
        return self.repo.agregar(titulo, anio)

    def obtener(self, libro_id: int) -> dict:
        libro = self.repo.obtener(libro_id)
        if libro is None:
            raise LibroNoEncontrado(f"El libro {libro_id} no existe")
        return libro

    def listar(self) -> list[dict]:
        return self.repo.listar()


# ===================== Capa de presentación (Router) =====================
servicio = ServicioLibros(RepositorioMemoria())
router = APIRouter(prefix="/api/libros", tags=["libros"])


@router.post("", response_model=LibroOut, status_code=status.HTTP_201_CREATED)
def crear_libro(datos: LibroIn):
    return servicio.crear(datos.titulo, datos.anio)


@router.get("", response_model=ListaLibros)
def listar_libros():
    return {"libros": servicio.listar()}


@router.get("/{libro_id}", response_model=LibroOut)
def obtener_libro(libro_id: int):
    return servicio.obtener(libro_id)


app = FastAPI(title="Biblioteca API - Arquitectura en Capas", version="1.1.0")
app.include_router(router)


@app.exception_handler(ErrorDominio)
async def manejar_error_dominio(request: Request, exc: ErrorDominio):
    return JSONResponse(
        status_code=exc.http_status,
        content={"error": {"codigo": exc.codigo, "mensaje": exc.mensaje}},
    )
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas de aislamiento de errores en lógica de negocio

**Resultado del Test Harness:** 9/9 casos correctos (A1 = prueba de humo de la versión artesanal; 01 en adelante = versión auditada `app_ia.py`).

| # | Caso | Detalle observado | Resultado |
| :-: | :--- | :--- | :-: |
| A1 | Artesano: creación válida | HTTP 201 | ✅ |
| 01 | POST válido → 201 | HTTP 201 {'id': 1, 'titulo': 'Rayuela', 'anio': 1963} | ✅ |
| 02 | Duplicado (regla de negocio) → 409 con sobre | HTTP 409 {'error': {'codigo': 'DUPLICADO', 'mensaje': "Ya existe un libro con el título 'rayuela'"}} | ✅ |
| 03 | Libro inexistente → 404 con sobre | HTTP 404 | ✅ |
| 04 | Título solo con espacios → 422 (servicio) | HTTP 422 {'error': {'codigo': 'VALIDACION', 'mensaje': 'El título no puede estar vacío'}} | ✅ |
| 05 | Año fuera de rango → 422 (DTO) | HTTP 422 | ✅ |
| 06 | Listado → 200 con 1 libro | HTTP 200 | ✅ |
| 07 | Servicio lanza error de dominio (no HTTPException) |  | ✅ |
| 08 | El servicio funciona con un repositorio falso (desacoplado) | {'id': 7, 'titulo': 'Aura', 'anio': 1962} | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> El harness quedó en verde (incluido el servicio probado sin HTTP con un repositorio falso). Hallazgos de la auditoría:
>
> - **`HTTPException` dentro del servicio:** es el error de mi versión artesanal (y el patrón más común en ejemplos de FastAPI): el servicio queda amarrado al framework y no se puede probar sin servidor. Por eso exigí en el prompt que el servicio no importara FastAPI y agregué los casos 07 y 08, que prueban el servicio directamente y con un repositorio falso. La solución es la jerarquía `ErrorDominio` con `codigo` y `http_status`, y un solo *handler* en la capa web.
> - **Constante deprecada:** `status.HTTP_422_UNPROCESSABLE_ENTITY` genera un aviso de deprecación en Starlette reciente (ahora se llama `HTTP_422_UNPROCESSABLE_CONTENT`). Usé el número `422` para que funcione en ambas versiones.
> - **El repositorio devolvía referencias internas:** si el router modificaba el diccionario devuelto, cambiaba el dato guardado. **Corrección:** el repositorio devuelve copias (`dict(libro)`).
> - **Servicio global:** `servicio = ServicioLibros(RepositorioMemoria())` se crea al importar el módulo; en la sesión 12 se reemplaza por inyección de dependencias.
>
> Sin errores 500.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> Lograr que el servicio no dependiera de FastAPI; tuve que pensar el error como algo del negocio («el libro está duplicado») y no como un código HTTP.

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Responder 409 Conflict para los títulos duplicados, regla que no venía en el reto original.

**¿Qué error detectó la verificación que tú no habías previsto?**
> Ninguno en la versión final; la prueba sin HTTP (caso 07) es la que fallaría si el servicio volviera a lanzar `HTTPException`.

---
**Firma digital:** FernandaPalomo

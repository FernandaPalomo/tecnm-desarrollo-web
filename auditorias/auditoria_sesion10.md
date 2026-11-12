# Bitácora de Requerimientos y Auditoría IA — Sesión 10
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Hito 1: Mock API RESTful con OpenAPI y Validation Harness  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Integración de Router y esquemas de validación  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Integrar todo el Bloque 1 en una sola API fue lo más pesado: el CRUD, la validación, los errores con sobre, el middleware y OpenAPI tenían que convivir. A mano tuve que llevar la cuenta de los ids, buscar el libro en cada operación y decidir qué responder en cada caso (204 sin cuerpo para el `DELETE`, 404 si ya no existe, 405 si el método no está permitido). Seguir la tabla de casos antes de escribir código me ayudó a no olvidar ninguno.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app.py`. Datos en memoria, sin base de datos. |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | CRUD de `/api/libros` con ids del servidor: `GET` lista (200, inicia vacía), `POST` (201), `GET/PUT /{id}` (200 o 404), `DELETE /{id}` (204 o 404). `titulo` 1-120 caracteres y `anio` opcional 1450-2100 → 422. Sobre de error uniforme para 404, 405 y 422. Middleware de traza. `/openapi.json` con el esquema `Libro`. Sin 500. |
| **Output (O)** | Solo código modular en `app.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: FastAPI 0.110+ y Python 3.10+ en Google Colab con uvicorn. Un solo archivo app.py, datos en memoria. Integra lo construido en las sesiones 04 a 09.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Mock API de libros del Hito 1.
1. CRUD completo de /api/libros con ids asignados por el servidor: GET (lista inicial vacía), POST (201), GET /{id} (200), PUT /{id} (200), DELETE /{id} (204).
2. Un libro inexistente responde 404 y un método no permitido (PATCH) responde 405.
3. Valida titulo (no vacío) y anio (entre 1450 y 2100) con 422.
4. Todos los errores usan el sobre {"error": {"codigo", "mensaje"}}.
5. Conserva el middleware de X-Process-Time y X-Request-ID.
6. /openapi.json debe incluir el esquema Libro. Ningún caso debe producir 500.

Output: Solo el código de app.py, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
*Versión final de `app.py`, ya con las correcciones de la auditoría (sección 4).*
```python
import re
import time
import uuid
from fastapi import FastAPI, Request, Response
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from starlette.exceptions import HTTPException as StarletteHTTPException
from pydantic import BaseModel, Field
from typing import Optional

app = FastAPI(title="Tu Mock API del Hito 1 se comportará así")

# AHORA VACIO, asi lo pide la verificacion
libros = []

# Solo se aceptan IDs cortos y seguros (evita inyección en logs y cabeceras)
PATRON_REQUEST_ID = re.compile(r"^[A-Za-z0-9._-]{1,64}$")

@app.middleware("http")
async def agregar_cabeceras(request: Request, call_next):
    inicio = time.perf_counter()
    # Si el cliente manda un X-Request-ID válido, se respeta; si no, se genera uno
    request_id = request.headers.get("x-request-id", "")
    if not PATRON_REQUEST_ID.match(request_id):
        request_id = uuid.uuid4().hex
    response = await call_next(request)
    proceso = time.perf_counter() - inicio
    response.headers["x-process-time"] = f"{proceso:.6f}"
    response.headers["x-request-id"] = request_id
    return response

# 404 de rutas inexistentes y 405 también usan el sobre de error (se conserva el header Allow)
@app.exception_handler(StarletteHTTPException)
async def http_handler(request: Request, exc: StarletteHTTPException):
    codigos = {404: "NO_ENCONTRADO", 405: "METODO_NO_PERMITIDO"}
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"codigo": codigos.get(exc.status_code, "ERROR_HTTP"), "mensaje": str(exc.detail)}},
        headers=getattr(exc, "headers", None),
    )

class LibroNoEncontrado(Exception):
    def __init__(self, libro_id: int):
        self.libro_id = libro_id
        super().__init__(f"El libro {libro_id} no existe")

@app.exception_handler(LibroNoEncontrado)
async def libro_no_encontrado_handler(request: Request, exc: LibroNoEncontrado):
    return JSONResponse(
        status_code=404,
        content={"error": {"codigo": "NO_ENCONTRADO", "mensaje": f"El libro {exc.libro_id} no existe"}}
    )

@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    # Se indica qué campo falló sin exponer el valor recibido
    detalles = [{"campo": ".".join(str(p) for p in e["loc"][1:]), "problema": e["msg"]} for e in exc.errors()]
    return JSONResponse(
        status_code=422,
        content={"error": {"codigo": "VALIDACION", "mensaje": "Revisa los datos enviados", "detalles": detalles}}
    )

# Esquema Libro: lo exige el caso 09 en components.schemas
class Libro(BaseModel):
    id: int
    titulo: str
    anio: Optional[int] = None

class LibroCrear(BaseModel):
    titulo: str = Field(..., min_length=1, max_length=120)
    anio: Optional[int] = Field(default=None, ge=1450, le=2100)

class LibroActualizar(BaseModel):
    titulo: str = Field(..., min_length=1, max_length=120)
    anio: Optional[int] = Field(default=None, ge=1450, le=2100)

def buscar_libro(libro_id: int):
    for libro in libros:
        if libro["id"] == libro_id:
            return libro
    return None

@app.get("/api/libros")
def listar_libros():
    return {"libros": libros}

@app.post("/api/libros", status_code=201, response_model=Libro)
def crear_libro(datos: LibroCrear):
    if not datos.titulo.strip():
        return JSONResponse(status_code=422, content={"error": {"codigo": "VALIDACION", "mensaje": "Revisa los datos enviados"}})
    nuevo_id = max([l["id"] for l in libros], default=0) + 1
    nuevo = {"id": nuevo_id, "titulo": datos.titulo.strip(), "anio": datos.anio}
    libros.append(nuevo)
    return nuevo

@app.get("/api/libros/{libro_id}", response_model=Libro)
def get_libro(libro_id: int):
    libro = buscar_libro(libro_id)
    if not libro:
        raise LibroNoEncontrado(libro_id)
    return libro

@app.put("/api/libros/{libro_id}", response_model=Libro)
def actualizar_libro(libro_id: int, datos: LibroActualizar):
    libro = buscar_libro(libro_id)
    if not libro:
        raise LibroNoEncontrado(libro_id)
    if not datos.titulo.strip():
        return JSONResponse(status_code=422, content={"error": {"codigo": "VALIDACION", "mensaje": "Revisa los datos enviados"}})
    libro["titulo"] = datos.titulo.strip()
    libro["anio"] = datos.anio
    return libro

@app.delete("/api/libros/{libro_id}", status_code=204)
def eliminar_libro(libro_id: int):
    libro = buscar_libro(libro_id)
    if not libro:
        raise LibroNoEncontrado(libro_id)
    libros.remove(libro)
    return Response(status_code=204)
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Suite integral de 10 pruebas HTTP destructivas

**Resultado del verificador:** 9/9 casos correctos.

| # | Caso | Esperado | Resultado |
| :-: | :--- | :-: | :-: |
| 01 | Listado inicial vacío (`GET /api/libros`) | 200 | ✅ |
| 02 | Creación válida (`POST /api/libros`) | 201 | ✅ |
| 03 | Consulta del libro creado (`GET /api/libros/1`) | 200 | ✅ |
| 04 | Actualización del libro (`PUT /api/libros/1`) | 200 | ✅ |
| 05 | Eliminación del libro (`DELETE /api/libros/1`) | 204 | ✅ |
| 06 | El libro eliminado ya no existe (`GET /api/libros/1`) | 404 | ✅ |
| 07 | Validación de datos inválidos (`POST /api/libros`) | 422 | ✅ |
| 08 | Método no permitido (`PATCH /api/libros/1`) | 405 | ✅ |
| 09 | Contrato OpenAPI disponible (`GET /openapi.json`) | 200 | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> Los 9 casos pasaron. Hallazgos:
>
> - **`anio` sin validar:** el contrato pedía validar `titulo` **y** `anio`, pero la IA solo validaba el título. El caso 07 pasó únicamente porque el título venía vacío; `{"titulo": "Aura", "anio": 1200}` se guardaba con 201. **Corrección:** `anio: Optional[int] = Field(default=None, ge=1450, le=2100)`.
> - **404 de ruta y 405 sin sobre:** respondían `{"detail": "Method Not Allowed"}`, distinto del resto de errores. **Corrección:** *handler* para `StarletteHTTPException` que conserva el estado y el encabezado `Allow`, con `codigo` `NO_ENCONTRADO` o `METODO_NO_PERMITIDO`.
> - **422 sin detalle y `X-Request-ID` sin validar:** se aplicaron las mismas correcciones de las sesiones 08 y 09.
> - **Detalle menor:** el encabezado `Allow` del 405 solo lista `GET` aunque la ruta acepta `PUT` y `DELETE`; es un comportamiento de Starlette, no del código.
>
> Sin errores 500. Pendiente del entregable: crear el tag `v0.1-hito1` en el repositorio.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> El 405, porque FastAPI lo genera solo y no pasaba por mi formato de error.

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Validar el rango del año también en el `PUT` y usar el mismo sobre para 404, 405 y 422.

**¿Qué error detectó la verificación que tú no habías previsto?**
> Ninguno en la suite; el año sin validar lo encontré probando un caso que no estaba en la tabla.

---
**Firma digital:** FernandaPalomo

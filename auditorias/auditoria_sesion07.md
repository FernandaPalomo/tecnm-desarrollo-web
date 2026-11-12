# Bitácora de Requerimientos y Auditoría IA — Sesión 07
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Especificación OpenAPI (Swagger) y ReDoc  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Anotación manual de metadata de endpoints  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Escribir la especificación OpenAPI a mano en JSON fue muy tedioso: hay que describir cada ruta, cada método, sus respuestas y los esquemas en `components.schemas`, y luego referenciarlos con `$ref`. Lo peor es que el documento y el código se desincronizan fácilmente: si cambio un campo en el endpoint y olvido actualizar el JSON, la documentación miente. También me confundió que un modelo solo aparece en `components.schemas` si algún endpoint lo usa.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app.py`. Datos en memoria, sin base de datos. |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | App `Biblioteca API` v1.0.0 con descripción. `GET /api/libros` (summary «Listar libros», 200 con `RespuestaLibros`) y `POST /api/libros` (summary «Crear un libro», 201 con `Libro`, 422 documentado). El esquema `Libro` aparece en `components.schemas`. `/openapi.json`, `/docs` y `/redoc` responden 200. |
| **Output (O)** | Solo código modular en `app.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: FastAPI 0.110+ y Python 3.10+ en Google Colab con uvicorn. Un solo archivo app.py, datos en memoria.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Publica una API de libros con documentación OpenAPI generada desde el código.
1. La aplicación se llama "Biblioteca API", versión 1.0.0, con la descripción "API de libros del curso DAB-2201".
2. GET /api/libros (summary "Listar libros") responde 200 con {"libros": [...]}.
3. POST /api/libros (summary "Crear un libro") responde 201 con el libro creado.
4. El modelo Pydantic Libro (id, titulo, autor) debe aparecer en components.schemas.
5. /openapi.json, /docs y /redoc deben responder 200. Documenta las respuestas 200, 201 y 422.

Output: Solo el código de app.py, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
*Versión final de `app.py`, ya con las correcciones de la auditoría (sección 4).*
```python
from fastapi import FastAPI, status
from pydantic import BaseModel, Field
from typing import List

app = FastAPI(
    title="Biblioteca API",
    version="1.0.0",
    description="API de libros del curso DAB-2201"
)

# Modelo de entrada: lo que el cliente envía
class LibroCrear(BaseModel):
    titulo: str = Field(..., min_length=1, max_length=120, examples=["Rayuela"])
    autor: str = Field(..., min_length=1, max_length=80, examples=["Julio Cortázar"])

# Modelo Libro solicitado por la especificación (aparece en components.schemas)
class Libro(LibroCrear):
    id: int

# Modelo de respuesta del listado
class RespuestaLibros(BaseModel):
    libros: List[Libro]

libros_db: List[dict] = []

@app.get(
    "/api/libros",
    response_model=RespuestaLibros,
    summary="Listar libros",
    responses={200: {"description": "Listado de libros registrados"}},
)
def obtener_libros():
    return {"libros": libros_db}

@app.post(
    "/api/libros",
    response_model=Libro,
    status_code=status.HTTP_201_CREATED,
    summary="Crear un libro",
    responses={201: {"description": "Libro creado"}, 422: {"description": "Datos inválidos"}},
)
def crear_libro(datos: LibroCrear):
    nuevo = {"id": max((l["id"] for l in libros_db), default=0) + 1, **datos.model_dump()}
    libros_db.append(nuevo)
    return nuevo
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Verificación de renderizado de interfaz Swagger UI

**Resultado del verificador:** 4/4 casos correctos.

| # | Caso | Esperado | Resultado |
| :-: | :--- | :-: | :-: |
| 01 | OpenAPI responde con metadatos (`GET /openapi.json`) | 200 | ✅ |
| 02 | Swagger UI está disponible (`GET /docs`) | 200 | ✅ |
| 03 | ReDoc está disponible (`GET /redoc`) | 200 | ✅ |
| 04 | El endpoint documentado responde (`GET /api/libros`) | 200 | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> Los 4 casos pasaron, pero el contrato publicado no coincidía con el ejemplo de la sesión:
>
> - **Faltaba `POST /api/libros`:** la IA solo implementó el `GET`, aunque el resultado esperado documenta `post` con summary «Crear un libro» y respuesta 201. El verificador no lo revisa, así que la documentación quedaba incompleta. **Corrección:** se agregó el `POST` con `status_code=201` y su modelo de entrada `LibroCrear`.
> - **`GET` devolvía siempre `[]`:** era una respuesta fija. **Corrección:** ahora lee la colección en memoria.
> - **Sin restricciones ni ejemplos:** los campos no tenían límites ni `examples`, así que Swagger mostraba `"string"`. **Corrección:** `Field(min_length, max_length, examples=[...])`.
> - **Respuestas de error no documentadas:** se agregó `responses={422: ...}` para que el contrato diga lo que el servidor realmente hace.
>
> Sin errores 500.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> Que `Libro` apareciera en `components.schemas`: con un `response_model` que solo devolvía una lista sin tipar, el esquema no se generaba.

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Separar `LibroCrear` (entrada, sin `id`) de `Libro` (salida, con `id`).

**¿Qué error detectó la verificación que tú no habías previsto?**
> Ninguno; la falta del `POST` la encontré comparando mi `/openapi.json` con el ejemplo del cuaderno.

---
**Firma digital:** FernandaPalomo

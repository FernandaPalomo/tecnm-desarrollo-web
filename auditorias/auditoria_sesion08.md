# Bitácora de Requerimientos y Auditoría IA — Sesión 08
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Respuestas Semánticas y Excepciones HTTP Personalizadas  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Lanzamiento manual de excepciones estructuradas  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Manejar los errores a mano significó escribir un `try/except` en cada endpoint y construir el mismo JSON de error una y otra vez. Lo difícil fue que FastAPI ya tiene su propio formato (`{"detail": ...}`) para los 404 y los 422, así que mi formato y el del framework se mezclaban: unas respuestas tenían `error` y otras `detail`. Para que todas fueran iguales tuve que entender que hay que registrar *handlers* globales en lugar de repetir el `JSONResponse` en cada función.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app.py`. Datos en memoria, sin base de datos. |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | Excepción de negocio `LibroNoEncontrado` con *handler* → 404 `{"error": {"codigo": "NO_ENCONTRADO", "mensaje"}}`. *Handler* de `RequestValidationError` → 422 con el mismo sobre, `codigo: VALIDACION` y `detalles` por campo. Título vacío → 422. Semilla: libro 1 «Rayuela». `GET /api/libros/1` → 200, `/api/libros/99` → 404, `POST` válido → 201. Ningún 500. |
| **Output (O)** | Solo código modular en `app.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: FastAPI 0.110+ y Python 3.10+ en Google Colab con uvicorn. Un solo archivo app.py, datos en memoria.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Implementa un manejo de errores uniforme.
1. Define la excepción LibroNoEncontrado y regístrala con @app.exception_handler para responder 404 con {"error": {"codigo": "NO_ENCONTRADO", "mensaje": "..."}}.
2. Registra un handler para RequestValidationError que responda 422 con el mismo sobre y codigo "VALIDACION".
3. Un título vacío se rechaza con 422.
4. La API inicia con el libro {"id": 1, "titulo": "Rayuela", "anio": 1963}.
5. GET /api/libros/{id} responde 200 o 404; POST /api/libros responde 201.
6. Ningún caso debe producir 500.

Output: Solo el código de app.py, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
*Versión final de `app.py`, ya con las correcciones de la auditoría (sección 4).*
```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import BaseModel, Field
from typing import Optional

app = FastAPI(title="Tus errores se verán exactamente así")

# Libro inicial para que el caso feliz exista
libros = [
    {"id": 1, "titulo": "Rayuela", "anio": 1963}
]

# Excepción de negocio
class LibroNoEncontrado(Exception):
    def __init__(self, libro_id: int):
        self.libro_id = libro_id
        super().__init__(f"El libro {libro_id} no existe")

# Handler 404 -> {"error": {"codigo": "NO_ENCONTRADO", ...}}
@app.exception_handler(LibroNoEncontrado)
async def libro_no_encontrado_handler(request: Request, exc: LibroNoEncontrado):
    return JSONResponse(
        status_code=404,
        content={"error": {"codigo": "NO_ENCONTRADO", "mensaje": f"El libro {exc.libro_id} no existe"}}
    )

# Handler 422 -> {"error": {"codigo": "VALIDACION", ...}}
@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    # Se indica qué campo falló sin exponer el valor recibido
    detalles = [{"campo": ".".join(str(p) for p in e["loc"][1:]), "problema": e["msg"]} for e in exc.errors()]
    return JSONResponse(
        status_code=422,
        content={"error": {"codigo": "VALIDACION", "mensaje": "Revisa los datos enviados", "detalles": detalles}}
    )

# Modelo: título vacío = 422
class LibroCrear(BaseModel):
    titulo: str = Field(..., min_length=1)
    anio: Optional[int] = None

@app.get("/api/libros/{libro_id}")
def get_libro(libro_id: int):
    for libro in libros:
        if libro["id"] == libro_id:
            return libro
    raise LibroNoEncontrado(libro_id)

@app.post("/api/libros", status_code=201)
def crear_libro(datos: LibroCrear):
    if not datos.titulo.strip():
        return JSONResponse(
            status_code=422,
            content={"error": {"codigo": "VALIDACION", "mensaje": "Revisa los datos enviados"}}
        )
    nuevo_id = max([l["id"] for l in libros], default=0) + 1
    nuevo = {"id": nuevo_id, "titulo": datos.titulo.strip(), "anio": datos.anio}
    libros.append(nuevo)
    return nuevo
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas de captura de errores sin crashing (HTTP 500)

**Resultado del verificador:** 4/4 casos correctos.

| # | Caso | Esperado | Resultado |
| :-: | :--- | :-: | :-: |
| 01 | Un libro existente responde 200 (`GET /api/libros/1`) | 200 | ✅ |
| 02 | Un libro inexistente responde 404 con sobre de error (`GET /api/libros/99`) | 404 | ✅ |
| 03 | Los datos inválidos responden 422 con sobre de error (`POST /api/libros`) | 422 | ✅ |
| 04 | Un libro válido sigue respondiendo 201 (`POST /api/libros`) | 201 | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> Los 4 casos pasaron. Observaciones:
>
> - **El 422 perdía la información útil:** el *handler* respondía siempre «Revisa los datos enviados» sin decir qué campo falló, así que el cliente no sabe qué corregir. **Corrección:** se agregó `detalles` con el campo y el problema (`exc.errors()`), **sin** devolver el valor recibido para no reflejar datos del usuario.
> - **Validación duplicada:** además de `min_length=1`, el endpoint vuelve a comprobar `titulo.strip()` y construye otro `JSONResponse` a mano. Funciona (rechaza `"   "`), pero lo correcto sería `str_strip_whitespace=True` en el modelo para que la regla viva en un solo lugar.
> - **Rutas inexistentes y 405 siguen con `{"detail"}`:** el sobre solo cubre las excepciones de negocio y de validación; los errores propios de Starlette conservan el formato por defecto. Lo corregí en el Hito 1 (sesión 10).
> - **`POST` sin `response_model`:** la salida no está documentada en OpenAPI.
>
> Sin errores 500.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> El 422 con sobre, porque FastAPI responde por defecto con `{"detail": [...]}` y tuve que reemplazar su *handler* de validación.

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Incluir `detalles` por campo en el error 422, sin exponer el valor enviado.

**¿Qué error detectó la verificación que tú no habías previsto?**
> Ninguno, aunque al principio olvidé sembrar el libro 1 y el caso feliz respondía 404.

---
**Firma digital:** FernandaPalomo

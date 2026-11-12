# Bitácora de Requerimientos y Auditoría IA — Sesión 09
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Middleware Web, Intercepción y Logging de Transacciones  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Middleware manual de tiempo de respuesta (X-Process-Time)  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Medir el tiempo en cada endpoint por separado era repetitivo: guardar la hora al inicio, calcular la diferencia al final y agregar los encabezados en cada `return`. Además, eso no cubría los 404 de rutas que no existen, porque ningún endpoint mío se ejecuta en ese caso. Lo que me costó entender es que el *middleware* envuelve toda la petición, incluso las respuestas que genera el propio framework.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app.py`. Datos en memoria, sin base de datos. |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | Middleware HTTP que agrega a **toda** respuesta (incluidos 404) `X-Process-Time` (segundos con decimales, medido con `perf_counter`) y `X-Request-ID`. Si el cliente envía un `X-Request-ID` válido (`[A-Za-z0-9._-]{1,64}`) se conserva; si no, se genera un UUID4. `GET /api/libros` → 200. |
| **Output (O)** | Solo código modular en `app.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: FastAPI 0.110+ y Python 3.10+ en Google Colab con uvicorn. Un solo archivo app.py, datos en memoria.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Agrega un middleware HTTP que mida el tiempo de cada petición y propague un identificador de traza.
1. Toda respuesta incluye X-Process-Time con los segundos transcurridos en formato decimal.
2. Toda respuesta incluye X-Request-ID; si la petición trae ese encabezado, se conserva su valor, si no, se genera uno.
3. El middleware también se aplica a las respuestas 404.
4. Conserva GET /api/libros con 200 y el manejo de errores de la sesión 08.

Output: Solo el código de app.py, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
*Versión final de `app.py`, ya con las correcciones de la auditoría (sección 4).*
```python
import re
import time
import uuid
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from pydantic import BaseModel, Field
from typing import Optional

app = FastAPI(title="Todas tus respuestas llevarán estas cabeceras")

libros = [
    {"id": 1, "titulo": "Rayuela", "anio": 1963}
]

# --- Middleware: agrega las 2 cabeceras a TODO ---
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

class LibroCrear(BaseModel):
    titulo: str = Field(..., min_length=1)
    anio: Optional[int] = None

@app.get("/api/libros")
def listar_libros():
    return {"libros": libros}

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
*Pruebas ejecutadas:* Pruebas de inyección de encabezados de rastreo

**Resultado del verificador:** 3/3 casos correctos.

| # | Caso | Esperado | Resultado |
| :-: | :--- | :-: | :-: |
| 01 | Una respuesta correcta trae tiempo y traza (`GET /api/libros`) | 200 | ✅ |
| 02 | El X-Request-ID del cliente se conserva (`GET /api/libros`) | 200 | ✅ |
| 03 | El middleware también cubre los 404 (`GET /api/no-existe`) | 404 | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> Los 3 casos pasaron. La auditoría encontró:
>
> - **`X-Request-ID` del cliente sin validar:** la IA copiaba cualquier valor recibido a la respuesta. Un cliente podría enviar textos enormes o caracteres especiales para ensuciar los *logs* (*log injection*). **Corrección:** solo se acepta un ID que cumpla `^[A-Za-z0-9._-]{1,64}$`; si no, se genera uno nuevo.
> - **IDs de 8 caracteres:** `uuid4().hex[:8]` reduce mucho la unicidad y puede repetirse con mucho tráfico. **Corrección:** UUID4 completo.
> - **`time.time()` para medir duraciones:** depende del reloj del sistema, que puede ajustarse. **Corrección:** `time.perf_counter()`, que es monotónico.
> - **El middleware no registra nada:** el tema es *logging* de transacciones, pero no se escribe ningún log. Faltaría un `logger.info` con método, ruta, estado, duración e ID.
>
> Sin errores 500.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> Que el middleware también cubriera los 404 de rutas inexistentes; tuve que confirmar que se ejecuta antes del ruteo.

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Validar el formato del `X-Request-ID` recibido.

**¿Qué error detectó la verificación que tú no habías previsto?**
> Ninguno; pero comprobé que los encabezados se comparan en minúsculas, por eso el verificador busca `x-request-id`.

---
**Firma digital:** FernandaPalomo

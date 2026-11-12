# Bitácora de Requerimientos y Auditoría IA — Sesión 06
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Validación Estricta, Constraints y Params Query/Body  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Filtro manual de rangos y longitudes de texto  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> A mano, cada regla de negocio se convirtió en un `if` con su propio mensaje: longitud del título entre 3 y 80, año entre 1450 y 2100 y precio mayor que 0. Lo complicado fue no confundir los límites incluyentes y excluyentes (el precio debe ser **mayor** que 0, no mayor o igual), y validar también el `limite` de la consulta, que llega como texto en la URL y hay que convertirlo antes de compararlo. Si la conversión falla, el servidor lanza una excepción y responde 500 en lugar de 422.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app.py`. Datos en memoria, sin base de datos. |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | `LibroCreate` con `Field`: `titulo` 3-80 caracteres, `anio` 1450-2100 (ambos incluyentes), `precio` > 0 (excluyente). `POST /api/libros` → 201 o 422. `GET /api/libros?limite=N` con `Query(default=10, ge=1, le=100)` → 200 con `{"libros": [...]}`; `limite` fuera de rango → 422. |
| **Output (O)** | Solo código modular en `app.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: FastAPI 0.110+ y Pydantic v2 (Python 3.10+) en Google Colab con uvicorn. Un solo archivo app.py y datos en memoria.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Implementa el contrato de libros con reglas de negocio explícitas en el DTO.
1. titulo: entre 3 y 80 caracteres. anio: entre 1450 y 2100. precio: mayor que 0. Usa Field().
2. POST /api/libros responde 201 con el libro creado y su id (empieza en 1); los datos inválidos responden 422.
3. GET /api/libros acepta el parámetro limite entre 1 y 100 con Query(); fuera de rango responde 422.
4. No repitas la validación dentro del endpoint: debe vivir en el DTO.

Output: Solo el código de app.py, modular y con códigos HTTP semánticos, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
*Versión final de `app.py`, ya con las correcciones de la auditoría (sección 4).*
```python
from fastapi import FastAPI, Query, status
from pydantic import BaseModel, Field
from typing import List

app = FastAPI(title="Tus límites de negocio responderán así")

# Simulación de base de datos en memoria
libros_db = []

# Modelo Pydantic para la validación estricta del cuerpo (Body)
class LibroCreate(BaseModel):
    titulo: str = Field(..., min_length=3, max_length=80)
    anio: int = Field(..., ge=1450, le=2100)
    precio: float = Field(..., gt=0)

class LibroResponse(BaseModel):
    id: int
    titulo: str
    anio: int
    precio: float

@app.post("/api/libros", response_model=LibroResponse, status_code=status.HTTP_201_CREATED)
def crear_libro(libro: LibroCreate):
    nuevo_id = max((l["id"] for l in libros_db), default=0) + 1
    nuevo_libro = {
        "id": nuevo_id,
        "titulo": libro.titulo,
        "anio": libro.anio,
        "precio": libro.precio
    }
    libros_db.append(nuevo_libro)
    return nuevo_libro

class ListaLibros(BaseModel):
    libros: List[LibroResponse]

@app.get("/api/libros", response_model=ListaLibros)
def listar_libros(limite: int = Query(default=10, ge=1, le=100)):
    return {"libros": libros_db[:limite]}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas de desbordamiento de cadenas y números negativos

**Resultado del verificador:** 6/6 casos correctos.

| # | Caso | Esperado | Resultado |
| :-: | :--- | :-: | :-: |
| 01 | Un libro dentro de los límites se crea (`POST /api/libros`) | 201 | ✅ |
| 02 | Un título demasiado corto produce 422 (`POST /api/libros`) | 422 | ✅ |
| 03 | Un año fuera de rango produce 422 (`POST /api/libros`) | 422 | ✅ |
| 04 | Un precio no positivo produce 422 (`POST /api/libros`) | 422 | ✅ |
| 05 | El límite de paginación debe ser al menos 1 (`GET /api/libros?limite=0`) | 422 | ✅ |
| 06 | Un límite válido responde 200 (`GET /api/libros?limite=5`) | 200 | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> Los 6 casos pasaron. Lo que el verificador no detecta:
>
> - **`limite` obligatorio:** la IA lo declaró como `Query(...)`, así que `GET /api/libros` sin parámetros respondía **422**. Un listado sin paginación explícita debería funcionar. **Corrección:** `Query(default=10, ge=1, le=100)`.
> - **Formato de listado inconsistente:** devolvía una lista directa en lugar de `{"libros": [...]}` como en las sesiones 02 a 05. **Corrección:** modelo `ListaLibros`.
> - **`id` con `len(libros_db) + 1`:** generaría ids repetidos en cuanto exista un `DELETE`. **Corrección:** `max(ids) + 1`.
> - **Títulos con espacios:** `"   "` (tres espacios) cumple `min_length=3`. Se podría agregar `str_strip_whitespace=True` en el `model_config`; lo dejo anotado para la sesión 08.
>
> Sin errores 500.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> El del precio, porque al principio usé `ge=0` y un precio de 0 se aceptaba. La diferencia entre `ge` y `gt` es pequeña pero cambia la regla de negocio.

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Darle a `limite` un valor por defecto de 10 para que el listado funcione sin parámetros.

**¿Qué error detectó la verificación que tú no habías previsto?**
> Ninguno en la suite; el 422 del listado sin `limite` lo descubrí al probar la ruta desde Swagger.

---
**Firma digital:** FernandaPalomo

# Bitácora de Requerimientos y Auditoría IA — Sesión 05
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Pydantic v2 / DTOs en PHP y Tipado Estático  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* DTO manual con validación de tipos primitivos  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Validar tipos a mano fue más trabajo del que pensaba. Para cada campo tuve que revisar con `isinstance` si era `str`, `int`, `float` o `bool`, y luego decidir qué hacer con casos raros: `"1966"` es texto pero representa un número, mientras que `"mil novecientos"` no. Además, en Python `True` es instancia de `int`, así que `isinstance(True, int)` da `True` y un booleano se colaba como año. También tenía que rechazar los campos que no estaban en el contrato (como `color`) recorriendo las llaves del JSON. Todo eso se vuelve un bloque enorme de `if` que se repite en cada endpoint.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app.py`. Datos en memoria, sin base de datos. |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | DTO de entrada `LibroIn` cerrado (`extra="forbid"`): `titulo` (str, no vacío), `anio` (int), `precio` (float), `disponible` (bool). La coerción de `"1966"` a `1966` se acepta; `"mil novecientos"` y los campos desconocidos responden 422. DTO de salida `LibroOut` con `id` asignado por el servidor. `POST /api/libros` → 201; `GET /api/libros` → 200 con `{"libros": [...]}`. |
| **Output (O)** | Solo código modular en `app.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: FastAPI 0.110+ con Pydantic v2 y Python 3.10+, en Google Colab con uvicorn. Un solo archivo app.py, datos en memoria.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Define el contrato de un libro con un DTO de entrada y otro de salida.
1. LibroIn exige titulo (str), anio (int), precio (float) y disponible (bool), y rechaza cualquier campo desconocido con model_config = ConfigDict(extra="forbid").
2. La cadena "1966" debe convertirse a 1966; "mil novecientos" debe responder 422.
3. POST /api/libros responde 201 con LibroOut: el id asignado por el servidor (empieza en 1) y los datos del libro, sin campos inventados.
4. GET /api/libros responde 200 con {"libros": [...]}.

Output: Solo el código de app.py, con los DTO de entrada y salida separados y códigos HTTP semánticos, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
*Versión final de `app.py`, ya con las correcciones de la auditoría (sección 4).*
```python
from fastapi import FastAPI, status
from pydantic import BaseModel, ConfigDict, Field
from typing import List

app = FastAPI(title="Tu DTO aceptará y rechazará exactamente esto")

# Memoria temporal para los libros
libros_db = []
next_id = 1

# DTO de entrada: Rechaza campos extra
class LibroIn(BaseModel):
    model_config = ConfigDict(extra="forbid")

    titulo: str = Field(..., min_length=1)
    anio: int
    precio: float
    disponible: bool

# DTO de salida
class LibroOut(BaseModel):
    id: int
    titulo: str
    anio: int
    precio: float
    disponible: bool

@app.post("/api/libros", response_model=LibroOut, status_code=status.HTTP_201_CREATED)
def crear_libro(libro: LibroIn):
    global next_id
    nuevo_libro = {
        "id": next_id,
        "titulo": libro.titulo,
        "anio": libro.anio,
        "precio": libro.precio,
        "disponible": libro.disponible
    }
    libros_db.append(nuevo_libro)
    next_id += 1
    return nuevo_libro

class ListaLibros(BaseModel):
    libros: List[LibroOut]

@app.get("/api/libros", response_model=ListaLibros)
def listar_libros():
    return {"libros": libros_db}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas de tipos incompatibles y coerción

**Resultado del verificador:** 5/5 casos correctos.

| # | Caso | Esperado | Resultado |
| :-: | :--- | :-: | :-: |
| 01 | Un libro válido se crea con 201 (`POST /api/libros`) | 201 | ✅ |
| 02 | La cadena numérica se convierte a entero (`POST /api/libros`) | 201 | ✅ |
| 03 | Un año no numérico produce 422 (`POST /api/libros`) | 422 | ✅ |
| 04 | Un campo extra produce 422 (`POST /api/libros`) | 422 | ✅ |
| 05 | El listado devuelve los libros creados (`GET /api/libros`) | 200 | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> Los 5 casos quedaron en verde. Hallazgos de la auditoría:
>
> - **Título vacío aceptado:** `{"titulo": ""}` se guardaba sin problema. **Corrección:** `titulo: str = Field(..., min_length=1)`.
> - **Coerción más permisiva de lo esperado:** en modo *lax*, Pydantic también acepta `"250.5"` como `precio`, `1963.0` como `anio` y `"true"` o `1` como `disponible`. El contrato solo pedía la coerción de cadenas numéricas a entero, así que si se quisiera un contrato totalmente estricto habría que usar `ConfigDict(strict=True)` o tipos `StrictBool`, aceptando que `"1966"` también se rechazaría.
> - **`GET` sin `response_model`:** el listado devolvía la lista interna tal cual. **Corrección:** se agregó el modelo `ListaLibros` para que la salida también pase por el DTO.
> - **Estado global con `global next_id`:** funciona en un solo proceso, pero no es seguro con varios *workers*; se resolverá al usar base de datos.
>
> No se generaron errores 500.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> El del campo extra, porque por defecto Pydantic ignora los campos desconocidos y responde 201. Tuve que agregar `extra="forbid"` para que el contrato fuera cerrado.

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Exigir un título no vacío y envolver el listado en un `response_model`.

**¿Qué error detectó la verificación que tú no habías previsto?**
> Ninguno; pero me sorprendió que el booleano `True` también pasa como `int` si no se usa el modo estricto.

---
**Firma digital:** FernandaPalomo

# Bitácora de Requerimientos y Auditoría IA — Sesión 02
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Gestión en GitHub Web y Métodos HTTP (GET, POST, PUT, DELETE)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Enrutamiento manual por verbo HTTP  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Al enrutar a mano por verbo HTTP, la dificultad principal es que hay que revisar el método y la ruta de cada petición con muchos `if`, y cualquier descuido cambia el resultado. Es fácil confundir 404 (la ruta o el recurso no existe) con 405 (la ruta existe, pero el verbo no está permitido), porque a mano hay que decidir cuál devolver en cada caso. También hay que extraer el `id` de la URL, convertirlo a número, buscar el libro en la lista y devolver el código correcto en cada operación (201 al crear, 200 al leer o reemplazar, 204 sin cuerpo al eliminar). Todo eso son muchas líneas repetidas y fáciles de equivocar.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app.py`. Los libros se guardan en memoria (sin base de datos). |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | CRUD de libros sobre `/api/libros`. `GET /api/libros` → 200 con `{"libros": [...]}` (inicia vacía). `POST /api/libros` con `{"titulo", "anio"}` → 201 con el libro creado y un `id` asignado por el servidor (empieza en 1). `GET /api/libros/{id}` → 200 con el libro o 404 si no existe. `PUT /api/libros/{id}` reemplaza el libro → 200 (404 si no existe). `DELETE /api/libros/{id}` → 204 sin cuerpo (404 si no existe). Un verbo no implementado sobre una ruta existente (por ejemplo `PATCH /api/libros/1`) debe responder 405, no 404. PUT y DELETE son idempotentes en cuanto al estado final del recurso. |
| **Output (O)** | Solo código modular en `app.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: Trabajo con FastAPI 0.110+ y Python 3.10+, ejecutando el servidor con uvicorn dentro de Google Colab. Todo debe quedar en un solo archivo llamado app.py. Los datos se guardan en memoria; no uses base de datos.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Implementa un CRUD de libros (campos: titulo de tipo string y anio de tipo entero).
1. GET /api/libros devuelve {"libros": [...]} con HTTP 200. La colección inicia vacía.
2. POST /api/libros crea un libro, asigna el id en el servidor (empezando en 1) y responde HTTP 201 con el libro creado.
3. GET /api/libros/{id} devuelve el libro con HTTP 200 o HTTP 404 si no existe.
4. PUT /api/libros/{id} reemplaza el libro y responde HTTP 200 (404 si no existe).
5. DELETE /api/libros/{id} elimina el libro y responde HTTP 204 sin cuerpo (404 si no existe).
6. Usa modelos de Pydantic para validar la entrada.
7. No programes PATCH: un verbo no implementado sobre una ruta existente debe responder 405, no 404.

Output: Entrégame únicamente el código de app.py, modular y con códigos HTTP semánticos, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel
from typing import List

app = FastAPI(title="Tu CRUD de libros")

# Modelo Pydantic para validar la entrada de datos
class LibroBase(BaseModel):
    titulo: str
    anio: int

# Estructura del libro almacenado con su ID
class Libro(LibroBase):
    id: int

# Base de datos simulada en memoria
libros_db: List[Libro] = []
id_counter = 1

@app.get("/api/libros", response_model=dict)
def listar_libros():
    return {"libros": libros_db}

@app.post("/api/libros", response_model=Libro, status_code=status.HTTP_201_CREATED)
def crear_libro(libro: LibroBase):
    global id_counter
    nuevo_libro = Libro(id=id_counter, titulo=libro.titulo, anio=libro.anio)
    libros_db.append(nuevo_libro)
    id_counter += 1
    return nuevo_libro

@app.get("/api/libros/{libro_id}", response_model=Libro)
def obtener_libro(libro_id: int):
    for libro in libros_db:
        if libro.id == libro_id:
            return libro
    raise HTTPException(status_code=404, detail="Libro no encontrado")

@app.put("/api/libros/{libro_id}", response_model=Libro)
def reemplazar_libro(libro_id: int, libro_actualizado: LibroBase):
    for i, libro in enumerate(libros_db):
        if libro.id == libro_id:
            libros_db[i] = Libro(id=libro_id, titulo=libro_actualizado.titulo, anio=libro_actualizado.anio)
            return libros_db[i]
    raise HTTPException(status_code=404, detail="Libro no encontrado")

@app.delete("/api/libros/{libro_id}", status_code=status.HTTP_204_NO_CONTENT)
def eliminar_libro(libro_id: int):
    for i, libro in enumerate(libros_db):
        if libro.id == libro_id:
            libros_db.pop(i)
            return
    raise HTTPException(status_code=404, detail="Libro no encontrado")
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas de métodos no permitidos (HTTP 405)

**Resultado del verificador:** 8/8 casos correctos.

| # | Caso | Esperado | Resultado |
| :-: | :--- | :-: | :-: |
| 01 | La colección inicia vacía (`GET /api/libros`) | 200 | ✅ |
| 02 | POST crea el libro con id 1 (`POST /api/libros`) | 201 | ✅ |
| 03 | GET lista el libro creado (`GET /api/libros`) | 200 | ✅ |
| 04 | GET por id encuentra el libro (`GET /api/libros/1`) | 200 | ✅ |
| 05 | PUT reemplaza el libro (`PUT /api/libros/1`) | 200 | ✅ |
| 06 | DELETE elimina el libro (`DELETE /api/libros/1`) | 204 | ✅ |
| 07 | El libro eliminado ya no existe (`GET /api/libros/1`) | 404 | ✅ |
| 08 | PATCH no está implementado: 405, no 404 (`PATCH /api/libros/1`) | 405 | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> El código pasó los 8 casos del verificador y respeta los códigos del contrato (201, 200, 204, 404 y 405). Al auditarlo encontré detalles que la IA no cuidó:
>
> - **Validación débil de los campos:** `titulo` acepta texto vacío y `anio` acepta cualquier entero (por ejemplo, `-500` o `99999`). Se puede corregir con `Field(min_length=1)` en el título y un rango razonable en el año.
> - **Código no pedido en el contrato:** un cuerpo inválido en POST o PUT devuelve 422 de forma automática por FastAPI, no por código escrito por la IA.
> - **Contador global:** `id_counter` es una variable global, por lo que si el servidor se reinicia los datos y los ids se pierden (es esperable porque la colección vive en memoria, pero no se podría usar así en producción).
> - **Respuesta del listado:** usa `response_model=dict`, que no describe la estructura real en la documentación OpenAPI. Un modelo con `libros: List[Libro]` sería más preciso.
> - **Título de la API:** quedó el texto de ejemplo `"Tu CRUD de libros"`.
>
> No apareció ningún error 500 ni falla de seguridad grave en este laboratorio. El 405 del PATCH lo resolvió FastAPI solo, tal como lo indicaba la pista del contrato.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> El de PATCH con 405 en lugar de 404, porque a primera vista parece que si no existe el método debería ser "no encontrado". Entendí que 404 significa que el recurso no existe y 405 que la ruta existe pero ese verbo no está permitido. FastAPI lo resuelve solo al no definir PATCH sobre `/api/libros/{id}`.

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Separar los modelos en `LibroBase` (lo que envía el cliente) y `Libro` (con el `id` asignado por el servidor), usar una lista en memoria con un contador global para los ids y devolver 404 también en PUT y DELETE cuando el libro no existe. El contrato solo pedía el 404 explícitamente en GET por id.

**¿Qué error detectó la verificación que tú no habías previsto?**
> La verificación no marcó ningún error en mi implementación. Lo que aprendí es que los casos dependen del orden: el libro debe tener `id` 1, así que el servidor tiene que iniciar con la colección vacía y el contador en 1, y que después de eliminar el libro el GET debe dar 404.

---
**Firma digital:** FernandaPalomo

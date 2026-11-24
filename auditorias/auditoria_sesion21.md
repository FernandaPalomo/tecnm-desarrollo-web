# Bitácora de Requerimientos y Auditoría IA — Sesión 21
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Fundamentos de Persistencia: SQLite / PostgreSQL y Drivers  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Conexión manual y ejecución de sentencias SQL  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Hacerlo a mano exige varios pasos que son fáciles de olvidar: abrir la conexión con `sqlite3.connect`, crear la tabla con `CREATE TABLE IF NOT EXISTS`, ejecutar el `INSERT`, llamar a `commit()` (sin él la fila no se guarda en el archivo) y cerrar la conexión. También tuve que decidir cómo devolver el `id` generado (`lastrowid`) y cómo evitar concatenar strings en el SQL usando placeholders `?`. Además, la validación de `titulo` y `anio` la tendría que escribir yo si no usara Pydantic.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+), SQLite con el módulo estándar `sqlite3`, archivo `biblioteca.db`, ejecución en Colab |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | `GET /api/libros` → 200 con `{"libros": [...]}`; `POST /api/libros` con `{titulo, anio}` → 201 con `{id, titulo, anio}`; tabla `libros` creada al iniciar; consultas parametrizadas; datos persistidos en `biblioteca.db` |
| **Output (O)** | Solo código modular en `app.py`, esquemas Pydantic y códigos HTTP semánticos |

### 2.2 Prompt Estructurado
```text
CONTEXTO: API REST con FastAPI 0.110+ (Python 3.10+) que persiste datos en un archivo SQLite llamado biblioteca.db usando el módulo sqlite3.

ROL: Actúa como arquitecto de software senior, especialista en APIs RESTful y seguridad web.

ESPECIFICACIÓN:
1. Al iniciar la aplicación, crea la tabla libros (id INTEGER PRIMARY KEY AUTOINCREMENT, titulo TEXT NOT NULL, anio INTEGER NOT NULL) si no existe, sin pasos manuales.
2. POST /api/libros recibe {"titulo": str, "anio": int}, inserta una fila y responde 201 con {"id", "titulo", "anio"}.
3. GET /api/libros lee las filas de la base y responde 200 con {"libros": [{"id", "titulo", "anio"}]}.
4. Usa consultas parametrizadas (placeholders ?), nunca concatenación de strings.
5. Define esquemas Pydantic para entrada y salida.
6. Llama a commit() después de insertar y cierra siempre la conexión.

OUTPUT: Solo código en un único archivo app.py, sin explicaciones fuera de comentarios breves.
```

---

## 3. Código Devuelto por el Modelo
```python
import sqlite3
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List

app = FastAPI(title="Tus libros sobrevivirán al reinicio")

DB_PATH = "biblioteca.db"

def init_db():
    con = sqlite3.connect(DB_PATH)
    cursor = con.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS libros (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            titulo TEXT NOT NULL,
            anio INTEGER NOT NULL
        )
    """)
    con.commit()
    con.close()

# Inicializar la base de datos al arrancar la aplicación
init_db()

class LibroBase(BaseModel):
    titulo: str
    anio: int

class LibroResponse(BaseModel):
    id: int
    titulo: str
    anio: int

class LibrosListResponse(BaseModel):
    libros: List[LibroResponse]

@app.get("/api/libros", response_model=LibrosListResponse)
def obtener_libros():
    con = sqlite3.connect(DB_PATH)
    cursor = con.cursor()
    cursor.execute("SELECT id, titulo, anio FROM libros")
    filas = cursor.fetchall()
    con.close()

    libros = [{"id": f[0], "titulo": f[1], "anio": f[2]} for f in filas]
    return {"libros": libros}

@app.post("/api/libros", response_model=LibroResponse, status_code=201)
def crear_libro(libro: LibroBase):
    con = sqlite3.connect(DB_PATH)
    cursor = con.cursor()
    cursor.execute(
        "INSERT INTO libros (titulo, anio) VALUES (?, ?)",
        (libro.titulo, libro.anio)
    )
    nuevo_id = cursor.lastrowid
    con.commit()
    con.close()

    return {"id": nuevo_id, "titulo": libro.titulo, "anio": libro.anio}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Verificación HTTP del contrato (3 casos) y comprobación de persistencia en el archivo SQLite. Resultado: **3/3 casos correctos** y **1/1 comprobación correcta**.

| Caso | Resultado |
| :--- | :--- |
| GET `/api/libros` con base vacía → 200 `{"libros": []}` | ✅ |
| POST `/api/libros` `{"titulo":"Rayuela","anio":1963}` → 201 con `id: 1` | ✅ |
| GET `/api/libros` lee la base → 200 con Rayuela | ✅ |
| La fila existe en `biblioteca.db` (consulta directa con `sqlite3`) | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`). *Los `400` y `422` no se probaron en esta sesión.*
* [ ] No existen respuestas HTTP `500 Internal Server Error` no capturadas. *No se probó: ningún caso provoca un fallo de la base.*
* [ ] Encabezados `Content-Type: application/json` correctamente configurados. *FastAPI lo envía por defecto, pero no lo verifiqué de forma explícita.*

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> El verificador solo cubre el camino feliz, y el código pasa en verde aunque tiene omisiones:
>
> 1. **Sin manejo de errores de la base.** Si `biblioteca.db` está bloqueada, corrupta o inaccesible, `sqlite3.OperationalError` sube sin capturar y el cliente recibe un `500`. Lo correcto sería un `503 Service Unavailable` con un mensaje controlado.
> 2. **Conexiones sin `try/finally`.** Si falla el `execute`, `con.close()` nunca se ejecuta y la conexión queda abierta.
> 3. **Validación débil.** Pydantic acepta `titulo` vacío (`""`) y `anio` negativo o absurdo, que se guardan tal cual. Falta `Field(min_length=1)` y un rango para `anio`.
> 4. **Lo que sí está bien:** el SQL usa placeholders `?`, así que no hay inyección SQL en `titulo`.
>
> Corrección propuesta:
> ```python
> from contextlib import closing
> from pydantic import Field
>
> class LibroBase(BaseModel):
>     titulo: str = Field(min_length=1, max_length=200)
>     anio: int = Field(ge=0, le=2100)
>
> @app.post("/api/libros", response_model=LibroResponse, status_code=201)
> def crear_libro(libro: LibroBase):
>     try:
>         with closing(sqlite3.connect(DB_PATH)) as con:
>             cur = con.execute(
>                 "INSERT INTO libros (titulo, anio) VALUES (?, ?)",
>                 (libro.titulo, libro.anio),
>             )
>             con.commit()
>             return {"id": cur.lastrowid, "titulo": libro.titulo, "anio": libro.anio}
>     except sqlite3.OperationalError:
>         raise HTTPException(status_code=503, detail="Base de datos no disponible")
> ```

---
**Firma digital:** FernandaPalomo ✔

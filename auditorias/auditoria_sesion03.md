# Bitácora de Requerimientos y Auditoría IA — Sesión 03
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Headers, Content-Type y Negociación de Contenido  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Lectura y emisión manual de encabezados HTTP  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Al leer y emitir los encabezados a mano, lo más difícil fue entender la diferencia entre `Content-Type` (el formato de lo que yo envío) y `Accept` (el formato que quiero recibir), porque se parecen y es fácil mezclarlos. Además, los valores no siempre llegan limpios: pueden traer parámetros como `application/json; charset=utf-8` o varios formatos separados por comas, así que comparar el texto exacto a mano no es confiable. También hay que recordar qué código devolver en cada caso: 415 cuando el formato que se envía no es soportado y 406 cuando el cliente pide un formato que el servidor no puede entregar. Si se olvida alguna de estas validaciones, la API acepta cosas que no debería o falla con un error 500.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app.py`. Los libros se guardan en memoria (sin base de datos). |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | Negociación de contenido sobre `/api/libros`. `POST /api/libros` solo acepta `Content-Type: application/json`; con cualquier otro tipo (por ejemplo `text/plain`) responde 415. Un POST válido con `{"titulo", "anio"}` responde 201 con el libro creado y su `id` (empieza en 1). `GET /api/libros` solo responde si el cliente acepta `application/json`; si pide otro formato (por ejemplo `application/xml`) responde 406; si lo acepta responde 200. Todas las respuestas correctas llevan `Content-Type: application/json`. Invariante: `Content-Type` describe lo que el cliente envía y `Accept` lo que el cliente acepta recibir. |
| **Output (O)** | Solo código modular en `app.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: Trabajo con FastAPI 0.110+ y Python 3.10+, ejecutando el servidor con uvicorn dentro de Google Colab. Todo debe quedar en un solo archivo llamado app.py. Los datos se guardan en memoria; no uses base de datos.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Implementa la creación y el listado de libros (campos: titulo de tipo string y anio de tipo entero) con negociación de contenido.
1. POST /api/libros solo acepta Content-Type: application/json. Si el Content-Type es otro (por ejemplo text/plain), responde HTTP 415.
2. Un POST válido responde HTTP 201 con el libro creado y un id asignado por el servidor (empezando en 1).
3. GET /api/libros solo responde si el cliente acepta application/json (cabecera Accept). Si pide otro formato, como application/xml, responde HTTP 406. Si acepta JSON, responde HTTP 200.
4. Las respuestas correctas deben llevar el encabezado Content-Type: application/json.
5. Inspecciona request.headers antes de procesar el cuerpo. Usa modelos de Pydantic para validar la entrada.

Output: Entrégame únicamente el código de app.py, modular y con códigos HTTP semánticos, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
```python
from fastapi import FastAPI, Request, Response, status
from pydantic import BaseModel
from typing import List

app = FastAPI(title="Tu API negociará el formato exactamente así")

# Base de datos en memoria para almacenar los libros creados
libros_db = []

class LibroBase(BaseModel):
    titulo: str
    anio: int

class Libro(LibroBase):
    id: int

@app.post("/api/libros", status_code=status.HTTP_201_CREATED)
async def crear_libro(request: Request, response: Response):
    # 1. Validar Content-Type para POST (debe ser application/json)
    content_type = request.headers.get("content-type", "")
    if "application/json" not in content_type:
        return Response(
            content="Content-Type no soportado. Debe ser application/json.",
            status_code=status.HTTP_415_UNSUPPORTED_MEDIA_TYPE
        )

    # 2. Intentar parsear el JSON
    try:
        data = await request.json()
        libro_in = LibroBase(**data)
    except Exception:
        return Response(
            content="Cuerpo JSON inválido o mal formado.",
            status_code=status.HTTP_400_BAD_REQUEST
        )

    # 3. Guardar el nuevo libro
    nuevo_id = len(libros_db) + 1
    nuevo_libro = Libro(id=nuevo_id, titulo=libro_in.titulo, anio=libro_in.anio)
    libros_db.append(nuevo_libro)

    # Retornar el libro creado en JSON
    return nuevo_libro

@app.get("/api/libros")
async def listar_libros(request: Request, response: Response):
    # 1. Validar cabecera Accept
    # Si no se envía Accept, por defecto se asume que acepta cualquier formato
    accept = request.headers.get("accept", "*/*")

    # Si se pide explícitamente algo que no sea JSON o comodín, se rechaza con 406
    if "application/json" not in accept and "*/*" not in accept:
        return Response(
            content="Formato no aceptable. Esta API solo responde en application/json.",
            status_code=status.HTTP_406_NOT_ACCEPTABLE
        )

    return libros_db
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas con tipos de contenido no soportados (415)

**Resultado del verificador:** 4/4 casos correctos.

| # | Caso | Esperado | Resultado |
| :-: | :--- | :-: | :-: |
| 01 | POST válido en JSON (`POST /api/libros`) | 201 + `Content-Type: application/json` | ✅ |
| 02 | POST en texto plano se rechaza (`Content-Type: text/plain`) | 415 | ✅ |
| 03 | GET que pide XML se rechaza (`Accept: application/xml`) | 406 | ✅ |
| 04 | GET que acepta JSON responde (`Accept: application/json`) | 200 + `Content-Type: application/json` | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> El código pasó los 4 casos del verificador, pero al auditarlo encontré varios puntos débiles que el verificador no revisa:
>
> - **Código 400 en lugar de 422:** cuando el JSON está mal formado o falta un campo, la IA devuelve 400 con un `except Exception` genérico. En la Sesión 01 el contrato pedía 422 para estos casos, así que la API quedó inconsistente. Además, el `except` genérico oculta el tipo real del error. Se corregiría capturando `json.JSONDecodeError` (400) y `ValidationError` de Pydantic (422) por separado, o dejando que FastAPI valide el cuerpo con el modelo.
> - **Errores sin `Content-Type: application/json`:** las respuestas 415, 406 y 400 se devuelven como texto plano, sin encabezado JSON. Una API que negocia JSON debería devolver también sus errores en JSON, por ejemplo con `JSONResponse(status_code=415, content={"detail": "..."})`.
> - **Comparación frágil de encabezados:** se usa `"application/json" in content_type` y `"application/json" in accept`. Eso no interpreta los valores `q` del `Accept` (por ejemplo `application/json;q=0` indicaría que NO lo acepta) ni comodines como `application/*`, que debería aceptarse. También dejaría pasar valores raros que solo contengan ese texto.
> - **Formato de listado distinto al de la Sesión 02:** `GET /api/libros` devuelve una lista directa en lugar de `{"libros": [...]}`. El contrato de esta sesión no lo definía, pero rompe la consistencia de la API.
> - **Generación del `id`:** `len(libros_db) + 1` funciona porque aún no hay DELETE, pero generaría ids repetidos si se eliminaran libros.
> - **Detalles menores:** el parámetro `response` y la importación `List` no se usan, y el título de la API quedó con el texto de ejemplo.
>
> No apareció ningún error 500 en este laboratorio. Esta auditoría me mostró que pasar las pruebas no significa que el código sea correcto en todos los casos.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> El del 406, porque tuve que distinguirlo del 415: el 415 depende de lo que yo envío (`Content-Type`) y el 406 de lo que el cliente quiere recibir (`Accept`). Al principio parecen lo mismo, pero se validan en cabeceras distintas y en rutas distintas (POST y GET).

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Aceptar el comodín `*/*` y la ausencia de `Accept` como si el cliente aceptara JSON, y devolver 400 cuando el cuerpo JSON viene mal formado. El contrato no decía qué hacer en esos casos.

**¿Qué error detectó la verificación que tú no habías previsto?**
> La verificación no marcó ningún error en mi implementación. Lo que sí tuve que tener en cuenta es que también revisa que las respuestas correctas lleven `Content-Type: application/json`, y que el primer POST debe devolver el `id` 1, por lo que el servidor debe iniciar con la colección vacía.

---
**Firma digital:** FernandaPalomo

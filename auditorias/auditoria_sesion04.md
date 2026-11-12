# Bitácora de Requerimientos y Auditoría IA — Sesión 04
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Anatomía RESTful, URIs Nombres y Recursos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Diseño de URIs jerárquicas sin verbos en la ruta  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Lo más difícil fue decidir qué va en la ruta y qué va en la consulta. Mi primer impulso fue escribir rutas como `/api/obtenerLibrosAutor`, porque así pienso las funciones en Python, pero en REST la acción ya la dice el método `GET`. También me costó distinguir cuándo un dato **identifica** un recurso (`autor_id`, va en la ruta) y cuándo solo **filtra** el resultado (`genero` y `limite`, van después del `?`). A mano, además, hay que recordar convertir el `autor_id` a entero y comprobar si existe antes de buscar sus libros, porque si no se hace la API responde una lista vacía en lugar de un 404.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app.py`. Datos en memoria, sin base de datos. |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | Recursos anidados `GET /api/autores/{autor_id}/libros` (200 con `{autor_id, libros}`; 404 `{"detail": "Autor no encontrado"}` si el autor no existe) y colección `GET /api/libros` (200 con `{libros}` ordenados por id). Query params opcionales: `genero` (filtro, sin distinguir mayúsculas) y `limite` (entero 1-100; 422 fuera de rango). Sin verbos en las rutas. Semilla fija: autor 1 García Márquez (libros 1 y 2), autor 2 Cortázar (libro 3). |
| **Output (O)** | Solo código modular en `app.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: Trabajo con FastAPI 0.110+ y Python 3.10+, ejecutando el servidor con uvicorn en Google Colab. Todo debe quedar en un solo archivo app.py y los datos se guardan en memoria.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Diseña una API de autores y libros donde las rutas nombren recursos y no acciones.
1. GET /api/autores/{autor_id}/libros responde 200 con {"autor_id": ..., "libros": [...]}. Cada libro tiene id, titulo y genero.
2. Parámetros de consulta opcionales: genero filtra por género y limite recorta la lista (entero entre 1 y 100).
3. Si el autor no existe responde 404 con {"detail": "Autor no encontrado"}.
4. GET /api/libros responde 200 con {"libros": [...]} con todos los libros ordenados por id.
5. Siembra exactamente: autor 1 Gabriel García Márquez con «Cien años de soledad» (novela, id 1) y «Doce cuentos peregrinos» (cuento, id 2); autor 2 Julio Cortázar con «Rayuela» (novela, id 3).
6. No uses verbos en las rutas; usa modelos Pydantic para la salida.

Output: Entrégame únicamente el código de app.py, modular y con códigos HTTP semánticos, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
*Versión final de `app.py`, ya con las correcciones de la auditoría (sección 4).*
```python
from fastapi import FastAPI, HTTPException, Query
from typing import Optional, List
from pydantic import BaseModel

app = FastAPI(title="Anatomía RESTful, URIs Nombres y Recursos")

# Base de datos simulada en memoria
DATABASE = {
    "autores": {
        1: {
            "id": 1,
            "nombre": "Gabriel García Márquez",
            "libros": [
                {"id": 1, "titulo": "Cien años de soledad", "genero": "novela"},
                {"id": 2, "titulo": "Doce cuentos peregrinos", "genero": "cuento"}
            ]
        },
        2: {
            "id": 2,
            "nombre": "Julio Cortázar",
            "libros": [
                {"id": 3, "titulo": "Rayuela", "genero": "novela"}
            ]
        }
    }
}

# Modelos de salida Pydantic para estructura correcta
class Libro(BaseModel):
    id: int
    titulo: str
    genero: str

class AutorLibrosResponse(BaseModel):
    autor_id: int
    libros: List[Libro]

class LibrosCompletosResponse(BaseModel):
    libros: List[Libro]

@app.get("/api/autores/{autor_id}/libros", response_model=AutorLibrosResponse)
def obtener_libros_autor(
    autor_id: int,
    genero: Optional[str] = None,
    limite: Optional[int] = Query(default=None, ge=1, le=100)
):
    # 1. Verificar si el autor existe
    if autor_id not in DATABASE["autores"]:
        raise HTTPException(status_code=404, detail="Autor no encontrado")

    autor = DATABASE["autores"][autor_id]
    libros_filtrados = autor["libros"]

    # 2. Filtrar por género si se solicita
    if genero:
        libros_filtrados = [l for l in libros_filtrados if l["genero"].lower() == genero.lower()]

    # 3. Limitar cantidad de resultados
    if limite is not None:
        libros_filtrados = libros_filtrados[:limite]

    return {
        "autor_id": autor_id,
        "libros": libros_filtrados
    }

@app.get("/api/libros", response_model=LibrosCompletosResponse)
def obtener_todos_los_libros():
    # Obtiene todos los libros de todos los autores de manera plana
    todos_libros = []
    for autor in DATABASE["autores"].values():
        todos_libros.extend(autor["libros"])

    # Ordenamos por ID para asegurar consistencia
    todos_libros = sorted(todos_libros, key=lambda x: x["id"])
    return {"libros": todos_libros}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas de peticiones a recursos inexistentes (404)

**Resultado del verificador:** 4/4 casos correctos.

| # | Caso | Esperado | Resultado |
| :-: | :--- | :-: | :-: |
| 01 | Libros del autor 1 (`GET /api/autores/1/libros`) | 200 | ✅ |
| 02 | Filtro por género y límite (`GET /api/autores/1/libros?genero=cuento&limite=1`) | 200 | ✅ |
| 03 | Un autor inexistente produce 404 (`GET /api/autores/99/libros`) | 404 | ✅ |
| 04 | La colección completa de libros (`GET /api/libros`) | 200 | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> El código pasó los 4 casos, pero la auditoría encontró detalles que el verificador no revisa:
>
> - **`limite` sin validar:** la IA lo dejó como `Optional[int]` sin restricciones. Con `?limite=-1` el *slicing* de Python devolvía todos los libros menos el último y con `?limite=0` una lista vacía, sin error. **Corrección:** `Query(default=None, ge=1, le=100)`, que ahora responde 422.
> - **Nombre de la función con verbo:** la ruta está bien (`/api/autores/{autor_id}/libros`), pero la función se llama `obtener_libros_autor`. No afecta al contrato HTTP, aunque el `operationId` de OpenAPI hereda ese nombre.
> - **Falta `GET /api/libros/{id}` y `GET /api/autores/{id}`:** el contrato no los pedía, pero una API jerárquica normalmente expone también el recurso individual.
> - **El filtro de género no se valida:** un género inexistente devuelve una lista vacía con 200, lo cual es correcto en REST (el recurso existe, el filtro no encontró coincidencias).
>
> No apareció ningún error 500.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> El 404 del autor inexistente, porque al principio la lista vacía me parecía una respuesta válida. Entendí que no es lo mismo «el autor no tiene libros de ese género» (200 con lista vacía) que «el autor no existe» (404).

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Hacer el filtro de género sin distinguir mayúsculas (`Cuento` = `cuento`) y limitar `limite` entre 1 y 100.

**¿Qué error detectó la verificación que tú no habías previsto?**
> Ninguno: los 4 casos pasaron a la primera. El problema de `limite` negativo lo encontré yo al probar valores fuera del contrato.

---
**Firma digital:** FernandaPalomo


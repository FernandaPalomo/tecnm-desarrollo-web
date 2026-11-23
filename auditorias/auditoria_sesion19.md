# Bitácora de Requerimientos y Auditoría IA — Sesión 19
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Sanitización de Entradas, Prevención de XSS e Inyecciones  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** 23531112  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Limpieza manual de HTML y caracteres especiales  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Al intentar limpiar el texto a mano me di cuenta de que no basta con buscar y borrar la palabra `<script>`: el atacante puede escribirla como `<ScRiPt>`, con espacios, o usar otras etiquetas como `<img src=x onerror=alert(1)>` que también ejecutan código. Hacer una lista de etiquetas prohibidas con `replace()` es interminable y siempre deja huecos. También me costó decidir en qué momento limpiar (al recibir, al guardar o al mostrar) y evitar el doble escape, que convierte `&lt;` en `&amp;lt;` y deja el texto ilegible. Además, validar que el campo exista, no esté vacío y no sea demasiado largo implicaba muchos `if` repetidos en el endpoint.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) o PHP 8.2 con servidor embebido |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | Sanitizador de entradas y escapes de salida JSON/HTML |
| **Output (O)** | Solo código modular, esquemas Pydantic/DTO y códigos HTTP semánticos |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Estoy construyendo una API REST con FastAPI 0.110+ y Python 3.10+. No uso base de datos; los comentarios se guardan en una lista en memoria.

ROL: Actúa como arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

ESPECIFICACIÓN: Implementa un endpoint de comentarios que neutralice HTML peligroso (XSS).
- POST /api/comentarios recibe JSON {"texto": str} y responde 201 Created con {"id": int, "texto": str}.
- El texto se escapa con html.escape ANTES de guardarse, de modo que lo almacenado y lo devuelto ya es seguro. No se debe volver a escapar al listar (evitar doble escape).
- GET /api/comentarios responde 200 OK con {"comentarios": [{"id": int, "texto": str}]} sin etiquetas ejecutables.
- Un comentario normal (por ejemplo "Hola mundo") no debe alterarse.
- Validar con Pydantic: texto obligatorio, entre 1 y 500 caracteres, sin espacios en blanco únicamente. Si es inválido responder 422 Unprocessable Entity.
- La respuesta nunca debe contener la cadena <script> sin escapar.
- Ninguna entrada debe provocar un 500 Internal Server Error.
- Content-Type de todas las respuestas: application/json.

OUTPUT: Devuelve solo código Python modular (esquemas Pydantic de entrada y salida, y los dos endpoints) con códigos HTTP semánticos. Sin explicaciones ni texto adicional.
```

---

## 3. Código Devuelto por el Modelo
```python
# Código devuelto por el LLM
from html import escape

from fastapi import FastAPI
from pydantic import BaseModel, Field, field_validator

app = FastAPI(title="API de Comentarios Seguros")

_comentarios: list[dict] = []
_siguiente_id = 1


class ComentarioIn(BaseModel):
    texto: str = Field(..., min_length=1, max_length=500)

    @field_validator("texto")
    @classmethod
    def texto_no_vacio(cls, v: str) -> str:
        v = v.strip()
        if not v:
            raise ValueError("El texto no puede estar vacío")
        return v


class ComentarioOut(BaseModel):
    id: int
    texto: str


class ListaComentariosOut(BaseModel):
    comentarios: list[ComentarioOut]


@app.post("/api/comentarios", response_model=ComentarioOut, status_code=201)
def crear_comentario(entrada: ComentarioIn):
    global _siguiente_id
    nuevo = {
        "id": _siguiente_id,
        # Escape de salida: <, >, &, " y ' se convierten en entidades HTML
        "texto": escape(entrada.texto, quote=True),
    }
    _siguiente_id += 1
    _comentarios.append(nuevo)
    return nuevo


@app.get("/api/comentarios", response_model=ListaComentariosOut)
def listar_comentarios():
    # El texto ya se guardó escapado; no se vuelve a escapar para evitar doble escape
    return {"comentarios": _comentarios}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas de inyección de scripts en payloads

- ✅ Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
- ✅ No existen respuestas HTTP `500 Internal Server Error` no capturadas.
- ✅ Encabezados `Content-Type: application/json` correctamente configurados.

Resultado del verificador: comentario normal `201`, payload `<script>alert(1)</script>` `201` con el texto escapado (`&lt;script&gt;alert(1)&lt;/script&gt;`), listado `200` sin scripts ejecutables.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> En una primera versión, la IA solo escapaba el texto y no validaba la entrada: un `texto` vacío, compuesto solo de espacios o muy largo se aceptaba con `201`, cuando lo correcto es rechazarlo con `422 Unprocessable Entity`. También escapaba el texto una segunda vez en el `GET`, lo que producía doble escape (`&amp;lt;script&amp;gt;`). Lo corregí agregando `min_length`, `max_length` y un `field_validator` en el esquema Pydantic para que los datos inválidos devuelvan `422`, y moví el escape a un solo punto (al guardar) para que el listado devuelva el texto tal como quedó almacenado. Además aclaré en el prompt que la validación de entrada no sustituye al escape de salida: son dos defensas distintas y se necesitan ambas.

---
**Firma digital:** FernandaPalomo

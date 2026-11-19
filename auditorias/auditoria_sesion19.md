Bitácora de Requerimientos y Auditoría IA — Sesión 19

Materia: Desarrollo de Aplicaciones Web (DAB-2201)
Tema: Sanitización de Entradas, Prevención de XSS e Inyecciones
Docente Titular: MCC. Iván Márquez Larios (ijmarquezl)
Estudiante: Fernanda Palomo
Matrícula: [23531112]
Track: Python/FastAPI

1. Fase 1: El Artesano (Dificultad Manual)

Reto: Limpieza manual de HTML y caracteres especiales

(Borrador, ajústalo a tu experiencia) Lo más difícil fue decidir en qué punto limpiar el texto y qué caracteres cubrir. Hacerlo con replace() es fácil de olvidar: si escapaba < y > pero no &, el resultado quedaba inconsistente o se escapaba dos veces. Además, validar que el campo no estuviera vacío y que el JSON llegara bien formado se volvía código repetido en cada endpoint. Una lista manual de etiquetas prohibidas (como <script>) falla con variantes como <img onerror=...>, por eso es mejor escapar todo que buscar lo peligroso.

2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)
2.1 Matriz de Especificación Formal
Elemento C-R-E-O	Definición Técnica
Contexto (C)	FastAPI 0.110+ (Python 3.10+), almacenamiento en memoria, contrato POST/GET /api/comentarios
Rol (R)	Arquitecto de software senior especialista en APIs RESTful y Seguridad Web
Especificación (E)	Sanitizador de entradas y escapes de salida JSON/HTML
Output (O)	Solo código modular, esquemas Pydantic y códigos HTTP semánticos
2.2 Prompt Estructurado
text
CONTEXTO: Estoy construyendo una API con FastAPI 0.110+ y Python 3.10+. El
almacenamiento es en memoria (una lista). Los comentarios los verá un
frontend que los inserta en HTML.

ROL: Actúa como arquitecto de software senior especialista en APIs RESTful
y seguridad web (OWASP, prevención de XSS).

ESPECIFICACIÓN:
1. POST /api/comentarios recibe {"texto": str} y responde 201 con
   {"id": int, "texto": str}.
2. El texto se escapa con html.escape (convierte <, >, &) ANTES de guardarse,
   y se devuelve ya escapado. Un comentario normal no debe alterarse.
3. GET /api/comentarios responde 200 con {"comentarios": [...]} sin
   etiquetas ejecutables.
4. Valida con Pydantic: texto obligatorio, sin espacios solos, máximo 1000
   caracteres. Entradas inválidas -> 422, nunca 500.
5. Ninguna respuesta debe contener la cadena "<script>" sin escapar.

OUTPUT: Solo código Python modular (esquemas Pydantic, endpoints), con
códigos HTTP semánticos. Sin explicaciones fuera de comentarios en el código.
3. Código Devuelto por el Modelo
python
from html import escape
from itertools import count

from fastapi import FastAPI
from pydantic import BaseModel, Field, field_validator

app = FastAPI(title="API de Comentarios")

_comentarios: list[dict] = []
_ids = count(1)


class ComentarioIn(BaseModel):
    texto: str = Field(..., min_length=1, max_length=1000)

    @field_validator("texto")
    @classmethod
    def no_vacio(cls, v: str) -> str:
        v = v.strip()
        if not v:
            raise ValueError("El texto no puede estar vacío")
        return v


class ComentarioOut(BaseModel):
    id: int
    texto: str


@app.post("/api/comentarios", response_model=ComentarioOut, status_code=201)
def crear_comentario(entrada: ComentarioIn):
    # Escape ANTES de guardar; quote=False para no alterar comentarios normales
    nuevo = {"id": next(_ids), "texto": escape(entrada.texto, quote=False)}
    _comentarios.append(nuevo)
    return nuevo


@app.get("/api/comentarios")
def listar_comentarios():
    # Ya se guardan escapados: se devuelven tal cual (evita doble escape)
    return {"comentarios": _comentarios}
4. Fase 3: El Auditor (Resultados del Test Harness HTTP)

Pruebas ejecutadas: Pruebas de inyección de scripts en payloads

Prueba sugerida (pytest + TestClient) para que tengas evidencia real:

python
from fastapi.testclient import TestClient
from main import app

c = TestClient(app)

def test_normal():
    r = c.post("/api/comentarios", json={"texto": "Hola mundo"})
    assert r.status_code == 201 and r.json()["texto"] == "Hola mundo"

def test_script():
    r = c.post("/api/comentarios", json={"texto": "<script>alert(1)</script>"})
    assert r.status_code == 201 and "<script>" not in r.text

def test_listado():
    r = c.get("/api/comentarios")
    assert r.status_code == 200 and "<script>" not in r.text

def test_vacio():
    assert c.post("/api/comentarios", json={"texto": "   "}).status_code == 422

Marca estos checkboxes solo después de ejecutarla:

[ ] Respuestas semánticas correctas (200 OK, 201 Created, 422 Unprocessable).
[ ] No existen respuestas HTTP 500 Internal Server Error no capturadas.
[ ] Encabezados Content-Type: application/json correctamente configurados.
Reporte Crítico

(Borrador, ajústalo a lo que observaste) La primera versión de la IA usaba escape(..., quote=True), lo que cambiaba apóstrofes y comillas en comentarios normales (it's → it&#x27;s) y violaba el requisito “un comentario normal no se altera”; lo corregí con quote=False. También advertí el riesgo de doble escape: si se escapa al guardar y otra vez al devolver, el texto queda como &amp;lt;. Lo resolví escapando una sola vez, al guardar. Finalmente, agregué validación de texto vacío o de solo espacios para devolver 422 en lugar de aceptar basura. Limitación: html.escape protege en contexto de texto HTML, pero para atributos, URLs o JavaScript se necesita otro tratamiento, y lo ideal es escapar también en el frontend (por ejemplo usando textContent en lugar de innerHTML).

Firma digital: FernandaPalomo

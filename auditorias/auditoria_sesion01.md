# Bitácora de Requerimientos y Auditoría IA — Sesión 01
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Encuadre, Diagnóstico HTTP y Servidor en Colab  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Procesamiento manual de JSON de petición sin IA  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Al hacerlo a mano, la parte más difícil es que hay que revisar todo uno mismo: que el cuerpo llegue como JSON válido, que exista la clave `mensaje`, que sea texto y qué responder cuando algo falla. Si el JSON llega mal formado (por ejemplo, solo `{`), un código escrito a mano se rompe fácilmente y el servidor devuelve un error 500 en lugar de un error controlado. También es fácil confundirse con los códigos de estado: no es obvio cuándo corresponde 400, 405 o 422. Esto toma muchas líneas de `if`/`try` que se pueden olvidar.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) ejecutado con uvicorn dentro de Google Colab, en un solo archivo `app.py`. |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | `GET /api/estado` devuelve `{"servidor": "ok", "curso": "DAB-2201"}` con HTTP 200. `POST /api/eco` recibe un JSON con la clave `mensaje` (texto) y devuelve `{"mensaje": <valor recibido>, "recibido": true}` con HTTP 200. Un JSON mal formado o sin la clave `mensaje` debe responder HTTP 422 (nunca 500). Un método no permitido (por ejemplo `GET /api/eco`) debe responder HTTP 405. |
| **Output (O)** | Solo código modular en un archivo `app.py`, con un esquema Pydantic para el cuerpo del POST y códigos HTTP semánticos. Sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: Trabajo con FastAPI 0.110+ y Python 3.10+, ejecutando el servidor con uvicorn dentro de Google Colab. Todo debe quedar en un solo archivo llamado app.py.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación:
1. GET /api/estado debe devolver {"servidor": "ok", "curso": "DAB-2201"} con HTTP 200.
2. POST /api/eco debe recibir un JSON con la clave "mensaje" (string) y devolver {"mensaje": <el valor recibido>, "recibido": true} con HTTP 200.
3. Usa un modelo de Pydantic para validar el cuerpo del POST.
4. Si el JSON está mal formado o falta la clave "mensaje", la API no debe caerse: debe responder HTTP 422, nunca 500.
5. Un método no permitido sobre una ruta existente (por ejemplo GET /api/eco) debe responder HTTP 405.

Output: Entrégame únicamente el código de app.py, modular y con códigos HTTP semánticos, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="Tu servidor responderá exactamente así")

# Definimos el modelo de Pydantic para validar el cuerpo de la petición POST
class MensajeEco(BaseModel):
    mensaje: str

# Ruta GET /api/estado
@app.get("/api/estado")
async def obtener_estado():
    return {
        "servidor": "ok",
        "curso": "DAB-2201"
    }

# Ruta POST /api/eco
@app.post("/api/eco")
async def duplicar_mensaje(datos: MensajeEco):
    return {
        "mensaje": datos.mensaje,
        "recibido": True
    }
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas de códigos de estado HTTP 200, 400 y 422

**Resultado del verificador:** 4/4 casos correctos.

| # | Caso | Esperado | Resultado |
| :-: | :--- | :-: | :-: |
| 01 | El servidor está vivo y responde JSON (`GET /api/estado`) | 200 | ✅ |
| 02 | El eco devuelve el mensaje recibido (`POST /api/eco`) | 200 | ✅ |
| 03 | Un JSON roto no tumba el servidor (`POST /api/eco` con `{`) | 422 | ✅ |
| 04 | El método incorrecto se rechaza (`GET /api/eco`) | 405 | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> El código pasó los 4 casos del verificador, pero al auditarlo encontré detalles que la IA dejó sin cuidar:
>
> - **Título de la API:** dejó `title="Tu servidor responderá exactamente así"`, que es el texto de ejemplo del esqueleto y no un nombre real. Conviene cambiarlo por algo descriptivo, como `"API Sesión 01 - DAB-2201"`.
> - **Validación del campo `mensaje`:** el modelo solo exige que sea texto, así que acepta un mensaje vacío (`""`) o de tamaño enorme. Una mejora sería usar `Field(min_length=1, max_length=500)` para limitar la entrada.
> - **Códigos de error implícitos:** el 422 y el 405 no los programó la IA; los devuelve FastAPI por defecto. Es importante saber que este comportamiento viene del framework y no del código escrito.
>
> No apareció ningún error 500 ni falla de seguridad grave en este laboratorio, pero la auditoría sirvió para no pegar el código de la IA sin revisarlo.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> El del JSON roto (HTTP 422), porque a mano es el más fácil de olvidar. Con el modelo de Pydantic, FastAPI lo resuelve solo, pero antes de eso había que entender por qué responde 422 y no 400.

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Definir el modelo `MensajeEco` con el campo `mensaje: str` y usar funciones `async def` en las rutas. El contrato solo decía que Pydantic validaba el cuerpo, pero no cómo nombrar ni estructurar el modelo.

**¿Qué error detectó la verificación que tú no habías previsto?**
> La verificación no marcó ningún error en mi implementación. Lo que sí aprendí es que el caso 405 (método incorrecto) también debe cumplirse, y que FastAPI lo maneja automáticamente al definir la ruta `/api/eco` solo para POST.

---
**Firma digital:** FernandaPalomo

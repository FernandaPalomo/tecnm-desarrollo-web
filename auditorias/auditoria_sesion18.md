# Bitácora de Requerimientos y Auditoría IA — Sesión 18
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Rate Limiting, Throttling y Prevención de Abuso  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** 23531112  
**Track:** [ **Python/FastAPI** | PHP 8+ ]  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Contador manual de peticiones por IP en memoria  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Llevar el conteo a mano obliga a decidir muchas cosas que no se ven a simple vista. Hay que identificar al cliente (por IP o por token), guardar los momentos de sus peticiones, descartar las que ya salieron de la ventana de 60 segundos y calcular cuántos segundos faltan para el encabezado `Retry-After`. Un contador simple que se reinicia cada minuto permite ráfagas en el cambio de ventana (por ejemplo, 3 peticiones al segundo 59 y 3 más al segundo 61), por eso conviene guardar la hora de cada petición (ventana deslizante). También hay que acordarse de que el diccionario vive solo en la memoria de un proceso: si la app se reinicia o corre con varios workers, el conteo se pierde o se divide.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) o PHP 8.2 con servidor embebido |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | Rate Limiter deslizante con encabezados Retry-After |
| **Output (O)** | Solo código modular, esquemas Pydantic/DTO y códigos HTTP semánticos |

### 2.2 Prompt Estructurado
```text
Contexto: Trabajo con FastAPI 0.110+ en Python 3.10+, en un solo archivo app.py, ejecutado en Google Colab. Sin librerías externas de rate limiting; el estado se guarda en memoria y se reinicia al arrancar la aplicación.

Rol: Actúa como arquitecto de software senior especialista en APIs RESTful y seguridad web.

Especificación: Implementa un limitador de peticiones de ventana deslizante.
- GET /api/recurso permite 3 peticiones por cliente (identificado por IP) en una ventana de 60 segundos.
- Las 3 primeras responden 200 con un JSON simple.
- La cuarta responde 429 con el cuerpo {"error": "límite de peticiones alcanzado"} y el encabezado Retry-After con los segundos que faltan para que se libere un lugar en la ventana.
- Cada cliente tiene su propio contador; un cliente no debe afectar a otro.
- Las peticiones rechazadas no deben contarse dentro de la ventana.

Output: Solo código Python modular en app.py, con códigos HTTP semánticos. Sin explicaciones adicionales.
```

---

## 3. Código Devuelto por el Modelo
```python
# Código devuelto por el LLM
import time
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse

app = FastAPI(title="Tu límite de peticiones se verá así")

# Diccionario en memoria para rastrear las peticiones por IP del cliente
# Estructura: { ip: [timestamp1, timestamp2, ...] }
request_history = {}
WINDOW_SECONDS = 60
MAX_REQUESTS = 3

@app.get("/api/recurso")
async def obtener_recurso(request: Request):
    client_ip = request.client.host if request.client else "127.0.0.1"
    now = time.time()

    # Inicializar el historial de la IP si no existe
    if client_ip not in request_history:
        request_history[client_ip] = []

    # Filtrar peticiones para mantener solo las que están dentro de la ventana de 60 segundos
    request_history[client_ip] = [t for t in request_history[client_ip] if now - t < WINDOW_SECONDS]

    # Verificar si excede el límite
    if len(request_history[client_ip]) >= MAX_REQUESTS:
        # Calcular el tiempo restante para que expire la petición más antigua del bloque
        oldest_request = request_history[client_ip][0]
        retry_after = int(WINDOW_SECONDS - (now - oldest_request))
        if retry_after <= 0:
            retry_after = 1

        return JSONResponse(
            status_code=429,
            content={"error": "límite de peticiones alcanzado"},
            headers={"Retry-After": str(retry_after)}
        )

    # Registrar la petición actual y responder exitosamente
    request_history[client_ip].append(now)
    return {"mensaje": "petición permitida"}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas de ráfaga masiva y bloqueo temporal (HTTP 429)

Resultado del verificador: **4/4 casos correctos**.

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

| # | Caso | Esperado | Resultado |
| :-- | :--- | :--- | :--- |
| 01 | Primera petición permitida | 200 | ✅ |
| 02 | Segunda petición permitida | 200 | ✅ |
| 03 | Tercera petición permitida | 200 | ✅ |
| 04 | La cuarta petición se limita | 429 + `Retry-After` + `{"error": "límite de peticiones alcanzado"}` | ✅ |

### Cálculo de la ventana
Con `WINDOW_SECONDS = 60` y `MAX_REQUESTS = 3`, el tiempo de espera se calcula con la petición más antigua que sigue dentro de la ventana:

`Retry-After = WINDOW_SECONDS − (now − petición_más_antigua)`

Ejemplo: un cliente hace peticiones en los segundos 0, 5 y 10 (las tres dan 200). En el segundo 12 hace la cuarta:

- `now − petición_más_antigua = 12 − 0 = 12`
- `Retry-After = 60 − 12 = 48` segundos

Al segundo 60 la petición del segundo 0 sale de la ventana y se libera un lugar. Como la ventana es deslizante, no se libera todo de golpe: en el segundo 65 sale la del segundo 5, y así sucesivamente.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> El código `429` y el encabezado `Retry-After` están bien. Las fallas están en la robustez del limitador, que el verificador no revisa porque solo hace 4 peticiones desde una misma IP:
>
> 1. **`Retry-After` redondeado hacia abajo.** El cálculo usa `int()`, que trunca. Si faltan 48.7 segundos, se responde 48 y el cliente que obedece el encabezado reintenta antes de tiempo y recibe otro 429. Corrección: `math.ceil`.
> 2. **Fuga de memoria.** `request_history` crea una entrada por cada IP y nunca la borra. Un atacante que cambie de IP (o un tráfico grande) hace crecer el diccionario sin límite, lo que puede tumbar la API por falta de memoria. Corrección: eliminar la IP del diccionario cuando su lista queda vacía.
> 3. **Identificación del cliente.** `request.client.host` detrás de un proxy o balanceador (como en Colab o en producción) puede ser la IP del proxy, y entonces todos los usuarios comparten un solo contador, que es justo el contador global que el contrato pide evitar. Se debe leer `X-Forwarded-For` solo si el proxy es de confianza, o limitar por token de usuario.
> 4. **Estado en memoria de un solo proceso.** Con varios workers cada uno lleva su propio conteo y el límite real se multiplica; además se pierde al reiniciar. El contrato lo permite, pero en producción se usaría Redis u otro almacén compartido.
> 5. **`time.time()`** depende del reloj del sistema, que puede ajustarse; para medir intervalos es más seguro `time.monotonic()`.
>
> Corrección propuesta para los puntos 1, 2 y 5:
>
> ```python
> import math, time
>
> @app.get("/api/recurso")
> async def obtener_recurso(request: Request):
>     client_ip = request.client.host if request.client else "127.0.0.1"
>     now = time.monotonic()
>
>     historial = [t for t in request_history.get(client_ip, []) if now - t < WINDOW_SECONDS]
>
>     if len(historial) >= MAX_REQUESTS:
>         request_history[client_ip] = historial
>         retry_after = max(1, math.ceil(WINDOW_SECONDS - (now - historial[0])))
>         return JSONResponse(
>             status_code=429,
>             content={"error": "límite de peticiones alcanzado"},
>             headers={"Retry-After": str(retry_after)},
>         )
>
>     historial.append(now)
>     request_history[client_ip] = historial
>     return {"mensaje": "petición permitida"}
> ```
>
> Nota: este código no limpia las IP inactivas que nunca vuelven a escribir; para eso haría falta una tarea periódica que borre las entradas cuyo historial ya expiró.

---
**Firma digital:** FernandaPalomo

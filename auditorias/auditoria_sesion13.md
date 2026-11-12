# Bitácora de Requerimientos y Auditoría IA — Sesión 13
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Manejo de Sesiones Stateful vs Stateless Tokens  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Simulador de tabla de sesiones en memoria  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> El simulador de tabla de sesiones en memoria funcionó, pero al revisarlo vi varios problemas: el `session_id` lo generaba con `uuid`, la cookie no tenía `HttpOnly` ni `SameSite`, y las sesiones expiradas nunca se borraban de la tabla. También me costó comparar los dos modelos: con la tabla, cerrar sesión es tan fácil como borrar la fila, pero con un token firmado el servidor no guarda nada, así que no hay nada que borrar y el token sigue funcionando hasta que expira.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app_ia.py`. Datos en memoria, sin base de datos. |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | **Stateful:** `POST /api/sesion/login` crea `session_id` con `secrets.token_urlsafe(32)` y cookie `HttpOnly`, `SameSite=strict`, `Max-Age` (Secure por variable de entorno); `GET /api/sesion/perfil` 200/401; expiración validada en el servidor; `POST /api/sesion/logout` 204 y revocación inmediata. **Stateless:** `POST /api/token/login` emite token HMAC-SHA256 con `sub`, `exp` y `jti`; `GET /api/token/perfil` 200/401; `POST /api/token/logout` agrega el `jti` a una *denylist*. 401 con `WWW-Authenticate` en todos los rechazos. |
| **Output (O)** | Solo código modular en `app_ia.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: FastAPI 0.110+ y Python 3.10+ en Google Colab con uvicorn. Un solo archivo app_ia.py, todo en memoria y sin librerías JWT (eso es la sesión 15).

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Implementa y compara dos mecanismos de autenticación para el usuario de prueba fernanda / secreto123.
A) Sesión stateful con cookie:
  1. POST /api/sesion/login crea un id de sesión aleatorio e impredecible, lo guarda en una tabla en memoria con su expiración (TTL_SESION, variable de entorno) y lo envía en una cookie HttpOnly y SameSite=Strict.
  2. GET /api/sesion/perfil responde 200 con el usuario, o 401 si la sesión no existe, fue revocada o expiró (la expiración se valida en el servidor).
  3. POST /api/sesion/logout borra la sesión (204).
B) Token stateless:
  4. POST /api/token/login emite un token firmado con HMAC-SHA256 que contiene sub, exp (TTL_TOKEN) y jti.
  5. GET /api/token/perfil valida el encabezado Authorization: Bearer, la firma y la expiración.
  6. POST /api/token/logout revoca el token agregando su jti a una denylist.
Todos los rechazos responden 401 con WWW-Authenticate. Las credenciales incorrectas responden 401.

Output: Solo el código de app_ia.py, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
*Versión final de `app_ia.py`, ya con las correcciones de la auditoría (sección 4).*
```python
"""Comparativa: sesión stateful (cookie + tabla en servidor) vs token stateless.

- Stateful: el servidor guarda la sesión; revocar = borrar la fila.
- Stateless: el token firmado (HMAC-SHA256) lleva sus datos y su expiración;
  el servidor no guarda nada, por lo que revocar requiere una lista de
  revocación (denylist) por identificador de token (jti).
"""
import base64
import hashlib
import hmac
import json
import os
import secrets
import time
from typing import Annotated

from fastapi import Cookie, Depends, FastAPI, Header, HTTPException, Response, status
from pydantic import BaseModel, Field

SECRETO = os.getenv("SECRETO_TOKENS", secrets.token_hex(32)).encode()
TTL_SESION = int(os.getenv("TTL_SESION", "1800"))
TTL_TOKEN = int(os.getenv("TTL_TOKEN", "900"))
COOKIE = "session_id"

# Usuario de prueba (en la sesión 14 se guardará un hash, no el texto plano)
USUARIOS = {"fernanda": "secreto123"}

SESIONES: dict[str, dict] = {}       # stateful
TOKENS_REVOCADOS: set[str] = set()   # denylist para stateless


class Credenciales(BaseModel):
    usuario: str = Field(..., min_length=1, max_length=50)
    password: str = Field(..., min_length=1, max_length=128)


def no_autorizado(detalle: str, esquema: str = "Cookie") -> HTTPException:
    return HTTPException(status.HTTP_401_UNAUTHORIZED, detail=detalle, headers={"WWW-Authenticate": esquema})


def validar_credenciales(datos: Credenciales) -> str:
    esperado = USUARIOS.get(datos.usuario, "")
    # compare_digest en ambos casos para no filtrar si el usuario existe
    if not (secrets.compare_digest(datos.password, esperado) and datos.usuario in USUARIOS):
        raise no_autorizado("Credenciales inválidas", "Basic")
    return datos.usuario


app = FastAPI(title="Sesiones Stateful vs Tokens Stateless", version="1.3.0")


# =================== STATEFUL: cookie de sesión ===================
@app.post("/api/sesion/login")
def login_sesion(datos: Credenciales, response: Response):
    usuario = validar_credenciales(datos)
    sid = secrets.token_urlsafe(32)                       # id impredecible
    SESIONES[sid] = {"usuario": usuario, "expira": time.time() + TTL_SESION}
    response.set_cookie(
        COOKIE, sid, max_age=TTL_SESION, httponly=True, samesite="strict",
        secure=os.getenv("COOKIE_SECURE", "0") == "1",    # 1 en producción (HTTPS)
    )
    return {"mensaje": "Sesión iniciada", "expira_en": TTL_SESION}


def usuario_de_sesion(session_id: Annotated[str | None, Cookie()] = None) -> str:
    sesion = SESIONES.get(session_id or "")
    if sesion is None:
        raise no_autorizado("Sesión inexistente o revocada")
    if sesion["expira"] < time.time():
        SESIONES.pop(session_id, None)                      # limpieza perezosa
        raise no_autorizado("Sesión expirada")
    return sesion["usuario"]


@app.get("/api/sesion/perfil")
def perfil_sesion(usuario: Annotated[str, Depends(usuario_de_sesion)]):
    return {"usuario": usuario, "modo": "stateful"}


@app.post("/api/sesion/logout", status_code=status.HTTP_204_NO_CONTENT)
def logout_sesion(response: Response, session_id: Annotated[str | None, Cookie()] = None):
    SESIONES.pop(session_id or "", None)                    # revocación inmediata
    response.delete_cookie(COOKIE)
    response.status_code = status.HTTP_204_NO_CONTENT
    return response


# =================== STATELESS: token firmado ===================
def _b64(datos: bytes) -> str:
    return base64.urlsafe_b64encode(datos).rstrip(b"=").decode()


def _b64_dec(texto: str) -> bytes:
    return base64.urlsafe_b64decode(texto + "=" * (-len(texto) % 4))


def emitir_token(usuario: str) -> str:
    payload = {"sub": usuario, "exp": int(time.time()) + TTL_TOKEN, "jti": secrets.token_hex(8)}
    cuerpo = _b64(json.dumps(payload, separators=(",", ":")).encode())
    firma = _b64(hmac.new(SECRETO, cuerpo.encode(), hashlib.sha256).digest())
    return f"{cuerpo}.{firma}"


def leer_token(token: str) -> dict:
    try:
        cuerpo, firma = token.split(".")
        esperada = _b64(hmac.new(SECRETO, cuerpo.encode(), hashlib.sha256).digest())
        if not hmac.compare_digest(firma, esperada):
            raise ValueError("firma")
        payload = json.loads(_b64_dec(cuerpo))
    except Exception:
        raise no_autorizado("Token inválido", "Bearer")
    if payload["exp"] < time.time():
        raise no_autorizado("Token expirado", "Bearer")
    if payload["jti"] in TOKENS_REVOCADOS:
        raise no_autorizado("Token revocado", "Bearer")
    return payload


def payload_de_token(authorization: Annotated[str | None, Header()] = None) -> dict:
    if not authorization or not authorization.startswith("Bearer "):
        raise no_autorizado("Falta el encabezado Authorization: Bearer", "Bearer")
    return leer_token(authorization.removeprefix("Bearer "))


@app.post("/api/token/login")
def login_token(datos: Credenciales):
    return {"access_token": emitir_token(validar_credenciales(datos)), "token_type": "bearer", "expira_en": TTL_TOKEN}


@app.get("/api/token/perfil")
def perfil_token(payload: Annotated[dict, Depends(payload_de_token)]):
    return {"usuario": payload["sub"], "modo": "stateless"}


@app.post("/api/token/logout", status_code=status.HTTP_204_NO_CONTENT)
def logout_token(payload: Annotated[dict, Depends(payload_de_token)]):
    TOKENS_REVOCADOS.add(payload["jti"])                   # sin esto, el token seguiría vivo
    return Response(status_code=status.HTTP_204_NO_CONTENT)
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas de expiración y revocación de sesión

**Resultado del Test Harness:** 11/11 casos correctos (A1 = prueba de humo de la versión artesanal; 01 en adelante = versión auditada `app_ia.py`).

| # | Caso | Detalle observado | Resultado |
| :-: | :--- | :--- | :-: |
| A1 | Artesano: perfil con cookie → 200 | HTTP 200 | ✅ |
| 01 | Login incorrecto → 401 | HTTP 401 | ✅ |
| 02 | Login → 200 con cookie HttpOnly + SameSite=strict | session_id=M7fKKCajjQcnULWgxSo_MyBAGtjXsLzZdQqX1jpNOd4; HttpOnly; Max-... | ✅ |
| 03 | Perfil con sesión → 200 | HTTP 200 | ✅ |
| 04 | Revocación: cookie robada tras logout → 401 | logout 204, reuso 401 | ✅ |
| 05 | Expiración en servidor: sesión vencida → 401 | HTTP 401 {'detail': 'Sesión expirada'} | ✅ |
| 06 | Token válido → 200 | HTTP 200 | ✅ |
| 07 | Token alterado → 401 | HTTP 401 | ✅ |
| 08 | Revocación stateless con denylist (jti) → 401 | logout 204, reuso 401 | ✅ |
| 09 | Expiración: token vencido → 401 | HTTP 401 {'detail': 'Token expirado'} | ✅ |
| 10 | Sin Authorization → 401 con WWW-Authenticate: Bearer | HTTP 401 | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> Los 11 casos quedaron en verde. Hallazgos:
>
> - **Expiración confiada al navegador:** la primera prueba de expiración (caso 05) falló porque el cliente borró la cookie por su `Max-Age` y nunca la envió; eso no prueba nada sobre el servidor. Reenvié la cookie «robada» a mano después de vencida y confirmé que el servidor la rechaza con «Sesión expirada».
> - **Logout stateless sin denylist no revoca nada:** si `POST /api/token/logout` solo respondiera 204, el mismo token seguiría dando 200 hasta expirar, porque el servidor no guarda nada. Esa es la gran diferencia con las sesiones. Por eso el token lleva `jti` y el logout lo agrega a una *denylist* (caso 08).
> - **Cookie sin `Secure`:** en Colab se usa HTTP, así que `Secure` se activa con `COOKIE_SECURE=1`; en producción debe ser obligatoria.
> - **Contraseña en texto plano** en el diccionario de usuarios: se corrige en la sesión 14 con bcrypt.
> - **La denylist y la tabla crecen sin límite:** habría que purgar periódicamente las entradas vencidas.
>
> Sin errores 500.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> La revocación del token stateless, porque por diseño el servidor no recuerda nada y tuve que agregar estado (la *denylist*) para poder invalidarlo.

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Usar `SameSite=Strict` y `secrets.token_urlsafe(32)` para el id de sesión.

**¿Qué error detectó la verificación que tú no habías previsto?**
> Que mi prueba de expiración dependía del cliente y no del servidor (caso 05).

---
**Firma digital:** FernandaPalomo

# Bitácora de Requerimientos y Auditoría IA — Sesión 15
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Autenticación Stateless con JSON Web Tokens (JWT)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Codificación Base64Url y firma HMAC-SHA256  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Construir un JWT a mano me ayudó a ver que no está cifrado, solo codificado: el *header* y el *payload* son JSON en Base64Url (sin el relleno `=`) y cualquiera puede leerlos. Lo que lo protege es la firma HMAC-SHA256 sobre `header.payload`. Me costó el detalle del relleno al decodificar (hay que volver a agregar los `=`), comparar la firma con `hmac.compare_digest` y validar la expiración. Al final comprobé que mi token manual lo acepta PyJWT, así que el formato quedó correcto.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app_ia.py`. Datos en memoria, sin base de datos. |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | `POST /api/token` valida credenciales (bcrypt, hash señuelo) y emite JWT HS256 con `sub`, `rol`, `iss`, `aud`, `iat`, `nbf`, `exp` (15 min) y `jti`. Secreto desde `SECRETO_JWT` (≥ 32 bytes). `GET /api/protegido` con `HTTPBearer`: `jwt.decode` con `algorithms=["HS256"]`, emisor y audiencia obligatorios. Token expirado → 401 «Token expirado»; alterado, otra firma, `alg=none`, otra audiencia o malformado → 401 «Token inválido»; sin encabezado → 401 con `WWW-Authenticate: Bearer`. |
| **Output (O)** | Solo código modular en `app_ia.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: FastAPI 0.110+, Python 3.10+ y PyJWT en Google Colab con uvicorn. Un solo archivo app_ia.py, usuario de prueba fernanda / secreto123 guardado con bcrypt.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Autenticación stateless con JWT.
1. POST /api/token valida las credenciales y responde 200 con {"access_token", "token_type": "bearer", "expira_en"}; 401 si son incorrectas.
2. El token usa HS256 y contiene sub, rol, iss, aud, iat, nbf, exp (15 minutos) y jti. El secreto se lee de la variable SECRETO_JWT; nunca debe quedar fijo en el código.
3. GET /api/protegido exige Authorization: Bearer <token> (usa HTTPBearer) y responde 200 con el usuario y su rol.
4. jwt.decode debe recibir una lista blanca de algoritmos y validar emisor, audiencia y expiración.
5. Responde 401 con WWW-Authenticate: Bearer si falta el token, si está alterado, si está firmado con otra clave, si usa alg=none, si es de otra audiencia, si está mal formado o si expiró ("Token expirado").

Output: Solo el código de app_ia.py, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
*Versión final de `app_ia.py`, ya con las correcciones de la auditoría (sección 4).*
```python
"""Autenticación stateless con JWT (PyJWT, HS256)."""
import os
import secrets
import uuid
from datetime import datetime, timedelta, timezone
from typing import Annotated

import bcrypt
import jwt
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from pydantic import BaseModel, Field

# El secreto viene del entorno; si falta, se genera uno aleatorio (nunca uno fijo en el código)
SECRETO_JWT = os.getenv("SECRETO_JWT") or secrets.token_urlsafe(48)
ALGORITMO = "HS256"
EMISOR = "tecnm-dab2201"
AUDIENCIA = "biblioteca-api"
MINUTOS_EXPIRACION = int(os.getenv("MINUTOS_EXPIRACION", "15"))

# Usuario de prueba guardado con hash bcrypt (sesión 14)
USUARIOS = {"fernanda": {"hash": bcrypt.hashpw(b"secreto123", bcrypt.gensalt(12)), "rol": "lector"}}
HASH_SENUELO = bcrypt.hashpw(secrets.token_bytes(16), bcrypt.gensalt(12))

app = FastAPI(title="Biblioteca API - Autenticación JWT", version="1.5.0")
esquema_bearer = HTTPBearer(auto_error=False)


class Credenciales(BaseModel):
    usuario: str = Field(..., min_length=1, max_length=50)
    password: str = Field(..., min_length=1, max_length=72)


class TokenOut(BaseModel):
    access_token: str
    token_type: str = "bearer"
    expira_en: int


def error_401(detalle: str) -> HTTPException:
    return HTTPException(status.HTTP_401_UNAUTHORIZED, detail=detalle, headers={"WWW-Authenticate": "Bearer"})


def crear_token(usuario: str, rol: str) -> str:
    ahora = datetime.now(timezone.utc)
    payload = {
        "sub": usuario, "rol": rol, "iss": EMISOR, "aud": AUDIENCIA,
        "iat": ahora, "nbf": ahora, "exp": ahora + timedelta(minutes=MINUTOS_EXPIRACION),
        "jti": uuid.uuid4().hex,
    }
    return jwt.encode(payload, SECRETO_JWT, algorithm=ALGORITMO)


def decodificar_token(token: str) -> dict:
    try:
        return jwt.decode(
            token, SECRETO_JWT,
            algorithms=[ALGORITMO],                 # lista blanca: bloquea alg=none y confusión de algoritmos
            audience=AUDIENCIA, issuer=EMISOR,
            options={"require": ["exp", "iat", "sub", "iss", "aud"]},
        )
    except jwt.ExpiredSignatureError:
        raise error_401("Token expirado")
    except jwt.InvalidTokenError:
        raise error_401("Token inválido")


def usuario_actual(cred: Annotated[HTTPAuthorizationCredentials | None, Depends(esquema_bearer)]) -> dict:
    if cred is None or cred.scheme.lower() != "bearer":
        raise error_401("Falta el encabezado Authorization: Bearer <token>")
    return decodificar_token(cred.credentials)


@app.post("/api/token", response_model=TokenOut)
def emitir_token(datos: Credenciales):
    registro = USUARIOS.get(datos.usuario)
    valido = bcrypt.checkpw(datos.password.encode(), registro["hash"] if registro else HASH_SENUELO)
    if not (registro and valido):
        raise error_401("Credenciales inválidas")
    return {"access_token": crear_token(datos.usuario, registro["rol"]), "expira_en": MINUTOS_EXPIRACION * 60}


@app.get("/api/protegido")
def recurso_protegido(payload: Annotated[dict, Depends(usuario_actual)]):
    return {"usuario": payload["sub"], "rol": payload["rol"], "mensaje": "Acceso concedido"}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas con tokens alterados o firma inválida

**Resultado del Test Harness:** 11/11 casos correctos (A1 = prueba de humo de la versión artesanal; 01 en adelante = versión auditada `app_ia.py`).

| # | Caso | Detalle observado | Resultado |
| :-: | :--- | :--- | :-: |
| A1 | Artesano: token manual válido y compatible con PyJWT | HTTP 200 | ✅ |
| 01 | Credenciales incorrectas → 401 | HTTP 401 | ✅ |
| 02 | Login → 200 con access_token tipo bearer | HTTP 200 | ✅ |
| 03 | Token válido → 200 | HTTP 200 {'usuario': 'fernanda', 'rol': 'lector', 'mensaje': 'Acceso concedido'} | ✅ |
| 04 | Payload alterado (rol=admin) → 401 | HTTP 401 | ✅ |
| 05 | Firmado con otra clave → 401 | HTTP 401 | ✅ |
| 06 | Ataque alg=none → 401 | HTTP 401 | ✅ |
| 07 | Token expirado → 401 'Token expirado' | HTTP 401 {'detail': 'Token expirado'} | ✅ |
| 08 | Token de otra audiencia → 401 | HTTP 401 | ✅ |
| 09 | Sin Authorization → 401 + WWW-Authenticate: Bearer | HTTP 401 | ✅ |
| 10 | Token malformado → 401 (no 500) | HTTP 401 | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> Los 11 casos pasaron, incluidos el payload alterado (`rol=admin`), la firma con otra clave y el ataque `alg=none`. Hallazgos:
>
> - **Secreto fijo en el código:** mi versión artesanal (y muchos ejemplos de JWT) tiene el secreto escrito en el archivo; cualquiera con acceso al repositorio podría firmar tokens válidos. **Corrección:** se lee de `SECRETO_JWT` y, si no existe, se genera uno aleatorio con `secrets.token_urlsafe(48)` (los tokens dejan de valer al reiniciar, lo cual es preferible a un secreto conocido).
> - **Clave corta:** mi primer secreto artesanal tenía 26 bytes y PyJWT mostró `InsecureKeyLengthWarning` (RFC 7518 pide al menos 32 bytes para HS256). Lo amplié a 40 bytes.
> - **Sin `aud` ni `iss`:** un token emitido para otra API con el mismo secreto se habría aceptado. **Corrección:** `audience` e `issuer` obligatorios (caso 08).
> - **Mensajes internos expuestos:** mi versión artesanal devuelve `str(exc)` al cliente, lo que filtra detalles de la validación. **Corrección:** solo «Token expirado» o «Token inválido».
> - **Sin revocación ni refresh token:** como vimos en la sesión 13, un JWT robado vale hasta que expira; por eso la expiración se dejó en 15 minutos.
>
> Sin errores 500.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> Entender por qué hay que pasar `algorithms=["HS256"]` aunque el token ya diga su algoritmo: si el servidor confía en el *header*, un atacante puede pedir `none`.

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Validar emisor y audiencia, y no exponer el mensaje interno de PyJWT.

**¿Qué error detectó la verificación que tú no habías previsto?**
> El aviso de clave corta de PyJWT, que no esperaba en mi versión artesanal.

---
**Firma digital:** FernandaPalomo

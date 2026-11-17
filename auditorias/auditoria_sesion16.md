# Bitácora de Requerimientos y Auditoría IA — Sesión 16
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** OAuth2 Password Flow y Header Authorization Bearer  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [23531112]  
**Track:** [ **Python/FastAPI** | PHP 8+ ]  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Extracción manual del encabezado Authorization  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Hacerlo a mano implica leer el encabezado `Authorization`, separarlo en esquema y token (`Bearer <token>`), comparar el esquema sin distinguir mayúsculas y minúsculas, y manejar los casos en que el encabezado no existe, viene vacío o usa otro esquema (por ejemplo `Basic`). En cada caso hay que responder `401` y acordarse de agregar `WWW-Authenticate: Bearer`, porque sin ese reto el cliente no sabe qué esquema usar. Además, el login no llega como JSON sino como formulario (`application/x-www-form-urlencoded`), así que también hay que decodificar `username` y `password` manualmente. Es fácil olvidar un caso borde y dejar una ruta mal protegida.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) o PHP 8.2 con servidor embebido |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | Esquema OAuth2PasswordBearer y endpoint /token |
| **Output (O)** | Solo código modular, esquemas Pydantic/DTO y códigos HTTP semánticos |

### 2.2 Prompt Estructurado
```text
Contexto: Trabajo con FastAPI 0.110+ en Python 3.10+, en un solo archivo app.py, ejecutado en Google Colab. Dependencias disponibles: fastapi, python-multipart, PyJWT.

Rol: Actúa como arquitecto de software senior especialista en APIs RESTful y seguridad web.

Especificación: Implementa el Password Flow de OAuth2.
- POST /token recibe username y password como formulario (application/x-www-form-urlencoded), no como JSON, usando OAuth2PasswordRequestForm. El usuario de prueba es "ana" con contraseña "secreta".
- Si las credenciales son correctas, responde 200 con {"access_token": "...", "token_type": "bearer"}. Si son incorrectas, responde 400.
- GET /api/perfil está protegida con OAuth2PasswordBearer(tokenUrl="token") y exige el encabezado Authorization: Bearer <token>. Con token válido responde 200 con {"usuario": "ana"}.
- Sin token, o con un esquema distinto de Bearer (por ejemplo Basic), responde 401 e incluye el encabezado WWW-Authenticate: Bearer.
- Un token inválido o alterado también responde 401 con WWW-Authenticate: Bearer.

Output: Solo código Python modular en app.py, con esquemas Pydantic para la respuesta del token y códigos HTTP semánticos. Sin explicaciones adicionales.
```

---

## 3. Código Devuelto por el Modelo
```python
# Código devuelto por el LLM
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

app = FastAPI(title="Tu OAuth2 Password Flow se verá así")

# Configuramos el esquema OAuth2 con la ruta del token de login
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    # El contrato exige: usuario "ana" con contraseña "secreta"
    if form_data.username == "ana" and form_data.password == "secreta":
        return {
            "access_token": form_data.username,  # Usamos su nombre como token simple para esta sesión
            "token_type": "bearer"
        }

    # Si las credenciales no son correctas, se responde con un error 400 o 401
    raise HTTPException(
        status_code=status.HTTP_400_BAD_REQUEST,
        detail="Incorrect username or password"
    )

@app.get("/api/perfil")
async def leer_perfil(token: str = Depends(oauth2_scheme)):
    # El contrato establece que el token del usuario debe corresponder a "ana"
    if token != "ana":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token inválido o expirado",
            headers={"WWW-Authenticate": "Bearer"}
        )
    return {"usuario": "ana"}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas de acceso no autorizado sin token (HTTP 401)

Resultado del verificador: **4/4 casos correctos** y **1/1 comprobación de archivo** (se usa el flujo OAuth2 de FastAPI).

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

| # | Caso | Esperado | Resultado |
| :-- | :--- | :--- | :--- |
| 01 | Login por formulario emite token | 200 + `token_type: bearer` | ✅ |
| 02 | Token Bearer abre el perfil | 200 + `{"usuario": "ana"}` | ✅ |
| 03 | Sin token | 401 + `WWW-Authenticate: Bearer` | ✅ |
| 04 | Esquema distinto (`Basic`) | 401 | ✅ |

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> La falla de seguridad más importante es que la IA usó el nombre de usuario como token (`"access_token": form_data.username`). Aunque el verificador dio 4/4, cualquier persona puede enviar `Authorization: Bearer ana` sin hacer login y entrar al perfil, porque el token no está firmado ni tiene expiración. Los casos del contrato solo comprueban que el flujo funcione, no que el token sea falsificable.
>
> **Corrección:** emitir un JWT firmado con expiración (PyJWT, que ya está instalado en el entorno) y validarlo en la ruta protegida, respondiendo 401 con `WWW-Authenticate: Bearer` si es inválido o está vencido:
>
> ```python
> import jwt
> from datetime import datetime, timedelta, timezone
>
> SECRET_KEY = "cambia-esta-clave"  # en producción, desde una variable de entorno
> ALGORITHM = "HS256"
>
> def crear_token(usuario: str) -> str:
>     exp = datetime.now(timezone.utc) + timedelta(minutes=30)
>     return jwt.encode({"sub": usuario, "exp": exp}, SECRET_KEY, algorithm=ALGORITHM)
>
> # En /token: "access_token": crear_token(form_data.username)
>
> # En /api/perfil:
> try:
>     payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
> except jwt.PyJWTError:
>     raise HTTPException(status_code=401, detail="Token inválido o expirado",
>                         headers={"WWW-Authenticate": "Bearer"})
> return {"usuario": payload["sub"]}
> ```
>
> Otras observaciones: las credenciales (`ana` / `secreta`) están escritas directamente en el código, algo aceptable solo para este laboratorio; en un sistema real se guardarían con hash (bcrypt) en una base de datos. Además, el login fallido responde 400, que es lo que usa la documentación de FastAPI, y el `401` queda reservado para el acceso a rutas protegidas.

---
**Firma digital:** [FernandaPalomo]

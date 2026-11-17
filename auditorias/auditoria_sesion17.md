# Bitácora de Requerimientos y Auditoría IA — Sesión 17
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Control de Acceso Basado en Roles (RBAC)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [23531112]  
**Track:** [ **Python/FastAPI** | PHP 8+ ]  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Verificación manual de permisos de usuario (Admin/User)  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Hacerlo a mano obliga a repetir en cada ruta la misma lógica: leer el encabezado `Authorization`, decodificar y validar el token, sacar el rol y comparar con el permitido. Lo más difícil es no mezclar las dos preguntas: primero hay que autenticar (¿quién eres?) y devolver `401` si no hay token o es inválido, y solo después autorizar (¿tu rol alcanza?) y devolver `403`. Si se revisa el rol antes de confirmar la identidad, a quien no tiene token se le puede responder `403` por error. Además, al copiar la verificación `if rol != "admin"` en cada endpoint es fácil olvidarla en alguno y dejar una ruta administrativa abierta.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) o PHP 8.2 con servidor embebido |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | Middleware de roles dinámico basado en claims JWT |
| **Output (O)** | Solo código modular, esquemas Pydantic/DTO y códigos HTTP semánticos |

### Matriz rol → permiso
| Rol | `POST /api/token` | `GET /api/perfil` | `GET /api/admin/panel` |
| :--- | :---: | :---: | :---: |
| Sin token (anónimo) | ✅ 200 (con credenciales válidas) | ❌ 401 | ❌ 401 |
| `user` (ana) | ✅ 200 | ✅ 200 | ❌ 403 |
| `admin` (admin) | ✅ 200 | ✅ 200 | ✅ 200 |

### 2.2 Prompt Estructurado
```text
Contexto: Trabajo con FastAPI 0.110+ en Python 3.10+, en un solo archivo app.py, ejecutado en Google Colab. Dependencias disponibles: fastapi, pydantic, PyJWT.

Rol: Actúa como arquitecto de software senior especialista en APIs RESTful y seguridad web.

Especificación: Implementa autenticación JWT con control de acceso basado en roles (RBAC).
- Usuarios de prueba: admin/admin123 (rol admin) y ana/secreta (rol user).
- POST /api/token recibe JSON {"usuario", "password"} y responde 200 con {"access_token", "token_type": "bearer"}. El token debe incluir en sus claims el usuario (sub), el rol y una expiración. Credenciales incorrectas responden 401.
- Crea una dependencia que valide el token Bearer y responda 401 con WWW-Authenticate: Bearer si falta, es inválido o expiró.
- Crea una dependencia reutilizable que exija un rol y responda 403 si el usuario autenticado no lo tiene.
- GET /api/admin/panel responde 200 con {"panel": "admin"} solo para rol admin; un usuario con rol user recibe 403 y sin token se recibe 401 (no 403).
- GET /api/perfil responde 200 con {"usuario", "rol"} para cualquier usuario autenticado.

Output: Solo código Python modular en app.py, con esquemas Pydantic para la petición de login y la respuesta del token, y códigos HTTP semánticos. Sin explicaciones adicionales.
```

---

## 3. Código Devuelto por el Modelo
```python
# Código devuelto por el LLM
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel
from datetime import datetime, timedelta
import jwt

# Configuración de JWT
SECRET_KEY = "super-secret-key-for-rbac-session-17"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

app = FastAPI(title="Sesión 17 - RBAC")
security = HTTPBearer(auto_error=False)

# Base de datos simulada en memoria
USERS_DB = {
    "admin": {"password": "admin123", "rol": "admin"},
    "ana": {"password": "secreta", "rol": "user"}
}

# Modelos de Pydantic
class LoginRequest(BaseModel):
    usuario: str
    password: str

class TokenResponse(BaseModel):
    access_token: str
    token_type: str

# Función para crear el token de acceso
def crear_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

# Dependencia para obtener y validar el usuario actual desde el JWT
def obtener_usuario_actual(credentials: HTTPAuthorizationCredentials = Depends(security)):
    if not credentials:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="No autenticado",
            headers={"WWW-Authenticate": "Bearer"},
        )
    token = credentials.credentials
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        usuario: str = payload.get("sub")
        rol: str = payload.get("rol")
        if usuario is None or rol is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Token inválido",
                headers={"WWW-Authenticate": "Bearer"},
            )
        return {"usuario": usuario, "rol": rol}
    except jwt.PyJWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token inválido o expirado",
            headers={"WWW-Authenticate": "Bearer"},
        )

# Dependencia para requerir un rol específico
class RequerirRol:
    def __init__(self, rol_permitido: str):
        self.rol_permitido = rol_permitido

    def __call__(self, usuario_actual: dict = Depends(obtener_usuario_actual)):
        if usuario_actual["rol"] != self.rol_permitido:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="No tiene permisos suficientes para acceder a este recurso"
            )
        return usuario_actual

# Endpoint de autenticación (Login)
@app.post("/api/token", response_model=TokenResponse)
def login(payload: LoginRequest):
    user = USERS_DB.get(payload.usuario)
    if not user or user["password"] != payload.password:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Credenciales incorrectas"
        )

    # Guardamos el rol y el usuario (sub) en los claims del token
    token = crear_access_token({"sub": payload.usuario, "rol": user["rol"]})
    return {"access_token": token, "token_type": "bearer"}

# Endpoint protegido para administradores
@app.get("/api/admin/panel", dependencies=[Depends(RequerirRol("admin"))])
def admin_panel():
    return {"panel": "admin"}

# Endpoint protegido para cualquier usuario autenticado
@app.get("/api/perfil")
def obtener_perfil(usuario_actual: dict = Depends(obtener_usuario_actual)):
    return {"usuario": usuario_actual["usuario"], "rol": usuario_actual["rol"]}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas de acceso denegado por rol insuficiente (HTTP 403)

Resultado del verificador: **6/6 casos correctos**.

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

| # | Caso | Esperado | Resultado |
| :-- | :--- | :--- | :--- |
| 01 | El administrador obtiene su token | 200 | ✅ |
| 02 | El usuario normal obtiene su token | 200 | ✅ |
| 03 | El admin entra al panel | 200 + `{"panel": "admin"}` | ✅ |
| 04 | El usuario normal recibe 403 | 403 | ✅ |
| 05 | Sin token recibe 401 | 401 | ✅ |
| 06 | El perfil funciona para el usuario normal | 200 + `{"usuario": "ana", "rol": "user"}` | ✅ |

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> Los códigos HTTP quedaron bien: la IA separó correctamente `401` (sin token, token inválido o vencido, con `WWW-Authenticate: Bearer`) de `403` (autenticado pero sin permiso). Las fallas están en la seguridad del código, que el verificador no revisa:
>
> 1. **Clave secreta escrita en el código.** `SECRET_KEY` está fija en `app.py`; si el archivo se sube a GitHub, cualquiera podría firmar tokens con rol `admin`. Corrección: leerla de una variable de entorno.
> 2. **`datetime.utcnow()` está obsoleto** desde Python 3.12 (el entorno usa 3.13) y genera una advertencia. Corrección: `datetime.now(timezone.utc)`.
> 3. **Contraseñas en texto plano** en `USERS_DB`. Aceptable solo para el laboratorio; en un sistema real se guardan con hash (bcrypt, ya instalado vía passlib).
> 4. **`RequerirRol` solo acepta un rol** y compara por igualdad, por lo que no escala si más adelante hay varios roles con acceso a una misma ruta. Corrección: recibir una lista de roles permitidos.
> 5. **El rol viaja en el token** y se confía en él hasta que expira (30 min): si a un usuario le quitan el rol de admin, su token anterior sigue funcionando hasta vencer.
>
> Corrección propuesta para los puntos 1 a 4:
>
> ```python
> import os
> from datetime import datetime, timedelta, timezone
>
> SECRET_KEY = os.environ.get("SECRET_KEY", "solo-para-desarrollo")
>
> def crear_access_token(data: dict):
>     to_encode = data.copy()
>     expire = datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
>     to_encode.update({"exp": expire})
>     return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
>
> class RequerirRol:
>     def __init__(self, *roles_permitidos: str):
>         self.roles_permitidos = roles_permitidos
>
>     def __call__(self, usuario_actual: dict = Depends(obtener_usuario_actual)):
>         if usuario_actual["rol"] not in self.roles_permitidos:
>             raise HTTPException(
>                 status_code=status.HTTP_403_FORBIDDEN,
>                 detail="No tiene permisos suficientes para acceder a este recurso",
>             )
>         return usuario_actual
>
> # Uso: Depends(RequerirRol("admin"))  o  Depends(RequerirRol("admin", "editor"))
> ```

---
**Firma digital:** FernandaPalomo

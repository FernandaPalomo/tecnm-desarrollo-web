Bitácora de Requerimientos y Auditoría IA — Sesión 20
Materia: Desarrollo de Aplicaciones Web (DAB-2201)
Tema: Hito 2: API con Inyección de Dependencias, Auth JWT y RBAC
Docente Titular: MCC. Iván Márquez Larios (ijmarquezl)
Estudiante: Fernanda de Jesús Pat Palomo
Matrícula: 23531112
Track: Python/FastAPI
1. Fase 1: El Artesano (Dificultad Manual)
Reto: Integración de autenticación y autorización en API
Dificultad encontrada al procesar la petición HTTP o validar campos a mano:
Lo más difícil de hacerlo a mano fue unir todas las piezas del flujo (registro, login, token, perfil y panel) sin repetir código. Tenerm que leer el encabezado Authorization, separar la palabra Bearer, decodificar el JWT, revisar que no hubiera expirado y buscar al usuario en cada ruta protegida generaba mucho código duplicado y fácil de olvidar en alguna ruta. También me costó distinguir cuándo responder 401 Unauthorized (no sé quién eres o tu token no sirve) y cuándo 403 Forbidden (sé quién eres, pero no tienes permiso). Además, hay que hashear la contraseña al registrar y cuidar que ni la contraseña ni el hash aparezcan nunca en una respuesta.
2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)
2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
Elemento C-R-E-O
Definición Técnica
Contexto (C)
FastAPI 0.110+ (Python 3.10+) o PHP 8.2 con servidor embebido
Rol (R)
Arquitecto de software senior especialista en APIs RESTful y Seguridad Web
Especificación (E)
Contrato de seguridad y protección de rutas privadas
Output (O)
Solo código modular, esquemas Pydantic/DTO y códigos HTTP semánticos
2.2 Prompt Estructurado
CONTEXTO: Estoy construyendo una API REST de libros con FastAPI 0.110+ y Python 3.10+. Los usuarios se guardan en un diccionario en memoria. Debo usar inyección de dependencias de FastAPI, contraseñas con bcrypt (passlib) y JWT (PyJWT).

ROL: Actúa como arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

ESPECIFICACIÓN: Implementa el contrato de seguridad completo (Hito 2):
- Usuario administrador precargado: admin / admin123 con rol "admin". Los demás usuarios se registran con rol "user".
- POST /api/registro {usuario, password} -> 201 Created. La contraseña se guarda hasheada (nunca en texto plano) y la respuesta NO incluye password ni hash. Si el usuario ya existe, responder 400.
- POST /api/token {usuario, password} -> 200 OK con {access_token, token_type}. El JWT incluye el usuario (sub) y el rol, y expira. Credenciales incorrectas -> 401.
- GET /api/perfil protegido con Authorization: Bearer <token> para cualquier usuario autenticado -> 200 con {usuario, rol}.
- GET /api/admin/panel solo para rol admin -> 200. Usuario normal -> 403 Forbidden.
- Sin token, token inválido o expirado -> 401 Unauthorized.
- Usar dependencias reutilizables (obtener_usuario_actual y requerir_rol) en lugar de repetir la validación en cada ruta.
- Nunca devolver 500 por entradas inválidas; las respuestas son application/json.

OUTPUT: Devuelve solo código Python modular en un archivo app.py (esquemas Pydantic, funciones de hash/JWT, dependencias y endpoints) con códigos HTTP semánticos. Sin explicaciones ni texto adicional.
3. Código Devuelto por el Modelo
# Código devuelto por el LLM
import datetime
from typing import Dict, Optional
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel, Field
from passlib.context import CryptContext
import jwt

app = FastAPI(title="Sesión 20 · Hito 2: API con Inyección de Dependencias, Auth JWT y RBAC")

# --- Configuración de Seguridad y Criptografía ---
SECRET_KEY = "super_secret_key_para_el_hito_2_desarrollo_web_cancun_2025"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
security = HTTPBearer()

# --- Base de Datos en Memoria (Simulada) ---
DB_USUARIOS: Dict[str, dict] = {
    "admin": {
        "usuario": "admin",
        "password_hash": pwd_context.hash("admin123"),
        "rol": "admin"
    }
}

# --- Modelos de Pydantic ---
class UserRegister(BaseModel):
    usuario: str
    password: str

class UserResponse(BaseModel):
    usuario: str
    rol: str

class UserLogin(BaseModel):
    usuario: str
    password: str

class TokenResponse(BaseModel):
    access_token: str
    token_type: str

# --- Funciones de Utilidad (JWT y Hash) ---
def verificar_password(password_plano: str, password_hash: str) -> bool:
    return pwd_context.verify(password_plano, password_hash)

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def crear_token_acceso(datos: dict) -> str:
    copia_datos = datos.copy()
    expiracion = datetime.datetime.now(datetime.timezone.utc) + datetime.timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    copia_datos.update({"exp": expiracion})
    return jwt.encode(copia_datos, SECRET_KEY, algorithm=ALGORITHM)

# --- Dependencias de FastAPI ---
def obtener_usuario_actual(credentials: HTTPAuthorizationCredentials = Depends(security)) -> dict:
    token = credentials.credentials
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Token inválido"
            )
    except jwt.PyJWTError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token inválido o expirado"
        )

    usuario = DB_USUARIOS.get(username)
    if usuario is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Usuario no encontrado"
        )
    return usuario

def requerir_rol(rol_requerido: str):
    def dependencia(usuario_actual: dict = Depends(obtener_usuario_actual)):
        if usuario_actual.get("rol") != rol_requerido:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="No tienes suficientes permisos para realizar esta acción"
            )
        return usuario_actual
    return dependencia

# --- Endpoints de la API ---

@app.post("/api/registro", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
def registrar_usuario(user_data: UserRegister):
    username = user_data.usuario.strip()
    if not username or not user_data.password:
        raise HTTPException(status_code=400, detail="Campos obligatorios vacíos")

    if username in DB_USUARIOS:
        raise HTTPException(status_code=400, detail="El usuario ya existe")

    nuevo_usuario = {
        "usuario": username,
        "password_hash": hash_password(user_data.password),
        "rol": "user"
    }
    DB_USUARIOS[username] = nuevo_usuario

    return UserResponse(usuario=nuevo_usuario["usuario"], rol=nuevo_usuario["rol"])

@app.post("/api/token", response_model=TokenResponse)
def login(credentials: UserLogin):
    usuario = DB_USUARIOS.get(credentials.usuario)
    if not usuario or not verificar_password(credentials.password, usuario["password_hash"]):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Credenciales incorrectas"
        )

    payload = {
        "sub": usuario["usuario"],
        "rol": usuario["rol"]
    }
    token = crear_token_acceso(payload)
    return TokenResponse(access_token=token, token_type="bearer")

@app.get("/api/perfil", response_model=UserResponse)
def obtener_perfil(usuario_actual: dict = Depends(obtener_usuario_actual)):
    return UserResponse(usuario=usuario_actual["usuario"], rol=usuario_actual["rol"])

@app.get("/api/admin/panel")
def obtener_panel_admin(usuario_actual: dict = Depends(requerir_rol("admin"))):
    return {"status": "acceso concedido al panel de administración"}
4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
Pruebas ejecutadas: Suite de pruebas con usuarios legítimos e intrusos
[x] Respuestas semánticas correctas (200 OK, 201 Created, 400 Bad Request, 422 Unprocessable).
[x] No existen respuestas HTTP 500 Internal Server Error no capturadas.
[x] Encabezados Content-Type: application/json correctamente configurados.
Caso del verificador
Resultado esperado
Resultado obtenido
Registro sin filtrar contraseñas
201, sin password ni hash en la respuesta
201 ✅
Login del usuario registrado (ana)
200 con token
200 ✅
El usuario accede a su perfil
200
200 ✅
El usuario no entra al panel admin
403
403 ✅
El administrador obtiene su token
200 con token
200 ✅
El administrador entra al panel
200
200 ✅
Sin token no hay acceso
401
401 ✅
Resumen del verificador: 7/7 casos correctos.
Reporte Crítico
¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?
Aunque el verificador dio 7/7, al auditar el código encontré puntos que la IA dejó débiles y que el contrato no medía. Primero, la SECRET_KEY del JWT quedó escrita directamente en el código; cualquiera con acceso al repositorio podría falsificar tokens, por lo que debe leerse de una variable de entorno (os.environ). Segundo, el registro solo revisaba que los campos no estuvieran vacíos: no había longitud mínima de contraseña ni máxima de usuario, que se corrige con Field(min_length=..., max_length=...) en UserRegister para responder 422 Unprocessable Entity. Tercero, un usuario duplicado respondía 400 Bad Request, cuando lo más semántico es 409 Conflict. Lo que sí hizo bien la IA y conservé fue: hashear con bcrypt, no devolver nunca la contraseña ni el hash, distinguir 401 (sin token o token inválido) de 403 (sin permiso) y reutilizar las dependencias obtener_usuario_actual y requerir_rol en lugar de repetir la validación en cada ruta. También verifiqué que el rol se vuelve a leer de la base de datos y no se confía únicamente en el rol escrito en el token.
Firma digital: FernandaPalomo

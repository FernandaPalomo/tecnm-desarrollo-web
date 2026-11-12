# Bitácora de Requerimientos y Auditoría IA — Sesión 14
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Hashing Seguro de Contraseñas (Bcrypt / Argon2)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Cifrado manual con Salting y Factor de Costo  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Implementé a mano el hash con `hashlib.pbkdf2_hmac` usando un *salt* aleatorio de 16 bytes y 200 000 iteraciones como factor de costo, y guardé todo en un texto `iteraciones$salt$hash`. Lo difícil fue entender por qué el *salt* es necesario: sin él, dos usuarios con la misma contraseña tienen el mismo hash y basta una *rainbow table* para descifrarlos. También tuve que usar `hmac.compare_digest` al comparar, y aun así mi versión respondía más rápido cuando el usuario no existía, porque ni siquiera calculaba el hash.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app_ia.py`. Datos en memoria, sin base de datos. |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | Hash con **bcrypt** costo 12 (`$2b$12$`, 60 caracteres), *salt* aleatorio por usuario. `POST /api/usuarios` → 201 sin devolver el hash; 409 si el usuario existe; 422 si la contraseña tiene menos de 8 caracteres o más de 72 **bytes**. `POST /api/login` → 200 o 401 con el **mismo mensaje** exista o no el usuario, y con un hash señuelo para igualar el tiempo de respuesta. Endpoint de depuración del hash solo con `MODO_LAB=1`. |
| **Output (O)** | Solo código modular en `app_ia.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: FastAPI 0.110+ y Python 3.10+ en Google Colab con uvicorn. Un solo archivo app_ia.py, usuarios en memoria.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Registro y login con hashing seguro de contraseñas.
1. Usa bcrypt con factor de costo 12; cada usuario tiene su propio salt aleatorio. Nunca guardes ni devuelvas la contraseña ni el hash.
2. POST /api/usuarios responde 201 con {"usuario": ...}; 409 si ya existe; 422 si la contraseña tiene menos de 8 caracteres o más de 72 bytes (límite de bcrypt).
3. POST /api/login responde 200 si las credenciales son correctas y 401 con el mismo mensaje tanto si la contraseña es incorrecta como si el usuario no existe.
4. Evita ataques de timing: el login debe tardar lo mismo exista o no el usuario.
5. Solo si MODO_LAB=1, expón GET /api/debug/hash/{usuario} para inspeccionar el formato del hash.

Output: Solo el código de app_ia.py, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
*Versión final de `app_ia.py`, ya con las correcciones de la auditoría (sección 4).*
```python
"""Hashing seguro de contraseñas con bcrypt (salt aleatorio + factor de costo).

Nota de auditoría: no se usa passlib.CryptContext porque passlib
no es compatible con bcrypt >= 4.1 (ValueError al inicializar). Se usa la librería
bcrypt directamente.
"""
import os
import secrets

import bcrypt
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel, Field, field_validator

COSTO_BCRYPT = int(os.getenv("COSTO_BCRYPT", "12"))   # 2^12 rondas
MAX_BYTES = 72                                        # límite real de bcrypt

app = FastAPI(title="Biblioteca API - Hashing de contraseñas", version="1.4.0")

USUARIOS: dict[str, bytes] = {}                       # usuario -> hash bcrypt
# Hash señuelo: se verifica cuando el usuario no existe para igualar tiempos
HASH_SENUELO = bcrypt.hashpw(secrets.token_bytes(16), bcrypt.gensalt(COSTO_BCRYPT))


def hashear(password: str) -> bytes:
    return bcrypt.hashpw(password.encode("utf-8"), bcrypt.gensalt(COSTO_BCRYPT))


def verificar(password: str, hash_guardado: bytes) -> bool:
    return bcrypt.checkpw(password.encode("utf-8"), hash_guardado)


class Registro(BaseModel):
    usuario: str = Field(..., min_length=3, max_length=50, pattern=r"^[a-zA-Z0-9_.-]+$")
    password: str = Field(..., min_length=8)

    @field_validator("password")
    @classmethod
    def longitud_en_bytes(cls, v: str) -> str:
        if len(v.encode("utf-8")) > MAX_BYTES:
            raise ValueError(f"La contraseña no puede exceder {MAX_BYTES} bytes")
        return v


class Login(BaseModel):
    usuario: str = Field(..., max_length=50)
    password: str = Field(..., max_length=256)


class UsuarioOut(BaseModel):
    usuario: str          # el hash nunca sale del servidor


@app.post("/api/usuarios", response_model=UsuarioOut, status_code=status.HTTP_201_CREATED)
def registrar(datos: Registro):
    if datos.usuario in USUARIOS:
        raise HTTPException(status.HTTP_409_CONFLICT, detail="El usuario ya existe")
    USUARIOS[datos.usuario] = hashear(datos.password)
    return {"usuario": datos.usuario}


@app.post("/api/login")
def login(datos: Login):
    hash_guardado = USUARIOS.get(datos.usuario)
    password = datos.password.encode("utf-8")[:MAX_BYTES].decode("utf-8", "ignore")
    # Siempre se ejecuta un checkpw (real o señuelo): mismo costo exista o no el usuario
    valido = verificar(password, hash_guardado or HASH_SENUELO) and hash_guardado is not None
    if not valido:
        raise HTTPException(
            status.HTTP_401_UNAUTHORIZED,
            detail="Credenciales inválidas",          # mismo mensaje para ambos casos
            headers={"WWW-Authenticate": "Bearer"},
        )
    return {"mensaje": f"Bienvenida, {datos.usuario}"}


# Solo para el laboratorio: permite al auditor inspeccionar el formato del hash
@app.get("/api/debug/hash/{usuario}")
def ver_hash(usuario: str):
    if os.getenv("MODO_LAB", "0") != "1" or usuario not in USUARIOS:
        raise HTTPException(status.HTTP_404_NOT_FOUND, detail="No encontrado")
    return {"hash": USUARIOS[usuario].decode()}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas contra ataques de Rainbow Tables y Timing

**Resultado del Test Harness:** 10/10 casos correctos (A1 = prueba de humo de la versión artesanal; 01 en adelante = versión auditada `app_ia.py`).

| # | Caso | Detalle observado | Resultado |
| :-: | :--- | :--- | :-: |
| A1 | Artesano (PBKDF2 manual): login correcto → 200 | HTTP 200 | ✅ |
| 01 | Registro → 201 sin exponer el hash | HTTP 201 {'usuario': 'fernanda'} | ✅ |
| 02 | Misma contraseña → hashes distintos (salt) | $2b$12$loxztDPj./CeQ7anGj1rcu… vs $2b$12$RQEgeEcI0dzK9/eCRqh7Bu… | ✅ |
| 03 | Formato bcrypt con costo 12 ($2b$12$) | $2b$12$ | ✅ |
| 04 | El hash no coincide con una rainbow table (MD5/SHA-256) |  | ✅ |
| 05 | Login correcto → 200 | HTTP 200 | ✅ |
| 06 | Contraseña incorrecta y usuario inexistente → 401 idénticos | {'detail': 'Credenciales inválidas'} / {'detail': 'Credenciales inválidas'} | ✅ |
| 07 | Timing: usuario existente vs inexistente (razón < 1.5) | 277 ms vs 251 ms (x1.10) | ✅ |
| 08 | Contraseña > 72 bytes → 422 (no se trunca en silencio) | HTTP 422 | ✅ |
| 09 | Usuario duplicado → 409 | HTTP 409 | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> Los 10 casos pasaron (hashes distintos para la misma contraseña, formato `$2b$12$`, sin coincidencia con MD5/SHA-256, y tiempos de login similares). Hallazgos:
>
> - **`passlib` roto con bcrypt actual:** la forma más común (la que aparece en tutoriales antiguos de FastAPI) es `CryptContext(schemes=["bcrypt"])` de *passlib*. Con bcrypt 5.0 (la versión que instala `pip` hoy) falla al iniciar con `ValueError: password cannot be longer than 72 bytes`, porque passlib ya no se mantiene. Lo comprobé en el entorno. **Corrección:** usar la librería `bcrypt` directamente (`hashpw`, `gensalt`, `checkpw`).
> - **Truncamiento silencioso a 72 bytes:** bcrypt ignora lo que pasa de 72 bytes, así que dos contraseñas largas con el mismo inicio serían equivalentes. Además, el límite es en **bytes**, no en caracteres: 40 `ñ` son 80 bytes. **Corrección:** validador que rechaza con 422 en el registro.
> - **Enumeración de usuarios por tiempo:** mi versión artesanal respondía en ~3 ms cuando el usuario no existía y en ~37 ms cuando sí (medido), porque no calculaba ningún hash para usuarios inexistentes; eso delata qué usuarios están registrados. **Corrección:** se verifica contra un hash señuelo; la razón medida en el harness quedó cerca de 1.1.
> - **Falta *rate limiting*:** el costo de bcrypt frena los ataques offline, pero el endpoint de login sigue permitiendo intentos ilimitados (tema de la sesión 18).
>
> Sin errores 500.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> La prueba de timing, porque tuve que entender que un 401 idéntico no basta si el tiempo de respuesta delata que el usuario existe.

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Rechazar contraseñas de más de 72 bytes en lugar de truncarlas.

**¿Qué error detectó la verificación que tú no habías previsto?**
> Que `passlib` ya no funciona con bcrypt 5; el error apareció apenas importé la app.

---
**Firma digital:** FernandaPalomo

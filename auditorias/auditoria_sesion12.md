# Bitácora de Requerimientos y Auditoría IA — Sesión 12
**Materia:** Desarrollo de Aplicaciones Web (DAB-2201)  
**Tema:** Inyección de Dependencias (FastAPI Depends / PHP Container)  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Fernanda de Jesús Pat Palomo  
**Matrícula:** [No. de Control]  
**Track:** Python/FastAPI  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Inyector de dependencias manual por funciones  
*Describe qué dificultad encontraste al procesar la petición HTTP o validar campos a mano:*
> Con el inyector manual, cada endpoint tenía que llamar a `obtener_config()`, `verificar_clave()` y `obtener_repositorio()` por su cuenta. Encontré dos problemas: si olvidaba la llamada a `verificar_clave` en una ruta nueva, esa ruta quedaba abierta; y `obtener_repositorio()` creaba un repositorio nuevo en cada llamada, así que los libros se perdían entre peticiones. Tampoco tenía forma de cerrar recursos al final de la petición ni de reemplazar una dependencia en las pruebas.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal (Contrato OpenAPI / DTO)
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | FastAPI 0.110+ (Python 3.10+) con uvicorn en Google Colab, en un solo archivo `app_ia.py`. Datos en memoria, sin base de datos. |
| **Rol (R)** | Arquitecto de software senior especialista en APIs RESTful y Seguridad Web |
| **Especificación (E)** | Cadena con `Depends()`: `get_settings` (cacheado con `lru_cache`, API key desde `API_KEY`) → `get_sesion` (generador con `yield`, una sesión por request que siempre se cierra en `finally`) → `get_repositorio` → `get_servicio`. `verificar_api_key` como dependencia del router (401 con `WWW-Authenticate` si falta o es incorrecta, comparación con `secrets.compare_digest`). Encabezado `X-Sesion-ID` por respuesta. Sustituible con `dependency_overrides`. |
| **Output (O)** | Solo código modular en `app_ia.py`, con esquemas Pydantic y códigos HTTP semánticos, sin explicaciones adicionales. |

### 2.2 Prompt Estructurado
```text
Contexto: FastAPI 0.110+ y Python 3.10+ en Google Colab con uvicorn. Un solo archivo app_ia.py, datos en memoria.

Rol: Actúa como un arquitecto de software senior especialista en APIs RESTful y Seguridad Web.

Especificación: Usa el contenedor de dependencias de FastAPI (Depends) para la API de libros.
1. get_settings() devuelve la configuración (API key leída de la variable de entorno API_KEY) y se cachea con lru_cache.
2. get_sesion() es un generador con yield: crea una sesión por request con un id único y la cierra siempre en un bloque finally. Lleva un contador de sesiones abiertas y cerradas expuesto en GET /api/metricas/dependencias.
3. get_repositorio() depende de la sesión y get_servicio() depende del repositorio. Los datos deben persistir entre requests.
4. verificar_api_key() lee el encabezado X-API-Key y responde 401 con WWW-Authenticate si falta o es incorrecta. Aplícala como dependencia del APIRouter /api/libros.
5. POST /api/libros responde 201 y GET /api/libros responde 200; ambas agregan el encabezado X-Sesion-ID.
6. Debe poder sustituirse cualquier dependencia con app.dependency_overrides.

Output: Solo el código de app_ia.py, sin explicaciones.
```

---

## 3. Código Devuelto por el Modelo
*Versión final de `app_ia.py`, ya con las correcciones de la auditoría (sección 4).*
```python
"""Inyección de dependencias con Depends().

Cadena: Settings -> Sesión por request (yield) -> Repositorio -> Servicio.
La API key se valida como dependencia del router: ninguna ruta puede olvidarla.
"""
import os
import secrets
import uuid
from functools import lru_cache
from typing import Annotated, Iterator

from fastapi import APIRouter, Depends, FastAPI, Header, HTTPException, Response, status
from pydantic import BaseModel, Field


# ---------- Configuración (singleton cacheado) ----------
class Settings(BaseModel):
    api_key: str = Field(default_factory=lambda: os.getenv("API_KEY", "clave-curso"))


@lru_cache
def get_settings() -> Settings:
    return Settings()


# ---------- Almacén compartido (vive lo mismo que la app) ----------
ALMACEN: dict[int, dict] = {}
METRICAS = {"sesiones_abiertas": 0, "sesiones_cerradas": 0}


# ---------- Sesión por request: se abre y SIEMPRE se cierra ----------
class SesionUnidadTrabajo:
    def __init__(self) -> None:
        self.id = uuid.uuid4().hex[:12]
        self.cerrada = False

    def cerrar(self) -> None:
        self.cerrada = True


def get_sesion() -> Iterator[SesionUnidadTrabajo]:
    sesion = SesionUnidadTrabajo()
    METRICAS["sesiones_abiertas"] += 1
    try:
        yield sesion
    finally:
        sesion.cerrar()
        METRICAS["sesiones_cerradas"] += 1


# ---------- Repositorio y servicio ----------
class RepositorioLibros:
    def __init__(self, sesion: SesionUnidadTrabajo) -> None:
        self.sesion = sesion

    def agregar(self, titulo: str) -> dict:
        nuevo_id = max(ALMACEN, default=0) + 1
        ALMACEN[nuevo_id] = {"id": nuevo_id, "titulo": titulo}
        return dict(ALMACEN[nuevo_id])

    def listar(self) -> list[dict]:
        return [dict(l) for l in ALMACEN.values()]


def get_repositorio(sesion: Annotated[SesionUnidadTrabajo, Depends(get_sesion)]) -> RepositorioLibros:
    return RepositorioLibros(sesion)


class ServicioLibros:
    def __init__(self, repo: RepositorioLibros) -> None:
        self.repo = repo

    def crear(self, titulo: str) -> dict:
        return self.repo.agregar(titulo.strip())

    def listar(self) -> list[dict]:
        return self.repo.listar()


def get_servicio(repo: Annotated[RepositorioLibros, Depends(get_repositorio)]) -> ServicioLibros:
    return ServicioLibros(repo)


# ---------- Seguridad como dependencia ----------
def verificar_api_key(
    settings: Annotated[Settings, Depends(get_settings)],
    x_api_key: Annotated[str | None, Header()] = None,
) -> None:
    # compare_digest: comparación en tiempo constante (evita ataques de timing)
    if x_api_key is None or not secrets.compare_digest(x_api_key, settings.api_key):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="API key ausente o inválida",
            headers={"WWW-Authenticate": "ApiKey"},
        )


# ---------- Router ----------
class LibroIn(BaseModel):
    titulo: str = Field(..., min_length=1, max_length=120)


class LibroOut(BaseModel):
    id: int
    titulo: str


router = APIRouter(prefix="/api/libros", tags=["libros"], dependencies=[Depends(verificar_api_key)])


@router.post("", response_model=LibroOut, status_code=status.HTTP_201_CREATED)
def crear_libro(
    datos: LibroIn,
    response: Response,
    servicio: Annotated[ServicioLibros, Depends(get_servicio)],
    sesion: Annotated[SesionUnidadTrabajo, Depends(get_sesion)],
):
    response.headers["X-Sesion-ID"] = sesion.id
    return servicio.crear(datos.titulo)


@router.get("")
def listar_libros(
    response: Response,
    servicio: Annotated[ServicioLibros, Depends(get_servicio)],
    sesion: Annotated[SesionUnidadTrabajo, Depends(get_sesion)],
):
    response.headers["X-Sesion-ID"] = sesion.id
    return {"libros": servicio.listar()}


app = FastAPI(title="Biblioteca API - Inyección de Dependencias", version="1.2.0")
app.include_router(router)


@app.get("/api/metricas/dependencias")
def metricas():
    return METRICAS
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness HTTP)
*Pruebas ejecutadas:* Pruebas de ciclo de vida de dependencias por request

**Resultado del Test Harness:** 9/9 casos correctos (A1 = prueba de humo de la versión artesanal; 01 en adelante = versión auditada `app_ia.py`).

| # | Caso | Detalle observado | Resultado |
| :-: | :--- | :--- | :-: |
| A1 | Artesano: creación con clave → 201 | HTTP 201 | ✅ |
| 01 | Sin API key → 401 | HTTP 401 | ✅ |
| 02 | API key incorrecta → 401 | HTTP 401 | ✅ |
| 03 | POST con clave → 201 | HTTP 201, {'id': 2, 'titulo': 'Aura'} | ✅ |
| 04 | Cada request recibe una sesión distinta | 7dcdc892c104 vs 8f46626f1919 | ✅ |
| 05 | Los datos persisten entre requests (almacén compartido) | {'libros': [{'id': 1, 'titulo': 'Rayuela'}, {'id': 2, 'titulo': 'Aura'}]} | ✅ |
| 06 | Toda sesión abierta se cerró (yield + finally) | {'sesiones_abiertas': 3, 'sesiones_cerradas': 3} | ✅ |
| 07 | Una sola sesión por request aunque 2 dependencias la pidan | Depends cachea la dependencia dentro del request | ✅ |
| 08 | dependency_overrides reemplaza servicio y configuración | HTTP 201 {'id': 999, 'titulo': 'Prueba'} | ✅ |

* [x] Respuestas semánticas correctas (`200 OK`, `201 Created`, `400 Bad Request`, `422 Unprocessable`).
* [x] No existen respuestas HTTP `500 Internal Server Error` no capturadas.
* [x] Encabezados `Content-Type: application/json` correctamente configurados.

### Reporte Crítico
*¿Qué código de estado HTTP o falla de seguridad omitió la IA y cómo lo corregiste?*
> El harness pasó completo (sesiones distintas por request, todas cerradas, una sola sesión aunque dos dependencias la pidan y `dependency_overrides` funcionando). Hallazgos:
>
> - **Comparación de la API key con `!=`:** la comparación normal se detiene en el primer carácter distinto, lo que en teoría permite un ataque de *timing*. **Corrección:** `secrets.compare_digest`.
> - **API key fija en el código:** la IA puso `"clave-curso"` como valor por defecto. Lo dejé solo para el laboratorio, pero en producción `API_KEY` debe ser obligatoria y fallar al arrancar si no existe.
> - **`/api/metricas/dependencias` sin protección:** está fuera del router, así que no exige API key. Para el laboratorio es útil, pero expone información interna.
> - **Ciclo de vida del almacén:** en mi versión artesanal el repositorio se creaba en cada llamada y los datos se perdían. Aquí el almacén vive con la aplicación y solo la sesión vive con el request; el caso 05 confirma que los libros persisten entre peticiones.
>
> Sin errores 500.

---

## 5. Cierre

**¿Qué resultado te costó más alcanzar y por qué?**
> Entender que `Depends` cachea la misma dependencia dentro de un request: aunque `get_servicio` y el endpoint pidan la sesión, solo se crea una.

**¿Qué decisión de diseño tomaste que no estaba escrita en el contrato?**
> Aplicar la API key a nivel de router para que ninguna ruta nueva quede abierta por olvido.

**¿Qué error detectó la verificación que tú no habías previsto?**
> La sesión debía cerrarse incluso si el endpoint falla; el contador de abiertas/cerradas lo hizo visible.

---
**Firma digital:** FernandaPalomo

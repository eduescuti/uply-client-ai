# UPLY CLIENT AI — CLAUDE.md

## Identidad del repositorio

Sistema de inteligencia orientada al usuario final de Uply. Responsable de career coaching, diagnóstico profesional, skill gap analysis, recomendación de cursos, simulación de entrevistas, sugerencias de perfil/CV, aprendizaje de inglés y predicción de empleabilidad. Es consumido **exclusivamente** por `uply-backend` — nunca directamente desde el frontend.

---

## Stack tecnológico

| Capa | Tecnología | Versión |
|------|-----------|---------|
| Lenguaje | Python | 3.12 |
| Framework web | FastAPI | 0.115.x |
| ORM | SQLAlchemy 2 (async) + Alembic | 2.x |
| Base de datos | PostgreSQL 16 + pgvector | — |
| Cache / Colas | Redis + Celery | — |
| Orquestación LLM | LangChain | 0.3.x |
| Abstracción LLM | LiteLLM | latest |
| LLM (desarrollo) | Ollama — llama3.2:3b o gemma2:2b | local, gratis |
| LLM (producción) | Claude Haiku 4.5 (`claude-haiku-4-5-20251001`) | Anthropic API |
| LLM (fallback prod) | Groq `llama-3.1-8b-instant` | gratis |
| Embeddings | sentence-transformers (all-MiniLM-L6-v2) | local, gratis |
| Búsqueda vectorial | pgvector (extensión de PostgreSQL) | — |
| Validación | Pydantic v2 | 2.x |
| Tests | pytest + httpx | — |
| Linting | Ruff | latest |
| Gestión de dependencias | uv | latest |

---

## Estrategia de LLM por entorno

| Entorno | Modelo | Costo |
|---------|--------|-------|
| Desarrollo local | `ollama/llama3.2:3b` | Gratis |
| PC 8GB RAM | `ollama/gemma2:2b` | Gratis (más liviano) |
| Producción | `claude-haiku-4-5-20251001` | ~$0.00036/sesión |
| Fallback producción | `groq/llama-3.1-8b-instant` | Gratis |

**Por qué Haiku en producción:** contexto de 200K tokens (ideal para sesiones largas de entrevistas), calidad muy superior a modelos gratuitos, costo insignificante hasta miles de usuarios.

El proveedor nunca está hardcodeado — se configura via `.env`:

```python
from litellm import completion

response = completion(
    model=settings.LLM_MODEL,
    fallbacks=["groq/llama-3.1-8b-instant"],   # fallback automático si Anthropic falla
    messages=[{"role": "user", "content": prompt}],
    api_base=settings.LLM_BASE_URL,             # None en prod (LiteLLM lo resuelve)
    stream=True,                                  # Siempre usar streaming en endpoints conversacionales
)
```

### LLMs con Ollama (desarrollo)

```bash
# Instalar Ollama (Linux)
curl -fsSL https://ollama.ai/install.sh | sh

# Descargar modelo según RAM disponible
ollama pull llama3.2:3b     # ~2GB RAM — recomendado (notebook 16GB)
ollama pull gemma2:2b       # ~1.5GB RAM — PC fija 8GB

# Verificar que corre
ollama list
ollama run llama3.2:3b "Hola"
```

### Embeddings con sentence-transformers (gratis, local)

```python
from sentence_transformers import SentenceTransformer

# Se descarga una sola vez (~80MB)
model = SentenceTransformer("all-MiniLM-L6-v2")
embeddings = model.encode(["texto a embeddear"])
# Dimension: 384 — almacenar en pgvector
```

---

## Estructura del proyecto

```
uply-client-ai/
├── alembic/
│   ├── env.py
│   └── versions/
├── app/
│   ├── main.py                     # Entry point FastAPI
│   ├── config.py                   # Settings con Pydantic BaseSettings
│   ├── database.py                 # Engine async SQLAlchemy + pgvector setup
│   ├── api/
│   │   └── v1/
│   │       ├── router.py           # Router principal v1
│   │       ├── coaching/           # Career coaching y orientación (SSE)
│   │       ├── diagnosis/          # Diagnóstico inicial de perfil
│   │       ├── skills/             # Skill gap analysis
│   │       ├── courses/            # Recomendación de cursos
│   │       ├── interviews/         # Simulación de entrevistas (SSE)
│   │       ├── resume/             # Sugerencias de perfil (NO genera archivos)
│   │       ├── english/            # Aprendizaje de inglés contextual (SSE)
│   │       └── employability/      # Predicción de empleabilidad
│   ├── core/
│   │   ├── llm/                    # Wrappers de LiteLLM y chains de LangChain
│   │   ├── embeddings/             # sentence-transformers + helpers de pgvector
│   │   ├── rag/                    # RAG pipelines (retrieval + generation)
│   │   └── prompts/                # Prompt templates como constantes
│   ├── models/                     # Modelos SQLAlchemy
│   ├── schemas/                    # Pydantic schemas (request / response)
│   ├── services/                   # Lógica de negocio de cada capacidad de IA
│   ├── repositories/               # Acceso a datos (SQLAlchemy)
│   └── tasks/                      # Tareas async con Celery
├── tests/
│   ├── unit/
│   └── integration/
├── pyproject.toml
└── alembic.ini
```

---

## Comandos esenciales

```bash
# Instalación
uv sync

# Desarrollo (http://localhost:8001)
uv run fastapi dev app/main.py
# Docs Swagger: http://localhost:8001/docs

# Base de datos
uv run alembic upgrade head                                    # Aplicar migraciones
uv run alembic revision --autogenerate -m "descripcion"        # Nueva migración
uv run alembic downgrade -1                                    # Revertir última

# Descargar modelo de embeddings (solo primera vez)
uv run python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')"

# Celery worker (para tareas async)
uv run celery -A app.tasks worker --loglevel=info

# Tests
uv run pytest                          # Todos los tests
uv run pytest tests/unit/              # Solo unitarios
uv run pytest tests/integration/       # Solo integración
uv run pytest --cov=app --cov-report=html   # Con cobertura

# Linting y formato
uv run ruff check .
uv run ruff format .
```

---

## Entorno de desarrollo — Docker Compose

El entorno completo se levanta desde el repositorio `uply-devops`:

```bash
# Levantar todos los servicios (ejecutar desde uply-devops/)
docker compose up -d

# Levantar solo infraestructura
docker compose up postgres redis ollama -d

# Este servicio corre en el contenedor:
#   uply-client-ai (FastAPI) → puerto 8001
```

**PC con 8GB RAM:** usar `gemma2:2b` en lugar de `llama3.2:3b` para Ollama.

---

## Variables de entorno

Crear `.env` en la raíz (nunca commitear):

```env
# App
ENVIRONMENT=development
PORT=8001

# Base de datos
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/uply_client_ai_dev

# Redis
REDIS_URL=redis://localhost:6379/1

# LLM — desarrollo (Ollama local, gratis)
LLM_MODEL=ollama/llama3.2:3b
LLM_BASE_URL=http://localhost:11434
ANTHROPIC_API_KEY=                    # vacío en desarrollo

# LLM — producción (descomentar en prod)
# LLM_MODEL=claude-haiku-4-5-20251001
# LLM_BASE_URL=https://api.anthropic.com
# ANTHROPIC_API_KEY=sk-ant-...
# GROQ_API_KEY=gsk_...               # fallback

# Embeddings — local con sentence-transformers (gratis)
EMBEDDING_MODEL=all-MiniLM-L6-v2
EMBEDDING_DIMENSION=384

# Autenticación interna (validar requests del backend)
INTERNAL_API_KEY=internal-secret-key-dev
```

---

## Streaming SSE — endpoints conversacionales

Los endpoints de coaching, entrevistas e inglés deben devolver respuestas en streaming usando `StreamingResponse` de FastAPI con LiteLLM `streaming=True`.

```python
from fastapi import APIRouter
from fastapi.responses import StreamingResponse
from litellm import completion

@router.post("/coaching/session")
async def stream_coaching_session(request: CoachingRequest):
    async def generate():
        response = completion(
            model=settings.LLM_MODEL,
            fallbacks=["groq/llama-3.1-8b-instant"],
            messages=build_messages(request),
            stream=True,
        )
        for chunk in response:
            content = chunk.choices[0].delta.content or ""
            if content:
                yield f"data: {content}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

**Endpoints que usan streaming:**
- `POST /v1/coaching/session`
- `POST /v1/interviews/simulate`
- `POST /v1/english/session`

---

## Módulo de diseño de CV — `/v1/resume/curate`

**No existe generación ni descarga de archivos de CV.**

Este módulo es responsable de **diseñar el contenido del CV de cada usuario individualmente**. El AI no solo sugiere — decide activamente qué secciones incluir, qué contenido agregar, qué eliminar y cómo estructurar el perfil para maximizar la empleabilidad del usuario. El template visual es el mismo para todos, pero el contenido varía por usuario según lo que el modelo decide.

```python
@router.post("/resume/curate", response_model=ResumeCuratedResponse)
async def curate_resume(request: ResumeProfileRequest):
    # Analiza el perfil raw del usuario
    # Decide qué agregar, qué eliminar, qué destacar
    # Devuelve el perfil curado con instrucciones de qué aplicar
    ...
```

Schema de entrada:
```python
class ResumeProfileRequest(BaseModel):
    user_id: str
    basics: dict       # nombre, titulo, email
    experiencia: list
    educacion: list
    skills: list[str]
    idiomas: list
    objetivo_profesional: str | None   # qué busca el usuario
```

Schema de salida (el backend aplica estos cambios a las tablas del perfil):
```python
class ResumeCuratedResponse(BaseModel):
    user_id: str
    secciones_a_incluir: list[str]       # qué secciones mostrar en el CV
    secciones_a_excluir: list[str]       # qué secciones omitir
    contenido_sugerido: dict             # texto mejorado por sección
    skills_a_destacar: list[str]         # skills más relevantes para el objetivo
    titulo_profesional_sugerido: str     # headline del CV curado por IA
    explanation: list[str]               # por qué se tomaron estas decisiones
```

**Nota:** El endpoint anterior `/v1/resume/suggestions` queda reemplazado por `/v1/resume/curate`. La diferencia es que curate devuelve decisiones estructuradas listas para aplicar, no solo sugerencias.

---

## Módulo de vinculación de empleos — `/v1/jobs/recommendations`

`client-ai` también es responsable de la **vinculación de empleos desde el lado del candidato**: encuentra y recomienda vacantes relevantes para el usuario basándose en su perfil curado y objetivos profesionales.

```python
@router.post("/jobs/recommendations", response_model=JobRecommendationsResponse)
async def recommend_jobs(request: JobRecommendationRequest):
    # Analiza el perfil curado del usuario
    # Busca vacantes afines vía embeddings (pgvector)
    # Rankea y explica por qué cada vacante es relevante
    ...

class JobRecommendationRequest(BaseModel):
    user_id: str
    profile: ProfileForAI    # perfil serializado por el backend
    vacantes_disponibles: list[dict]   # vacantes activas del backend

class JobRecommendationsResponse(BaseModel):
    user_id: str
    recomendaciones: list[JobRecommendation]

class JobRecommendation(BaseModel):
    job_id: str
    score: float             # 0.0 a 1.0
    explanation: list[str]   # por qué esta vacante es relevante para el usuario
    skills_match: list[str]  # skills del usuario que hacen match
    skills_gap: list[str]    # skills que le faltan para esta vacante
```

**Diferencia con enterprise-ai:** `client-ai` recomienda vacantes **para un candidato** (perspectiva del candidato). `enterprise-ai` rankea candidatos **para una vacante** (perspectiva de la empresa). Son dos flujos complementarios.

---

## Convenciones de código y arquitectura

### Estructura de un endpoint de IA

```python
# router: define la ruta
@router.post("/diagnosis", response_model=DiagnosisResponse)
async def run_diagnosis(request: DiagnosisRequest, service: DiagnosisService = Depends()):
    return await service.diagnose(request)

# service: lógica de IA
class DiagnosisService:
    async def diagnose(self, request: DiagnosisRequest) -> DiagnosisResponse:
        # 1. Recuperar contexto relevante del usuario (pgvector)
        context = await self.repo.get_user_context(request.user_id)
        # 2. Construir prompt
        prompt = DIAGNOSIS_PROMPT.format(profile=context, ...)
        # 3. Llamar al LLM via LiteLLM
        response = await self.llm.complete(prompt)
        # 4. Parsear respuesta con Pydantic
        return DiagnosisResponse.model_validate(response)
```

### RAG Pipeline estándar

```
Request del usuario →
  Buscar contexto por similitud en pgvector →
  Construir prompt con contexto recuperado →
  LLM genera respuesta →
  Post-procesar y estructurar con Pydantic →
  Retornar respuesta con explicación
```

### Schemas Pydantic
- Requests: `XxxRequest`
- Responses: `XxxResponse`
- Siempre incluir campo `explanation: list[str]` en respuestas de IA (explicabilidad)
- Nunca exponer modelos SQLAlchemy directamente

### Testing — TDD obligatorio
```
1. RED    → escribir test que falla
2. GREEN  → implementar lo mínimo para que pase
3. REFACTOR → mejorar sin romper tests
```
- **Mockear siempre las llamadas al LLM** en tests unitarios (no gastar tokens)
- Tests de integración con PostgreSQL real (base de datos `uply_client_ai_test`)
- Evaluar calidad de respuestas del LLM en tests separados de evaluación

### Seguridad
- Validar el header `x-internal-api-key` en todos los endpoints (solo `uply-backend` debe llamar)
- Nunca loguear contenido del usuario en texto claro
- Sanitizar inputs antes de incluirlos en prompts (evitar prompt injection)

---

## Deployment — DigitalOcean Droplet

- **Plataforma:** DigitalOcean Droplet — 4 vCPU / 8GB RAM / 160GB SSD (compartido)
- **Costo estimado:** ~$24/mes total del Droplet (con backend y enterprise-ai)
- En producción corre el mismo `docker-compose.yml` de `uply-devops`

**Requerimientos para deployment:**
- `Dockerfile` en la raíz del repositorio
- Health check endpoint: `GET /health`
- Logs a stdout (no a archivos)

---

## Protocolo de sesión para el agente

**Al inicio de cada sesión:**
1. Leer este `CLAUDE.md` completo
2. Leer el `README.md` de este repositorio — el agente MAIN deja allí las instrucciones y prioridades para esta sesión
3. Ejecutar `git log --oneline -10`
4. Si el repositorio está vacío → ejecutar el scaffolding (ver Estado actual)

**Durante la sesión:**
- TDD obligatorio — test primero, siempre
- Un PR por capacidad de IA implementada o mejorada
- Formato de PR: `[CLIENT-AI] descripción breve`

**Al finalizar la sesión:**
- Commitear y abrir PR hacia `main`
- **Los PRs NO se mergean directamente** — se abren y el agente MAIN los revisa y mergea en su sesión posterior
- Actualizar el `README.md` de este repositorio con el resumen de la sesión:
  - Capacidades implementadas o mejoradas
  - Endpoints disponibles
  - Calidad observada de las respuestas del LLM
  - Qué queda pendiente

---

## Estado actual del proyecto

> **PRIMERA SESIÓN — Repositorio vacío.**
> La primera sesión debe ejecutar el scaffolding completo:
>
> ```bash
> # 1. Inicializar proyecto con uv
> uv init
> uv python pin 3.12
>
> # 2. Instalar dependencias
> uv add fastapi uvicorn sqlalchemy[asyncio] asyncpg alembic pgvector
> uv add langchain langchain-community litellm sentence-transformers
> uv add pydantic-settings redis celery httpx
> uv add --dev pytest pytest-asyncio pytest-cov httpx ruff
>
> # 3. Crear estructura de carpetas base
> # 4. Configurar FastAPI con middleware de autenticación interna
> # 5. Configurar SQLAlchemy async + pgvector
> # 6. Configurar Alembic
> # 7. Implementar health check endpoint
> # 8. Implementar primera capacidad: diagnóstico de perfil profesional (con streaming)
> ```

---

## Integración con otros repositorios

| Repositorio | Relación |
|-------------|----------|
| `uply-backend` | Único cliente autorizado. Valida requests con header `x-internal-api-key` |
| `uply-enterprise-ai` | Independiente. No se llaman entre sí |
| `uply` (frontend) | No llama a este servicio directamente |
| `uply-devops` | Docker Compose centralizado — levantar desde ahí |

# UPLY CLIENT AI — CLAUDE.md

## Identidad del repositorio

Sistema de inteligencia orientada al usuario final de Uply. Responsable de career coaching, diagnóstico profesional, skill gap analysis, recomendación de cursos, simulación de entrevistas, optimización de CV, aprendizaje de inglés y predicción de empleabilidad. Es consumido **exclusivamente** por `uply-backend` — nunca directamente desde el frontend.

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
| Embeddings | sentence-transformers (all-MiniLM-L6-v2) | local, gratis |
| Búsqueda vectorial | pgvector (extensión de PostgreSQL) | — |
| Validación | Pydantic v2 | 2.x |
| Tests | pytest + httpx | — |
| Linting | Ruff | latest |
| Gestión de dependencias | uv | latest |

**Todos gratuitos. Los LLMs corren localmente con Ollama, sin costo de API.**

---

## Arquitectura de IA — sin costo

### LLMs con Ollama (gratis, local)

```bash
# Instalar Ollama (Linux)
curl -fsSL https://ollama.ai/install.sh | sh

# Descargar modelo según RAM disponible
ollama pull llama3.2:3b     # ~2GB RAM — recomendado
ollama pull gemma2:2b       # ~1.5GB RAM — más liviano

# Verificar que corre
ollama list
ollama run llama3.2:3b "Hola"
```

El código nunca tiene el proveedor hardcodeado — se configura via `.env`:

```python
# LiteLLM es agnóstico al proveedor
from litellm import completion

response = completion(
    model=settings.LLM_MODEL,       # "ollama/llama3.2:3b" en dev
    messages=[{"role": "user", "content": prompt}],
    api_base=settings.LLM_BASE_URL  # "http://localhost:11434" para Ollama
)
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
│   │       ├── coaching/           # Career coaching y orientación
│   │       ├── diagnosis/          # Diagnóstico inicial de perfil
│   │       ├── skills/             # Skill gap analysis
│   │       ├── courses/            # Recomendación de cursos
│   │       ├── interviews/         # Simulación de entrevistas
│   │       ├── resume/             # Optimización y generación de CV
│   │       ├── english/            # Aprendizaje de inglés contextual
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

# LLM — Ollama local (gratis)
LLM_MODEL=ollama/llama3.2:3b
LLM_BASE_URL=http://localhost:11434

# Embeddings — local con sentence-transformers (gratis)
EMBEDDING_MODEL=all-MiniLM-L6-v2
EMBEDDING_DIMENSION=384

# Autenticación interna (validar requests del backend)
INTERNAL_API_KEY=internal-secret-key-dev
```

### Alternativas gratuitas para LLM (si Ollama no está disponible)

```env
# Groq — free tier: 14.400 requests/día
LLM_MODEL=groq/llama-3.1-8b-instant
GROQ_API_KEY=gsk_...

# Google Gemini — free tier: 1.500 requests/día
LLM_MODEL=gemini/gemini-1.5-flash
GEMINI_API_KEY=AIza...
```

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

## Protocolo de sesión para el agente

**Al inicio de cada sesión:**
1. Leer este `CLAUDE.md` completo
2. Ejecutar `git log --oneline -10`
3. Abrir el Google Docs "Uply" y leer la sección **"Pendientes Client AI"**
4. Si el repositorio está vacío → ejecutar el scaffolding (ver Estado actual)

**Durante la sesión:**
- TDD obligatorio — test primero, siempre
- Un PR por capacidad de IA implementada o mejorada
- Formato de PR: `[CLIENT-AI] descripción breve`

**Al finalizar la sesión:**
- Commitear y abrir PR hacia `main`
- Anotar en Google Docs "Uply" → sección **"Pendientes Client AI"**:
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
> # 8. Implementar primera capacidad: diagnóstico de perfil profesional
> ```

---

## Integración con otros repositorios

| Repositorio | Relación |
|-------------|----------|
| `uply-backend` | Único cliente autorizado. Valida requests con header `x-internal-api-key` |
| `uply-enterprise-ai` | Independiente. No se llaman entre sí |
| `uply` (frontend) | No llama a este servicio directamente |

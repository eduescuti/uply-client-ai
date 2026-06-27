# uply-client-ai

Sistema de IA orientada al usuario final — FastAPI, Python 3.12, LiteLLM, LangChain, pgvector.
Consumido exclusivamente por `uply-backend`. Nunca llamado directamente desde el frontend.

---

<!-- ================================================================ -->
<!-- SECCIÓN MAIN — Solo el agente MAIN escribe esta sección.       -->
<!-- Agente Client-AI: LEE esta sección, NUNCA la modifiques.       -->
<!-- ================================================================ -->

## 📋 Instrucciones MAIN

> *Última actualización: 2026-06-22*

**FASE ACTUAL:** Fase 1 — Scaffolding inicial (Repositorio vacío)

### 🎯 Responsabilidad
Construir la inteligencia que acompaña a cada candidato. Eres responsable del coaching profesional, análisis de CV, recomendaciones de empleos y predicción de empleabilidad.

### ⚠️ BLOQUEADO POR
**Espera a que la infraestructura esté corriendo Y que el Backend esté listo.** Necesitas:
- PostgreSQL corriendo en `localhost:5432` con la base de datos `uply_client_ai_dev` creada
- Redis corriendo en `localhost:6379`
- Ollama corriendo en `localhost:11434` con modelo descargado
- Backend respondiendo en `http://localhost:4000/health`

**El agente MAIN actualiza este README cuando es momento de iniciar.** Hasta entonces, puedes preparar el scaffolding de código y estructura de directorios, pero NO ejecutar migraciones ni levantar el servidor.

### 🚀 Prioridad Fase 1 (Orden estricto)

1. **Scaffolding FastAPI**
   - `uv init` + `uv python pin 3.12`
   - Instalar dependencias del CLAUDE.md
   - Estructura: `app/`, `tests/`, `alembic/`

2. **Configurar PostgreSQL + pgvector**
   - SQLAlchemy async con asyncpg
   - Alembic para migraciones
   - **CRÍTICO:** Primera migración: `CREATE EXTENSION IF NOT EXISTS vector`

3. **Configurar LiteLLM**
   - Desarrollo: `ollama/llama3.2:3b` (o `gemma2:2b` si 8GB RAM)
   - Fallback: `groq/llama-3.1-8b-instant`
   - Mockear SIEMPRE en tests

4. **Descargar Embeddings**
   - `uv run python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')"`
   - ~80MB, descarga una sola vez

5. **Middleware esencial**
   - Validar `x-internal-api-key` en TODOS los endpoints
   - Logs estructurados a stdout

6. **Endpoints Fase 1**
   - `GET /health` → `{ status: "ok" }`
   - `POST /v1/diagnosis` — análisis inicial de empleabilidad
   - Swagger en `GET /docs`

7. **Tests**
   - TDD: RED → GREEN → REFACTOR
   - LLM mocked en TODOS los tests unitarios

### ✅ Definición de "listo"
- [ ] Proyecto corre sin errores: `uv run fastapi dev app/main.py`
- [ ] `GET /health` devuelve `{ status: "ok" }`
- [ ] `GET /docs` abre Swagger
- [ ] Tests pasan: `uv run pytest`
- [ ] LLM mocked en tests unitarios
- [ ] Middleware valida `x-internal-api-key`
- [ ] README.md en rama `develop` actualizado

### 📝 Notas importantes
- **`/v1/resume/curate`** diseña activamente el CV de cada usuario (decide qué secciones incluir).
- **`/v1/jobs/recommendations`** recomienda vacantes PARA el candidato (perspectiva candidato).
- **Diferencia con enterprise-ai:** Client-AI = empleos para candidato. Enterprise-AI = candidatos para empresa.
- **Mockear LLM en tests:** `completion()` debe ser mock en unitarios. No gastar tokens.

### 📚 Documentación de referencia

Leer al inicio de cada sesión desde el repositorio `uply-devops`:
- `ARQUITECTURA_INTEGRACION.md` — Flujos completos, endpoints, protocolo de comunicación
- `DESARROLLO_ROADMAP.md` — Fase actual, tareas asignadas, criterios de éxito
- `TROUBLESHOOTING.md` — Ollama, pgvector, Alembic, LLM mocking y más

---

## 📊 Estado del proyecto

**Fase:** Fase 1 — Scaffolding inicial
**Último PR mergeado:** —
**Endpoints implementados:** 0 / 7
**Tests:** 0
**LLM Tests:** Todos mocked
**Próxima sesión:** Scaffolding + PostgreSQL + Diagnosis
**Dependencias:** PostgreSQL + Redis + Ollama corriendo + Backend listo (MAIN confirma aquí)

---

<!-- ================================================================ -->
<!-- SECCIÓN AGENTE — Solo el agente Client-AI escribe esta sección.  -->
<!-- MAIN: LEE esta sección para ver el estado. No la modifiques.     -->
<!-- ================================================================ -->

## 📝 Última sesión

> *Fecha: —*

**Completado en esta sesión:** —

**Estado actual:**
- Endpoints implementados: 0 / 7
- Tests: 0
- Scaffolding: pendiente

**Pendiente / Próxima sesión:** Primera sesión — scaffolding + health + diagnosis

**Problemas encontrados:** —

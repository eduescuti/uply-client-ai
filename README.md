# uply-client-ai

Sistema de IA orientada al usuario final — FastAPI, Python 3.12, LiteLLM, LangChain, pgvector.
Consumido exclusivamente por `uply-backend`. Nunca llamado directamente desde el frontend.

---

## 📋 Instrucciones del Agente MAIN

> *Última actualización: —*

**PRIMERA SESIÓN — Repositorio vacío.**

Ejecutar el scaffolding completo según las instrucciones de la sección "Estado actual" del `CLAUDE.md`.

**Prioridad Fase 1:**
1. Scaffolding FastAPI con el stack definido en `CLAUDE.md` (uv, Python 3.12)
2. Configurar LiteLLM → Ollama/llama3.2:3b en desarrollo (gemma2:2b para 8GB RAM)
3. Descargar sentence-transformers `all-MiniLM-L6-v2` (~80MB)
4. Configurar SQLAlchemy async + pgvector + Alembic
5. Middleware de autenticación interna (`x-internal-api-key`)
6. Endpoint `GET /health`
7. Endpoint `POST /v1/diagnosis`: recibe `ProfileForAI`, devuelve diagnóstico de empleabilidad con `explanation`

**Notas importantes:**
- Mockear siempre las llamadas al LLM en tests unitarios (no gastar tokens)
- Endpoint `/v1/resume/curate` diseña activamente el CV de cada usuario (mismo template, contenido distinto)
- Endpoint `/v1/jobs/recommendations` recomienda vacantes PARA el candidato (perspectiva candidato)
- Diferencia con enterprise-ai: client-ai busca empleos para el candidato; enterprise-ai rankea candidatos para la empresa

---

## Estado del proyecto

**Fase actual:** Fase 1 — Scaffolding inicial  
**Último PR mergeado:** —  
**Endpoints disponibles:** —  
**Próxima sesión:** Ejecutar scaffolding + health + diagnosis

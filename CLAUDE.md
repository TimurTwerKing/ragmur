# Instrucciones de trabajo — Ragmur

## Contexto

Ragmur es un servicio RAG expuesto como API HTTP: ingiere documentos, los indexa y responde consultas citando la fuente. Multi-tenant. El LLM se ejecuta dentro del servicio a partir de la fase 2.

Antes de escribir código: leer `ARCHITECTURE.md` (contratos y modelo de datos) y `ROADMAP.md` (tarea actual). `DECISIONS.md` guarda el porqué de cada decisión y las alternativas ya descartadas: consultarlo antes de proponer un cambio de diseño, para no reabrir discusiones cerradas.

Idioma: español en conversación y documentación. Código, identificadores y mensajes de commit en inglés.

## Reglas de arquitectura

No se modifican sin acordarlo previamente:

1. **Fases secuenciales.** No implementar nada de la fase 2 o 3 mientras la fase 1 no esté cerrada. Si algo de una fase posterior desbloquea la actual, indicarlo y esperar confirmación.
2. **La fase 1 no usa LLM.** Ni para trocear, ni para reescribir consultas, ni para verificar. Esto es una restricción de la fase 1, no un principio permanente del proyecto: la fase 2 integra el LLM dentro del servicio y `/answer` pasa a ser el endpoint principal. Si una solución de fase 1 requiere un LLM, la solución es incorrecta para esa fase.
3. **`tenant_id` en toda consulta a Qdrant.** Todo test de recuperación incluye comprobación de aislamiento entre tenants, y entre `owner_id` dentro de un mismo tenant.
4. **El tenant se deriva de la API key**, nunca del cuerpo de la petición. El `owner_id` sí llega en la petición: subdivide el espacio del tenant por usuario final, y autenticarlo es responsabilidad de la aplicación cliente.
5. **Dirección de dependencias:** `api/` importa de `ingestion/`, `retrieval/`, `verification/` y `llm/`; nunca al revés. La lógica de dominio no conoce FastAPI.
6. **Sin frameworks RAG de alto nivel.** Nada de LangChain o LlamaIndex como framework. Se acepta `langchain-text-splitters` como utilidad aislada.
7. **Toda llamada a un LLM pasa por la capa `llm/provider.py`.** Ningún módulo invoca directamente a un SDK de proveedor.
8. **Todo acceso a Qdrant pasa por `retrieval/store.py`.** Ningún otro módulo importa `qdrant_client`, y toda función pública de `store.py` recibe `tenant_id` como primer argumento obligatorio. El filtro por tenant se aplica en la capa de aplicación: una consulta que lo olvide devuelve datos ajenos sin dar ningún error.
9. **Dependencias nuevas justificadas.** Proponer, explicar qué aporta frente a lo existente, esperar confirmación.

## Convenciones

- Python 3.12 gestionado con `uv`
- Anotaciones de tipo en toda firma pública; `mypy` debe pasar
- `ruff` para formato y linting
- Pydantic v2 para toda entrada y salida de la API
- SQLAlchemy 2.0 declarativo tipado sobre `asyncpg`; migraciones con Alembic
- Configuración vía `pydantic-settings`; ningún valor hardcodeado que deba ser configuración
- Modelos de ML con carga perezosa y reutilizados entre peticiones, nunca instanciados por petición
- Logging estructurado; sin `print`
- Commits semánticos: `feat:`, `fix:`, `refactor:`, `test:`, `docs:`, `chore:`

## Tests

- `pytest`, con estructura espejo de `src/`
- Toda ruta nueva requiere al menos un test de camino correcto y uno de error
- Qdrant y PostgreSQL se levantan con Docker en los tests de integración; no se mockean
- Los modelos de ML sí se mockean en tests unitarios

## Comandos

```bash
uv sync
docker compose up -d
uv run uvicorn ragmur.main:app --reload
uv run pytest
uv run ruff check --fix . && uv run ruff format .
uv run mypy src/
uv run python -m eval.run
uv run alembic upgrade head
```

## Método de trabajo

- Una tarea del roadmap por vez; marcar su casilla al completarla
- Si hay más de un enfoque razonable, exponer brevemente las opciones y su compromiso antes de implementar. Si solo hay uno sensato, proceder
- Cuando un cambio invalide algo de `ARCHITECTURE.md`, actualizar el documento en el mismo commit
- Si una decisión previa resulta equivocada al implementarla, señalarlo en lugar de rodearla
- Sin abstracciones para necesidades hipotéticas: la interfaz mínima que resuelve el caso actual

## Restricciones

- No dar por válida la calidad de recuperación sin pasarla por `eval/`
- No ajustar troceado, `top_k` ni umbrales sin datos de evaluación
- No publicar una cifra de recuperación sin indicar sobre qué etapa se calcula, ni una de latencia sin indicar el hardware
- No commitear `.env`, claves ni modelos descargados. De `eval/results/` se versionan los resúmenes por ejecución; los volcados por consulta se ignoran
- No modificar el esquema de base de datos sin migración
- No dejar `TODO` en el código: si es trabajo pendiente, va al roadmap

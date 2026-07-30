# Ragmur

Servicio RAG expuesto como API HTTP. Multi-tenant, con búsqueda híbrida, reranking y verificación de citas. El proveedor de LLM es intercambiable en caliente entre modelos locales (Ollama) y remotos (OpenAI, Anthropic, Gemini).

> **Estado: en construcción, fase 0.** El diseño está cerrado y documentado; todavía no hay código. Este README describe el sistema al que se dirige el proyecto, no lo que ya funciona. El detalle de qué está hecho y qué no está en [`ROADMAP.md`](ROADMAP.md).

## Qué hace

Ingiere documentos, los indexa y responde consultas en lenguaje natural **citando la fuente exacta** de cada afirmación. Las aplicaciones que lo consumen —un portfolio, un chatbot, un panel de gestión— no necesitan implementar nada de recuperación: llaman a la API.

## Modos de uso

Ragmur expone dos niveles. El LLM que redacta vive **dentro del servicio**.

| Endpoint | Qué devuelve | Para quién |
|---|---|---|
| `POST /v1/answer` | Respuesta redactada + citas + veredicto de verificación | Uso normal |
| `POST /v1/query` | Fragmentos relevantes con su fuente, sin redactar | Clientes que ya tienen su propio LLM |
| `POST /v1/search` | Igual que `query` pero sin capa agéntica | Depuración y evaluación |

`/search` y `/query` no son restos de una versión anterior: existen para poder medir la recuperación de forma aislada, sin que el modelo generativo enturbie la métrica. Es la razón por la que el proyecto tiene cifras y no impresiones.

## Arquitectura

```
Ingesta:     fichero → extracción → troceado → embeddings denso + disperso → Qdrant

Consulta:    consulta → [router: reescribe y reintenta si el resultado es pobre]
                      → búsqueda densa ─┐
                      → búsqueda BM25 ──┴→ fusión RRF → reranking → fragmentos
                      → redacción con LLM → verificación de citas → respuesta
```

## Aislamiento

Dos niveles, ambos aplicados como filtro en Qdrant:

- **`tenant_id`** separa aplicaciones cliente (portfolio, lazytripz, chatbot). Se deriva de la API key y nunca se acepta como parámetro de entrada.
- **`owner_id`** separa a los usuarios finales dentro de una misma aplicación. Es lo que hace que las facturas que sube un usuario no sean recuperables por otro del mismo tenant. Lo envía la aplicación cliente, que es quien autentica a sus usuarios.

## Estado de construcción

El desarrollo es estrictamente secuencial. Detalle y criterios de cierre en [`ROADMAP.md`](ROADMAP.md).

| Fase | Contenido | Estado |
|---|---|---|
| 0 | Esqueleto: FastAPI, Docker, migraciones, CI | — |
| 1 | Núcleo de recuperación sin LLM: ingesta, híbrida, reranking, verificación NLI, multi-tenancy, evaluación | — |
| 2 | LLM dentro del servicio: proveedor intercambiable, router agéntico, `/answer` | — |
| 3 | Ingesta multimodal: OCR y visión sobre documentos escaneados | — |

**La fase 1 no incluye ningún LLM. Es una etapa intermedia, no el diseño final.** Se construye así para poder medir la calidad de recuperación de forma aislada: si un fallo aparece con el LLM ya integrado, resulta imposible saber si falló la búsqueda o el modelo. La fase 2 introduce el LLM dentro del servicio y `/answer` pasa a ser el endpoint principal.

## Stack

| Capa | Tecnología |
|---|---|
| Lenguaje | Python 3.12, `uv` |
| API | FastAPI, Pydantic v2, Uvicorn |
| Base vectorial | Qdrant (vectores densos + dispersos, fusión RRF, multi-tenancy) |
| Metadatos | PostgreSQL, SQLAlchemy 2.0 sobre `asyncpg`, migraciones con Alembic |
| Embeddings | bge-m3 (multilingüe, 1024 dim) |
| Léxica | BM25 vía FastEmbed (vectores dispersos, IDF calculado por Qdrant) |
| Reranking | bge-reranker-v2-m3 (cross-encoder) |
| Verificación | mDeBERTa-v3-base-xnli (NLI) |
| Capa LLM | LiteLLM sobre Ollama, OpenAI, Anthropic, Gemini |
| Evaluación | Golden set propio; RAGAS a partir de la fase 2 |
| Despliegue | Docker Compose |

Sin frameworks RAG de alto nivel: el pipeline está escrito a mano. Única excepción, `langchain-text-splitters` como utilidad aislada de troceado.

## Arranque

```bash
cp .env.example .env
docker compose up -d          # Qdrant + PostgreSQL
uv sync
uv run alembic upgrade head
uv run uvicorn ragmur.main:app --reload
```

OpenAPI en `http://localhost:8000/docs`.

## Uso

```bash
# Ingesta
curl -X POST http://localhost:8000/v1/documents \
  -H "X-API-Key: $RAGMUR_API_KEY" \
  -F "file=@contrato.pdf"

# Ingesta en el espacio de un usuario final
curl -X POST http://localhost:8000/v1/documents \
  -H "X-API-Key: $RAGMUR_API_KEY" \
  -F "file=@factura.pdf" \
  -F "owner_id=u_4471"

# Respuesta redactada y verificada
curl -X POST http://localhost:8000/v1/answer \
  -H "X-API-Key: $RAGMUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "¿Cuál es el plazo de entrega?"}'

# Solo fragmentos, sin redactar
curl -X POST http://localhost:8000/v1/query \
  -H "X-API-Key: $RAGMUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "plazo de entrega", "top_k": 5}'
```

Cada tenant es un espacio aislado: documentos, índice y configuración de LLM propios. El tenant se deriva de la API key.

Cada resultado indica su origen mediante un localizador: página en PDF, sección o desplazamiento en DOCX, Markdown y texto plano. No todos los formatos tienen páginas, así que la precisión de la cita depende del formato de origen.

## Evaluación

```bash
uv run python -m eval.run
```

Golden set versionado en `eval/golden_set.yaml`, con consultas reales y con consultas negativas —sin respuesta en el corpus— para medir si el sistema sabe decir "no consta".

Cada métrica se aplica a la etapa que realmente mide. **recall@10** mide la recuperación: si el fragmento correcto entra en la lista. El reranking no añade documentos, reordena los que ya están, así que se mide con **nDCG@10** y **MRR**.

| Configuración | recall@10 | nDCG@10 | MRR |
|---|---|---|---|
| Solo densa | — | — | — |
| Solo BM25 | — | — | — |
| Híbrida (RRF) | — | — | — |
| Híbrida + reranking | — | — | — |

Con un golden set de 25–30 consultas, cada una vale unos 4 puntos porcentuales y el intervalo de confianza es más ancho que la diferencia entre configuraciones. Por eso se publica también el desglose de victorias, derrotas y empates por consulta, que con muestra pequeña es más honesto que un porcentaje agregado.

Toda cifra de latencia se publica indicando el hardware. Referencia del proyecto: RTX 5080. Los modelos funcionan también en CPU, pero es el modo degradado.

## Documentación

- [`ROADMAP.md`](ROADMAP.md) — fases, tareas y criterios de cierre
- [`ARCHITECTURE.md`](ARCHITECTURE.md) — contratos, modelo de datos y decisiones técnicas
- [`DECISIONS.md`](DECISIONS.md) — por qué el sistema es así, con las alternativas descartadas
- [`CLAUDE.md`](CLAUDE.md) — reglas para agentes de codificación

## Licencia

MIT

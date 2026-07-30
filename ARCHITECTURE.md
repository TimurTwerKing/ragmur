# Arquitectura

Qué se construye y con qué contratos. El porqué de cada decisión está en [`DECISIONS.md`](DECISIONS.md); el orden en [`ROADMAP.md`](ROADMAP.md).

## Estructura del repositorio

```
ragmur/
├── src/ragmur/
│   ├── main.py
│   ├── core/
│   │   ├── config.py           # pydantic-settings
│   │   ├── security.py         # resolución de tenant por API key
│   │   └── logging.py
│   ├── api/
│   │   ├── deps.py
│   │   ├── schemas.py
│   │   └── routes/
│   │       ├── health.py
│   │       ├── documents.py
│   │       ├── search.py
│   │       ├── verify.py
│   │       ├── query.py        # fase 2
│   │       └── answer.py       # fase 2
│   ├── ingestion/
│   │   ├── extractors/         # pdf.py, docx.py, text.py
│   │   ├── chunker.py
│   │   └── pipeline.py
│   ├── retrieval/
│   │   ├── embedder.py         # bge-m3
│   │   ├── sparse.py           # BM25 vía FastEmbed
│   │   ├── store.py            # cliente Qdrant — único punto de acceso
│   │   ├── hybrid.py           # fusión RRF
│   │   ├── reranker.py         # cross-encoder
│   │   └── router.py           # fase 2: reescritura y reintento
│   ├── verification/
│   │   ├── splitter.py
│   │   └── nli.py
│   ├── llm/                    # fase 2
│   │   ├── provider.py         # abstracción LiteLLM
│   │   └── generator.py
│   └── db/
│       ├── models.py
│       └── migrations/
├── eval/
│   ├── golden_set.yaml
│   ├── run.py
│   └── results/
├── tests/
├── docker-compose.yml
├── Dockerfile
└── pyproject.toml
```

Dirección de dependencias: `api/` importa de `ingestion/`, `retrieval/`, `verification/` y `llm/`. Nunca al revés. La lógica de dominio no conoce FastAPI.

Dos puntos de estrangulamiento obligatorios, ambos verificables con un test de imports:

- **Qdrant** solo se toca desde `retrieval/store.py`. Ningún otro módulo importa `qdrant_client`.
- **LLM** solo se invoca desde `llm/provider.py`. Ningún módulo importa un SDK de proveedor.

## Flujo

```
Ingesta
  fichero → extracción → troceado → embedding denso + disperso → Qdrant

Recuperación (fase 1)
  consulta → embedding denso ─┐
           → vector disperso ─┴→ RRF → reranking → fragmentos ordenados

Recuperación agéntica (fase 2)
  consulta → recuperación
           → si resultados < umbral: reescritura con LLM → reintento → fusión

Respuesta completa (fase 2)
  recuperación agéntica → redacción con LLM → verificación NLI → respuesta con citas
```

## Aislamiento: dos niveles

| Nivel | Campo | Origen | Obligatorio |
|---|---|---|---|
| Aplicación cliente | `tenant_id` | Derivado de la API key, nunca del cuerpo | Sí, en toda consulta |
| Usuario final | `owner_id` | Enviado por la aplicación cliente | No |

`tenant_id` separa aplicaciones (portfolio, lazytripz, chatbot). `owner_id` separa a los usuarios finales **dentro** de una aplicación: es lo que hace que las facturas de un usuario no sean recuperables por otro del mismo tenant.

Una consulta sin `owner_id` recorre todo el espacio del tenant, que es el comportamiento correcto para un corpus compartido como el del portfolio. Una consulta con `owner_id` filtra por `tenant_id AND owner_id`.

**Asunción explícita:** Ragmur confía en que la aplicación cliente envía el `owner_id` correcto, igual que confía en su API key. Autenticar al usuario final es responsabilidad de la aplicación cliente. Ver `DECISIONS.md` §3.8.

## Modelo de datos

### PostgreSQL

Acceso mediante SQLAlchemy 2.0 (declarativo tipado) sobre `asyncpg`. Migraciones con Alembic.

```sql
tenants
  id            uuid primary key
  name          text not null
  created_at    timestamptz not null default now()

api_keys
  id            uuid primary key
  tenant_id     uuid not null references tenants(id) on delete cascade
  key_id        text not null unique    -- prefijo público, viaja en la clave
  key_hash      text not null           -- HMAC-SHA256, no bcrypt
  name          text
  created_at    timestamptz not null default now()
  last_used_at  timestamptz
  revoked_at    timestamptz

documents
  id              uuid primary key
  tenant_id       uuid not null references tenants(id) on delete cascade
  owner_id        text                -- null = documento del tenant, no de un usuario
  filename        text not null
  mime_type       text not null
  content_sha256  text not null
  status          text not null   -- pending | processing | indexed | failed | unsupported
  page_count      int             -- null en formatos sin paginación
  chunk_count     int
  error           text
  created_at      timestamptz not null default now()

  unique (tenant_id, owner_id, content_sha256)
  index  (tenant_id, owner_id)

llm_configs                      -- fase 2
  tenant_id     uuid primary key references tenants(id) on delete cascade
  provider      text not null    -- ollama | openai | anthropic | gemini
  model         text not null
  fallback      jsonb
  updated_at    timestamptz not null default now()
```

Las claves de API viven en su propia tabla para permitir rotación, claves distintas por entorno y revocación sin cortar el servicio al tenant. El formato es `rgm_<key_id>_<secret>`: se localiza el registro por `key_id`, que está indexado, y se compara el secreto en tiempo constante.

El hash es HMAC-SHA256 con una clave de servidor, no bcrypt ni argon2: una API key es un secreto aleatorio de alta entropía, no una contraseña humana, y el coste deliberado de bcrypt se pagaría en cada petición. Ver `DECISIONS.md` §3.11.

`content_sha256` se calcula sobre los bytes del fichero. Una segunda subida idéntica dentro del mismo espacio devuelve el documento existente en lugar de duplicar sus chunks, que contaminarían el `top_k` y las cifras de evaluación.

### Qdrant

Colección única `ragmur_chunks` con vectores nombrados:

- `dense` — 1024 dimensiones, distancia coseno (bge-m3)
- `sparse` — vector disperso BM25, **declarado con `modifier=IDF`**

El `modifier=IDF` no es opcional. FastEmbed emite frecuencias de término; el componente IDF —el que da más peso a los términos raros— lo aplica Qdrant en el servidor a partir de las estadísticas de la colección. Sin él, la rama léxica puntúa solo por repetición y pierde exactamente lo que aporta frente a la búsqueda densa. Falla en silencio: el sistema funciona y recupera peor.

Consecuencia de la colección única: el IDF es global sobre todos los tenants, así que el corpus de uno influye en la puntuación léxica de otro. Es una asunción aceptada; si un tenant llega a dominar el corpus, se revisa.

Payload:

```json
{
  "tenant_id": "uuid",
  "owner_id": "u_4471",
  "document_id": "uuid",
  "filename": "contrato.pdf",
  "locator": { "type": "page", "start": 4, "end": 4 },
  "position": 17,
  "text": "..."
}
```

Índices de payload obligatorios sobre `tenant_id` (con `is_tenant=True`, que además reorganiza el almacenamiento por tenant) y sobre `owner_id`. Sin ellos, el filtrado degrada el rendimiento al crecer la colección.

Se descarta una colección por tenant: multiplica el consumo de memoria y complica el mantenimiento sin aportar aislamiento adicional real.

### Localizador

Los chunks no guardan un número de página, sino un localizador con tipo. Cada extractor rellena el que su formato puede sostener:

```json
{ "locator": { "type": "page",    "start": 4, "end": 5 } }
{ "locator": { "type": "section", "value": "3.2 Plazos" } }
{ "locator": { "type": "offset",  "start": 12040, "end": 12840 } }
```

| Formato | Tipo de localizador |
|---|---|
| PDF | `page` — `start` y `end` porque un chunk puede cruzar la frontera de página |
| DOCX | `section` si hay encabezados, `offset` si no. **No `page`:** la paginación la calcula el renderizador al maquetar, no está en el fichero |
| Markdown | `section` a partir del encabezado más cercano |
| TXT | `offset` |

El consumidor siempre recibe algo con lo que señalar la fuente; la precisión depende del formato de origen.

### Identidad de los chunks

`chunk_id` es válido dentro de una sesión de consulta: identifica un punto de Qdrant y sirve para encadenar `/search` con `/verify`. **No es estable entre reindexados.** Cambiar los parámetros de troceado —que es justo lo que hace la tarea 1.7 con los datos de evaluación— regenera todos los chunks con identificadores nuevos. Un consumidor que quiera guardar una cita debe guardar `document_id` + `locator`, no `chunk_id`.

## Contratos de API

Toda petición requiere `X-API-Key`. El tenant se deriva de la clave y nunca se acepta como parámetro de entrada.

### `POST /v1/documents`

Multipart: `file`, y opcionalmente `owner_id`.

Respuesta `202` — documento aceptado, ingesta en curso:

```json
{ "document_id": "uuid", "status": "pending" }
```

Respuesta `200` — el fichero ya existe en este espacio (mismo `content_sha256`):

```json
{ "document_id": "uuid", "status": "indexed", "duplicate": true }
```

### `GET /v1/documents` · `GET /v1/documents/{id}` · `DELETE /v1/documents/{id}`

Listado, detalle y borrado. El listado acepta `owner_id` como filtro. El borrado elimina el registro y todos sus puntos en Qdrant.

### `POST /v1/search` — fase 1

Recuperación directa, sin capa agéntica ni generación.

```json
{
  "query": "condiciones de entrega",
  "top_k": 5,
  "candidates": 30,
  "rerank": true,
  "owner_id": "u_4471",
  "document_ids": ["uuid"]
}
```

`candidates` es el tamaño de la lista **tras la fusión RRF** y antes del reranking. Cada rama recupera internamente lo necesario para alimentarla.

```json
{
  "results": [
    {
      "chunk_id": "uuid",
      "document_id": "uuid",
      "filename": "contrato.pdf",
      "locator": { "type": "page", "start": 4, "end": 4 },
      "text": "...",
      "score": 0.87,
      "dense_rank": 2,
      "sparse_rank": 1
    }
  ],
  "timings_ms": { "embed": "...", "search": "...", "rerank": "..." }
}
```

Los rangos por rama y los tiempos se devuelven a propósito: evidencian que la búsqueda es efectivamente híbrida y permiten depurar sin instrumentación externa. Este endpoint es la base de la evaluación.

Los valores de `timings_ms` se rellenan con mediciones reales al cerrar la tarea 1.6, indicando el hardware. Ver "Rendimiento" más abajo.

### `POST /v1/verify` — fase 1

```json
{
  "answer": "El plazo de entrega es de 48 horas y el transporte va incluido.",
  "chunk_ids": ["uuid-1", "uuid-2"],
  "owner_id": "u_4471"
}
```

Los `chunk_ids` se resuelven aplicando el mismo filtro que la búsqueda: `tenant_id` siempre, y `owner_id` si se envía. Un `chunk_id` que no pertenezca al espacio del solicitante se ignora en lugar de puntuarse. Sin esto, `/verify` sería un oráculo con el que sondear fragmentos ajenos dentro del mismo tenant.

```json
{
  "claims": [
    { "text": "El plazo de entrega es de 48 horas.", "verdict": "supported",    "score": 0.94, "best_chunk_id": "uuid-1" },
    { "text": "El transporte va incluido.",         "verdict": "contradicted", "score": 0.88, "best_chunk_id": "uuid-2" }
  ],
  "summary": { "supported": 1, "weak": 0, "unsupported": 0, "contradicted": 1 }
}
```

Cuatro veredictos, según umbrales configurables:

| Veredicto | Significado |
|---|---|
| `supported` | La fuente respalda la afirmación (entailment por encima del umbral) |
| `weak` | Respaldo por debajo del umbral, sin contradicción |
| `unsupported` | Ninguna fuente dice nada al respecto (neutral) |
| `contradicted` | Alguna fuente afirma lo contrario (contradiction) |

`contradicted` se separa de `unsupported` a propósito: son fallos de gravedad muy distinta, y el modelo NLI ya distingue las tres clases. Colapsarlos tiraría la señal más valiosa del verificador.

### `POST /v1/query` — fase 2

Igual que `/search` pero con router agéntico. Añade a la respuesta:

```json
{
  "rewritten_queries": ["experiencia con Docker", "administración de servidores"],
  "attempts": 2
}
```

### `POST /v1/answer` — fase 2

Endpoint principal del servicio.

```json
{
  "query": "¿Cuál es el plazo de entrega?",
  "verifier": "nli",
  "top_k": 5,
  "owner_id": "u_4471"
}
```

```json
{
  "answer": "El plazo de entrega estándar es de 48 horas.",
  "citations": [
    {
      "chunk_id": "uuid-1",
      "document_id": "uuid",
      "filename": "contrato.pdf",
      "locator": { "type": "page", "start": 4, "end": 4 }
    }
  ],
  "verification": { "supported": 1, "weak": 0, "unsupported": 0, "contradicted": 0 },
  "rewritten_queries": [],
  "provider": "ollama/qwen3:8b",
  "timings_ms": { "retrieval": "...", "generation": "...", "verification": "..." }
}
```

### `PUT /v1/config/llm` — fase 2

```json
{ "provider": "anthropic", "model": "claude-sonnet-4-6", "fallback": { "provider": "ollama", "model": "qwen3:8b" } }
```

Aplica en la siguiente petición, sin reinicio del servicio. `provider` y `model` se validan contra una lista permitida: llegan desde la API y determinan a quién se factura.

## Decisiones técnicas

Resumen. El razonamiento completo, con las alternativas descartadas, está en [`DECISIONS.md`](DECISIONS.md).

### El LLM se integra en la fase 2, no en la 1

El sistema final ejecuta el LLM dentro de Ragmur. La fase 1 se construye sin él por una razón de método: con generación activa, un resultado incorrecto no permite distinguir si falló la recuperación o el modelo. Midiendo primero la recuperación aislada se obtiene una línea base sobre la que atribuir cualquier cambio posterior.

Los endpoints sin generación permanecen tras la fase 2 como interfaz de bajo nivel y como base de la suite de evaluación.

### Sin frameworks RAG de alto nivel

No se usan LangChain ni LlamaIndex como framework. El pipeline es corto y escribirlo directamente mantiene cada paso explícito y depurable. Se admite `langchain-text-splitters` como utilidad aislada.

### Fusión RRF en lugar de combinación de scores

Los scores de similitud coseno y BM25 no son comparables; normalizarlos requiere calibración frágil y dependiente del corpus. RRF combina posiciones, no puntuaciones, y funciona sin ajuste. Qdrant lo implementa de forma nativa.

### Verificación por NLI

Un modelo NLI clasifica pares (premisa, hipótesis) como entailment, neutral o contradiction. Frente a un LLM para la misma tarea: coste nulo por llamada, ejecución en CPU, sin dependencia externa y sin riesgo de que el verificador alucine.

Limitaciones asumidas:

1. La segmentación en frases aproxima la descomposición en afirmaciones atómicas, que un LLM realizaría mejor.
2. El contexto del modelo es limitado (~512 tokens); los fragmentos largos se recorren por ventanas conservando la puntuación máxima.
3. Los modelos XNLI se entrenaron con premisas de una sola frase. Un chunk de 800 caracteres como premisa es un régimen distinto, y los umbrales no se heredan de la literatura: se calibran sobre el set de prueba propio.
4. Una frase que arranca con una referencia anafórica ("este plazo", "dicha cláusula") es inverificable aislada y producirá falsos `unsupported`. Mitigación prevista: concatenar la frase anterior a la hipótesis.

La fase 2 añade `verifier=llm` como alternativa y mide el grado de acuerdo entre ambos. Esa comparación es un resultado del proyecto.

### bge-m3 para embeddings

Multilingüe (corpus mixto español/inglés), buen rendimiento en recuperación, ejecutable en CPU. La interfaz `embedder.py` aísla la decisión por si conviene migrar a embeddings por API.

### Multi-tenancy desde el inicio, en dos niveles

`tenant_id` se propaga desde la API key hasta el filtro de Qdrant; `owner_id` subdivide el espacio del tenant por usuario final. Todo test de búsqueda comprueba ambos aislamientos. Los dos campos viven en el payload de Qdrant, así que introducirlos más tarde obligaría a reindexar el corpus completo.

### Ingesta en tareas de fondo

La fase 1 procesa la ingesta con `BackgroundTasks` de FastAPI, con un límite de concurrencia explícito. Una cola dedicada (Redis + arq) se introduce solo cuando el volumen lo justifique.

Consecuencia asumida: un reinicio del proceso pierde los trabajos en vuelo. Al arrancar, los documentos que lleven demasiado tiempo en `processing` se marcan como `failed` para que no queden colgados indefinidamente.

## Rendimiento

Hardware de referencia del proyecto: **RTX 5080** en homelab, que aloja además Ollama para la fase 2.

Los tres modelos de la fase 1 funcionan también en CPU, y ese modo se mantiene soportado para que el proyecto sea reproducible sin GPU. Pero es el modo degradado, no el objetivo: el cross-encoder de reranking sobre 30 candidatos es el coste dominante de una consulta, y la diferencia entre CPU y GPU supera el orden de magnitud.

Regla: toda cifra de `timings_ms` publicada indica sobre qué hardware se midió. Los ejemplos de este documento llevan `"..."` hasta que existan mediciones reales.

## Evaluación

`eval/golden_set.yaml`:

```yaml
- id: q001
  query: "¿Cuál es el plazo de entrega estándar?"
  expected:
    - document: "contrato-proveedor.pdf"
      locator: { type: page, start: 4 }
  note: "aparece como 'plazo máximo de suministro'; la palabra entrega no consta"

- id: q014
  query: "¿Cuál es la penalización por retraso?"
  expected: []
  note: "negativa: el contrato no menciona penalizaciones. El sistema debe no recuperar nada relevante"
```

`expected` es una lista: una consulta puede tener varias fuentes válidas, y tratarla como una sola convierte recall@k en un simple hit-rate. Una lista vacía marca una **consulta negativa**, sin respuesta en el corpus: son las que miden si el sistema sabe decir "no consta", y son lo primero que probará quien vea la demo.

El campo `note` documenta por qué una consulta es difícil, y evita que el golden set degenere en casos triviales que siempre pasan.

### Qué métrica mide qué etapa

| Etapa | Métrica | Motivo |
|---|---|---|
| Recuperación (densa, BM25, híbrida) | recall@k | Mide si el fragmento correcto entra en la lista |
| Reranking | nDCG@10, MRR | El reranker no añade documentos, reordena los ya recuperados. recall@k apenas se mueve y haría parecer inútil la capa |
| Sistema completo | latencia p95 | Con el hardware indicado |

Se ejecuta sobre las cuatro configuraciones —densa, BM25, híbrida, híbrida + reranking— para atribuir la mejora a cada capa.

Con un golden set de 25–30 consultas, cada una vale unos 4 puntos porcentuales y el intervalo de confianza es más ancho que la diferencia que se busca. Por eso se reporta también el **desglose de victorias, derrotas y empates por consulta** entre configuraciones: con muestra pequeña es más honesto y más informativo que un porcentaje agregado.

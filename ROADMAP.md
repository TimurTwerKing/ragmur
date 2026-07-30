# Roadmap

## Reglas de ejecución

1. Las fases son secuenciales. No se abre la fase 2 hasta que la 1 cumple todos sus criterios de cierre, evaluación incluida.
2. Dentro de cada fase las tareas siguen orden de dependencia. Cada una debe ejecutarse y verificarse antes de pasar a la siguiente.
3. Al completar una tarea, se marca su casilla en este documento.

## Estado final del sistema

Para evitar malentendidos durante la construcción: **el destino del proyecto es un servicio que recibe una pregunta y devuelve una respuesta redactada, citada y verificada, con el LLM ejecutándose dentro de Ragmur.**

La fase 1 se construye sin LLM porque es la única forma de medir la calidad de recuperación de forma aislada. Es una etapa intermedia deliberada, no una decisión de diseño permanente. Los endpoints que devuelven solo fragmentos (`/search`, `/query`) se conservan después como interfaz de bajo nivel y como base de la evaluación.

---

## Qué se hace y qué se obtiene en cada fase

Resumen en lenguaje llano, antes del detalle técnico. Cada fase termina con algo que funciona y que se puede enseñar.

### Fase 0 — El esqueleto

**Qué se hace.** Montar el proyecto vacío: dependencias, contenedores de Qdrant y PostgreSQL, una aplicación FastAPI que arranca, migraciones de base de datos, linter, tests y CI.

**Qué se obtiene.** Un repositorio donde `docker compose up` y `uvicorn` levantan un servicio que responde en `/health` diciendo si la base de datos y la base vectorial están vivas. Todavía no hace nada útil, pero a partir de aquí todo lo que se escriba tiene tests, tipos y CI detrás.

**Qué no hay todavía.** Nada relacionado con documentos ni con búsqueda.

**Se termina cuando.** `/health` devuelve 200 con las dos dependencias conectadas.

### Fase 1 — El buscador

Es la fase larga, y la que da su valor al proyecto.

**Qué se hace.**

1. Tenants y claves de API, para que cada aplicación tenga su espacio aislado *(1.1)*.
2. Subir documentos y extraerles el texto: PDF, DOCX, TXT y Markdown *(1.2)*.
3. Partir ese texto en fragmentos manejables, cada uno sabiendo de qué página o sección viene *(1.3)*.
4. Convertir cada fragmento en dos representaciones —una que captura el significado, otra que captura las palabras exactas— y guardarlas en Qdrant *(1.4)*.
5. Buscar por las dos vías a la vez y fusionar los dos rankings en uno *(1.5)*.
6. Coger los mejores candidatos y reordenarlos con un modelo más preciso y más lento *(1.6)*.
7. Medir cuánto aporta cada capa, con un conjunto fijo de preguntas de referencia *(1.7)*.
8. Verificar afirmaciones: dado un texto y unos fragmentos, decir si esos fragmentos lo respaldan *(1.8)*.

**Qué se obtiene.**

- Subes un PDF por HTTP y, unos segundos después, puedes preguntar por su contenido.
- Preguntas por "condiciones de entrega" y recibes **los párrafos concretos** donde se trata, con el nombre del fichero y la página. Encuentra también los que usan otras palabras —"plazo máximo de suministro"—, que es exactamente lo que un buscador por palabras clave no hace.
- Aislamiento real y probado con tests: cada aplicación ve solo sus documentos, y cada usuario final solo los suyos dentro de su aplicación.
- Un endpoint al que le pasas una frase y unos fragmentos y te responde si esos fragmentos la respaldan, la contradicen o no dicen nada al respecto.
- **Una tabla con números.** Cuánto recupera la búsqueda por significado sola, cuánto la de palabras sola, cuánto las dos combinadas y cuánto añade el reordenado. Reproducible, con fecha y hardware. Es lo que separa este proyecto de una demo.
- Todo desplegado en el homelab sobre documentos reales, con una demo accesible públicamente.

**Qué no hay todavía.**

**No hay respuesta redactada.** Preguntas "¿cuál es el plazo de entrega?" y recibes el párrafo donde lo pone, no la frase "el plazo es de 48 horas". La demo pública de esta fase es un buscador con citas, no un chatbot.

Y la verificación funciona al revés de lo que parece: como todavía no hay nada que redacte, el texto a verificar lo envía quien llama a la API. Sirve para demostrar el mecanismo —pegas una afirmación y ves si el corpus la sostiene— pero aún no está enganchada a una generación propia.

**Por qué se hace así.** Con el LLM presente desde el principio, una respuesta mala no permite saber si falló la búsqueda o el modelo. Midiendo primero la búsqueda sola queda una línea base contra la que comparar todo lo que venga después.

**Se termina cuando.** Hay una cifra reproducible de recall@10 para las cuatro configuraciones, la demo está en pie y los tests de aislamiento pasan.

### Fase 2 — El que responde

**Qué se hace.**

1. Una capa que habla con cualquier proveedor de LLM —Ollama en local, o OpenAI, Anthropic y Gemini— y que se cambia sin reiniciar el servicio *(2.1)*.
2. Un router que, cuando la búsqueda devuelve poco, reformula la pregunta con el LLM y lo reintenta *(2.2)*.
3. `/answer`: busca, redacta la respuesta, la verifica frase a frase y la devuelve con sus citas *(2.3)*.

**Qué se obtiene.**

- Preguntas en lenguaje natural y recibes **una respuesta escrita**, con las citas de dónde sale cada afirmación y un veredicto por frase que dice si la fuente la respalda.
- Cambiar de un modelo local en la GPU propia a Claude o GPT con una sola petición HTTP, sin tocar el servicio ni reiniciarlo.
- Segunda tanda de números: cuánto mejora la recuperación al dejar que el LLM reformule la pregunta, y cuánto coinciden el verificador NLI y un verificador LLM sobre las mismas afirmaciones.

**Qué no hay todavía.** Los documentos escaneados o fotografiados siguen rechazándose con estado `unsupported`.

**Se termina cuando.** `/answer` responde con citas verificadas, el cambio de proveedor funciona en caliente y las dos cifras comparadas están publicadas.

### Fase 3 — Los documentos que hoy se rechazan

Solo se aborda si aparece un corpus real que lo exija.

**Qué se hace.** OCR para escaneados limpios, extracción de tablas y estructura, y un LLM de visión para los documentos degradados. Cada documento sale con una puntuación de confianza, y los que quedan por debajo del umbral pasan por una cola de revisión manual.

**Qué se obtiene.** Que la foto de una factura deje de dar `unsupported` y pase a ser consultable como cualquier otro documento.

**Qué no cambia.** Ni la búsqueda ni la generación. Esta fase solo produce texto, y ese texto entra por el mismo sitio que el de un PDF normal.

### Resumen

| Al terminar | Qué tienes en la mano |
|---|---|
| Fase 0 | Un repositorio limpio, con CI en verde y un servicio que arranca |
| Fase 1 | Un buscador sobre documentos propios, con citas exactas, aislamiento multi-tenant y una tabla de cifras que justifica cada capa |
| Fase 2 | Un servicio que responde preguntas redactando y citando la fuente, con el proveedor de LLM intercambiable en caliente |
| Fase 3 | Lo mismo, aceptando además documentos escaneados y fotografiados |

---

## Fase 0 — Esqueleto

- [ ] `pyproject.toml` con `uv`, Python 3.12
- [ ] Estructura de paquetes según `ARCHITECTURE.md`
- [ ] `docker-compose.yml` con Qdrant y PostgreSQL
- [ ] Configuración con `pydantic-settings` y `.env.example`
- [ ] SQLAlchemy 2.0 + `asyncpg`, y Alembic configurado con la primera migración
- [ ] App FastAPI con `GET /health` que reporta estado de Qdrant y PostgreSQL
- [ ] Logging estructurado en JSON
- [ ] `ruff` y `mypy` configurados
- [ ] `pytest` con test de humo sobre `/health`
- [ ] GitHub Actions: lint y tests en cada push

**Cierre:** `docker compose up` levanta el conjunto, `alembic upgrade head` aplica la migración inicial y `/health` devuelve 200 con ambas dependencias conectadas.

---

## Fase 1 — Núcleo de recuperación

Sin LLM en ninguna tarea de esta fase.

> **Nota de método.** Las tareas 1.2 a 1.6 conviene recorrerlas dos veces: una primera pasada mínima que atraviese el camino completo —un fichero TXT, troceado fijo, solo búsqueda densa, sin reranking— hasta tener `POST /v1/documents` y `POST /v1/search` respondiendo de punta a punta, y una segunda pasada añadiendo cada capa sobre algo que ya funciona y ya se puede medir.
>
> No altera el contenido ni el orden de las tareas, solo su granularidad. La ventaja es tocar embeddings y Qdrant desde el principio, y añadir BM25, RRF y reranking sobre un sistema observable en lugar de sobre teoría.

### 1.1 Multi-tenancy y autenticación

- [ ] Tabla `tenants` (id, nombre, timestamps)
- [ ] Tabla `api_keys` (tenant_id, key_id, key_hash, nombre, revoked_at, last_used_at)
- [ ] Formato de clave `rgm_<key_id>_<secret>`: búsqueda por `key_id` indexado, comparación del secreto en tiempo constante
- [ ] Hash con HMAC-SHA256 y clave de servidor, **no bcrypt ni argon2** (se pagaría ~100 ms en cada petición, y una API key no es una contraseña humana)
- [ ] Dependencia FastAPI que resuelve el tenant desde `X-API-Key` y rechaza claves revocadas
- [ ] CLI para crear tenant, emitir API key y revocarla
- [ ] Test: sin clave → 401; clave revocada → 401; con clave de otro tenant → no accede a datos ajenos

El aislamiento se implementa aquí. Añadirlo después obliga a reindexar todo el corpus.

**Cierre:** dos tenants creados desde la CLI, y la clave de uno no permite ver nada del otro.

### 1.2 Ingesta

- [ ] Tabla `documents` (id, tenant_id, owner_id, filename, mime, content_sha256, status, page_count, chunk_count, error, timestamps)
- [ ] `POST /v1/documents` — multipart con `file` y `owner_id` opcional, responde 202 con `document_id`
- [ ] Deduplicación por `content_sha256`: subida repetida en el mismo espacio devuelve 200 con el documento existente, sin reindexar
- [ ] Extractor PDF (PyMuPDF) con localizador de tipo `page`, con `start` y `end` porque un chunk puede cruzar la frontera de página
- [ ] Extractor DOCX (python-docx) con localizador de tipo `section` u `offset`
- [ ] Extractor TXT y Markdown con localizador `offset` y `section` respectivamente
- [ ] PDF sin capa de texto → estado `unsupported` con mensaje explícito. Definir el umbral de detección (caracteres extraídos por página)
- [ ] Límites de subida: tamaño máximo en bytes, número máximo de páginas y timeout de extracción
- [ ] Validación del MIME por contenido real, no por la cabecera del multipart
- [ ] Límite de concurrencia en las `BackgroundTasks` (semáforo de 1–2): sin él, N subidas simultáneas compiten por la GPU y se degradan entre sí
- [ ] Al arrancar el proceso, marcar como `failed` los documentos que lleven demasiado tiempo en `processing`. Sin esto, un reinicio los deja colgados para siempre
- [ ] `GET /v1/documents` (con filtro por `owner_id`), `GET /v1/documents/{id}`, `DELETE /v1/documents/{id}` con borrado en cascada de vectores

**Nota.** `python-docx` no puede dar número de página: la paginación la calcula el renderizador al maquetar y no existe en el fichero. TXT y Markdown tampoco tienen páginas. Por eso el localizador tiene tipo y cada extractor rellena el que su formato sostiene.

**Cierre:** un PDF se ingiere conservando la página de origen; un DOCX se ingiere conservando sección o desplazamiento; subir dos veces el mismo fichero no duplica chunks.

### 1.3 Troceado

- [ ] Troceador recursivo por caracteres, tamaño y solape configurables
- [ ] Cada chunk conserva `document_id`, `tenant_id`, `owner_id`, localizador, posición ordinal y texto
- [ ] Valores iniciales 800 / 120, ajustados posteriormente con resultados de 1.7
- [ ] Tests: sin pérdida de texto, sin chunks vacíos, localizador coherente con el texto del chunk

**Nota.** Cambiar los parámetros de troceado tras 1.7 regenera todos los chunks con `chunk_id` nuevos y obliga a reindexar el corpus. Está previsto, y por eso 1.4 incluye un comando de reindexado.

**Cierre:** un documento troceado y vuelto a unir reproduce el texto original, y cada chunk apunta a un localizador correcto.

### 1.4 Indexado

- [ ] Servicio de embeddings bge-m3 con carga perezosa del modelo
- [ ] Generación de vectores dispersos BM25 con FastEmbed
- [ ] Colección `ragmur_chunks` con vectores nombrados `dense` y `sparse`
- [ ] **El vector `sparse` se declara con `modifier=IDF`** — ver nota, es el fallo silencioso más probable de toda la fase
- [ ] Índice de payload sobre `tenant_id` con `is_tenant=True`
- [ ] Índice de payload sobre `owner_id`
- [ ] Escritura por lotes
- [ ] Transición del documento a estado `indexed`
- [ ] Comando de reindexado (un documento, un tenant o todo el corpus) que reaplica troceado e indexado desde el fichero original

**Nota sobre `modifier=IDF`.** FastEmbed emite frecuencias de término; el componente IDF —el que da más peso a los términos raros del corpus— lo aplica Qdrant en el servidor, y solo si el vector disperso se declaró con ese modificador. Sin él, la rama léxica puntúa por pura repetición y pierde exactamente aquello que justifica tenerla junto a la búsqueda densa. No lanza ningún error: el sistema funciona y recupera peor. Solo se detecta con evaluación.

Consecuencia de la colección única: el IDF es global sobre todos los tenants, así que el corpus de uno influye en la puntuación léxica de otro. Asunción aceptada, documentada en `DECISIONS.md`.

**Cierre:** los chunks de un documento ingerido existen en Qdrant con ambos vectores, su `tenant_id` y su `owner_id`.

### 1.5 Búsqueda híbrida

- [ ] `POST /v1/search` con `query`, `top_k`, `candidates`, `rerank`, `owner_id` y filtro por documento
- [ ] Rama densa: embedding de consulta + búsqueda vectorial
- [ ] Rama léxica: vector disperso BM25
- [ ] Fusión RRF mediante la Query API de Qdrant
- [ ] `candidates` definido como el tamaño de la lista **tras la fusión** y antes del reranking; cada rama recupera internamente lo necesario para alimentarla
- [ ] Filtro por `tenant_id` obligatorio en toda consulta, con test que lo garantice
- [ ] Filtro adicional por `owner_id` cuando la petición lo incluye, con test de aislamiento entre dos `owner_id` del mismo tenant
- [ ] Ningún módulo importa `qdrant_client` salvo `retrieval/store.py`, y toda función pública de `store.py` recibe `tenant_id` como primer argumento obligatorio. Test de imports que lo verifique
- [ ] Respuesta con `chunk_id`, `document_id`, filename, localizador, texto, score y rango por rama

**Nota.** El filtro por tenant se aplica en la capa de aplicación: una consulta que lo olvide devuelve datos ajenos sin dar ningún error. De ahí el punto de estrangulamiento en `store.py`, que es la misma idea que la regla del `llm/provider.py` aplicada a Qdrant.

**Cierre:** una consulta con sinónimos recupera fragmentos que la rama léxica no encuentra, y un término exacto poco frecuente recupera lo que la rama densa no prioriza.

### 1.6 Reranking

- [ ] Cross-encoder bge-reranker-v2-m3 sobre los candidatos de la fusión
- [ ] `candidates` por defecto 30 → `top_k` por defecto 5
- [ ] Los candidatos se puntúan en un único lote, no en bucle
- [ ] Activable y desactivable por petición
- [ ] Latencia añadida registrada en `timings_ms`
- [ ] Objetivo de latencia p95 de `/search` con reranking activo, fijado y medido, indicando el hardware

**Nota.** El reranking es el coste dominante de una consulta. Sin un objetivo de latencia declarado, la capa entra en el sistema sin presupuesto y no hay forma de decidir si compensa.

**Cierre:** la misma consulta con `rerank: true` y `rerank: false` devuelve los mismos fragmentos en distinto orden, y la diferencia de latencia queda registrada.

### 1.7 Evaluación

- [ ] `eval/golden_set.yaml` con 25–30 consultas reales y sus fuentes esperadas
- [ ] `expected` es una **lista** de fuentes, no una sola: una consulta puede tener varios fragmentos válidos, y tratarla como única convierte recall@k en un simple hit-rate
- [ ] Incluir **consultas negativas** (`expected: []`), sin respuesta en el corpus
- [ ] `eval/run.py` calculando recall@k, MRR, nDCG@10 y latencia p95
- [ ] Ejecución sobre cuatro configuraciones: densa, BM25, híbrida, híbrida + reranking
- [ ] Desglose de victorias, derrotas y empates por consulta entre configuraciones
- [ ] Resúmenes versionados en `eval/results/` con fecha, configuración y hardware
- [ ] Reajuste de troceado, `top_k` y umbrales con los resultados obtenidos, reindexado y segunda medición
- [ ] Tabla del README rellenada con cifras reales

**Qué métrica mide qué etapa.** recall@k mide la **recuperación**: si el fragmento correcto entra en la lista. El reranking no añade documentos, reordena los que ya están, así que recall@k apenas se mueve al activarlo y la fila "híbrida + reranking" parecería una capa inútil. El orden lo miden **nDCG@10 y MRR**. Publicar una cifra sin decir sobre qué etapa se calcula no vale.

**Sobre el tamaño de la muestra.** Con 25–30 consultas, cada una vale unos 4 puntos porcentuales y el intervalo de confianza es más ancho que la diferencia que se busca entre configuraciones. Por eso el desglose por consulta no es un extra: con muestra pequeña, "híbrida gana en 9, pierde en 2, empata en 14" es más honesto y más informativo que "62 % frente a 58 %".

Esta tarea no es opcional ni se pospone. Es la que permite ajustar troceado, `top_k` y umbrales con datos en lugar de intuición.

**Cierre:** existen cifras reproducibles de recall@10 para las cuatro configuraciones, y de nDCG@10 y MRR para híbrida con y sin reranking.

### 1.8 Verificación de citas

- [ ] Segmentador de respuesta en frases, sin LLM
- [ ] Servicio NLI con mDeBERTa-v3-base-xnli
- [ ] Los pares (premisa, hipótesis) se puntúan en lote, no en bucle
- [ ] Ventaneo para fragmentos que exceden el contexto del modelo, conservando puntuación máxima
- [ ] Cuatro veredictos: `supported`, `weak`, `unsupported`, `contradicted`
- [ ] `POST /v1/verify` — entrada: texto redactado + `chunk_ids` + `owner_id` opcional; salida: veredicto por frase con score
- [ ] Los `chunk_ids` se resuelven filtrando por `tenant_id` y, si se envía, por `owner_id`. Un `chunk_id` fuera del espacio del solicitante se ignora, no se puntúa
- [ ] Umbrales configurables, **calibrados sobre el set de prueba propio**
- [ ] Mitigación de correferencia: concatenar la frase anterior a la hipótesis
- [ ] Set de prueba con afirmaciones deliberadamente falsas y afirmaciones deliberadamente contradichas

**Por qué `contradicted` va separado.** El modelo NLI ya distingue tres clases. Que la fuente no diga nada sobre una afirmación (`unsupported`) y que la fuente afirme lo contrario (`contradicted`) son fallos de gravedad muy distinta, y el segundo es la mejor demostración de lo que hace el verificador. Colapsarlos tira la señal más valiosa.

**Por qué los umbrales se calibran y no se copian.** Los modelos XNLI se entrenaron con premisas de una sola frase. Un chunk de 800 caracteres como premisa es un régimen distinto al de entrenamiento, y los umbrales publicados en la literatura no se transfieren.

**Falsos positivos esperados.** Una frase que empieza por "este plazo" o "dicha cláusula" es inverificable aislada y se clasificará como `unsupported` sin que haya nada mal en ella. De ahí la mitigación por correferencia.

**Cierre:** una afirmación inventada sobre un documento real se clasifica como `unsupported`, y una afirmación que contradice al documento se clasifica como `contradicted`.

### 1.9 Cierre de fase

- [ ] Tests sobre ingesta, búsqueda, aislamiento entre tenants y aislamiento por `owner_id`
- [ ] `/health` (proceso vivo) separado de `/ready` (modelos cargados y dependencias conectadas)
- [ ] Rate limiting por API key
- [ ] Borrado completo de un tenant, incluidos sus puntos en Qdrant
- [ ] README con arquitectura, arranque y cifras de evaluación
- [ ] Desplegado en el homelab con un corpus real como primer tenant
- [ ] Demo accesible públicamente

**Nota sobre `/ready`.** Los modelos tardan decenas de segundos en cargar. Si el health check devuelve 200 antes de que estén listos, el orquestador manda tráfico y las primeras peticiones expiran.

**Nota sobre el borrado de tenant.** `documents` cascadea desde `tenants` por clave foránea, pero los puntos de Qdrant no: hay que borrarlos explícitamente. Con usuarios finales subiendo facturas, el borrado a petición es un requisito, no un extra.

**Cierre de la fase 1.** Se cumplen a la vez: los criterios de cierre de 1.1 a 1.8; la suite de tests pasa incluyendo aislamiento por tenant y por `owner_id`; el servicio está desplegado en el homelab con un corpus real; la demo es accesible; y el README publica las cifras de evaluación con su fecha y su hardware. Hasta entonces no se abre la fase 2.

---

## Fase 2 — LLM dentro del servicio

### 2.1 Capa de proveedor

- [ ] LiteLLM como interfaz única sobre Ollama, OpenAI, Anthropic y Gemini
- [ ] Tabla `llm_configs`: proveedor y modelo por tenant
- [ ] Lista permitida de `provider/model`: llegan desde la API y determinan a quién se factura
- [ ] Tabla de uso: tokens y coste por petición y por tenant
- [ ] Presupuesto por tenant con corte al superarlo
- [ ] `PUT /v1/config/llm` — cambio de proveedor y modelo sin reiniciar el servicio
- [ ] Fallback configurable ante fallo del proveedor primario
- [ ] Test con Ollama local y con un proveedor remoto

**Nota.** `llm_configs` no guarda credenciales del proveedor. Si un tenant configura `provider: openai`, se gasta la cuenta del operador. De ahí la lista permitida y el presupuesto; si en algún momento hacen falta credenciales por tenant, se cifran en reposo.

### 2.2 Router de recuperación

- [ ] Detección de recuperación pobre: número de resultados o score máximo bajo umbral
- [ ] Reescritura de la consulta en 2–4 sub-consultas mediante el LLM
- [ ] Reintento contra la búsqueda híbrida y fusión RRF de los resultados de todas las sub-consultas
- [ ] Límites de reintentos y de latencia total
- [ ] `POST /v1/query` — recuperación agéntica; devuelve las sub-consultas empleadas
- [ ] Trazas de decisión en la respuesta

### 2.3 Generación y verificación integradas

- [ ] `POST /v1/answer` — recupera, redacta con el LLM configurado, verifica y devuelve respuesta con citas
- [ ] Toda afirmación clasificada `unsupported` o `contradicted` se marca o se elimina según política configurable
- [ ] Modo `verifier=nli|llm`
- [ ] Métrica de acuerdo entre verificador NLI y verificador LLM sobre el golden set
- [ ] Delimitación explícita del contexto recuperado en el prompt

**Nota sobre inyección de prompt.** El contenido ingerido llega al LLM, así que un documento con instrucciones incrustadas puede manipular la redacción. **La verificación NLI no protege de esto:** una respuesta manipulada puede ser enteramente `supported` por los chunks citados. Se documenta como riesgo asumido y se mitiga parcialmente delimitando el contexto en el prompt.

`/answer` es el endpoint principal del servicio a partir de esta fase.

### 2.4 Métricas complementarias

- [ ] Integración de RAGAS

**Nota.** RAGAS va aquí y no en la fase 1 porque sus métricas principales (`faithfulness`, `answer_relevancy`, `context_precision`, `context_recall`) usan un LLM como juez, lo que choca con la regla de que la fase 1 no usa LLM. Si en algún momento interesa adelantarlas, solo son admisibles las variantes no-LLM (`NonLLMContextRecall`, `NonLLMContextPrecisionWithReference`).

### 2.5 Cierre de fase

- [ ] Subconjunto del golden set marcado como **consultas difíciles**: aquellas en las que la búsqueda directa recupera poco y el router debe dispararse
- [ ] Evaluación comparada: recall@10 de `/search` frente a `/query`, sobre el golden set completo y sobre el subconjunto difícil
- [ ] Cifra de acuerdo NLI vs LLM documentada
- [ ] README actualizado con ambas mejoras medidas

**Nota.** El router solo actúa cuando la recuperación es pobre. Medido sobre el golden set completo —construido en 1.7 con consultas que la búsqueda directa ya resuelve— la comparación `/search` frente a `/query` saldrá plana por construcción, igual que recall@10 salía plano para el reranking. De ahí el subconjunto difícil: es donde la capa agéntica puede demostrar algo.

**Cierre de la fase 2.** `/answer` devuelve respuesta redactada, citas y veredicto de verificación; el proveedor de LLM se cambia sin reiniciar y el fallback funciona; y están publicadas las dos cifras: mejora del router sobre el subconjunto difícil, y grado de acuerdo entre verificador NLI y verificador LLM.

---

## Fase 3 — Ingesta multimodal

Se aborda únicamente cuando exista un corpus real que lo exija.

- [ ] Enrutado de los documentos ya marcados `unsupported` en 1.2 hacia el pipeline de OCR
- [ ] OCR con Tesseract para documentos limpios
- [ ] Extracción de tablas y estructura con docling
- [ ] Extracción con LLM de visión vía LiteLLM para documentos degradados
- [ ] Puntuación de confianza por documento
- [ ] Cola de revisión humana bajo umbral de confianza
- [ ] Endpoints de revisión: listar pendientes, aprobar, corregir

El output de esta fase es texto que entra en el pipeline de ingesta existente. No modifica la búsqueda ni la generación.

**Cierre de la fase 3.** Un PDF escaneado y una fotografía de un documento se ingieren, se indexan y son recuperables con la misma calidad de cita que un PDF con capa de texto, o quedan en la cola de revisión con su puntuación de confianza.

---

## Posteriores

- Servidor MCP sobre Ragmur para clientes compatibles
- Ingesta incremental y reindexado sin downtime
- Cache de embeddings de consultas frecuentes
- Panel de administración de tenants y documentos

# Contexto de diseño

Registro de las decisiones tomadas durante el diseño de Ragmur, con sus alternativas descartadas y el motivo. Sirve para que quien retome el proyecto —persona o agente— entienda **por qué** el sistema es como es y no vuelva a abrir discusiones ya cerradas.

Este documento no sustituye a `ARCHITECTURE.md` (qué se construye) ni a `ROADMAP.md` (en qué orden). Explica el razonamiento detrás de ambos.

---

## 1. Objetivo del proyecto

Ragmur es un proyecto de portfolio con un objetivo doble:

- **Técnico:** un servicio RAG reutilizable, consumido por varias aplicaciones propias (portfolio personal, lazytripz, chatbot con datos personales) desde una única instancia multi-tenant.
- **Profesional:** demostrar competencia en RAG con capas de calidad medibles, en un mercado donde abundan las demos superficiales.

La segunda condiciona el diseño más de lo que parece: **el proyecto debe producir cifras, no impresiones.** De ahí que la evaluación sea una tarea de primer nivel y no un extra.

## 2. Origen del alcance

El punto de partida fueron tres "proyectos RAG" presentados como opciones alternativas:

1. RAG con búsqueda híbrida, reranking y verificación de citas.
2. RAG multimodal con OCR y extracción de documentos degradados.
3. RAG agéntico con router de recuperación y reescritura de consultas.

**Decisión: no son tres proyectos alternativos, sino tres capas del mismo sistema con dependencia entre sí.** La opción 3 opera sobre los resultados de la 1; la opción 2 alimenta el índice del que bebe la 1. De ahí el orden de fases: núcleo → agéntico → multimodal.

Motivo del orden: construir lógica agéntica sobre una recuperación deficiente produce resultados igual de malos pero más caros y más difíciles de depurar.

## 3. Decisiones y alternativas descartadas

### 3.1 Orden de fases

**Decidido:** núcleo de recuperación → capa agéntica → ingesta multimodal. Estrictamente secuencial, sin solapamiento.

**Descartado:** empezar por lo agéntico (más vistoso) o por la ingesta multimodal (más aparatoso). La ingesta multimodal es ingeniería de datos, no RAG, y absorbe semanas peleando con parseo de PDF sin aportar aprendizaje de recuperación.

### 3.2 Momento de integrar el LLM

**Decidido:** el sistema final ejecuta el LLM **dentro** de Ragmur (endpoint `/answer`). La fase 1 se construye sin LLM como etapa intermedia deliberada.

**Motivo:** con generación activa desde el principio, un resultado incorrecto no permite distinguir si falló la recuperación o el modelo. Midiendo primero la recuperación aislada se obtiene una línea base sobre la que atribuir cualquier cambio posterior.

**Punto que causó confusión durante el diseño y conviene dejar fijado:** "sin LLM en fase 1" describe una etapa, no un principio del proyecto. Los endpoints sin generación (`/search`, `/query`) permanecen tras la fase 2 como interfaz de bajo nivel y como base de la suite de evaluación, no como modo de uso principal.

### 3.3 Verificación de citas sin LLM

La verificación de citas requiere una afirmación redactada que contrastar. Al no generar texto en fase 1, se resolvió exponiendo `POST /v1/verify`: el cliente envía un texto redactado más los `chunk_ids` citados, y Ragmur devuelve un veredicto por frase.

**Decidido:** verificador basado en un modelo NLI (mDeBERTa-v3-base-xnli), que clasifica pares (premisa, hipótesis) como entailment / neutral / contradiction.

**Descartado:** usar un LLM como verificador por defecto. Un LLM verificando a otro LLM introduce el mismo riesgo de alucinación que se pretende eliminar, además de coste y dependencia externa.

**Limitaciones asumidas y documentadas:**
- La segmentación en frases aproxima la descomposición en afirmaciones atómicas, que un LLM haría mejor.
- El contexto del modelo NLI es limitado (~512 tokens); los fragmentos largos se recorren por ventanas conservando la puntuación máxima.
- Los modelos XNLI se entrenaron con premisas de una sola frase. Un chunk de 800 caracteres como premisa es un régimen distinto al de entrenamiento, y la calibración de los umbrales no se transfiere: hay que calibrarlos sobre el set de prueba propio.
- Una frase que empieza con una referencia anafórica ("este plazo", "dicha cláusula") es inverificable aislada, y producirá falsos `unsupported`. Mitigación prevista: concatenar la frase anterior a la hipótesis.

La fase 2 añade `verifier=llm` como modo alternativo y **mide el grado de acuerdo entre ambos verificadores**. Esa comparación es un resultado del proyecto, no un detalle de implementación.

### 3.4 Multi-tenancy desde el primer día

**Decidido:** `tenant_id` propagado desde la API key hasta el filtro de Qdrant, implementado en la fase 1.

**Motivo:** el requisito real es servir a varios proyectos propios con corpus separados desde una única instancia. Añadir aislamiento a posteriori obliga a reindexar todo el corpus.

**Descartado:** una colección de Qdrant por tenant. Multiplica el consumo de memoria y complica el mantenimiento sin aportar aislamiento efectivo adicional frente a una colección única con índice de payload sobre `tenant_id`.

Ver también 3.8: el tenant por sí solo no cubre el aislamiento entre usuarios finales.

### 3.5 Sin frameworks RAG de alto nivel

**Decidido:** pipeline escrito a mano. Única excepción, `langchain-text-splitters` como utilidad aislada de troceado.

**Motivo:** el pipeline es corto, y escribirlo directamente mantiene cada paso explícito, depurable y explicable. Para el objetivo profesional del proyecto, "implementado sin framework" pesa más que "invocada una función de librería".

### 3.6 Fusión RRF

**Decidido:** Reciprocal Rank Fusion para combinar la rama densa y la léxica.

**Descartado:** normalizar y sumar scores. Los valores de similitud coseno y BM25 no son comparables, y su calibración es frágil y dependiente del corpus. RRF combina posiciones, no puntuaciones, y funciona sin ajuste.

### 3.7 Lenguaje y stack

**Decidido:** Python 3.12 con FastAPI.

**Motivo:** es el ecosistema de la IA, y FastAPI es el estándar actual para APIs nuevas de este tipo. Complementa la experiencia previa en PHP/Laravel en lugar de sustituirla; el perfil resultante (backend + IA + infraestructura propia) es menos común que el de especialista en un solo lenguaje.

**Contrapartida asumida:** Python concentra una gran oferta de perfiles junior. La diferenciación no viene del lenguaje sino de la especialización: capas de calidad medidas y arquitectura multi-tenant.

---

Las decisiones siguientes se incorporaron tras una revisión del diseño, antes de escribir código.

### 3.8 Segundo nivel de aislamiento: `owner_id`

**Problema detectado.** El requisito de que el usuario final suba sus propios documentos (por ejemplo, facturas) y los consulte "dentro de su propio espacio" no se cumple con `tenant_id` solo. El tenant se deriva de la API key, y la API key la tiene la **aplicación cliente**, no la persona. Si lazytripz tiene N usuarios con facturas privadas, los N comparten `tenant_id` y cualquiera recupera los documentos del resto.

**Descartado:** un tenant por usuario final. Obligaría a un endpoint de aprovisionamiento de tenants, a que la aplicación cliente gestionase N API keys y a custodiar una credencial por usuario. Desproporcionado.

**Decidido:** un campo opcional `owner_id` —texto opaco, definido por la aplicación cliente, típicamente su propio identificador de usuario— que subdivide el espacio del tenant.

- Se acepta en la ingesta y en la búsqueda.
- Si la consulta lo incluye, el filtro de Qdrant es `tenant_id AND owner_id`.
- Si no lo incluye, el filtro es solo `tenant_id`: el tenant ve todo su espacio, que es lo que necesita una aplicación con corpus compartido como el portfolio.

**Asunción explícita:** Ragmur confía en que la aplicación cliente envía el `owner_id` correcto, igual que confía en su API key. La autenticación del usuario final es responsabilidad de la aplicación cliente, no de Ragmur. Este límite es deliberado: llevar la identidad del usuario final dentro de Ragmur exigiría OAuth o JWT propios, y convierte el servicio en un proveedor de identidad, que no es lo que es.

**Motivo de resolverlo ahora y no después:** el mismo que en 3.4. `owner_id` va en el payload de Qdrant, y añadirlo a posteriori obliga a reindexar todo el corpus.

### 3.9 Localizador genérico en lugar de `page`

**Problema detectado.** El diseño original guardaba un número de página por chunk. No es representable fuera del PDF:

- `python-docx` no puede dar número de página: la paginación la calcula el renderizador al maquetar, no está en el XML del documento.
- TXT y Markdown no tienen páginas en absoluto.
- Con chunks de 800 caracteres, un chunk cruza frontera de página con frecuencia, así que un único número de página por chunk es incorrecto incluso en PDF.

**Decidido:** un localizador con tipo, que cada extractor rellena con lo que su formato puede sostener.

```json
{ "locator": { "type": "page",    "start": 4, "end": 5 } }
{ "locator": { "type": "section", "value": "3.2 Plazos" } }
{ "locator": { "type": "offset",  "start": 12040, "end": 12840 } }
```

El consumidor siempre recibe algo con lo que señalar la fuente; qué precisión tenga depende del formato de origen. `documents.page_count` pasa a nullable por el mismo motivo.

### 3.10 Deduplicación por hash de contenido

**Problema detectado.** Subir dos veces el mismo fichero duplica sus chunks en el índice. Los duplicados copan el `top_k`, degradan la recuperación real e **inflan artificialmente el recall del golden set**. Dado que el valor del proyecto son sus cifras, contaminar las cifras es un fallo de primer orden.

**Decidido:** `content_sha256` calculado sobre los bytes del fichero, con unicidad por `(tenant_id, owner_id, content_sha256)`. Una segunda subida idéntica devuelve `200` con el `document_id` existente, en lugar de `202` y un documento nuevo.

### 3.11 Claves de API en tabla propia

**Problema detectado.** El diseño original guardaba `api_key_hash` en la tabla `tenants`, con restricción de unicidad: una clave por tenant, para siempre. Sin rotación, sin claves distintas por entorno, y sin poder revocar una clave filtrada salvo cortando el servicio a esa aplicación.

**Decidido:** tabla `api_keys` separada, con el modelo habitual de token con prefijo público (equivalente a Laravel Sanctum). La clave tiene forma `rgm_<key_id>_<secret>`: se localiza el registro por `key_id`, que está indexado, y se compara el secreto en tiempo constante.

**Decidido también:** el hash es **HMAC-SHA256 con una clave de servidor, no bcrypt ni argon2.** Los algoritmos de hash de contraseñas están diseñados para ser lentos a propósito, porque defienden secretos de baja entropía elegidos por humanos. Una API key es un secreto aleatorio de alta entropía generado por el servidor: no hay ataque de diccionario que frenar, y el coste deliberado (~100 ms) se pagaría en **cada petición** al servicio.

### 3.12 Capa de acceso a PostgreSQL

**Decidido:** SQLAlchemy 2.0 en estilo declarativo tipado, con `asyncpg` como driver y Alembic para migraciones.

**Motivo:** SQLAlchemy 2.0 tiene soporte de tipos real, que es lo que permite que `mypy` cubra también la capa de datos. Alembic es el equivalente directo a las migraciones de Laravel. La decisión estaba implícita en el árbol de directorios (`db/models.py`, `db/migrations/`) y en los comandos documentados, pero sin fijar; se fija aquí.

**Descartado:** `asyncpg` en crudo con SQL a mano. Menos dependencias, pero pierde las migraciones versionadas, que son un requisito del proyecto.

## 4. Aclaraciones conceptuales fijadas

Puntos que se aclararon durante el diseño y que conviene no volver a confundir:

| Confusión | Aclaración |
|---|---|
| "Entrenar los embeddings" | Los modelos de embeddings **no se entrenan**. Se usa un modelo preentrenado (bge-m3) para convertir texto en vectores. |
| "La búsqueda vectorial busca palabras literales" | La búsqueda densa opera por **significado**; BM25 es la que opera por coincidencia literal. La búsqueda híbrida combina ambas porque cubren fallos distintos. |
| "La fase 1 no usa IA" | La fase 1 no usa **LLM**. Sí usa modelos: embeddings, cross-encoder de reranking y NLI de verificación. Ninguno requiere GPU; todos funcionan en CPU. |
| "La fase 3 hace falta para subir PDFs" | Ingerir PDF con capa de texto es extracción programática y pertenece a la fase 1. La fase 3 solo resuelve documentos **escaneados o fotografiados**, que carecen de texto extraíble. |
| "Cliente" en la documentación | Significa **aplicación cliente** (la que consume la API), no cliente de pago. |
| "La redacción la hace el cliente" | Cierto **solo en fase 1**. En el estado final, `/answer` redacta dentro de Ragmur. |
| "El tenant aísla a cada usuario" | El tenant aísla a cada **aplicación cliente**. El aislamiento entre usuarios finales de una misma aplicación lo da `owner_id` (ver 3.8). |
| "recall@10 mide todo el pipeline" | recall@10 mide la **recuperación**. El reranking no añade documentos, reordena los ya recuperados, y se mide con nDCG@10 y MRR (ver 6.5). |

## 5. Requisitos explícitos

- Desarrollo modular y estrictamente secuencial: no se abre una fase hasta cerrar la anterior por completo, evaluación incluida.
- Proveedor de LLM intercambiable **en caliente**, sin reinicio: modelos locales (Ollama sobre RTX 5080) y remotos (OpenAI, Anthropic, Gemini). Implementado con LiteLLM y configuración por tenant.
- Varios proyectos propios consumiendo la misma instancia con corpus completamente aislados.
- Ingesta de documentos por parte del usuario final (por ejemplo, facturas) consultables dentro de su propio espacio. Resuelto mediante `owner_id` (ver 3.8).

## 6. Criterios de calidad no negociables

1. Ninguna afirmación sobre la calidad de recuperación sin una cifra que la respalde, procedente de `eval/`.
2. Parámetros de troceado, `top_k` y umbrales ajustados con datos de evaluación, nunca por intuición.
3. Todo test de recuperación comprueba el aislamiento entre tenants, y entre `owner_id` dentro de un mismo tenant.
4. Las limitaciones conocidas se documentan en lugar de ocultarse; son parte del resultado del proyecto.
5. Cada métrica se aplica a la etapa que realmente mide: recall@k a la recuperación, nDCG@10 y MRR al reranking. Una cifra publicada sin indicar sobre qué etapa se calcula no vale.
6. Toda cifra de latencia se publica indicando el hardware sobre el que se midió.

## 7. Hardware de referencia

Las mediciones del proyecto se toman sobre el homelab propio: **RTX 5080**, que aloja además Ollama para la fase 2.

Los modelos de la fase 1 (bge-m3, bge-reranker-v2-m3, mDeBERTa-v3-base-xnli) funcionan también en CPU, y ese modo se mantiene soportado para que el proyecto sea reproducible sin GPU. Pero es el **modo degradado, no el objetivo**: el cross-encoder de reranking sobre 30 candidatos es el coste dominante de una consulta, y la diferencia entre CPU y GPU es de más de un orden de magnitud. Cualquier cifra de `timings_ms` que aparezca en documentación debe indicar sobre cuál de los dos se midió.

## 8. Trabajo posterior contemplado

Solo tras cerrar las tres fases: servidor MCP sobre Ragmur, ingesta incremental sin downtime, cache de embeddings de consultas frecuentes, panel de administración.

---

## Glosario

- **RAG** — Retrieval-Augmented Generation. Recuperar fragmentos relevantes de un corpus propio y aportarlos como contexto al generar una respuesta.
- **Embedding** — Representación numérica de un texto que permite comparar significados por proximidad vectorial.
- **BM25** — Algoritmo clásico de recuperación léxica, por coincidencia de términos. Sin modelo de por medio.
- **IDF** — *Inverse Document Frequency*. Componente de BM25 que da más peso a los términos raros del corpus que a los frecuentes. Sin él, BM25 puntúa solo por repetición y pierde precisamente lo que aporta frente a la búsqueda densa.
- **Búsqueda híbrida** — Combinación de recuperación densa (semántica) y léxica (BM25).
- **RRF** — Reciprocal Rank Fusion. Fusión de varios rankings por posición en lugar de por puntuación.
- **Cross-encoder** — Modelo que procesa consulta y fragmento conjuntamente para puntuar relevancia con mayor precisión que la comparación de vectores. Se aplica solo a un conjunto reducido de candidatos por su coste.
- **Reranking** — Reordenación de los candidatos recuperados mediante un cross-encoder.
- **NLI** — Natural Language Inference. Clasificación de un par (premisa, hipótesis) como entailment, neutral o contradiction. Base de la verificación de citas.
- **Chunk** — Fragmento en que se divide un documento antes de indexarlo.
- **Tenant** — Espacio aislado dentro del servicio, correspondiente a una aplicación cliente: sus documentos, su índice y su configuración.
- **`owner_id`** — Subdivisión opcional del espacio de un tenant, correspondiente a un usuario final de la aplicación cliente.
- **Localizador** — Referencia a la posición de un chunk dentro de su documento de origen: página, sección o desplazamiento, según lo que el formato permita.
- **recall@k** — Proporción de consultas cuyo fragmento correcto aparece entre los k primeros resultados. Métrica principal de la etapa de recuperación.
- **MRR** — Mean Reciprocal Rank. Media del inverso de la posición del primer resultado correcto. Sensible al orden, no solo a la presencia.
- **nDCG@k** — Normalized Discounted Cumulative Gain. Mide la calidad del orden de los k primeros resultados. Es la métrica adecuada para el reranking, que reordena sin añadir.
- **Golden set** — Conjunto versionado de consultas con su fuente esperada, usado para medir la recuperación de forma reproducible.

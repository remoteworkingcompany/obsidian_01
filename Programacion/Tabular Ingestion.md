# Tabular Ingestion — Excel y CSV en Knowledge BC

> **Estado:** v1 funcional end-to-end (2026-05-05). User sube xlsx/csv → se chunkea → se vectoriza → topics → RAG lo encuentra → chat cita con "hoja X · filas A-B". **Pendiente:** `/v1/preview` (UI), DuckDB sink y comando reingest (calidad), `/v1/regions` v2a (estructurales), v2b (LLM classifier), v3 (chat tool SQL). **Bounded Context:** `Knowledge` (no es BC nuevo — extiende el existente). **Servicio nuevo:** `services/tabular/` (Python/FastAPI + openpyxl + DuckDB). **Documento padre:** `[KNOWLEDGE_BC.md](./KNOWLEDGE_BC.md)`. Esta doc detalla el subsistema tabular; el resto del BC (RAG, lentes, retrieval) no cambia.

---

## 1. Visión y decisiones de alto nivel

Knowledge BC procesa hoy solo PDFs. Esta extensión añade soporte para **XLSX y CSV** como formatos de primera clase, manteniendo el mismo modelo de retrieval RAG y añadiendo un segundo modo de consulta (SQL-on-file) cuando aplica.

### Decisiones cerradas

|   |   |   |
|---|---|---|
|#|Decisión|Razonamiento corto|
|D1|Microservicio aparte (`tabular`), no extender `pdf-chunker`|Feature reutilizable más allá de Knowledge; responsabilidades distintas; deps distintas|
|D2|Sin BC nuevo — vive dentro de Knowledge|La invariante de negocio (corpus consultable per-owner) no cambia; solo cambia la extracción|
|D3|1 Excel = 1 KnowledgeItem (hojas como sub-locators)|Preserva el modelo de ownership por blob; cita en chat reconstruye sheet/range desde locator|
|D4|Chunking híbrido: narrative + row_blocks + parsed_values|Combina recall amplio (RAG) con drill-down (SQL) y agrupación|
|D5|DuckDB como sink secundario desde v1 (no v3)|Disocia persistencia de exposure; admin queries y debug desde día 1|
|D6|Postgres jsonb es SoT; DuckDB es vista materializada regenerable|Single source of truth; DuckDB rebuild es idempotente|
|D7|openpyxl read_only desde v1 (no calamine)|Migrar parser entre v1 y v2 sería autoinfligirse dolor; v2 necesita formato/estilo que calamine no expone|
|D8|v1 incluye detección heurística de header (no asume fila 1)|Sin esto, EFRAG/CSRD/ESG son inválidos en MVP — se pierde el caso real|
|D9|Vocabulario `kind` cerrado en v1/v2|Type-safe, todos los consumidores conocen el universo; abrir solo si hay evidencia|
|D10|`schema_fingerprint` strict + loose desde v1|Strict capta versiones idénticas; loose capta evoluciones del template|

---

## 2. Arquitectura

### 2.1 Microservicio `tabular`

Vive en `services/tabular/`, paralelo a `services/pdf-chunker/` y `services/embeddings/`. Misma forma hex que `pdf-chunker` (ports + adapters, FastAPI, RFC 7807 errores, healthcheck).

```
services/tabular/
├── Dockerfile                       # Python 3.12-slim
├── pyproject.toml
├── app/
│   ├── api/routes/
│   │   ├── inspect.py               # POST /v1/inspect
│   │   ├── extract.py               # POST /v1/extract
│   │   ├── regions.py               # POST /v1/regions    (v2a)
│   │   ├── preview.py               # POST /v1/preview
│   │   ├── profile.py               # POST /v1/profile    (v3)
│   │   ├── query.py                 # POST /v1/query      (v3)
│   │   └── sample.py                # POST /v1/sample     (v3)
│   ├── application/                 # use cases
│   ├── domain/
│   │   ├── workbook.py              # Workbook, Sheet, Region
│   │   └── table.py                 # Table, Column, dtype
│   ├── ports/
│   │   ├── tabular_reader.py        # interface
│   │   ├── region_detector.py       # interface (v2a)
│   │   └── sql_engine.py            # interface (v3)
│   └── infrastructure/
│       ├── readers/
│       │   ├── xlsx_openpyxl.py     # read_only + valores calculados
│       │   └── csv_pandas.py        # detección encoding/delimiter
│       └── sql/
│           └── duckdb_engine.py     # v1: write sink. v3: query.
└── tests/
```

### 2.2 Encaje en Knowledge BC

```
                   FILE UPLOADED
                         │
            ┌────────────▼────────────┐
            │  OnFileUploadedIngest   │   ← strategy por mime ya prevista
            │  (Knowledge subscriber) │
            └────────────┬────────────┘
                         │
                  ¿qué KnowledgeItemKind?
                         │
        ┌────────────────┼────────────────┐
        ▼                                  ▼
  IngestPdfPipeline                IngestTabularPipeline
  (PDF, actual)                    (Tabular, nuevo)
  • InspectPdf                     • InspectTabular
  • SemanticChunking               • RegionDetect (v2a) + Chunk
  • PersistChunks ◄────── ESTE ES COMPARTIDO ──────► PersistChunks
                         │
                         ▼
              EmbedKnowledgeChunksJob  (sin cambios)
                         │
                         ▼
       IdentifyKnowledgeItemTopicsJob  (sin cambios)
                         │
                         ▼
                KnowledgeIngested
                         │
                         ▼
             Lentes per-owner topics  (cross-format)
```

### 2.3 Cambios en el dominio Knowledge

**Nuevo VO:**

```php
// app/Domain/Knowledge/ValueObjects/KnowledgeItemKind.php
enum KnowledgeItemKind: string
{
    case Pdf     = 'pdf';
    case Tabular = 'tabular';   // xlsx, xls, csv, ods
}
```

**Migración (v1 inicial):**

```sql
ALTER TABLE knowledge_items
    ADD COLUMN kind varchar(16) NOT NULL DEFAULT 'pdf',
    ADD COLUMN schema_fingerprint_strict varchar(64) NULL,
    ADD COLUMN schema_fingerprint_loose  varchar(64) NULL;

ALTER TABLE knowledge_chunks
    ADD COLUMN locator_kind varchar(16) NOT NULL DEFAULT 'page';
    -- locator data + parsed_values van en la columna `metadata` jsonb existente
    -- bajo claves bien definidas: metadata.locator y metadata.parsed_values
```

**Nuevos puertos (Domain/Knowledge/Ports/):**

```
TabularInspectionPort        → POST tabular:/v1/inspect
TabularRegionDetectionPort   → POST tabular:/v1/regions   (v2a)
TabularExtractionPort        → POST tabular:/v1/extract
TabularQueryPort             → POST tabular:/v1/query     (v3)
```

Mismo patrón que los `Pdf*Port` actuales. Adapters en `Infrastructure/Knowledge/Adapters/HttpTabular*Client.php`.

### 2.4 Pipelines paralelos

`Application/Knowledge/Pipeline/` se reorganiza:

```
Pipeline/
├── Pdf/                              # actual, sin tocar
│   ├── IngestPdfPipelineExecutor.php
│   └── Steps/
│       ├── InspectPdfStepHandler.php
│       └── SemanticChunkingStepHandler.php
├── Tabular/                          # nuevo
│   ├── IngestTabularPipelineExecutor.php
│   └── Steps/
│       ├── InspectTabularStepHandler.php
│       ├── DetectRegionsStepHandler.php   (v2a)
│       └── TabularChunkingStepHandler.php
└── Shared/
    ├── PersistChunksStepHandler.php   # mime-agnostic
    ├── PipelineRunRepositoryPort.php
    ├── StepHandler.php
    └── StepStatus.php
```

`PersistChunksStepHandler` se mantiene compartido — recibe una lista plana de `KnowledgeChunk` y la escribe a BBDD sin importar de qué pipeline vienen. Es el punto de convergencia del polimorfismo.

---

## 3. Modelo de chunking híbrido

Cada hoja procesada produce, según el resultado de la detección de regiones:

### 3.1 Narrative chunk (1 por hoja, opcional)

Descripción generada por LLM ligero (`mlx_fast` o `claude-haiku-4-5`):

> _"Hoja 'Ventas Q3' con 1238 registros de transacciones por región y mes. Total facturado 3.4M€, mes pico octubre. 4 regiones (Norte, Sur, Este, Oeste). Columna 'Notas' contiene texto narrativo extenso (avg 287 chars)."_

**Skip heurístico** (no genera narrative chunk si):

- columns ≥ 3
    
- mean(header_chars) > 8 ∧ headers human-readable
    
- ninguna columna detectada como narrative
    
- las primeras 5 filas de datos tienen dtypes consistentes
    

Esto cubre el caso "Excel limpio de KPIs" donde el row_block ya habla por sí solo. Reduce coste LLM en el caso típico empresarial.

**Cache por hash:** narrative cacheado por `sha256(file_blob)`. Re-upload del mismo blob = 0 llamadas LLM.

### 3.2 Row-block chunks (N por tabla)

Bloques de 30-50 filas serializados como markdown con headers prefijados:

```
[Sheet: Ventas Q3 | Cols: Región, Mes, Importe, Cliente, Notas]
| Región | Mes        | Importe | Cliente   | Notas (resumen)         |
|--------|------------|---------|-----------|-------------------------|
| Norte  | Septiembre | 41200€  | Acme Corp | Cliente histórico, ...  |
| Norte  | Octubre    | 38500€  | Acme Corp | Renovación contrato ... |
...
```

Tamaño del bloque ajustable por config (`KNOWLEDGE_TABULAR_ROWS_PER_CHUNK`, default 40). Si una columna `narrative=true` tiene celdas con >300 chars, se chunkea con bloque más pequeño (10-15 filas) para que el chunk no se infle.

### 3.3 parsed_values en metadata

Cada row_block chunk lleva los valores parseados con dtypes en `metadata.parsed_values`:

```json
{
  "locator": {"kind": "row_block", "sheet": "Ventas Q3", "rows": [12, 51]},
  "parsed_values": {
    "columns": ["Región", "Mes", "Importe", "Cliente", "Notas"],
    "dtypes":  ["string", "string", "currency_eur", "string", "string"],
    "rows": [
      ["Norte", "Septiembre", 41200.0, "Acme Corp", "Cliente histórico..."],
      ["Norte", "Octubre", 38500.0, "Acme Corp", "Renovación..."]
    ]
  }
}
```

Lo consume el `QueryTabularKnowledgeTool` (v3) y los rebuilds de DuckDB.

---

## 4. Detección de regiones (v2a)

Hojas Excel rara vez son tabla pura. La detección clasifica el contenido en regiones por hoja antes de decidir qué chunker aplicar.

### 4.1 Niveles de detección

|   |   |   |
|---|---|---|
|Nivel|Trigger|Coste|
|**Heurística columnar**|Siempre. Por columna: longitud media, cardinalidad, dtype, narrative ratio|<50ms/hoja|
|**Estructural por rangos**|Siempre. Detecta múltiples tablas separadas por blank rows; lookup sheets|100-500ms/hoja|
|**LLM-as-classifier**|Solo si la heurística marca región como ambigua|1 llamada LLM/región ambigua. Cacheada por hash de la región|

### 4.2 Endpoint `/v1/regions` — contrato

```
POST /v1/regions
Content-Type: multipart/form-data
Body: file=<binary>, source_hash=<sha256>

Response 200:
{
  "schema_fingerprint_strict": "sha256:...",
  "schema_fingerprint_loose":  "sha256:...",
  "warnings": [
    {"code": "header_ambiguous", "sheet": "Ventas Q3", "detail": "..."}
  ],
  "sheets": [
    {
      "name": "Ventas Q3",
      "index": 0,
      "dimensions": {"rows": 1240, "cols": 8},
      "regions": [
        {
          "id": "ventas_q3#r1",
          "kind": "narrative_block",
          "range": "A1:H2",
          "char_count": 410,
          "preview": "Documento confidencial — distribución..."
        },
        {
          "id": "ventas_q3#r2",
          "kind": "table",
          "range": "A3:H1240",
          "header_row": 3,
          "row_count": 1238,
          "schema": {
            "columns": [
              {"name": "Región",  "letter": "A", "dtype": "string",
               "narrative": false, "cardinality": 4, "null_ratio": 0.0},
              {"name": "Importe", "letter": "C", "dtype": "currency_eur",
               "narrative": false, "min": 1240.5, "max": 412000.0},
              {"name": "Notas",   "letter": "G", "dtype": "string",
               "narrative": true, "avg_chars": 287, "p95_chars": 612}
            ]
          }
        }
      ]
    },
    {
      "name": "Definiciones",
      "index": 1,
      "regions": [
        {
          "id": "definiciones#r1",
          "kind": "lookup",
          "range": "A1:B45",
          "key_column": "A",
          "value_column": "B",
          "row_count": 44
        }
      ]
    }
  ]
}
```

### 4.3 Vocabulario `kind` (cerrado)

|   |   |   |
|---|---|---|
|`kind`|Descripción|Versión|
|`table`|Tabla estructurada con columnas tipadas (puede tener `narrative=true`)|v1|
|`narrative_block`|Texto libre sin estructura tabular|v2a|
|`lookup`|Diccionario clave→valor|v2a|
|`qa_pair_table`|Par pregunta/respuesta detectado|v2b|
|`pivot`|Pivot table|v3 (opcional)|
|`unclassified`|Fallback cuando la heurística falla — el cliente decide|v2a|

**Por qué cerrado, no abierto:** type-safety, todos los consumidores (cliente PHP, frontend, BD) saben el universo en compile-time. Añadir un kind requiere cambio de código coordinado, lo cual es deseable porque cada kind necesita un chunker que lo entienda. Si en algún momento aparece evidencia de necesitar extensibilidad por config (improbable en este dominio), se migra a registry pattern entonces. **Hasta v3 inclusive: cerrado**.

---

## 5. DuckDB

### 5.1 Rol como sink secundario

Desde v1, cada `IngestTabularPipelineExecutor` escribe a un fichero `.duckdb` per-owner además de a Postgres. Postgres es SoT (ownership, retrieval, RAG); DuckDB es vista materializada para análisis ad-hoc, debug y preparación de v3.

**Almacenamiento:**

- Storage: MinIO bucket `tabular-duckdb/` (mismo backend que Files)
    
- Path: `{owner_id}/store.duckdb`
    
- Tablas dentro: `t_{knowledge_item_id}_{sheet_slug}` con metadata columns `_chunk_id, _knowledge_item_id, _file_id, _sheet, _row_range`
    

**Generación:** lazy. El fichero se crea al ingerir el primer Excel del owner.

### 5.2 Concurrencia y locking

DuckDB admite **un único writer concurrente** y múltiples readers (MVCC). Con `.duckdb` per-owner, el caso problemático es: el mismo owner sube dos Excels en paralelo y ambos `IngestTabularPipelineExecutor` intentan escribir al mismo fichero a la vez.

**Decisión: serialización per-owner vía Laravel queue lock.**

Implementación: el job `IngestTabularPipelineJob` (o el Step que persiste a DuckDB) usa el middleware `WithoutOverlapping` con `owner_id` como clave:

```php
public function middleware(): array
{
    return [
        (new WithoutOverlapping("tabular-duckdb-{$this->ownerId}"))
            ->releaseAfter(60)
            ->expireAfter(180),
    ];
}
```

Esto serializa las escrituras DuckDB del mismo owner sin bloquear el resto del pipeline (chunking, embedding, topics): solo el step de write-to-duckdb queda detrás del lock. Si el lock no se libera en 60s, el job se reintenta.

**Plan de fallback (3 capas):**

1. **Si DuckDB write falla** (cualquier excepción): se loguea como warning, se
    
    marca el knowledge_item con flag `metadata.duckdb_dirty = true`, y la ingesta **continúa con éxito** (Postgres ya tiene los datos canónicos). El RAG sigue funcionando; solo se pierde la queryability SQL temporal.
    
2. **Job nightly de reconciliación** (`knowledge:tabular:reconcile-duckdb`)
    
    recorre items con `duckdb_dirty=true` y los re-vuelca desde Postgres. Si tiene éxito, limpia el flag.
    
3. **Comando manual de rebuild total** (`knowledge:tabular:rebuild-duckdb
    
    {owner_id}`): drop del` .duckdb` del owner y reconstrucción desde Postgres. Idempotente. Para casos de corrupción o cambio de schema.
    

Observabilidad: cada fallo de DuckDB write emite un span con `kind=db, attribute.error=...` en la traza del knowledge ingest, y un warning en `account_actions.payload.warnings`.

### 5.3 Reconciliación con Postgres

|   |   |   |
|---|---|---|
|Aspecto|Postgres|DuckDB|
|**Rol**|Source of truth operativa|Vista materializada regenerable|
|**Ownership / autorización**|Sí (queries con FK a `files.owner_id`)|No (no se valida; solo lectura para análisis)|
|**Retrieval RAG**|Sí (vector index, `RetrievableChunkReadPort`)|No|
|**Queries analíticas (SUM, JOIN, agregaciones)**|Posible vía jsonb operators pero feo|Sí (SQL completo nativo)|
|**Si hay drift**|Postgres gana|Se regenera desde Postgres|
|**Backup**|El de la BBDD|No requiere backup (regenerable)|

Los consumidores de DuckDB (admin dashboards, futuro `QueryTabularTool`) deben asumir que el dato puede ir 0-N segundos detrás de Postgres durante una ingesta activa. **Nunca usar DuckDB para validaciones críticas.**

---

## 6. Internacionalización

El dataset esperado mezcla idiomas (español, inglés, francés, portugués) en sheet names, headers, contenido narrativo y lookup values. Decisiones explícitas en cada capa:

### 6.1 Sheet names → table slugs (DuckDB)

Una hoja llamada `"Análisis Q3 — Reñido"` no puede ser nombre de tabla DuckDB literal. Slugificación:

1. Normalize Unicode NFKD
    
2. ASCII fold (acentos → letras base, ñ → n)
    
3. Reemplazar non-alphanumeric por `_`
    
4. Lowercase
    
5. Truncar a 32 chars
    
6. Si colisiona con otra hoja del mismo item, sufijo `_2`, `_3`, etc.
    

`"Análisis Q3 — Reñido"` → `analisis_q3_renido`. El sheet name **original** se preserva en `parsed_values.sheet_original` y en el locator.

### 6.2 Headers multilingües

Los headers se preservan **tal cual** en `parsed_values.columns` y en el texto serializado del row_block. **No se normalizan ni se traducen.** El embedding multilingüe (`bge-m3`) maneja correctamente queries cross-language (usuario pregunta en inglés, contenido en español, mismo embedding space).

Para `schema_fingerprint`:

- **Strict** usa los headers literales (incluye case, acentos, espacios) → un rename de `"Importe"` a `"Amount"` rompe el strict.
    
- **Loose** usa solo cardinalidad de columnas + dtype signature → resistente a renames. Ver §7.1.
    

### 6.3 Encoding CSV

Detección automática:

1. Intentar UTF-8 (95% del caso real).
    
2. Si falla: usar `chardet` para detectar (Latin-1, Windows-1252, etc.).
    
3. Confidence < 0.8 → fallback a Latin-1 con `errors='replace'` y warning.
    
4. El encoding detectado se persiste en `metadata.encoding` para auditoría.
    

**Delimiter** detection: `csv.Sniffer` con muestra de las primeras 8KB. Soporta `, ; \t |`. Si Sniffer falla (CSV sin headers consistentes), fallback a `,` con warning.

### 6.4 Keywords y topics

Los keywords/topics generados por el LLM en chunks tabular siguen las mismas reglas que los chunks PDF: el LLM responde en el idioma del contenido predominante del chunk. Una tabla mayoritariamente en español genera keywords en español. **No se traducen** — el embedding multilingüe neutraliza la diferencia en retrieval.

### 6.5 Lookup keys

Las claves de un `kind=lookup` se preservan literales (un código `"Q2"` o una sigla `"NACE-A"`). No se traducen. El enriquecimiento (v2a) que las expande en otros chunks usa el valor literal de la columna `value`, también sin tocar.

### 6.6 LLM prompts del servicio

Los prompts internos del microservicio (region classifier, narrative generator) se escriben **en inglés** (más estable para los modelos). Pero incluyen instrucción explícita: _"Respond in the predominant language of the input content."_ El output del LLM (label de cluster, narrative chunk) queda en el idioma del contenido.

---

## 7. Series y versionado

### 7.1 Schema fingerprint: strict + loose

Ambos se calculan en `/v1/inspect` y se persisten en `knowledge_items`.

**Strict** — sensible a cualquier cambio:

```
sha256(json.dumps({
    "sheets": [
        {"name": <sheet_name>, "headers": [<header_text>, ...]}
        for sheet in sheets
    ]
}, sort_keys=True))
```

**Loose** — resistente a renames y cambios cosméticos:

```
sha256(json.dumps({
    "sheets": [
        {
            "name_slug": slug(sheet_name),
            "col_count": len(columns),
            "dtype_sig": "|".join(c.dtype for c in columns),
            "narrative_count": sum(1 for c in columns if c.narrative)
        }
        for sheet in sheets
    ]
}, sort_keys=True))
```

**Política:**

- v1: **persistir ambos**, no actuar sobre ellos.
    
- v2a: actuar primero sobre `strict`. Si match perfecto en otro item del mismo owner → suggest "nueva versión de [item X]".
    
- v2a: si solo `loose` matchea → suggest "evolución del template [item X], ¿son la misma serie?".
    
- v2b+: `loose` matches con confidence más alta usando LLM para validar el emparejamiento (compara samples de las primeras filas).
    

Esto permite a v1 acumular datos sin compromiso. Cuando v2a llega, ya hay evidencia real de cuántos items de la base actual encajarían en series.

### 7.2 Mini-spec UX para series

> Diseño de UX para v2a. **No se implementa en v1** — solo se persisten los fingerprints para que cuando v2a llegue, la lógica tenga datos.

**Trigger del flujo:** usuario sube fichero tabular cuyo `schema_fingerprint` matchea (strict o loose) ≥1 item existente del mismo owner.

**Pantalla 1 — Modal post-subida (antes de procesar):**

```
┌──────────────────────────────────────────────────────────┐
│  Hemos detectado que este Excel se parece a uno          │
│  que ya tienes:                                          │
│                                                          │
│   📊 Informe ESG 2026 Q2  (subido el 12 abril)           │
│      Mismo template — 4 hojas, 28 columnas               │
│                                                          │
│  ¿Qué quieres hacer?                                     │
│                                                          │
│   ◉  Tratar como nueva versión de "Informe ESG 2026"     │
│      → Se agrupa en una serie. Por defecto el chat       │
│        usa la versión más reciente, pero puedes          │
│        consultar versiones anteriores.                   │
│                                                          │
│   ○  Tratar como documento separado                      │
│      → Se ingiere independiente. No se relaciona con     │
│        el otro Excel.                                    │
│                                                          │
│           [ Cancelar subida ]   [ Continuar ]            │
└──────────────────────────────────────────────────────────┘
```

**Default:** "documento separado" (conservador — el sistema no decide por el usuario; le ofrece la opción).

**Si match es solo loose** (template evolucionado):

```
   📊 Informe ESG 2026 Q2  (subido el 12 abril)
      Template parecido — 4 hojas, columnas similares
      pero con renames detectados:
        · "Importe" → "Amount"
        · "Trimestre" → "Quarter"
```

**Pantalla 2 — Vista de la serie en `/conocimiento`:**

Items que pertenecen a la misma serie se agrupan bajo una card expandible:

```
📁 Serie: Informe ESG 2026                           [3 versiones]
    ├─ 🟢 Q3 2026  (subido hoy)              ← versión actual
    ├─ ⚪ Q2 2026  (12 abril)
    └─ ⚪ Q1 2026  (15 enero)
```

Click expande/colapsa. Cada versión es accesible individualmente. Acción "Cambiar versión actual" para volver a una anterior si la última es errónea.

**Retrieval default:** versión `is_current = true` de cada serie. Configurable por proyecto (un proyecto puede pedir "todas las versiones") y por turno de chat (tool `searchKnowledge` acepta param `series_scope: 'current' | 'all'`).

**Naming de la serie:** derivado del `display_name` del primer item, con heurística para limpiar timestamps (`"Informe Q1 2026.xlsx"` → serie `"Informe ESG 2026"`). El usuario puede renombrarla.

**Modelo de datos (v2a):**

```sql
CREATE TABLE knowledge_item_series (
    id bigserial PRIMARY KEY,
    owner_id bigint NOT NULL,
    name varchar(255) NOT NULL,
    schema_fingerprint_strict varchar(64) NOT NULL,
    -- loose se almacena agregada vía la tabla pivot; no aquí
    created_at timestamptz, updated_at timestamptz
);

ALTER TABLE knowledge_items
    ADD COLUMN series_id bigint NULL REFERENCES knowledge_item_series(id),
    ADD COLUMN series_position int NULL,
    ADD COLUMN is_current_version boolean NOT NULL DEFAULT true;
```

Items sin serie llevan `series_id = NULL`. La inmensa mayoría de usuarios nunca verán este modelo (PDFs sueltos no entran en series).

---

## 8. Tools del chat

### 8.1 RAG actual (sin cambios)

`SearchProjectKnowledgeTool` sigue siendo el tool default. Funciona con chunks tabular y PDF indistintamente — el embedding es uniforme. Una pregunta narrativa sobre un Excel que tiene narrative chunks o columnas narrativas encuentra los chunks por similitud y los pasa al LLM como contexto.

### 8.2 `QueryTabularKnowledgeTool` (v3)

Tool nuevo para queries cuantitativas/agregativas:

```php
// app/Ai/Tools/QueryTabularKnowledgeTool.php
description: "Query tabular files (xlsx/csv) with SQL when the user asks
              for sums, averages, filters, comparisons, or specific row
              lookups. Use this instead of search for analytical questions."

params:
  - file_id: int  (required, validated against project scope)
  - sql: string   (required, SELECT only, validated)
  - limit: int    (optional, default 100, max 1000)

returns:
  - rows: array    (parsed values)
  - columns: array (with dtypes)
  - sql_executed: string (echo back para auditoría)
  - row_count: int
```

Flujo: validar ownership en Knowledge BC → resolver `knowledge_item_id` desde `file_id` → llamar a `tabular:/v1/query` con el `.duckdb` del owner + filtro a la tabla del item. El SQL se valida con un parser AST que solo permite SELECT (sin INSERT/UPDATE/DELETE/DROP).

**Routing del agente:** la decisión RAG vs SQL la hace el LLM por descripción de tool. Heurística natural:

- "¿qué dice X sobre Y?" / "explícame Z" → `searchKnowledge` (RAG)
    
- "suma", "media", "filas con", "compara", "máximo" → `queryTabular` (SQL)
    
- Casos mixtos: el agente puede llamar primero RAG para encontrar el fichero relevante, luego SQL sobre él.
    

---

## 9. Costes

Estimación grosso modo del coste LLM extra de la ingesta tabular vs PDF:

|   |   |   |   |
|---|---|---|---|
|Operación|Modelo|Calls/file|Tokens estimados|
|Region classifier (v2b, solo regiones ambiguas)|`mlx_fast` o Haiku|0-3|2k input / 100 output cada|
|Narrative chunk (skipped en Excel limpio)|Haiku o `mlx_fast`|0-N hojas|4k input / 500 output cada|
|Topic naming (cluster) — no es tabular-específico|Haiku|1-N clusters|igual que PDF actual|

**Mitigaciones ya decididas:**

- Cache narrativo por `sha256(file_blob)` — re-uploads gratis
    
- Skip heurístico de narrative para hojas muy estructuradas
    
- Modelos baratos por config (`KNOWLEDGE_TABULAR_NARRATIVE_MODEL`, `KNOWLEDGE_TABULAR_CLASSIFIER_MODEL`)
    
- Cache del classifier por `sha256(region_content)` — regiones idénticas entre versiones de la misma serie no re-clasifican
    

**Pendiente medir** con dataset real antes de v2 GA: latencia p95 de ingesta, coste por documento típico, ratio de cache hits.

---

## 10. Roadmap

|   |   |   |
|---|---|---|
|Versión|Alcance|Tiempo estimado|
|**v1 — tabular puro**|CSV + XLSX con header detection heurística. Endpoints `inspect`, `extract`, `preview`. Chunkers `Tabular` y `MixedColumns` con detección de columnas narrativas. Cache narrative por hash. Skip heurístico narrative. `schema_fingerprint` strict+loose persistido (sin acción). DuckDB sink secundario. openpyxl read_only.|4-6 semanas|
|**v2a — regiones estructurales**|Endpoint `/v1/regions`. Detección de múltiples tablas/hoja, blank rows, lookup sheets + enrichment cross-sheet. `KnowledgeItemSeries` y UX de series (mini-spec §7.2). Chunkers `NarrativeBlock` y `Lookup`. Soporte hojas mixtas. Sin LLM aún.|3-4 semanas|
|**v2b — LLM classifier (implementación)**|Region classifier para casos ambiguos. Chunkers `QAPairTable` y `DocumentSheet`. Cache de clasificación por hash de región.|3-4 semanas|
|**v2b — tuning continuo**|**No es un milestone, es un proceso permanente.** Revisión periódica de prompts, expansión de few-shot examples, A/B de versiones, benchmark contra corpus de test. Tracked via test suite y dashboard de métricas.|Continuo, ~1 semana/trimestre|
|**v3 — query y chat tabular**|DuckDB query exposure: endpoints `query`, `profile`, `sample`. `QueryTabularKnowledgeTool` en chat. Routing RAG vs SQL en agente. Series-aware retrieval (latest version default, override por param).|3-4 semanas|

**Total core (sin tuning continuo): 13-18 semanas.** v1 entrega valor real; v2a desbloquea documentos corporativos con rarezas; v2b cubre los retorcidos; v3 abre el chat tabular real.

### Distinción importante: v2b implementación vs tuning

**Implementación de v2b** = código que despacha región ambigua → LLM → clasificación → chunker correspondiente. Eso se hace, se mide, se shippea. Es un milestone con criterio de cierre claro (suite de tests pasa con N fixtures de regiones ambiguas conocidas).

**Tuning de v2b** = mejorar la calidad de la clasificación con datos reales. Es **iterativo** y no termina nunca:

- Cada trimestre: revisión de regiones que cayeron en `unclassified`
    
- Cada nuevo dominio de cliente: ampliación de few-shot examples
    
- Cada cambio de modelo (Haiku → nuevo): re-benchmark de la suite
    
- Métrica de tracking: % de regiones clasificadas correctamente vs ground truth (corpus de test mantenido manualmente)
    

Esto se trata como **deuda viva**, no como deuda técnica acumulable. Se asigna 1 sprint/trimestre a tuning. Si se descuida, la calidad degrada silenciosamente.

---

## 11. Open questions — resueltas (2026-05-05)

Las cinco open questions del diseño se cierran con los siguientes defaults para no bloquear v1. Se pueden revisar cuando haya datos de uso real.

1. **Bucket DuckDB.** Decisión: **reusar `MINIO_BUCKET=locaia`** con prefix
    
    `tabular-duckdb/`. Path completo: `{owner_id}/store.duckdb`. Razón: evita gestión de un segundo bucket en prod, las políticas IAM ya están resueltas para `locaia`. Si en prod aparece la necesidad de IAM diferenciada (e.g. backup distinto, ACLs específicas), se separa entonces — la migración es un `mc mirror` puntual.
    
2. **Tamaño máximo de Excel.** Decisión: **100MB de fichero / 100k filas
    
    por hoja** como límites duros en v1. Excels arriba de eso se rechazan con `KnowledgeItemTooLargeException` y reason claro: _"Fichero excede 100MB / 100k filas. Para volúmenes mayores contactar soporte."_. Sin batch processing en v1; queda como mejora si la base de usuarios lo pide.
    
3. **Cuándo se calcula `schema_fingerprint`.** Decisión: **en `/v1/inspect`**.
    
    Razones:
    
    - Es barato (solo lee headers, no datos)
        
    - Permite que Knowledge BC tome decisiones tempranas (ej. detectar dedupe semántico antes de procesar)
        
    - El servicio ya tiene el workbook abierto en ese momento `/v1/regions` (v2a) no recalcula — usa el fingerprint persistido.
        
4. **Encrypted XLSX.** Decisión: **rechazo en v1** con
    
    `KnowledgeItemProtectedException`. Reason: _"Fichero protegido con contraseña. Quita la protección y vuelve a subirlo."_. v2 evalúa añadir soporte vía `msoffcrypto-tool` con UI de prompt de contraseña per-upload — solo si hay demanda.
    
5. **Idempotencia de re-ingesta.** Decisión: comando Artisan
    
    `php artisan knowledge:tabular:reingest {knowledge_item_id}` que:
    
    - Marca el item como `pending`, purga sus chunks y topics
        
    - Re-encola `IngestTabularKnowledgeItemJob`
        
    - Se incluye en v1 (~30 líneas). El comando equivalente para PDF (`knowledge:reingest`) ya existe — mismo patrón.
        
    
    Esto cubre el caso "cambió el chunker" o "el LLM mejoró". Si un usuario re-sube el mismo blob, la dedupe actual de Knowledge BC sigue cortando el procesamiento — comportamiento intencional.
    

---

## 12. Decisiones explícitas — checklist

- D1: Microservicio aparte (`tabular`)
    
- D2: No es BC nuevo — vive en Knowledge
    
- D3: 1 Excel = 1 KnowledgeItem (hojas como locators)
    
- D4: Chunking híbrido (narrative + row_blocks + parsed_values)
    
- D5: DuckDB sink desde v1
    
- D6: Postgres = SoT, DuckDB = vista materializada
    
- D7: openpyxl read_only desde v1, no calamine
    
- D8: v1 con header detection heurística (no asume fila 1)
    
- D9: Vocabulario `kind` cerrado en v1/v2/v3 (revisar en v4 si surge evidencia)
    
- D10: `schema_fingerprint` strict + loose desde v1
    
- D11: DuckDB locking via `WithoutOverlapping` per-owner
    
- D12: 3 capas de fallback DuckDB (warning + nightly reconcile + manual rebuild)
    
- D13: i18n — slugs ASCII para tablas DuckDB, headers literales en parsed_values
    
- D14: Skip heurístico narrative para Excels limpios
    
- D15: Cache narrative + classifier por hash
    
- D16: v2b implementación y tuning son cosas distintas
    
- D17: Open questions §11 cerradas con defaults (2026-05-05)
    

---

## 13. Histórico

|                     |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Fecha               | Cambio                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| 2026-05-05          | Diseño inicial cerrado. Iteración con feedback estructurado en chat — pivot de "extender pdf-chunker" a "microservicio aparte", DuckDB adelantado a v1, header detection en v1, fingerprint loose añadido, v2 partido en v2a/v2b, mini-spec UX series, decisiones explícitas i18n y locking DuckDB.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| 2026-05-05 (audit)  | **Observability + LlmAudit completo.** El LLM call del `TabularNarrativeAgent` se envuelve ahora en `TracedLlmCall::run()` (mismo patrón que `SemanticChunkingStepHandler` en PDF), produciendo:  <br>· Span hijo `llm.tabular_narrative.sheet_NN` (kind=llm_call) con tokens/bytes/duración dentro del span padre `pipeline.chunking`  <br>· `LlmCallRecord` en `llm_call_records` con `purpose=classification`, input prompt completo, output texto + tokens — auditoría de calidad como cualquier otro LLM call del sistema  <br>· Evento `LlmCallCompleted` por bus que `LlmAuditSubscriber` persiste (Capa 2 del audit)  <br>· Resto del audit (`AuditTrailSubscriber`, `BroadcastSubscriber`, `ActivityLogSubscriber`, etc.) ya cubrían los eventos `KnowledgeIngestionStarted`, `KnowledgeItemChunked`, `KnowledgeItemRejected` por escuchar `IsDomainEvent::class` universalmente — sin trabajo extra  <br>· Helpers `responseToOutput()` y `providerFromModel()` portados del step PDF                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| 2026-05-05 (hotfix) | **Fix `varchar(64)` → `varchar(80)` en fingerprints.** Migración `2026_05_05_000005_widen_schema_fingerprint_columns.php`. La migración original ponía `varchar(64)` pero el formato emitido por el servicio (`sha256:` + 64 hex = 71 chars) no cabía. Primer upload real (Excel EFRAG IG3) lo destapó. Migración original también corregida para installs frescos. Mantenemos el prefijo `sha256:` por self-describing — habilita otros algoritmos (blake3) sin romper el contrato.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| 2026-05-05 (e2e)    | **v1 funcional end-to-end.** Bloques 1-3 completos:  <br>· `docker-compose.dev.yml` con servicio `tabular` (puerto 8005, healthcheck, bind-mount + uvicorn --reload)  <br>· `/v1/extract` operativo: parser tipado (`_typed_values.py`) con coerción cross-locale (formato europeo/anglo de números, fechas ISO, booleans bilingües)  <br>· Migración: `knowledge_chunks.locator_kind` y `metadata` jsonb (ya existía)  <br>· `ChunkPersistencePort` extendido para aceptar `locator_kind` + `metadata`; PDF mantiene `page_start/end` por compat  <br>· `TabularNarrativeAgent` (LLM) con prompt instructivo de idioma del contenido  <br>· `TabularChunkingStepHandler`: row_blocks de 30-50 filas con headers prefijados como markdown table; narrative chunk per-hoja con skip heurístico (≥3 cols + headers descriptivos + sin columnas narrative); narrativa con bloque más pequeño (12 filas) cuando hay columnas narrativas  <br>· `PersistTabularChunksStepHandler`: persiste chunks, mueve item a `chunked`, emite `KnowledgeItemChunked` (lo recoge subscriber existente que encola embedding)  <br>· `IngestTabularPipelineExecutor` con los 3 steps cableados  <br>· Bindings DI en `AppServiceProvider` para `TabularChunkingStepHandler` (3 params: narrative_model, rows_per_chunk, rows_per_chunk_narrative_column)  <br>· `config/knowledge.php` con bloque `tabular_chunking`  <br>· `KnowledgeRetrievalAdapter` lee `locator_kind` de SQL e incluye `metadata.locator` en resultados; `buildDisplayContext` polimórfico  <br>· `SendRagMessageHandler::retrievalResultToChunkArray` propaga `locator_kind` + `locator`  <br>· `GetRagReferencesHandler::mapChunks` los expone al frontend  <br>· Composable `useChunkLocator.js` con `locatorLabel()` (PDF: "p. 12-15"; row_block: "'Ventas Q3' · filas 4-43"; sheet_summary: "'Ventas Q3' · resumen")  <br>· `RagReferencesTable` y `RagReferenceModal`: header de columna dinámico ("Páginas" si todo PDF, "Ubicación" si mezclado), modal con visor PDF solo cuando aplica (tabular muestra solo el chunk en monospace + descarga del fichero) |
| 2026-05-05 (cierre) | Foundation de v1 aterrizada en código:  <br>· Migración `2026_05_05_000004_add_tabular_ingestion_columns.php` (kind, locator_kind, schema_fingerprint_strict/loose)  <br>· `KnowledgeItemKind` enum + `KnowledgeItem` entity y modelo Eloquent extendidos  <br>· Ports `TabularInspectionPort` + `TabularExtractionPort` + excepciones tipadas (`KnowledgeItemTooLargeException`, `KnowledgeItemUnsupportedFormatException`)  <br>· `KnowledgeItemRepositoryPort::updateSchemaFingerprints()`  <br>· `services/tabular/` (Python/FastAPI) con dominio, dispatcher de readers, error mapping RFC 7807  <br>· `/v1/inspect` operativo con header detection heurística (CSV via stdlib + chardet; XLSX via openpyxl read_only) y cálculo de fingerprints strict+loose  <br>· `/v1/extract` y `/v1/preview` con placeholder 501 (próxima sesión)  <br>· Adapters `HttpTabularInspectionClient` + `HttpTabularExtractionClient` + `AbstractTabularServiceClient`  <br>· `IngestTabularPipelineExecutor` + `InspectTabularStepHandler` + Command/Handler/Job paralelos al pipeline PDF  <br>· `OnFileUploadedIngest` despacha por `KnowledgeItemKind` (PDF/Tabular)  <br>· `config/knowledge.php` con `tabular_service` y `allowed_mime_types` extendido  <br>· `.env.example` con `KNOWLEDGE_TABULAR_*`  <br>· `AppServiceProvider` con `bindTabularServiceClient`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
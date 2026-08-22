# Module: normalizer

**Path:** `src/api/v2/modules/normalizer/`
**Files:** `router.js`, `service.js`, `worker.js`, `llm.js`, `file-parser.js`, plus `templates/` subdirectory (preset schema templates, e.g. `farmshop.js`)
**Agents:** `agents/classifier.js`, `agents/extractor.js`, `agents/resolver.js`, `agents/architect.js`, `agents/populator.js`, `agents/doc-writer.js`, `agents/dash-builder.js`, `agents/swarm-writer.js`, `agents/extractors/` (including `agents/extractors/natasha-ner.js`)
**Base URL:** `/api/v2/:db/normalize/...`
**Auth:** JWT required. All endpoints: `editor`.

## Purpose

AI-powered folder normalization pipeline. Processes documents from a workspace file folder, classifies them, extracts structured data, and populates EAV tables. Supports `auto` mode (fully automated) and `assisted` mode (HITL at each stage). BullMQ-based; workers run in a **separate process** (`backend/scripts/start-worker.js`) independent of the HTTP server. Falls back to in-process execution if Redis is unavailable.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/normalize` | List all normalization jobs for workspace |
| POST | `/normalize/start` | Start normalization job. Body: `{ folderId, mode: "auto"\|"assisted", hints? }` |
| GET | `/normalize/:jobId` | Get job status and current stage |
| GET | `/normalize/:jobId/schema` | Get synthesized schema payload (available after architecting stage) |
| GET | `/normalize/:jobId/resolved` | Get resolved entity clusters (available after resolving stage) |
| POST | `/normalize/:jobId/confirm-schema` | HITL: confirm proposed schema, advance to populating stage |
| POST | `/normalize/:jobId/confirm-resolution` | HITL: confirm entity cluster review (with optional overrides) |
| DELETE | `/normalize/:jobId` | Cancel/delete a job |
| POST | `/normalize/setup-object-layer` | Idempotent Product + ProductAlias type bootstrap (admin only). Calls `ensureObjectLayerTypes()`, returns `{ ok, data }` with created/existing type IDs and column mappings. Safe to call repeatedly. |

`hints` — optional free-text string with user-provided extraction hints (e.g. "company name is Ромашка, prices are in RUB"). Persisted to job data and forwarded to the extractor and populator agents.

## Pipeline Stages

```
Upload → [coordinator] → [classifier] → [extractor (two-pass)] → HITL UI → [resolver] → [populate] → Done
                                                              ↑
                                                        [architect] (if new schema needed)
                                                                                          ↓
                                                                                     [docwrite]
                                                                                     [dashbuild]
```

### Queues and Workers

All workers run inside `backend/scripts/start-worker.js` — a separate process started by `dev-start.sh`. This process has **no `--watch`** flag, so it survives HTTP server restarts.

| Queue | Concurrency | lockDuration | Role |
|-------|-------------|-------------|------|
| `normalizer` | 3 | 60s | Coordinator — dispatches to sub-queues |
| `normalizer-classify` | 10 | 120s | LLM: identify document type, target table |
| `normalizer-extract` | 5 | 900s | LLM: extract field values from document |
| `normalizer-resolve` | 2 | 600s | LLM: resolve ambiguities, conflicting values |
| `normalizer-architect` | 2 | 600s | LLM: propose schema if no target type exists |
| `normalizer-populate` | 2 | 600s | Write extracted data to EAV |
| `normalizer-docwrite` | 3 | 600s | LLM: generate structured documents |
| `normalizer-dashbuild` | 2 | 600s | LLM: build dashboards from extracted data |

### Stage Details

**Classify**: LLM determines what type of document it is and which EAV type to target (or suggests creating a new one).

**Extract (two-pass)**: Two separate LLM calls — first extracts structured entities (field values for EAV), second extracts document content blocks (headings, paragraphs, lists for docwrite). This avoids context-window bloat and lets each pass focus on its task.

**Schema-aware extraction**: Extractor receives the full column schema of the target type, enabling direct field mapping without guessing column names. Includes upsert deduplication — existing records are matched by key fields before creating new ones.

**HITL Review**: Frontend shows extracted fields to user. User corrects mistakes, confirms, and triggers populate.

**Architect** (optional): If no matching EAV type exists, LLM proposes a schema; user confirms; schema is created. In **append mode**, the architect fetches all existing workspace tables and includes them in the LLM prompt. The LLM can return `existingTypeId` on a table entry to reuse an existing table instead of creating a new one. The populator skips `createType` for tables with `existingTypeId`.

**Resolver** (`agents/resolver.js`): Workspace-aware entity resolution — matches extracted values against existing EAV records using exact string match first, then LLM fuzzy matching. Prevents duplicate records for the same real-world entity (e.g. same client under slightly different names).

**Populate**: Creates/updates EAV objects with the confirmed field values. Reads `hints` from job data for additional context.

**Docwrite** (`agents/doc-writer.js`): Universal docwrite — LLM suggests appropriate folder path and document tags based on content. Dynamically creates folders and tags if they don't exist. Runs after populate.

## File Parser (`file-parser.js`)

Supports: PDF (pdf-parse), DOCX (mammoth), Excel/XLSX (xlsx library), CSV (papaparse). Has file size limit and encoding detection. Returns plain text for LLM processing.

A PDF with no text layer is a scan and goes through the shared OCR chain
(`files/ocr-chain.js`, `format: 'pdf-scanned'`) — the same engines in the same order as
`doc-processor`. This file used to keep its own vision → mistral pair, so the same scan
read through the normaliser and through file processing came back different.

`extractFileContent(filePath, { workspaceId })` passes the workspace down into the chain, and
it matters since `vision-pdf` joined the PDF order: that engine spends LLM tokens, and without
a workspace the spend lands in `llm_calls` with `workspace_id = null`, outside both the quota
and the closed-contour check. Callers already have it — `classifyBatch` takes `db`, and the
extractor dispatcher (`agents/extractor.js`) passes `db` as the fourth argument to every
extractor, which is why `manual.js`, `specification.js` and `bom.js` only had to declare the
parameter they were already being handed.

## Extractors (`agents/extractors/`)

Specialised extractors for common document types: `generic.js`, `invoice.js`, `bom.js`, `specification.js`, `manual.js`. Generic extractor handles all other types.

## Natasha NER (`agents/extractors/natasha-ner.js`)

Deterministic Russian named-entity recognition via an external microservice (configured via `NATASHA_NER_URL` env var). Runs in parallel with LLM extraction inside the generic extractor (`agents/extractors/generic.js`). Results are merged with the LLM output using the `mergeNerWithLlm` strategy, which lets the high-precision rule-based NER results complement and correct the LLM-extracted entities.

## LLM Layer (`llm.js`)

Wrapper around the configured LLM provider. Structured output with Zod schema validation. Retries on malformed JSON. Reads workspace-level model override and logs cost per workspace.

## Job State Machine

```
listing → classifying → extracting → resolving → architecting → populating → writing → completed
                                                                                    ↓
                                                                              error / cancelled
```

State stored in `public.agent_memory` with key pattern `job:<id>:<suffix>` (e.g. `job:<id>:status`, `job:<id>:meta`). Uses direct SQL UPSERT (bypassing swarm-memory consolidation to avoid cross-job data corruption).

## Object Layer (ADR-016)

Three files under `src/api/v2/modules/normalizer/` implement a product identity layer on top of EAV.

### `object-layer-setup.js` — idempotent type bootstrap

Primary export: `ensureCanonicalTypes(pool, db, config = {})`. Alias: `ensureObjectLayerTypes`.

**Config options** (all optional, sensible defaults):
- `canonicalName` (default: `'Product'`) — canonical type name
- `canonicalColumns` — column definitions for canonical type
- `aliasName` (default: `'ProductAlias'`) — alias type name
- `aliasColumns` — column definitions for alias type
- `icons` — `{ canonical, alias }` icons

**Default Product columns:** `gtin` (unique), `article`, `sku`, `unit`, `description`

**Default ProductAlias columns:** `role` (free-text SHORT column, e.g. `bookkeeping`, `shop`, `sales`), `source_system`, `source_pk`, `source_code`

Safe to call on every startup — skips columns that already exist.

### `object-layer-upsert.js` — deterministic upsert

`upsertProductBatch(pool, db, records, { productTypeId, aliasTypeId, colAliasToId, keyFields })` resolves duplicates by strong keys in priority order: GTIN → article → SKU. The options object requires type IDs and key field mappings from `ensureCanonicalTypes()`. Internally uses `findCanonicalByKey(pool, db, productTypeId, colAliasToId, record, keyFields)` for each candidate key before inserting.

Exported functions: `findCanonicalByKey()` (alias: `findProductByKey()`), `upsertProductBatch()`

### `object-layer.js` — AI tools (not yet wired to an agent)

| Tool | Description |
|------|-------------|
| `resolve_product_aliases` | Role-based alias resolver — returns aliases filtered by role (`bookkeeping`, `shop`, `sales`) |
| `resolve_product_identity` | Probabilistic entity resolution — scores candidate products against input attributes |
| `apply_confirmed_identity` | Applies a confirmed merge decision, updating ProductAlias links |
| `get_product_movement` | Cross-source movement tracking — aggregates stock/sales events by product identity |

---

## DB Tables (per-workspace, lazy-init)

- Job state stored in `public.agent_memory` — keyed by `job:<id>:<suffix>` (agent_id = `normalizer`). Suffixes: `status`, `meta`, `entities`, `classified`, `resolved`, etc.

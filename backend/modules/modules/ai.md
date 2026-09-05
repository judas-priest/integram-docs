# Module: ai

**Path:** `src/api/v2/modules/ai/`
**Files:** `router.js`, `service.js`, `starters.js`, `autofill-worker.js`, `ai-column-worker.js`, `ai-column-listeners.js`, `llm-router.js`, `dlp-service.js`, `contour-guard.js`, plus `agent/` and `background-scans/` subdirectories
**Base URL:** `/api/v2/:db/ai/...`
**Auth:** JWT required for all endpoints.

## Purpose

The AI hub. Provides the multi-agent chat interface (SSE streaming), LLM completion, AI autofill, AI computed columns, formula generation, NL→data query, HITL confirmation, MCP tool execution, and model/usage management.

## Endpoints

### Agent chat

| Method | Path | Description |
|--------|------|-------------|
| POST | `/ai/agent-chat` | Multi-agent SSE stream (main AI chat). Body: `{ message, conversationId, model, attachment, pageContext, agentSlug?, topicId?, searchMode?, loopMode?, fileIds? }`. `searchMode` (optional): `'precise'` / `'balanced'` / `'full'` — controls Meta KB search precision. `fileIds` (optional): array of file IDs previously uploaded via `POST /ai/upload` — attached as context for the message. When `agentSlug` is set (e.g. `'teamchat-agent'`): bypasses orchestrator, calls the named agent directly, saves Q&A to teamchat room `meta-kb`, returns `topicId` in the `done` SSE event. **Domain role guard:** when both `agentSlug` and `topicId` are provided, the system resolves the topic's room, checks the user's `domain_role` membership, and verifies the agent slug is in that role's `agent_slugs` list. If the user's role does not permit the requested agent, an error is returned and the request is terminated. `null` domain role (no role assigned) allows all agents. |
| POST | `/ai/agent-resume` | Resume a HITL-pending operation after user confirms |
| POST | `/ai/agent-elicit` | Resume after elicitation form fill |
| POST | `/ai/mcp-resume` | HITL resume for MCP (no CSRF) |

### Generated file download

| Method | Path | Description |
|--------|------|-------------|
| GET | `/:db/ai-files/f/<exp>-<code>-<sig>` | Download a file produced by `create_file` / `create_excel`. Short form: digits, latin letters and hyphens only. |
| GET | `/:db/ai-files/<path>?t=<exp>.<sig>` | Same file by its relative path. Kept working for links already handed out. |

Mounted in `api/v2/index.js` **before** `requireJwt` on purpose. The agent prints the link and the
user *clicks* it; a browser navigation carries no `Authorization` header, and the workspace JWT
lives in the tab, not in a cookie — so a route behind `requireJwt` answers 401 to everyone, always
(observed on production 12.08.2026: six clicks, six 401s). The right to download is therefore
carried by the link itself: an HMAC over workspace + file + expiry (30 days), signed with
`JWT_SECRET`. The trade-off is stated plainly — whoever holds the link gets the file until it
expires, the same model as a presigned URL.

The signature is **hex**: base64url contains `_` and `-`, and Telegram's Markdown eats the
underscores, so a link broke in transit rather than on the server. The short form carries no file
name at all — it is found by a fingerprint of its path — because percent-encoded Cyrillic and
spaces are what messengers truncate.

Files are written per caller surface: `ai-chat/<clientId>/` for portal callers (they have the
`portal_jwt` cookie and get the portal download URL), `ai-chat/w<userId>/` for workspace callers.
Portal client IDs and workspace user IDs are numbered independently, so without the `w` prefix
client 31 and employee 31 shared one directory.

### MCP tool interface

| Method | Path | Description |
|--------|------|-------------|
| GET | `/ai/tools` | List all available tools (for MCP server) |
| POST | `/ai/tool` | Execute a single tool by name (`{ name, args, skipHitl, callId }`) |

### LLM utilities

| Method | Path | Description |
|--------|------|-------------|
| POST | `/ai/chat` | Raw LLM chat completion (not agentic) |
| POST | `/ai/fill-table` | AI fill for all rows of a table column |
| POST | `/ai/column-agent` | Per-column AI agent prompt |
| POST | `/ai/autofill-batch` | Batch AI autofill (sync or via BullMQ) |
| POST | `/ai/run-button` | Execute AI button for a specific row |
| POST | `/ai/generate-formula` | Generate formula from natural language description |
| POST | `/ai/suggest` | Suggest values for a field based on context |
| POST | `/ai/query` | NL → data query (returns structured EAV results) |
| POST | `/ai/generate-delta` | Generate Quill Delta document from prompt |
| POST | `/ai/upload` | Upload a file for AI chat context. Body: `multipart/form-data` with `file` field. Returns `{ fileId, filename, mimeType, size }`. Used with `agent-chat` body param `fileIds` (array of uploaded file IDs). |

### AI column config

| Method | Path | Description |
|--------|------|-------------|
| GET | `/ai/column-config/:typeId/:reqId` | Get AI button/column config |
| PUT | `/ai/column-config/:typeId/:reqId` | Set AI button/column config |

### Models and usage

| Method | Path | Description |
|--------|------|-------------|
| GET | `/ai/models` | List available LLM models |
| GET | `/ai/usage` | LLM token usage statistics |
| GET | `/ai/audit-log` | Tool call audit log |
| GET | `/ai/quota` | LLM quota: remaining tokens/requests for current billing period |

### Dynamic chat starters

| Method | Path | Description |
|--------|------|-------------|
| GET | `/ai/starters` | Dynamic suggestion buttons for chat empty state. Returns `{ ok, data: [{ label, prompt, category, partial? }] }` |

Generates 4-6 contextual starters based on workspace profile (tables, documents, connectors, portal, automations, reports), user role, swarm memory behavioral patterns, and audit log tool popularity. No LLM — template engine with ranking. Response cached 60s per workspace.

### Conversations

| Method | Path | Description |
|--------|------|-------------|
| GET | `/ai/conversations` | List user's conversations |
| POST | `/ai/conversations` | Create conversation |
| DELETE | `/ai/conversations/:id` | Archive conversation |
| GET | `/ai/conversations/:id/messages` | Get message history |
| POST | `/ai/feedback` | Submit thumbs up/down feedback on a response |

## Agent Architecture (`agent/`)

```
POST /ai/agent-chat
        ↓
   Orchestrator (agent/orchestrator.js)
   - Semantic routing (utterance matching → target agent)
   - Injects shared tools (memory, graph, KAG)
   - council.js: cross-domain / parallel query handling
   - semantic-router.js: utterance-based agent routing
        ↓
   Specialized Agent Runner (agent/runner.js)
   - Streams LLM SSE (multi-provider)
   - Calls tool-executor.js on tool_use blocks
   - Returns real toolCalls array (name + input + result) for each tool invoked
   - Mid-turn context compression: prune → merge → LLM-summarize (swarm-extract) при превышении 20000 токенов
        ↓
   tool-executor.js (626 switch cases)
   - Middleware pipeline: lessons → portalGrants → HITL → permission → audit
   - Calls service layer (objects, schema, reports, ...)
   - Returns structured JSON responses
        ↓
   Orchestrator post-delegation check
   - Checks agent result for contradictions with existing memory and other agent results
   - May call shouldConveneCouncil() → triggers 3-round council deliberation if contradictions detected
   - Council deliberation is not limited to PR review (review-gate); it applies to any cross-agent conflict
```

### Middleware and Subsystems

| File | Purpose |
|------|---------|
| `lessons-middleware.js` | Pre-execution guardrails from swarm-memory lessons. Injects constraints before tool execution. |
| `object-grounding.js` | Context enrichment: resolves object references to concrete EAV data for agent prompts. |
| `council.js` | Multi-agent deliberation protocol (~200 lines). Synthesizes positions from parallel agent runs, retains disagreements (✓/✗/?). Used by review-gate. |
| `zones.js` | Deterministic diff classification by ZONES.md. Returns `zones_touched`, `seam_change`, `owners_to_ack`. |
| `contour-guard.js` | DLP integration: enforces data loss prevention policies on agent outputs. See `dlp.md`. |
| `tools/platform-docs.js` | Справочник по платформе: `docs_map`, `docs_read`, `docs_search`, `docs_tool`. Раздел «Platform Docs Corpus» ниже. |

### 24 Specialized Agents

<!-- BEGIN:agent-tool-counts — числа генерируются backend/scripts/sync-tool-counts.mjs; описания правятся руками и сохраняются -->
| Slug | Tools | Responsibilities |
|------|-------|-----------------|
| `tables` | 73 | EAV CRUD, schema, bulk ops, computed, history, comments, graph nodes, lookups |
| `docs` | 38 | Documents, blocks, versions, sharing, folders, tags |
| `meta-kb` | 38 | Meta KB mode: debate engine, streaming 3-phase analysis (research→debate→synthesis), knowledge curation, KAG entity management, change requests |
| `teamchat` | 29 | Team discussions, decisions, codespace integration |
| `telegram` | 27 | Telegram bot management, messages, moderation, payments, stories |
| `codespace` | 25 | Git repositories, branches, commits, files, PRs, GitHub Sync |
| `admin` | 21 | Members, settings, backups, trash, notifications, audit, workspace info, templates |
| `files` | 21 | Files, CSV import/export, connectors, AI connector gen, download_file, reconcile_cdek |
| `portal` | 19 | Portal config, publish, orders, metrics, client profiles, custom Vue components |
| `automation` | 17 | Automations, webhooks, forms |
| `dashboard` | 17 | Dashboards, widgets, views, public sharing |
| `reports` | 17 | Report create/edit, columns, filters, aggregation, run |
| `analyst` | 16 | Data analysis, pattern detection, conflict identification |
| `reviewer` | 13 | Code review, PR analysis, quality assessment |
| `kag` | 12 | Knowledge graph: KAG entity management, graph traversal, ontology, anomalies |
| `grants` | 11 | Roles, ACL grants, row-level rules |
| `advisor` | 8 | Platform usage advice, grounded in the docs corpus (`docs_map`, `docs_read`, `docs_search`, `docs_tool`) |
| `sandbox-engineer` | 7 | Code generation agent: generates code, sends results as code cells to teamchat |
| `normalizer` | 5 | Normalization pipeline: start, status, confirm |
| `sandbox-researcher` | 5 | Research agent for sandbox: RAG knowledge base, decision search, teamchat search, table data |
| `activity` | 3 | Team activity intelligence: activity tracking, analytics, insights |
| `sandbox-controller` | 3 | Compliance check: verifies code matches requirements and project standards |
| `sandbox-opponent` | 3 | Blind review opponent: critiques generated code, finds contradictions with team decisions |
| `timeseries` | 3 | Record and query timeseries data |
<!-- END:agent-tool-counts -->

> **Tool counts** above are agent-defined tools only. Each agent also receives 24 runtime-injected shared tools (memory, graph, KAG, delegation, clarification) from the orchestrator — not counted here.

> **Note:** Object-layer tools (ADR-016: role aliases, identity resolution, product movement) are implemented in `normalizer/object-layer.js`, `agent/tool-executor.js`, and `agent/risk-tiers.js` and are callable via MCP and the `POST /ai/tool` endpoint, but are not yet included in any agent's tool list.

### Shared Tools (all agents)

`remember`, `recall`, `forget`, `share_insight`, `list_contradictions`, `resolve_contradiction`, `find_procedure`, `list_agents`, `delegate_to_agent`, `get_related`, `graph_query`, `kag_search`, `kag_traverse`, `kag_ask`, `kag_stats`, `kag_browse`, `kag_clusters`, `kag_anomalies`, `kag_import_entities`, `kag_import_relations`, `kag_import_ontology`, `kag_update_tags`, `kag_delete`, `ask_clarification`

### Workspace Tools

Per-workspace custom tools registered in `_v2_workspace_tools` are integrated into the AI tool pipeline:

- **Discovery**: `GET /ai/tools` calls `listWorkspaceToolDefs(pool, db)` and merges workspace tools into the platform catalog via `mergeTools()`. Workspace tools carry `_workspace: true` marker.
- **Dispatch**: `tool-executor.js` default case — when a tool name does not match any platform `TOOL_DEFS`, `executeWorkspaceTool(pool, db, toolName, args)` is called as the final fallback.
- **Execution**: Tools run in an isolated-vm V8 sandbox (`workspace-tools/sandbox.js`). Pure functions use cached isolates; tools with capabilities use `executeBridged()` with fresh isolates.
- **MCP gate**: `POST /ai/tool` checks `hasWorkspaceTool()` after `TOOL_DEFS` and `AgentCatalog`, returns 400 if no match.

### Per-Question Agent Selection (`enabledAgents`)

Portal and frontend can pass `enabledAgents` (array of agent slugs) to filter which agents participate in a meta-kb debate. Flow:
- Portal: `POST /portal/api/agent/run` body includes `enabledAgents` → stored as `enabledAgentSlugs` in portal agent context
- Frontend: `POST /ai/chat` can include `enabledAgents` in the message payload
- `mk_start_debate` TOOL_DEF accepts optional `agents` array; tool-executor merges it with `toolCtx.enabledAgentSlugs`
- If both are empty, all active workspace agents participate

Management AI tools (group: `workspace-tools`):

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `list_workspace_tools` | TIER_LOW | List custom tools for the workspace |
| `register_workspace_tool` | TIER_HIGH | Register a new custom tool |
| `update_workspace_tool` | TIER_HIGH | Update an existing tool by ID |
| `delete_workspace_tool` | TIER_HIGH | Delete a custom tool |
| `import_tool_pack` | TIER_HIGH | Bulk upsert tool definitions |

See `backend/docs/modules/workspace-tools.md` for full documentation.

### Tool Response Format

All tools return structured JSON:
```js
// Create: id + type + message
{ id: 123, type: "table", name: "Clients", message: "Таблица создана." }
// List: items + total
{ items: [...], total: 5 }
// Error: error code + message
{ error: "NOT_FOUND", message: "Таблица не найдена." }
```

`semantic_search` returns `{ items, docs, objects, total }` — `docs` and `objects` are included alongside `items` for downstream consumers (MCP and chat).

`read_file` supports pagination via `offset` (character offset, default 0) and `maxChars` (max characters to return, default 12000) params for reading large files in chunks. Returns `{ fileId, filename, mimeType, extractedText, totalChars, offset, hasMore, nextOffset?, classifiedType, confidence, extractedFields, processingStatus, ocrEngine }`. When `hasMore=true`, use `nextOffset` for the next page.

### list_objects — Summary Aggregation & Pagination

`list_objects` returns an enriched response with aggregation and pagination metadata:

```js
{
  typeName: "Дроны",
  columns: [{ alias: "Страна", reqId: 301, type: "ref" }, ...],
  total: 1520,           // total matching rows
  hasMore: true,          // true if more pages exist
  page: 1,                // current page
  pageSize: 20,           // rows per page (default 20, max 100)
  rows: [{ id, name, ...fields }, ...],
  summary: {              // aggregation by ref columns (top 15 values each)
    "Страна": [{ value: "Россия", count: 1200 }, { value: "Китай", count: 200 }, ...],
    "Тип": [{ value: "FPV", count: 800 }, ...]
  },
  _truncated: true,       // true when hasMore=true — data is incomplete
  _hint: "Показана страница 1. Всего 1520 записей. Используй page=2 для продолжения."
}
```

**summary** — агрегация по ref-колонкам (макс. 5 колонок, топ-15 значений в каждой). Агент должен использовать summary для ответа на вопросы «сколько X», «какие Y есть» вместо перебора rows.
**Pagination** — при `hasMore: true` агент должен запросить `page: 2, 3, ...` для сбора всех данных.
**`_truncated`** — structured truncation pattern: если `true`, ответ агента неполон и нужна дополнительная пагинация.

### Smart Result Truncation (`_compactToolResult`)

Large tool results (>16000 chars) are automatically compacted before being added to conversation history to prevent context bloat:

- **`read_file` / `download_file`** — text body is truncated, metadata preserved; adds `_textTruncated: true`, `_originalTextLength`, `_hint` with pagination guidance, and synthesizes `hasMore`/`nextOffset` if needed
- **List results** (with `items` array) — keeps first 20 items, adds `_compacted` note with total count
- **Generic fallback** — valid JSON envelope with `_truncated: true` and partial content up to the budget

Defined in `agent/runner.js`.

### HITL (Human-in-the-Loop)

Risky operations return `{ status: "pending_confirmation", ... }` and pause execution. Resume via `POST /ai/agent-resume`. Risk tiers defined in `agent/risk-tiers.js`:
- **LOW (1)** — read-only, no HITL
- **MEDIUM (2)** — write, reversible, HITL in chat mode
- **HIGH (3)** — irreversible or external, always HITL

`skipHitl=true` on `POST /ai/tool` auto-confirms HITL (also controlled by workspace setting `settings.agent.skip_hitl`). **Only `owner` and `admin` roles can use `skipHitl`** — the flag is silently ignored for `editor`/`viewer`. Pending state stored in `agent/history.js`.

> **MCP-local HITL pattern:** `delete_workspace` uses a local HITL queue in `mcp-server/index.js` with an `onApprove` callback, bypassing the backend risk-tier/mcp-resume path.

### Pre-action Data Snapshots

Before destructive operations (`delete_table`, `bulk_delete`, `bulk_update`), the agent automatically saves a snapshot of affected EAV data to `_v2_data_snapshots`. This is non-disableable — snapshots are always created regardless of HITL settings.

- Storage: `_v2_data_snapshots` table (per workspace schema)
- Retention: 30 days + max 50 snapshots per table
- Module: `backend/src/api/v2/modules/schema/data-snapshots.js`
- Wired in: `resumeAfterHitl()` in `agent/index.js`

## Review Gate (`agent/review-gate.js`)

Ревью-ворота эпика #28 (A1, issue #33): состязательное мульти-агентное ревью диффа PR.

```
POST /codespace/review-gate { repo, pr }   (зовёт инфраструктура по событию PR — контракт шва B3/B4)
        ↓
   runReviewGate(pool, db, { repoSlug, prNumber })
   - дифф PR из codespace (getDiff target...source, бюджет 60K символов)
   - A3 (#35): линза швов — zones.js читает ZONES.md ревьюируемого репо,
     детерминированно даёт zones_touched / seam_change / owners_to_ack
   - A4 (#36): searchSimilarDecisionsTool → релевантные решения _v2_decisions
     передаются агентам как контекст; конфликты → rule: "decision-conflict"
     + decision_ref; реализуемые решения → decision_refs (id фильтруются по
     реально найденным)
   - параллельно через runner: reviewer-agent + sandbox-opponent (mode: automation)
   - синтез: council.synthesizePositions — разногласия удерживаются (✓/✗/?)
   - структуризация в вердикт по схеме A2 (review-verdict.js), 1 ретрай,
     невалидный вердикт в карточку не уходит (REVIEW_GATE_INVALID_VERDICT)
        ↓
   { verdict, evidence }  — verdict: контракт A2 (docs/audits/review-verdict-schema.md),
                            evidence: тексты агентов, синтез, решения, зоны
```

Вердикт — евиденция для человека-ЛПР, не приговор: финальной власти у агентов нет.

## AI Column Workers

### `autofill-worker.js`

BullMQ consumer on `ai-batch` queue. Processes batch autofill jobs: for each object in the batch, calls the AI column prompt and writes the result back to the EAV requisite.

### `ai-column-worker.js`

BullMQ consumer on `ai-column` queue. Triggered by `object.created` events when a type has AI columns configured. Runs the AI prompt for each AI column and writes results.

### `ai-column-listeners.js`

Subscribes to `object.created` event. For each type that has AI column configs, enqueues an `ai-column` job.

### Agent Mode

AI buttons support an **agent mode** toggle (`agentMode: true` in config). When enabled:

- Instead of `this.chat()` (single LLM call), `runButton()` routes to the full orchestrator via `opts.agentRunner`
- The interpolated prompt (all `[ColName]`, `[ID]`, `[VAL]` placeholders substituted) becomes the agent's user message
- The agent has access to all tools: `web_search`, table read/write, documents, etc.
- HITL is disabled (`skip_hitl: true`) — buttons run non-interactively
- `opts.agentRunner` is injected by callers (router.js, ai-column-worker.js, tool-executor.js via toolCtx.agentRunner) to avoid circular imports: `service.js` ← `tool-executor.js` ← `service.js`
- MCP path (`run_ai_button` tool): `agentRunner` is passed via `toolCtx.agentRunner = runAgent` set in `index.js` — no circular import since index.js owns both runAgent and toolCtx construction

Config field: `agentMode: boolean` — stored in `_v2_column_ai_config` alongside other button config.

**Note:** `autofillBatch()` does not pass `agentRunner`, so agent mode silently falls back to chat mode for batch autofill. Add explicitly if needed in the future.

### `llm-router.js`

LLM routing layer. Selects model, provider, and parameters based on workspace settings and request type (chat vs autofill vs agent vs quality). Supports multiple providers: DeepSeek, Kodacode, OpenAI, Claude, xAI — all via an OpenAI-compatible API interface. Normalizes requests before forwarding to the selected provider. After the retry loop, the last error is thrown if all providers fail. `AbortError` from a timeout bypasses retries and is thrown immediately.

### Friendly Error Messages

`llm-router.js` converts provider HTTP errors into user-readable messages:
- **402** (insufficient balance) — treated like 429 for provider fallback; triggers cooldown via `makeCooldownError`
- **429** (rate limited) — triggers provider cooldown and falls back to the next candidate; error message includes provider/model name
- **Quota exceeded** — `routedFetch` pre-checks workspace quota before making the LLM call; throws a detailed Russian-language message with used/limit/period info and `code: 'QUOTA_EXCEEDED'`, `status: 429`

### Sampling parameters a model no longer accepts

Anthropic dropped `temperature`, `top_p` and `top_k` from Opus 4.7 onward (Opus 4.8/5,
Sonnet 5, Fable 5): a request carrying one gets a `400`. Through an OpenAI-compatible proxy
it reads ``` `temperature` is deprecated for this model ``` — measured 14–15.08.2026 on
`auth2api/claude-sonnet-5`, the fallback candidate of the `vision` alias, which is why
image recognition returned nothing at all on that stand.

`routedFetch` fixes this by behaviour, not by a model list — a list rots silently, and the
"opus in the name" test already failed once (`ai/agent/llm.js`). On a `400` whose body names
one of `SAMPLING_PARAMS`, the router drops **only the named field**, repeats the request
once, and records the `provider:model` pair in `_rejectedParams`; every later request to
that pair goes out without the field, so the price of learning is one round trip per
process. A `400` that names nothing triggers no retry — otherwise every failure would
double the load. The memo is cleared by `_resetConfig()`.

`ai/agent/llm.js` keeps its own one-shot `temperature` retry. It is now a second net rather
than the only one: the router's attempt happens first, and by the time `callLLM` inspects
the response the field is already gone.

### 429 — pause before moving on

A `429` is a request to wait, not a verdict on the model. Switching to the next candidate
instantly burns the chain for nothing (the next candidate may be pricier, worse, or absent),
so the router pauses and repeats **the same candidate once**: `Retry-After` seconds when the
header is present, otherwise `LLM_RATE_LIMIT_WAIT_MS` (default 2000), capped by
`LLM_RATE_LIMIT_MAX_WAIT_MS` (default 15000). The cap matters because a provider may ask for
an hour while the OCR engine only lives two minutes — waiting past the caller's own deadline
just trades a rate limit for a timeout. The pause does not survive the caller's `AbortSignal`.
If the repeat is refused too, the pre-existing cooldown takes over and the next candidate
runs; long-running shortages are the cooldown's job, not a wait loop's.

### LLM Quota System

Per-workspace token/request quotas configured via workspace settings (`settings.ai.quota_tokens`, `settings.ai.quota_period`). Quota periods: `day`, `week`, `month` (default). Token usage tracked in the `llm_calls` table.

`GET /ai/quota` (admin only) returns: `{ quota_tokens, quota_period, period_start, period_end, used_tokens, remaining_tokens }`. Returns `remaining_tokens: null` if no quota is configured.

`routedFetch` in `llm-router.js` calls `checkQuota()` before each LLM request when `opts.workspaceId` and `opts.workspaceSettings` are provided. When quota is exhausted, throws with `code: 'QUOTA_EXCEEDED'`, `status: 429`.

**Both fields are required, and omitting them is silent.** A caller that passes only `workspaceId` gets no quota check and no closed-contour check — there is no warning, the request simply goes out unmetered. HTTP routes take the settings from `req.workspace.settings` (populated by `middleware/db.js`). Background callers have no `req`, so `getWorkspaceSettings(db)` in `ai/service.js` reads them from `_v2_workspaces` by `db_name` or `slug` and caches the result for 60 seconds; settings edited in the admin UI take effect within that minute. It returns `null` on a read failure — distinguishable from `{}` ("workspace has no settings") — and the caller then proceeds unmetered rather than failing the job.

Document OCR was such a caller until 2026-08-12: `ocrAndExtractViaVision` reached `routedFetch` with neither field, so the one LLM call in the file pipeline bypassed both the quota and the closed contour. See `files.md`, section "Workspace context".

### DLP Rules CRUD

Admin-only endpoints for managing Data Loss Prevention rules. All require `admin` role + CSRF.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/ai/dlp/rules` | List DLP rules for workspace |
| POST | `/ai/dlp/rules` | Create rule. Body: `{ rule_type, value, description?, severity?, enabled?, model?, timeout_ms? }`. `rule_type`: `keyword`, `regex`, `type_block`, `llm_classify`. `severity`: `block`, `warn`, `audit`. Regex values are validated. Invalidates contour-guard cache. |
| PATCH | `/ai/dlp/rules/:id` | Update rule fields. At least one field required. Returns 404 if not found. |
| DELETE | `/ai/dlp/rules/:id` | Delete rule. Returns 404 if not found. |

### TTS Tools

Text-to-speech via server-side Piper TTS engine. All tools are `TIER_LOW` (no HITL).

| Tool | Description |
|------|-------------|
| `speak_text` | Convert text to speech. Args: `{ text, voice?, speed? }`. Returns `{ audio (base64 PCM), format, sampleRate, channels, text }`. Returns error if Piper is not installed. |
| `list_tts_voices` | List available TTS voices. Returns `{ available, engine, voices[], note? }`. |
| `get_tts_status` | Check TTS engine availability and status. Returns engine info and readiness. |

REST endpoints: `POST /tts/synthesize`, `GET /tts/voices` (module: `tts/router.js`). Implementation: `agent/tools/tts.js`.

### Specs Tools

Tools for data spec evaluation and enforcement. Specs define validation/quality rules per type.

| Tool | Tier | Description |
|------|------|-------------|
| `list_specs` | LOW | List data specs defined for a type |
| `check_record` | LOW | Evaluate a record against all its data specs |
| `run_spec` | LOW | Run a single data spec check on a record |

### Resolution Tools

Client identity resolution and verification.

| Tool | Tier | Description |
|------|------|-------------|
| `resolve_client` | MEDIUM | Resolve client to golden record (dedup/merge) |
| `verify_client_shipping` | LOW | Verify client shipping address |

### Excel Tool

| Tool | Tier | Description |
|------|------|-------------|
| `create_excel` | MEDIUM | Generate XLSX file from data. Global tool (tool-executor.js), available to any agent. |

### Web Search Tool

| Tool | Tier | Group | Description |
|------|------|-------|-------------|
| `web_search` | LOW | core | Web search via Tavily API with SearXNG fallback when Tavily keys are exhausted |

### Platform Docs Corpus (advisor group)

Справочник по самой платформе: документация репозитория и каталог инструментов.
Реализация — `agent/tools/platform-docs.js`, все четыре инструмента `TIER_LOW`.

| Tool | Что отдаёт |
|------|-----------|
| `docs_map` | Карта корпуса: путь, слой, модуль, заголовок, однострочное описание, дата последнего коммита файла. Фильтры `area`, `module`, `query` |
| `docs_read` | Документ целиком или один раздел. Раздел режется от своего заголовка до следующего того же или более высокого уровня. Пагинация `offset`/`maxChars` → `hasMore`/`nextOffset` |
| `docs_search` | Гибридный поиск (`hybridSearchDocs`) по фрагментам с цитатой `path#section`. Пустая выдача объявляется как «в найденном нет», а не «в документации нет» |
| `docs_tool` | Карточка инструмента платформы из **живых** `TOOL_DEFS` и `risk-tiers.js`: группа, риск-тир, описание, параметры. Копии в корпусе нет намеренно |

Четыре инструмента, а не один поиск: единственный непрозрачный `search_docs`
не даёт агенту увидеть структуру корпуса, и он не может понять, чего не
посмотрел. Полные карточки инструментов ограничены 25 на вызов — 626 схем
это ~87k токенов, и вываливать их в разговор значит испортить выбор инструмента.

**Хранение.** Корпус общий для всех воркспейсов и лежит под зарезервированным
ключом `__platform_docs__` в глобальных таблицах `platform_docs` (карта + полный
текст) и `doc_chunks` (фрагменты, `source_type = 'platform_doc'`). Именем
воркспейса этот ключ быть не может: `DB_NAME_RE` в `middleware/db.js` требует
буквы в начале.

**Сборка и заливка.**

```bash
npm run docs:corpus       # scripts/build-docs-corpus.mjs → backend/.docs-corpus/
npm run docs:corpus:load  # scripts/load-docs-corpus.mjs  → platform_docs + doc_chunks
npm run docs:corpus:eval  # scripts/eval-docs-corpus.mjs  → Recall@5 по эталонному набору
```

Сборщик падает на документе без заголовка или без описания. Текст документов
едет внутри артефакта: на прод уезжает только `backend/`, каталогов `docs/`,
`portal/docs/`, `frontend/docs/` там нет. В корпус не идут `docs/superpowers`,
`docs/audits`, `docs/other`, `docs/archive`, `docs/plans`, `docs/research`,
`docs/e2e`, `docs/testing` и списки техдолга — в выдаче поиска зачёркнутый
закрытый пункт читается как описание текущего поведения.

Заливка переживает недоступный провайдер эмбеддингов: фрагменты пишутся без
векторов, документ помечается `vectors_ok = false`, следующий запуск достраивает.
`docs_search` в таком состоянии возвращает поле `_degraded`.

**ask_advisor** больше не носит зашитый перечень возможностей платформы: правила
моделирования берутся из общего `SCHEMA_GUIDE_TEXT`, факты — из `docs_search`.

## MCP Integration

MCP server (`mcp-server/`) calls:
- `GET /ai/tools` to discover available tools
- `POST /ai/tool` to execute a tool

Uses the same `tool-executor.js` as the agent runner. HITL is handled via `POST /ai/mcp-resume`.

Tool groups include `core` (platform-wide tools), per-agent groups, and `orgs` (organization-level MCP tools exposed to external MCP clients).

## DB Tables (per-workspace, lazy-init)

- `_v2_ai_conversations` — conversation metadata
- `_v2_ai_messages` — message history (role, content, tool calls)
- `_v2_ai_usage` — token counts per request
- `_v2_ai_audit_log` — tool call log (name, args, result, risk_tier, duration_ms, HITL flags). The caller is split across two columns: `user_id BIGINT` for a real `_v2_users.id`, and `actor_ref VARCHAR(255)` for callers that have no user row — `a2a`, `agent:<slug>`, or a bare username. Both are written by `logToolCall()` via `splitAuditActor()`; binding a string straight to `user_id` used to make PostgreSQL reject the INSERT and drop the whole row. A refused write is counted in `getAuditDropStats()` and reported as `meta.droppedSinceStart` by `GET /:db/ai/audit-log`
- `_v2_ai_pending_hitl` — one pending confirmation/elicitation per `(thread_id, actor_key)`, which is also its UNIQUE key. `actor_key` is the canonical owner produced by `hitlActorKey()` on top of `splitAuditActor()`: `u:<id>` for a real `_v2_users.id`, `ref:<s>` for a caller with no user row (`a2a`, `agent:<slug>`, a bare username, `portal`). `user_id BIGINT` is kept alongside it for numeric users only. Every lookup (`getPendingHitl`, `takeAndDeletePendingHitl`, `deletePendingHitl`) matches on `actor_key`, so an actor can only ever claim its own pending — the previous `UNIQUE (thread_id)` let a second actor upsert its payload onto the first actor's row, and the first actor's confirmation then executed somebody else's action
- `_v2_ai_feedback` — user thumbs up/down
- `_v2_ai_messages` — columns include `file_ids JSONB` (array of uploaded file IDs attached to the message)
- `_v2_column_ai_config` — AI button/column prompt configs
- `llm_calls` — per-request LLM call log (global table). The caller is split the same way as in `_v2_ai_audit_log`: `user_id BIGINT` for a real `_v2_users.id`, `actor_ref TEXT` for callers with no user row (`a2a`, `agent:<slug>`, a username), both via `splitAuditActor()`. Before that split the column was `INTEGER` and a string actor made PostgreSQL refuse the INSERT, so every LLM call from an A2A or teamchat context lost its cost, latency and `trace_id` — and the workspace token quota under-counted by exactly those calls. A refused write is counted in `getLlmLogDropStats()` and logged at error level. Columns: `workspace_id`, `user_id`, `actor_ref`, `feature`, `agent_id`, `model`, `provider`, `prompt_tokens`, `compl_tokens`, `cost_usd` (computed via `calcCost()` using per-model pricing), `latency_ms`, `routing_method`, `routing_ms`, `recall_ms`, `total_pipeline_ms`, `cache_hit`, `prompt_cache_hit_tokens`, `prompt_cache_miss_tokens`, `tool_calls_count`, `llm_rounds`, `trace_id`, `contour`, `status`, `blocked_reason`, `created_at`. Written fire-and-forget by `logLlmCall()` in `service.js` — errors logged as warnings, never thrown

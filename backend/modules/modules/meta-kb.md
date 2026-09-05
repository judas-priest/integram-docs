# Meta KB Module

## Overview

Meta KB provides debate-based knowledge curation for the AI assistant's Meta KB mode. When users fixate decisions, workspace-specific expert agents can debate the answer before it is saved to the knowledge base.

## Architecture

- **Meta KB agent** (`agents/meta-kb.js`): separate agent with 34 core tools, streaming 3-phase analysis (research → debate → synthesis), cross-debate streaming, markdown rendering
- **KAG auto-indexing**: decisions are indexed to `kag_entities` automatically via a bus listener in `decisions/listeners.js` (events: `decision.created`, `decision.updated`, `decision.deleted`). Uses OR semantics for FTS (`|` instead of `&` in tsquery).
- **Debate engine**: parallel agent opinions (Phase 1) + LLM synthesis (Phase 2), streaming results via SSE
- **Debates stored** in global `_v2_mk_debates` table filtered by `db` column for workspace isolation
- **Embedding**: uses RouterAI API for vector search
- **Welcome endpoint** (`GET /:db/meta-kb/welcome`): returns `{ stats: {entities, relations}, recentTopics, recentDecisions, pendingChanges, roomId, starters }`. Uses `wsTable()` for schema-qualified table access.

## Files

```
backend/src/api/v2/modules/meta-kb/
  router.js          — REST endpoints: welcome, topics, debates, analytics, changes, snapshots, rules, export
  service.js         — debate engine: ensureTables, loadInternalAgents, runDebate, runDebateLoop, saveDebate, listDebates
  analytics.js       — workspace knowledge analytics
  change-requests.js — knowledge change request management
  research.js        — concept research in KAG
  rules.js           — knowledge governance rules
  snapshots.js       — knowledge base snapshots
  export.js          — debate export: Markdown and DOCX file generation, `mdToDocxBuffer(md)` for generic conversion
  welcome.js         — welcome endpoint data: stats, recent topics/decisions, pending changes, starters
  iterations.js      — iteration CRUD: list, update status, link debates to decisions
  appropriation.js   — Socratic appropriation gate: question generation, answer evaluation, covenant acts
  gift.js            — gift matrix: covenant act tracking, closed gifts listing
backend/src/api/v2/modules/ai/agent/
  agents/meta-kb.js  — Meta KB agent: system prompt, tool set, 3-phase debate orchestration
  tools/meta-kb.js   — 22 mk_* tool handlers
```

## Tables

### `_v2_mk_debates` (global)

| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT IDENTITY | Primary key |
| db | TEXT | Workspace slug |
| decision_id | BIGINT | Optional link to a decision |
| question | TEXT | Debate question |
| answer_text | TEXT | Context/answer being evaluated |
| opinions | JSONB | Object `{ opinions: [{ slug, name, text, latency }], crossTurns: [{ slug, name, text }] }` |
| consensus | TEXT | Synthesized conclusion from moderator LLM |
| verdict | VARCHAR(16) | Optional verdict tag |
| created_by | VARCHAR(128) | Username or 'system' |
| topic_id | BIGINT | Optional link to a teamchat topic |
| created_at | TIMESTAMPTZ | Creation time |

### `_v2_mk_iterations` (global)

| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT IDENTITY | Primary key |
| db | TEXT | Workspace slug |
| topic_id | BIGINT | Optional link to a teamchat topic |
| query | TEXT | Debate question / iteration query |
| status | VARCHAR(16) | Status: `in_progress`, `resolved`, etc. (default `in_progress`) |
| decision_id | BIGINT | Optional link to a fixated decision |
| proposed_at | TIMESTAMPTZ | When the iteration was proposed |
| resolved_by | VARCHAR(128) | Username who resolved |
| resolved_at | TIMESTAMPTZ | When the iteration was resolved |
| created_by | VARCHAR(128) | Username or 'system' |
| created_at | TIMESTAMPTZ | Creation time |

## Service API

### `ensureTablesOnce(pool)` (memoized)
Creates **both** global tables if they do not exist, delegating to `ensureTables(pool)`:

- `_v2_mk_debates` plus indexes `idx_mk_debates_db` (on `db`) and `idx_mk_debates_topic` (on `topic_id`);
- `_v2_mk_iterations` plus indexes `idx_mk_iterations_db` (on `db, created_at DESC`) and `idx_mk_iterations_status` (on `db, status`). The iterations block is guarded by a probe — `SELECT 1 FROM _v2_mk_iterations LIMIT 0` — and skipped entirely when the table already answers, so its DDL does not run on every cold start the way the debates DDL does.

Memoized — only runs DDL once per process lifetime via an internal `_tablesReady` flag.

### `loadInternalAgents(pool, db)`
Returns agents from `_v2_agent_registry` where `type = 'internal'` AND `active = true`. These are the workspace-specific expert personas used for debate positions.

### `runDebate(pool, db, question, answerText, agents, user, send, signal, topicId)`
Runs a multi-agent debate:
1. **Phase 1 (agentic loop)**: each agent in `agents` runs `debateAgentLoop()` — a multi-round agentic loop of up to `MAX_DEBATE_ROUNDS = 5` rounds, with access to `DEBATE_TOOLS` (tool calls allowed each round). Not a simple single LLM call.
2. **Phase 2 (cross-debate)**: agents respond to each other's opinions (SSE event: `cross-debate`)
3. **Phase 3 (synthesis)**: runs parallel KAG search + web search, injects results into the moderator prompt, then a moderator LLM call synthesizes all opinions into a `consensus`

Each agent has a per-agent timeout of `AGENT_TIMEOUT_MS = 180_000` (3 minutes). If an agent exceeds this, its opinion is recorded as a timeout error and the debate continues with remaining agents.

Incremental save for crash recovery: positions are saved after Phase 1, cross-turns and consensus are updated after Phase 3.

Returns `{ opinions: Array, crossTurns: Array, consensus: string, debateId: number }`.

- Throws `Error` if `agents` is empty or `question` is blank
- Agent LLM errors are caught and stored in the opinion as `{ error, text: '[Error: ...]' }` — debate continues
- `send` callback: called when each opinion/cross-turn completes (used for SSE streaming)
- `signal`: AbortSignal for cancellation
- `topicId`: optional topic ID for debate persistence
- After debate completes, `recordDebateGiftActs` is called fire-and-forget to record gift acts

### `saveDebate(pool, db, data, user)`
Persists a debate result to `_v2_mk_debates`. Calls `ensureTablesOnce` first.

Parameters in `data`: `{ decisionId?, question, answerText?, opinions, crossTurns?, consensus, verdict?, topicId? }`

Returns the new debate `id` (number).

### `listDebates(pool, db, limit = 10)`
Returns recent debates for a workspace, ordered by `created_at DESC`. Calls `ensureTablesOnce` first.

### `getDebateById(pool, db, debateId)`
Returns a single debate by its ID, or `null` if not found.

### `listDebatesByTopic(pool, db, topicId)`
Returns all debates linked to a specific teamchat topic, ordered by `created_at DESC`.

### `listDebatesByDecision(pool, db, decisionId)`
Returns the debates carrying that `decision_id`, ordered by `created_at DESC` — the backlink written by `linkDebateToDecision`. Selects `id`, `topicId`, `question`, `consensus`, `verdict`, `created_by`, `created_at` (no `opinions`, no `answer_text`). Calls `ensureTablesOnce` first.

### `mergeTools(agent, pool, db)` (async)
Resolves agent's `tools` string array — looks up each name first from platform TOOL_DEFS, then falls back to workspace `getCachedDefs`. Called by the debate engine before each agent run.

### `getDebateByTopic(pool, db, topicId)`
Returns the most recent debate linked to a specific teamchat topic, or `null` if none exists. Calls `ensureTablesOnce` first.

### `linkDebateToDecision(pool, db, topicId, decisionId, createdBy, opts = {})`
Stamps `decision_id` onto every debate of the topic that has none yet, then creates one `_v2_mk_iterations` row (status `proposed`, `query` taken from the topic's newest debate question, or the literal `'Meta KB debate'` when there is none) unless a row for that `(db, topic_id, decision_id)` already exists. Used when a debate outcome is fixated as a decision.

**`opts.requireDebate` makes it refuse instead of writing.** With that flag set, the function first checks whether the topic has any debate at all, and on finding none **returns `false` having written nothing** — no stamping, no iteration. It exists for the link-by-conversation-id path of the ordinary AI chat, where a topic may legitimately hold no debate and an empty iteration would be a lie. Every other outcome returns `true`. The return value is a two-state answer, not a status: read it.

## Loop Mode

### `evaluateConsensus(question, consensus, signal)` (internal)
LLM self-evaluation of debate quality. Scores the consensus on completeness, specificity, argumentation, and practicality (1–10). Returns `{ pass: boolean, score: number, feedback: string }`. `pass = true` if `score >= 7`. On parse failure, defaults to `{ pass: true, score: 7 }` to avoid infinite loops.

### `runDebateLoop(pool, db, question, answerText, agents, user, send, signal, topicId)`
Iterative refinement wrapper around `runDebate`. Calls `runDebate` up to `MAX_LOOP_ITERATIONS = 3` times, evaluating consensus quality between iterations via `evaluateConsensus`. If the evaluation passes (score >= 7) or the iteration limit is reached, the loop stops and returns the last `runDebate` result. On each failed evaluation, the next iteration receives the previous consensus and feedback as additional context.

Triggered by the `loopMode` body parameter on `POST /:db/ai/agent-chat`.

SSE events emitted: `loop-iteration`, `loop-evaluating`, `loop-evaluation`, `loop-done`.

## AI Tools

Defined in `backend/src/api/v2/modules/ai/agent/tools/meta-kb.js`:

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `mk_start_debate` | TIER_MEDIUM | Runs a multi-agent debate on a question with workspace expert agents. TOOL_DEF parameter `context` maps to `answerText` in the service. Optional `agents` array parameter selects specific agent slugs for the debate (empty = all active). Debate agents have access to workspace-registered tools via `mergeTools()`. |
| `mk_analytics` | TIER_LOW | Workspace knowledge analytics: entity/relation counts, sources, isolated nodes |
| `mk_research` | TIER_LOW | Research a concept in KAG: find entities, graph neighbors, knowledge gaps |
| `mk_propose_change` | TIER_MEDIUM | Propose a knowledge base change (add/update/delete entity), creates a review request |
| `mk_revoke_entity` | TIER_MEDIUM | Marks all KAG entities from a decision as `status='rejected'` in `kag_entities` |
| `mk_list_debates` | TIER_LOW | Lists recent debates for the current workspace |
| `mk_list_snapshots` | TIER_LOW | List knowledge base snapshots with date, label, and statistics |
| `mk_create_snapshot` | TIER_HIGH | Create a snapshot of the current knowledge base state for later comparison |
| `mk_diff_snapshots` | TIER_LOW | Compare two snapshots: added/removed entities and statistics delta |
| `mk_review_change` | TIER_HIGH | Approve or reject a proposed knowledge base change |
| `mk_export_debate` | TIER_LOW | Export a debate to Markdown format |
| `mk_list_topics` | TIER_LOW | List topics in the meta-kb room |
| `mk_appropriate_decision` | TIER_MEDIUM | Socratic appropriation gate: step 1 (no answers) generates questions about the consensus; step 2 (with answers) evaluates and, on success, records a covenant act in the gift matrix |
| `mk_welcome` | TIER_LOW | Get Meta-KB summary on open: stats, recent debates, recommendations |
| `mk_list_rules` | TIER_LOW | List knowledge base validation rules |
| `mk_run_rules` | TIER_LOW | Run validation rules on specified entities, returns violations |
| `mk_gift_matrix` | TIER_LOW | Gift matrix: covenant acts between debate participants |
| `mk_gift_closed` | TIER_LOW | Participants with balanced give/receive ratio over a period |
| `mk_create_rule` | TIER_MEDIUM | Create a knowledge base validation rule (condition + action as JSON) |
| `mk_delete_rule` | TIER_HIGH | Delete a knowledge base validation rule by ID |
| `mk_list_iterations` | TIER_LOW | List Meta-KB iterations with optional status filter |
| `mk_get_debate` | TIER_LOW | Get full debate by ID: question, opinions, consensus, verdict |

Additional non-mk tools available to the meta-kb agent (28 of the 34 names in `coreTools` — everything that is not `mk_*`): search — `kag_search`, `semantic_search`, `search_similar_decisions`, `search_teamchat`, `web_search`; decisions — `create_decision`, `analyze_decision_conflicts`; files for a human — `create_file` (md/txt/docx/pdf/pptx, link in `downloadUrl`), `create_excel`; data — `list_tables`, `get_table_schema`, `list_objects`, `get_object`, `list_documents`, `get_report`, `list_reports`; topics — `list_topics`, `delete_topic`; workspace tools — `register_workspace_tool`, `list_workspace_tools`, `update_workspace_tool`, `delete_workspace_tool`; codespace — `list_repos`, `get_file_tree`, `read_blob`, `commit_file`, `commit_multi_files`, `run_script`.

`create_object`, `update_object`, `create_document` and `append_block` are defined in the agent's `tools` map but are **absent from `coreTools`**, and `coreTools` is what `runner.js` shows the model — so the meta-kb agent cannot call them. This is deliberate: the system prompt tells it to hand writes over to the AI Assistant via `[switch-to-assistant: …]`. There are no `create_topic` / `update_topic` tools in this agent at all.

## REST Endpoints (additional)

### `GET /:db/meta-kb/topics/:topicId/debates`

List all debates for a topic. Multiple debates can exist per topic (debate history). Returns `{ debates: [...] }` ordered by `created_at DESC`.

### `GET /:db/meta-kb/decisions/:decisionId/debates`

Role `editor`. Lists the debates linked to a decision — the other side of the
backlink written by `linkDebateToDecision`. Returns `{ debates: [...] }` ordered
by `created_at DESC`, via `listDebatesByDecision()`.

### `GET /:db/meta-kb/topics/:topicId/debate`

Load the most recent saved debate for a teamchat topic. Returns `{ debate }` (or `{ debate: null }` if none exists). Uses `getDebateByTopic()` from service.js.

### `GET /:db/meta-kb/topics`

List topics in the meta-kb room. Finds the room named `'meta-kb'`, then returns up to 50 topics ordered by `updated_at DESC` with message counts. Returns `{ topics: [] }` if no meta-kb room exists.

### `DELETE /:db/meta-kb/topics/:topicId`

Delete a topic by ID.

### `GET /:db/meta-kb/debates/:id/export`

Download a debate as a Markdown file. Returns the file as an attachment (`Content-Disposition: attachment`).

### `GET /:db/meta-kb/debates/:id/export/docx`

Download a debate as a DOCX file. Returns the file as an attachment.

### `POST /:db/meta-kb/export/docx`

Generic markdown-to-DOCX conversion. Body: `{ markdown, filename }`. Returns DOCX buffer as attachment.

### Iterations

- `GET /:db/meta-kb/iterations` — list iterations for the workspace
- `PATCH /:db/meta-kb/iterations/:id` — update iteration status/resolution

### Changes

- `GET /:db/meta-kb/changes` — list proposed knowledge changes
- `POST /:db/meta-kb/changes` — create a new change proposal
- `POST /:db/meta-kb/changes/:id/review` — approve or reject a change

### Snapshots

- `GET /:db/meta-kb/snapshots` — list knowledge base snapshots
- `POST /:db/meta-kb/snapshots` — create a new snapshot
- `GET /:db/meta-kb/snapshots/diff` — compare two snapshots (query params: `a`, `b`)

### Rules

- `GET /:db/meta-kb/rules` — list governance rules
- `POST /:db/meta-kb/rules` — create a governance rule
- `DELETE /:db/meta-kb/rules/:id` — delete a governance rule
- `POST /:db/meta-kb/rules/run` — run rules against current knowledge base

### Other

- `GET /:db/meta-kb/debates/:id` — get a single debate by ID
- `POST /:db/meta-kb/debates/:id/appropriate` — Socratic appropriation gate (question generation / answer evaluation)
- `GET /:db/meta-kb/gift-matrix` — gift matrix: covenant acts between participants
- `GET /:db/meta-kb/gift-closed` — list of closed (completed) gifts

## Integration Points

- **`ai/router.js`**: `POST /:db/ai/agent-chat` with `agentSlug: 'meta-kb'` — routes to meta-kb agent directly (bypasses orchestrator). `GET /:db/meta-kb/welcome` — workspace stats for the meta-kb UI.
- **`agents/meta-kb.js`**: meta-kb agent system prompt with dedup rules, anti-phantom citation rules, aggregation awareness, 3-phase streaming (research → debate → synthesis)
- **`decisions/listeners.js`**: bus listener on `decision.created/updated/deleted` auto-indexes decisions to KAG (`kag_entities` table via swarm-memory)
- **`agents/teamchat.js`**: updated prompt instructs the agent to search KAG before web fallback
- **`agent/index.js`** (TOOL_DEFS): all 22 mk_* tools registered (`mk_start_debate`, `mk_analytics`, `mk_research`, `mk_propose_change`, `mk_revoke_entity`, `mk_list_debates`, `mk_welcome`, `mk_list_rules`, `mk_run_rules`, `mk_gift_matrix`, `mk_gift_closed`, `mk_create_rule`, `mk_delete_rule`, `mk_list_iterations`, `mk_get_debate`, etc.)
- **`tool-executor.js`**: switch cases for all 22 tools
- **`risk-tiers.js`**: `mk_list_debates`/`mk_analytics`/`mk_research`/`mk_welcome`/`mk_list_rules`/`mk_run_rules`/`mk_gift_matrix`/`mk_gift_closed`/`mk_list_iterations`/`mk_get_debate` = TIER_LOW, `mk_start_debate`/`mk_propose_change`/`mk_revoke_entity`/`mk_create_rule`/`mk_appropriate_decision` = TIER_MEDIUM, `mk_delete_rule`/`mk_create_snapshot`/`mk_review_change` = TIER_HIGH

## LLM Usage

`runDebate` uses `callLLM` from `agent/llm.js`. Each agent opinion uses `model = agent.model || 'fast'`. Synthesis uses `model='smart'`. Temperature and maxTokens use defaults from the LLM router.

## Portal Proxy (`/portal/api/meta-kb/*`)

**File:** `portal/meta-kb-proxy.js`
**Auth:** `portal_jwt` required + `metaKb.enabled` config flag
**Rate limit:** 30 req/min per portal client

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/welcome` | Stats + starter prompts |
| GET | `/topics` | List topics from meta-kb room |
| GET | `/topics/:topicId/debate` | Latest debate for topic |
| GET | `/topics/:topicId/debates` | All debates for topic |
| GET | `/debates/:id` | Debate by ID |
| GET | `/debates/:id/export` | Markdown export |
| GET | `/debates/:id/export/docx` | DOCX export |
| POST | `/debates/:id/appropriate` | Socratic evaluation |
| GET | `/analytics` | KAG analytics |
| GET | `/iterations` | Iteration list |
| PATCH | `/iterations/:id` | Update iteration status |
| GET | `/changes` | Change request list |
| POST | `/changes` | Propose change (entityId, changeType, proposed) |
| POST | `/changes/:id/review` | Approve/reject change |

### Debate Data Structure

Returned by `getDebateByTopic` / `getDebateById`:

```json
{
  "id": 123,
  "question": "...",
  "answerText": "...",
  "opinions": [{"name": "Agent", "text": "...", "slug": "agent-slug", "latency": 1234, "toolCalls": [...], "error": null}],
  "crossTurns": [{"name": "Agent", "text": "...", "slug": "agent-slug", "toolCalls": [...]}],
  "consensus": "Synthesis text...",
  "verdict": "accepted|rejected|null",
  "createdAt": "ISO date"
}
```

**Note:** field is `opinions` NOT `experts`, `consensus` NOT `synthesis`.

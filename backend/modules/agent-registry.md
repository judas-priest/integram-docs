# Module: agent-registry

**Path:** `src/api/v2/modules/agent-registry/`
**Files:** `router.js`, `service.js`, `a2a.js`
**Base URL:** `/api/v2/:db/agent-registry/...`
**Auth:** JWT required. List/get requires `editor`. Create/update/delete requires `admin`.

## Purpose

Registry of AI agents — both external (HTTP callback) and internal (Meta KB debate personas). Workspace admins register agents through UI; the orchestrator delegates tasks to external agents via `delegate_to_agent` tool.

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/agent-registry` | editor | List agents (all, not just active). `?roomId=N` filters by domain role for a teamchat room |
| POST | `/agent-registry` | admin | Register a new external agent |
| GET | `/agent-registry/:id` | editor | Get agent details |
| PATCH | `/agent-registry/:id` | admin | Update agent config |
| DELETE | `/agent-registry/:id` | admin | Delete agent registration |
| GET | `/agent-registry/:id/metrics` | editor | Agent performance metrics from swarm memory |
| GET | `/agent-registry/:id/tasks` | editor | Task history for an agent |
| POST | `/agent-registry/:id/rotate-secret` | admin | Rotate HMAC callback secret |
| POST | `/agent-registry/:id/invoke` | admin | Manually invoke agent for testing |
| POST | `/agent-registry/:id/health` | admin | Ping agent health endpoint |
| POST | `/agent-registry/callback` | HMAC or service key | Receive async task result from agent |

## Agent Config Schema

```json
{
  "slug": "my-agent",
  "name": "My Agent",
  "description": "...",
  "capabilities": ["analyze", "summarize"],
  "utterances": ["analyze this", "summarize report"],
  "callback_url": "https://agent.example.com/invoke",
  "callback_auth": {
    "type": "bearer",
    "token": "secret"
  },
  "protocol": "integram",
  "type": "external",
  "active": true,
  "mode": "sync",
  "timeout_ms": 30000
}
```

- `mode`: `sync` (wait for response) or `async` (agent calls `/callback` later)
- `utterances`: example phrases used by orchestrator for semantic routing to this agent
- `callback_auth`: supports `bearer`, `basic`, `header` auth types

## Internal Agents

Agents with `type: 'internal'` are used for Meta KB debates (`debateAgentLoop`). They run inside the platform — no external HTTP calls.

### Config

```json
{
  "slug": "analyst",
  "name": "Аналитик",
  "type": "internal",
  "system_prompt": "Ты — аналитик рынка...",
  "tools": ["kag_search", "web_search", "read_room"],
  "model": "fast"
}
```

- `type`: `'internal'` — no `callback_url` required
- `system_prompt`: LLM system message (required, max 10000 chars)
- `tools`: subset of debate tools (`kag_search`, `web_search`, `search_similar_decisions`, `search_teamchat`, `read_room`, `semantic_search`). Workspace-registered tools from `_v2_workspace_tools` can be mixed with platform tools — they are merged at debate time via `mergeTools()`.
- `model`: `'fast'` or `'smart'` (default: `'fast'`)
- `memory`: `'shared'` (default) or `'off'` — see **Memory mode** below
- `type` is immutable — set at creation, cannot be changed via PATCH

### Running an internal agent outside debates

Since issue #309 an internal agent is also a workspace-defined agent (ADR-024): the row is
assembled by `ai/agent/workspace-agent.js` and executed by the shared runner. Three doors lead
to the same run: the portal (`POST /portal/api/agent/run` with `agentSlug`, and
`/portal/api/agents/invoke-many`), `service.invoke()` (REST `/:id/invoke`, the
`delegate_to_agent` / `invoke_agent` automation actions) and the orchestrator's direct route.
Role and grants are the caller's, without elevation; a HITL request from a tool fails the run
when there is nobody to confirm (automations).

### Memory mode (`memory`)

Workspace memory is recalled by `(db, agentId)` plus the shared namespace — **not** per
interlocutor. For an agent that talks to *different people* (a customer-facing bot) this mixes
conversations: one customer's question ends up in the context of another customer's answer
(reproduction in issue #318). `memory: 'off'` closes both directions:

- **no reads** — the runner skips all three recalled blocks: workspace rules (`__rules__`),
  facts (`recallFacts`, including behavioral records and the exploration slot) and guardrails;
- **no writes** — no session insights at the end of the run, no `swarm-extract` during
  context compression (plain summarization is used instead), no episodic record from the
  `delegate_to_agent` automation action, no "routing feedback" for external agents.

What is *not* memory and still works with `off`: the agent's own system prompt and tools, and
the history of its conversation thread (below). The agent card that the registry syncs to
shared memory for routing (`agent:<slug>`) and its metrics are kept too — they carry no
conversation text.

The value lives in the `memory` column of `_v2_agent_registry`; rows created before the
column read as `shared`, i.e. behave as before.

### Conversation thread (`threadKey`)

`delegate_to_agent` runs are stateless unless the rule names a thread:

```json
{ "type": "delegate_to_agent", "agentSlug": "client-bot", "task": "{{_message}}", "threadKey": "tg:{{_chat_id}}" }
```

The key is a template resolved against the run environment. Past exchanges of the **same
agent and the same key** are passed to the model as conversation history (the last 6
completed exchanges, not older than 24 hours). There is no separate storage: the history *is*
the registry task journal — the question is `task`, the answer is `result_json`, the key is
the `thread_key` column of `_v2_agent_tasks`. Who shares a conversation is therefore decided
by the rule author: one chat — one key. A key longer than 200 characters is not truncated
(truncation would merge different threads) — the run proceeds without a thread and logs a
warning. External agents ignore the key.

### Restrictions

- Not visible in `list_agents` AI tool or A2A discovery endpoints
- `delegate_to_agent` runs them through the shared runner (issue #309); nested agent-to-agent
  calls are limited to depth 3
- `healthCheck()` returns healthy without making an HTTP request
- Secret rotation is not meaningful (secret is generated but unused in callbacks)

## Callback Authentication

Two options for the `/callback` endpoint:
1. **HMAC signature** (`X-Agent-Signature` header) — per-agent secret, verified via `service.verifyCallbackHmac`
2. **Service key** — `Bearer smk_...` fallback for backwards compatibility

## Task Flow (async mode)

1. Orchestrator calls `POST /agent-registry/:id/invoke`
2. Backend forwards to agent's `callback_url` with `taskId` and context
3. Agent processes asynchronously, then calls `POST /agent-registry/callback` with result
4. Backend resolves the pending task and returns result to orchestrator

## Dependencies

- `middleware/service-key-auth.js` — fallback auth for callbacks
- `modules/swarm-memory/` — metrics storage
- Event bus: none

## Write-back

When an async agent completes, `receiveCallback` checks the `source_object_id` and `write_back_field` columns on the task row. If both are set, the result is written back to the EAV object — the agent output is stored in the specified field of the source object.

## AI Tools

| Tool | Tier | Description |
|------|------|-------------|
| `list_agents` | TIER_LOW | List registered external agents and their capabilities |
| `get_agent` | TIER_LOW | Get agent details: endpoint, capabilities, metrics. Params: `id` (required) |
| `check_agent_health` | TIER_LOW | Ping external agent health endpoint. Params: `id` (required) |
| `get_agent_tasks` | TIER_LOW | Task history for an agent. Params: `id` (required), `limit`, `status` |
| `delegate_to_agent` | TIER_MEDIUM | Delegate a task to an external agent by slug. Params: `agentSlug` (required), `task` (required), `context` |

## Typed Tools (TD-015, ADR-040)

Внешний агент (`type: 'external'`, `protocol: 'integram'`) может приносить свои
инструменты: `tools` хранит массив определений `{name, description, parameters}`
(parameters — JSON Schema c `type: "object"`). У `type: 'internal'` колонка
по-прежнему хранит перечень ИМЁН (см. Internal Agents) — валидатор различает по
типу агента.

- **Discovery:** инструменты появляются в `GET /:db/ai/tools` под именами
  `ag_<slug>__<tool>` (MCP-клиенты получают их автоматически, `fetchTools`).
- **Вызов:** `POST /:db/ai/tool` с именем `ag_<slug>__<tool>` и `args`; платформа
  зовёт callback агента телом `{taskId, tool, args, context, workspace}`. В
  async-режиме результат приходит обычным callback'ом по `taskId`. Поля `task` и
  `tool` взаимоисключающи; `delegate_to_agent(slug, task)` остаётся задачным
  каналом и в tool-режим не смешивается.
- **Ограничения:** ≤ 50 инструментов на агента; имя `[a-z0-9_]{1,64}` без `__`;
  description ≤ 2000; `parameters.type` обязан быть `"object"`; a2a-агент typed
  tools носить не может. Регистрация/правка — admin (`POST/PATCH /agent-registry`),
  вызов — admin (паритет с `delegate_to_agent`).
- **Риск:** TIER_HIGH (внешний вызов); подтверждения HITL нет — заслон ролью,
  аудитом и портальными грантами (`allowDelete`).

## DDL Ownership

`ensureAgentRegistryTable()` in `service.js` owns the DDL for both `_v2_agent_registry` and `_v2_agent_tasks` (per ADR-013: module owns its own DDL). Bootstrap does not create these tables — they are lazily initialized on first access.

## DB Tables

- `_v2_agent_registry` (per-workspace) — agent definitions, callback URL, auth config, secret hash. `memory` (`shared` | `off`, default `shared`) — memory mode, see above
- `_v2_agent_tasks` (per-workspace) — task invocation history. Columns include `source_object_id` (EAV object that triggered the task), `write_back_field` (reqId to write result back to), `trace_id` (correlation ID for distributed tracing), `thread_key` (conversation thread of an internal agent; partial index `idx_agent_tasks_thread` on `(agent_id, thread_key, created_at DESC)`)
- `_v2_agent_leases` (per-workspace) — leases on dangerous operations (PM-351). Columns: `id` (identity), `db`, `lease_key` (UNIQUE, e.g. `object:42`), `holder_agent_id`, `acquired_at`, `expires_at`

## Leases (PM-351)

Coordination primitive so two agents never mutate the same object concurrently.

- **Table, not advisory locks.** `pg_try_advisory_lock` is session-bound, but `execSql`
  runs each query on any pooled connection — a lock taken on one connection is invisible
  to another and cannot be released from it. Instead, a single atomic statement arbitrates:
  `INSERT .. ON CONFLICT (lease_key) DO UPDATE .. WHERE <tbl>.expires_at <= NOW() RETURNING ..`
  — free key inserts, expired lease is stolen by the same statement (lazy expiry, no
  separate sweeper before acquire), a held key yields an empty `RETURNING` → `LEASE_HELD` 409.
  Two concurrent acquires cannot race: the unique index arbitrates inside one statement.
- **Clocks — DB `now()` only.** Both `expires_at` and the expiry check live in Postgres;
  clock skew between backend instances is irrelevant. `Date.now()` is banned in the lease
  code (guard test in `leases.test.js`).
- **TTL**: default 60s, min 1s, max 3600s. A holder that dies without `release` frees the
  key within TTL.
- **release** — `DELETE .. WHERE lease_key = ? AND holder_agent_id = ?`; `rowCount = 0` →
  `LEASE_HELD` (a foreign lease cannot be released).
- **Service API** (`agent-registry/service.js`): `acquireLease(pool, db, key, agentId, ttlSec)`,
  `releaseLease(pool, db, key, agentId)`, `isLeased(pool, db, key)` (informational only —
  any decision built on it reproduces the TOCTOU race that acquire eliminates),
  `withLease(pool, db, key, agentId, ttlSec, fn)` (release in `finally`; release failure
  is logged, never swallows fn's own error), `cleanupExpiredLeases(pool, db)` (background,
  300s grace, via `sideEffect` — an acquire never fails because of it).
- **Integration**: `update_object` in `tool-executor.js` wraps its write in `withLease`
  with key `object:<objId>`, holder = `toolCtx.agentId` (fallback `userId`). The lease is
  taken after the full pre-execution chain (permission/HITL/audit) and lives only for the
  write itself. A held lease surfaces as a 4xx envelope `{ok:false, error:{code:'LEASE_HELD'}}`
  on `POST /:db/ai/tool` and as `{error: <message>}` in chat tool results — never a 500.
- **Lazy DDL**: `ensureTables` creates the table via `ensureLeasesTable` (after the registry
  ensure, which owns `CREATE SCHEMA`).
- **No REST routes, deliberately**: an acquire endpoint would let a foreign process wedge
  keys indefinitely; the lease lives inside one tool-call lifecycle on the server.
- **Follow-ups**: lease on `bulk_update`/`pm_bulk_update` (key `bulk:<threadId>`), read-only
  `list_leases` inspection tool.

## A2A Agent Card Endpoints

### GET /:db/agent-registry/workspace-card

Returns an A2A-compatible Agent Card for the entire workspace. No authentication required.
Skills list **external agents only** — agents with `type: 'internal'` (debate/IC experts, which
have no callback URL) are filtered out of every publicly readable card.

### GET /:db/agent-registry/:slug/agent-card

Returns an A2A-compatible Agent Card for a specific registered agent by slug. No authentication required. Returns 404 if agent not found or if the agent is `type: 'internal'`.

Both endpoints follow A2A spec v0.3 card format: `{ name, description, version, url, capabilities, skills, authentication }`.
All cards declare `capabilities.streaming = false` — no SSE / `message/stream` transport is
implemented; the A2A endpoint answers with a single JSON-RPC response.

## A2A Support

Agents can be registered with `protocol: 'a2a'` (default: `'integram'`). When A2A protocol is set, `invoke()` sends A2A JSON-RPC `message/send` format instead of Integram format.

### Standard Discovery Endpoints (no auth)

- `GET /:db/agent-registry/workspace-card` — workspace A2A card with active external agents as skills (existing)
- `GET /:db/agent-registry/:slug/agent-card` — per-agent A2A card (existing)
- `GET /:db/.well-known/agent-card.json` — workspace A2A card (RFC 8615 standard path)
- `GET /:db/agent-registry/:slug/.well-known/agent-card.json` — per-agent A2A card (A2A standard path)

### A2A JSON-RPC Request Format

When protocol is `'a2a'`, `invoke()` sends to the agent's callback URL:

```json
{
  "jsonrpc": "2.0",
  "method": "message/send",
  "params": {
    "id": "task-uuid",
    "message": {
      "role": "user",
      "parts": [
        { "kind": "text", "text": "task description" }
      ]
    }
  }
}
```

Agent responds with A2A JSON-RPC result containing artifacts and messages.

### Incoming A2A Requests

Peer agents (A2A protocol) send requests to `POST /:db/portal/api/a2a`. See `backend/docs/modules/portal.md` for endpoint details.

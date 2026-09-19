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
- `type` is immutable — set at creation, cannot be changed via PATCH

### Restrictions

- Not visible in `list_agents` AI tool or A2A discovery endpoints
- `delegate_to_agent` returns a clear error for internal agents
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

## DDL Ownership

`ensureAgentRegistryTable()` in `service.js` owns the DDL for both `_v2_agent_registry` and `_v2_agent_tasks` (per ADR-013: module owns its own DDL). Bootstrap does not create these tables — they are lazily initialized on first access.

## DB Tables

- `_v2_agent_registry` (per-workspace) — agent definitions, callback URL, auth config, secret hash
- `_v2_agent_tasks` (per-workspace) — task invocation history. Columns include `source_object_id` (EAV object that triggered the task), `write_back_field` (reqId to write result back to), `trace_id` (correlation ID for distributed tracing)

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

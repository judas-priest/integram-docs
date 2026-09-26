# Module: workspace-tools

**Path:** `src/api/v2/modules/workspace-tools/`
**Files:** `router.js`, `service.js`, `sandbox.js`, `executor.js`, `bridges.js`
**Base URL:** `/api/v2/:db/workspace-tools`
**Auth:** JWT required. GET endpoints require `editor`. POST/PATCH/DELETE require `admin` + CSRF token.

## Purpose

Per-workspace custom tool registry. Each workspace defines its own JavaScript tools that the AI agent can call at runtime. Tools are stored in `_v2_workspace_tools`, executed inside isolated V8 sandboxes (isolated-vm), and exposed to the agent alongside platform tools via `GET /ai/tools`.

This eliminates hardcoding domain-specific logic into the platform core. A UAV analytics workspace might register `calc_flight_range`; a legal workspace might register `check_deadline`. Each workspace manages its own tool library independently.

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/workspace-tools` | editor | List all tools for the workspace. `?active=true` returns only active tools |
| POST | `/workspace-tools` | admin + CSRF | Register a new tool |
| POST | `/workspace-tools/import` | admin + CSRF | Bulk upsert a tool pack (array of tool definitions) |
| GET | `/workspace-tools/:id` | editor | Get a single tool by ID |
| PATCH | `/workspace-tools/:id` | admin + CSRF | Update an existing tool |
| DELETE | `/workspace-tools/:id` | admin + CSRF | Delete a tool |

## Tool Code Format

Tool code is a JavaScript function body that receives `args` (the tool arguments) and returns a serializable value. The code runs inside an IIFE in an isolated V8 context — `args` is injected via `globalThis.__args__` and cleaned up immediately.

`args` **is already declared for you** by the wrapper (`workspace-tools/sandbox.js`), in the same scope your code is spliced into. Declaring it again is a compile-time error, not a style issue:

```js
// Function body pattern — `args` is already in scope, just use it
// ... your logic ...
return result;
```

```js
// WRONG — the wrapper already did this one line above your code:
const args = __args__;   // SyntaxError: Identifier 'args' has already been declared
```

### Phase 1: Pure Functions (no external capabilities)

Pure math, string manipulation, or data transformation. No network, no DB, no LLM calls. These run in a reused Isolate with a compiled and cached Script.

**Example: `calc_multirotor`**

```js
// args: { payload_kg, battery_wh, efficiency_whpkm }
const range = args.battery_wh / (args.efficiency_whpkm * (1 + args.payload_kg / 10));
return { range_km: Math.round(range * 10) / 10 };
```

### Phase 2: Tools with Capabilities (`executeBridged`)

Tools that declare capabilities in their `capabilities` array gain access to bridge functions injected into the sandbox context via `executeBridged()` in `sandbox.js`. Bridge functions are created by `createBridge()` from `automations/isolated-runner.js` and injected as `ivm.Reference` objects. The sandbox runs `BOOTSTRAP_CODE` (also from `isolated-runner.js`) which provides the async unwrap layer (`__unwrap`) so user code can `await` bridge calls naturally.

After bootstrap, bridge references are cleaned from the global context to prevent the tool code from re-exposing them. Each bridged execution uses a **fresh isolate** (not cached) because bridge references are call-specific.

**Example: fetch + AI tool**

```js
// capabilities: ["fetch", "ai"]
// args: { url, question }
const html = await fetch(args.url).then(r => r.text());
const answer = await ai.ask(args.question, { context: html });
return { answer };
```

## Capabilities

Capabilities are declared in the tool's `capabilities` array. `executeBridged()` injects the corresponding bridge functions into the sandbox for tools that declare them.

| Capability | Bridge functions | Call budget | Auto risk tier |
|------------|-----------------|-------------|----------------|
| _(none)_ | — | no bridge is injected at all | TIER_LOW (1) |
| `query` | `query`, `queryPage`, `getRecord`, `kag_search`, `kag_relations`, `search_decisions`, `listDocuments`, `getDocument` | counter `query` | TIER_LOW (1) |
| `fetch` | `fetch`, `browse` | counter `fetch` | TIER_MEDIUM (2) |
| `ai` | `ai` | counter `ai` | TIER_MEDIUM (2) |
| `write` | `createRecord`, `updateRecord`, `moveChildren`, `createDocument` | counter `mutation` — **shared with `delete` and `kag_write`** | TIER_MEDIUM (2) |
| `kag_write` | `kag_import_entities`, `kag_import_relations` | counter `mutation` | TIER_MEDIUM (2) |
| `delete` | `deleteRecord` | counter `mutation` | **TIER_HIGH (3)** |
| `agent` | `delegateToAgent` | counter `agent` | TIER_MEDIUM (2) |
| `teamchat` | `sendTeamchatMessage` | counter `teamchat` | TIER_MEDIUM (2) |

Sources: `CAPABILITY_BRIDGES` and `BRIDGE_COUNTER` in `utils/sandbox-profiles.js`
(bridges and counters), `CAPABILITY_TIERS` in `workspace-tools/bridges.js` (tiers).

> **Budgets are enforced, and they are counted PER EXECUTION, not per minute.**
> `checkLimit(kind)` in `automations/isolated-runner.js` bumps the counter on
> every bridge call and **throws** — `Rate limit exceeded: max N <kind> calls per
> execution` — the moment it passes the budget. The budgets themselves are the
> `limits` of `SANDBOX_PROFILES.workspaceTool` (`utils/sandbox-profiles.js`) and
> are deliberately **not copied here**: a second copy of those numbers drifts
> from the first silently. Read them there.
>
> Two consequences of the counters being coarser than the capabilities:
> `write`, `delete` and `kag_write` all bill `mutation`, so a tool that deletes
> spends the same budget its writes do; the separate `delete` capability exists
> to raise the risk tier, not to hand out a second budget.
>
> A counter with **no** entry in the profile's `limits` is unavailable, not
> unlimited — the bridge refuses with `Sandbox counter "<kind>" is not available
> in sandbox profile "<name>"`. That is how a forbidden capability reads at
> runtime; a budget of 0 is never the mechanism.

## Sandbox Architecture

### Security Model

- Each tool executes in an isolated V8 Isolate via **isolated-vm**. There is no shared prototype chain between the host and the sandboxed code.
- `args` are passed via `new ivm.ExternalCopy(args).copyInto()` — a deep copy into the isolate heap; no references to host objects leak in.
- Bridge functions (Phase 2) are injected as `ivm.Reference` objects. The references are cleaned from the global after bootstrap so the tool code cannot re-expose them.
- Admin-only registration is defense-in-depth only. The actual security boundary is the V8 isolate.
- `onCatastrophicError` calls `process.exit(1)` to prevent corrupt-isolate execution.

### Performance Model

| Mode | Isolate | Context | Script |
|------|---------|---------|--------|
| Phase 1 (pure functions) | One per workspace, cached | Fresh per call | Compiled once, cached per tool |
| Phase 2 (capabilities) | Fresh per call | Fresh per call | Not cached (different bridges each time) |

The Phase 1 model amortizes V8 compilation cost across calls. Phase 2 uses fresh isolates because bridge references are call-specific and must not persist between invocations.

### Limits

| Resource | Limit |
|----------|-------|
| Memory per isolate | 128 MB |
| Execution timeout (default) | 5 seconds |
| Execution timeout (max, capabilities) | 30 seconds |
| Concurrent isolates | 20 (LRU eviction) |
| Idle TTL before dispose | 10 minutes |
| Code size | 50,000 characters |

## Caching

Two-level cache eliminates redundant DB queries and V8 compilation:

**Level 1 — Definition cache (`service.js`):**
`_defCache: Map<db, Map<toolName, {description, parameters, group, risk_tier, code, capabilities}>>`
— `capabilities` defaults to `[]` when the column is NULL.
Built on first `getCachedDefs()` call per workspace. Holds all active tool definitions for a workspace. `invalidateCache(db)` drops both the def cache and the DDL ready flag for a workspace.

**Level 2 — Sandbox cache (`sandbox.js`):**
`cache: Map<db, {isolate, scripts: Map<toolName, Script>, lastUsed}>`.
One entry per workspace. Compiled `Script` objects are stored inside the isolate entry and reused across calls for the same tool.

**Invalidation flow:**
1. Any write operation (create/update/delete/import) calls `invalidateCache(db)` in `service.js`.
2. `invalidateCache` drops the def cache entry and dynamically imports `sandbox.js` to call `disposeWorkspace(db)`, removing the compiled script cache and disposing the isolate.
3. Next `getCachedDefs()` call re-queries the DB and rebuilds the def map.
4. Next `executeSandboxed()` call recompiles the updated script into a fresh isolate entry.

## Integration Points

| Component | File | What happens |
|-----------|------|--------------|
| `executeTool` default case | `ai/agent/tool-executor.js` | When a tool name does not match any platform `TOOL_DEFS`, `executeWorkspaceTool(pool, db, toolName, args, toolCtx)` is called as the final fallback. Returns `null` if tool not found, causing the executor to return `UNKNOWN_TOOL`. |
| `POST /ai/tool` gate | `ai/router.js` | Before executing a tool by name (MCP path), the router checks `TOOL_DEFS`, then `AgentCatalog`, then `hasWorkspaceTool()`. Returns 400 if none match. |
| `GET /ai/tools` | `ai/router.js` | Calls `listWorkspaceToolDefs(pool, db)` and merges workspace tools into the platform tool catalog **inline in the handler** — a `seen` set dedupes by name, so a platform name always wins. There is no `mergeTools` function on this path. Workspace tools carry `_workspace: true` and `riskTier`. |
| `mergeTools` | `meta-kb/service.js` | A **different** merge, for a different reader: it builds one debate expert's tool list — `DEBATE_TOOLS` plus, for each name in `agent.tools`, the platform `TOOL_DEFS` entry or, failing that, the workspace def from `getCachedDefs`. Unknown names are dropped with `continue`. Not involved in `GET /ai/tools`. |
| `toolRiskTier` | `ai/agent/risk-tiers.js` | Management tools (`register_workspace_tool`, `update_workspace_tool`, `delete_workspace_tool`, `import_tool_pack`) are `TIER_HIGH`. `list_workspace_tools` is `TIER_LOW`. **Workspace tool names are not in `TOOL_TIERS` by construction** — they are created as data, not as code — so `toolRiskTier` answers `TIER_MEDIUM` for every one of them. Their declared `risk_tier` reaches the gates by a separate route, below. |
| Declared tier of a workspace tool | `workspace-tools/executor.js` → `ai/agent/runner.js`, `orchestrator.js`, `middleware.js` | `listWorkspaceToolDefs` ships `riskTier` with each def; `runner.js` and `orchestrator.js` fold those into the `ctx._wsTierByName` map, and three gates read it — `execOneTool`, the orchestrator's workspace-tool wrapper, and `portalGrantsMiddleware` in `PRE_CHECK_ORDER`. The declared tier applies **only** to names with no row of their own in `TOOL_TIERS`; it never overrides the platform classification. This was TD-047, closed 14.08.2026 — the in-code comment in `executor.js` still says the field is read by nobody, and that comment is stale. |
| HITL middleware | `ai/router.js` | High-risk management tools go through the standard HITL confirmation flow before execution. |
| Audit middleware | `ai/router.js` | All tool calls through `POST /ai/tool` are logged to `_v2_ai_audit_log` with `tool_name`, `risk_tier`, `user_id`, `session_id`. |

## AI Tools

Defined in `ai/agent/index.js` (group: `"workspace-tools"`), executed via `ai/agent/tool-executor.js`.

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `list_workspace_tools` | TIER_LOW | List custom tools for the current workspace. Optional `active: true` filter. |
| `register_workspace_tool` | TIER_HIGH | Register a new custom tool (name, description, parameters JSON schema, code, optional group_name, capabilities and `storeInCodespace` — when true the tool code is also committed to the codespace git repo `tools` for version control; default false). |
| `update_workspace_tool` | TIER_HIGH | Update an existing tool by ID. Accepts partial fields: name, description, parameters, code, active, capabilities, group_name, risk_tier. |
| `delete_workspace_tool` | TIER_HIGH | Delete a custom tool by ID. Invalidates cache. |
| `import_tool_pack` | TIER_HIGH | Bulk upsert an array of tool definitions. Returns `{imported, updated, skipped, errors}` — `updated` lists existing tools overwritten in place, and `skipped` is a deprecated alias of the very same array (nothing is ever silently skipped: a name collision always becomes an UPDATE). |

## DB Tables

### `_v2_workspace_tools` (per-workspace)

Created lazily by `ensureTables(pool, db)` on first access. Keyed by workspace schema (`wsTable(db, '_v2_workspace_tools')`).

| Column | Type | Description |
|--------|------|-------------|
| `id` | `BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY` | Auto-increment primary key |
| `name` | `VARCHAR(64) NOT NULL UNIQUE` | Snake_case tool name. Must match `/^[a-z][a-z0-9_]{1,62}$/`. Cannot shadow platform `TOOL_DEFS`. |
| `group_name` | `VARCHAR(32) NOT NULL DEFAULT 'custom'` | Logical grouping label for UI display |
| `description` | `TEXT NOT NULL DEFAULT ''` | Tool description shown to the LLM (max 2000 chars) |
| `parameters` | `JSONB NOT NULL DEFAULT '{}'` | JSON Schema for tool parameters |
| `code` | `TEXT NOT NULL DEFAULT ''` | JavaScript function body (max 50,000 chars) |
| `capabilities` | `TEXT[] NOT NULL DEFAULT '{}'` | Array of capability strings required by the tool |
| `risk_tier` | `SMALLINT NOT NULL DEFAULT 1` | Risk tier for HITL: 1=LOW, 2=MEDIUM, 3=HIGH |
| `active` | `BOOLEAN NOT NULL DEFAULT TRUE` | Inactive tools are excluded from the def cache and agent tool list |
| `source` | `VARCHAR(16) NOT NULL DEFAULT 'inline'` | Where the code lives: `inline` (in this row) or `codespace` (also committed to git). Added by `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` in `ensureTables`, so old installations get it. Surfaced by `mapRow` as `source`. |
| `codespace_path` | `VARCHAR(255) NULL` | For `source = 'codespace'`: `<repoSlug>:<branch>:<file>`, e.g. `tools:main:calc_flight_range.js`. NULL otherwise. Surfaced by `mapRow` as `codespacePath`. |
| `created_by` | `VARCHAR(128) NULL` | User ID or username of the creator |
| `created_at` | `TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP` | Creation timestamp |
| `updated_at` | `TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP` | Last modification timestamp (updated automatically on PATCH) |

`source` and `codespace_path` are written together by one path only —
`createFromCodespace` (`service.js`). It creates the `tools` repo if absent,
commits `<name>.js` to it, and only then calls `createTool` with
`source: 'codespace'`. The admin/owner check runs **before** the git write, so a
refusal cannot leave a committed file with no row behind it. Every other
creation path leaves the default `inline` / NULL.

## Tool Pack Format

`POST /workspace-tools/import` accepts a JSON body with `tools` array and optional `pack` label. Existing tools (matched by `name`) are updated in-place (upsert). Returns `{imported, updated, skipped, errors}` — `updated` lists existing tools that were overwritten; `skipped` is a deprecated alias of `updated` (nothing is ever silently skipped).

```json
{
  "pack": "uav-analytics-v1",
  "tools": [
    {
      "name": "calc_flight_range",
      "group_name": "uav",
      "description": "Calculate multirotor flight range from battery and payload parameters.",
      "parameters": {
        "type": "object",
        "properties": {
          "payload_kg":        { "type": "number", "description": "Payload weight in kilograms" },
          "battery_wh":        { "type": "number", "description": "Battery capacity in watt-hours" },
          "efficiency_whpkm":  { "type": "number", "description": "Energy consumption in Wh/km" }
        },
        "required": ["payload_kg", "battery_wh", "efficiency_whpkm"]
      },
      "code": "const range = args.battery_wh / (args.efficiency_whpkm * (1 + args.payload_kg / 10));\nreturn { range_km: Math.round(range * 10) / 10 };",
      "capabilities": [],
      "risk_tier": 1
    },
    {
      "name": "estimate_mission_cost",
      "group_name": "uav",
      "description": "Estimate mission cost given flight parameters and cost per kWh.",
      "parameters": {
        "type": "object",
        "properties": {
          "distance_km":  { "type": "number" },
          "efficiency_whpkm": { "type": "number" },
          "cost_per_kwh": { "type": "number" }
        },
        "required": ["distance_km", "efficiency_whpkm", "cost_per_kwh"]
      },
      "code": "const energy_kwh = (args.distance_km * args.efficiency_whpkm) / 1000;\nreturn { cost_usd: Math.round(energy_kwh * args.cost_per_kwh * 100) / 100 };",
      "capabilities": [],
      "risk_tier": 1
    }
  ]
}
```

## Server Functions (cross-reference)

Codespace **server functions** (`api/*.js` in codespace repos) are a SEPARATE execution environment from workspace tools. They share the same bridge factory (`createBridge` from `automations/isolated-runner.js`) and bridge selection (`selectBridges` from `workspace-tools/bridges.js`), but differ in isolation model:

- **Different isolates:** server functions always use fresh isolates (never cached), workspace tools cache isolates per workspace (Phase 1).
- **Different limits:** server functions have 64 MB memory / 15s timeout / stricter per-execution rate limits. Workspace tools have 128 MB / 30s.
- **Different OOM policy:** server functions dispose gracefully. Workspace tools call `process.exit(1)`.
- **Different callers:** server functions are called from portal custom code (`api.callFunction`). Workspace tools are called by AI agents and MCP.

Full reference: [`docs/guides/codespace-server-functions.md`](../../../docs/guides/codespace-server-functions.md)

## Dependencies

- **`isolated-vm`** (npm) — V8 isolate creation, script compilation, ExternalCopy for args transfer. Requires Node 24; fails on Node 26.
- **`automations/isolated-runner.js`** — separate isolated execution path for automation scripts. Workspace tools use their own sandbox pipeline (`sandbox.js`), not the automation runner.
- **`ai/agent/index.js`** (`TOOL_DEFS`) — management tools (`list_workspace_tools`, `register_workspace_tool`, etc.) are declared here. The `validate()` check in `service.js` imports `TOOL_DEFS` to prevent workspace tools from shadowing platform names.

## Workflow Example

```bash
# 1. Import a tool pack into the workspace
curl -X POST https://api.example.com/api/v2/mydb/workspace-tools/import \
  -H "Authorization: Bearer $JWT" \
  -H "X-CSRF-Token: $CSRF" \
  -H "Content-Type: application/json" \
  -d '{"pack":"uav-v1","tools":[{"name":"calc_flight_range","description":"...","parameters":{},"code":"return {range_km: args.battery_wh / args.efficiency_whpkm};","capabilities":[]}]}'
# Response: { "ok": true, "data": { "imported": ["calc_flight_range"], "updated": [], "skipped": [], "errors": [] } }

# 2. Verify the tool appears in the AI tool catalog
curl https://api.example.com/api/v2/mydb/ai/tools \
  -H "Authorization: Bearer $JWT"
# Response includes: { "name": "calc_flight_range", "_workspace": true, "group": "uav", ... }

# 3. Configure an agent to use the workspace tool
#    (agents with "workspace" in their tools list will see workspace tools automatically)
#    — no additional config needed; tool-executor default case handles it

# 4. Debate agent calls workspace tool during a Meta KB debate
#    → Agent LLM emits tool_call: { name: "calc_flight_range", args: { payload_kg: 2, battery_wh: 500, efficiency_whpkm: 8 } }
#    → tool-executor.js default case → executeWorkspaceTool(pool, db, "calc_flight_range", args)
#    → sandbox.js: reuses cached Isolate, creates fresh Context, runs compiled Script
#    → returns: '{"range_km": 55.6}'

# 5. MCP client calls workspace tool directly
curl -X POST https://api.example.com/api/v2/mydb/ai/tool \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"name":"calc_flight_range","args":{"payload_kg":2,"battery_wh":500,"efficiency_whpkm":8}}'
# → router checks hasWorkspaceTool() → passes HITL/audit middleware → executes via tool-executor default case
# Response: { "ok": true, "data": { "range_km": 55.6 } }
```

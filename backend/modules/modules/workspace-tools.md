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

```js
// Function body pattern — args is available, return your result
const args = __args__;
// ... your logic ...
return result;
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

| Capability | Bridge functions | Rate limit | Auto risk tier |
|------------|-----------------|------------|----------------|
| _(none)_ | — | none | TIER_LOW (1) |
| `query` | `query`, `getRecord`, `kag_search`, `search_decisions` | 100 req/min | TIER_LOW (1) |
| `fetch` | `fetch`, `browse` | 30 req/min | TIER_MEDIUM (2) |
| `ai` | `ai` | 20 req/min | TIER_MEDIUM (2) |
| `write` | `createRecord`, `updateRecord` | 20 req/min | TIER_MEDIUM (2) |
| `delete` | `deleteRecord` | 10 req/min | TIER_MEDIUM (2) |
| `agent` | `delegateToAgent` | 5 req/min | TIER_MEDIUM (2) |
| `teamchat` | `sendTeamchatMessage` | 20 req/min | TIER_MEDIUM (2) |

> **Note:** Rate limits are planned but not yet enforced.

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
`_defCache: Map<db, Map<toolName, {description, parameters, group, risk_tier, code}>>`.
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
| `GET /ai/tools` | `ai/router.js` | Calls `listWorkspaceToolDefs(pool, db)` and merges workspace tools into the platform tool catalog, deduplicating by name. Workspace tools carry `_workspace: true`. |
| `mergeTools` | `ai/router.js` | Workspace tool defs are appended to the flat platform tool list before being returned to the MCP client. |
| `toolRiskTier` | `ai/agent/risk-tiers.js` | Management tools (`register_workspace_tool`, `update_workspace_tool`, `delete_workspace_tool`, `import_tool_pack`) are `TIER_HIGH`. `list_workspace_tools` is `TIER_LOW`. Workspace-registered tools inherit the `risk_tier` stored in `_v2_workspace_tools` (default: 1 = TIER_LOW). |
| HITL middleware | `ai/router.js` | High-risk management tools go through the standard HITL confirmation flow before execution. |
| Audit middleware | `ai/router.js` | All tool calls through `POST /ai/tool` are logged to `_v2_ai_audit_log` with `tool_name`, `risk_tier`, `user_id`, `session_id`. |

## AI Tools

Defined in `ai/agent/index.js` (group: `"workspace-tools"`), executed via `ai/agent/tool-executor.js`.

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `list_workspace_tools` | TIER_LOW | List custom tools for the current workspace. Optional `active: true` filter. |
| `register_workspace_tool` | TIER_HIGH | Register a new custom tool (name, description, parameters JSON schema, code, optional group_name and capabilities). |
| `update_workspace_tool` | TIER_HIGH | Update an existing tool by ID. Accepts partial fields: name, description, parameters, code, active, capabilities, group_name, risk_tier. |
| `delete_workspace_tool` | TIER_HIGH | Delete a custom tool by ID. Invalidates cache. |
| `import_tool_pack` | TIER_HIGH | Bulk upsert an array of tool definitions. Returns `{imported, skipped, errors}`. Existing tools (by name) are updated in-place. |

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
| `created_by` | `VARCHAR(128) NULL` | User ID or username of the creator |
| `created_at` | `TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP` | Creation timestamp |
| `updated_at` | `TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP` | Last modification timestamp (updated automatically on PATCH) |

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
# Response: { "ok": true, "data": { "imported": ["calc_flight_range"], "skipped": [], "errors": [] } }

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

# Module: audit-export

**Path:** `src/api/v2/modules/audit-export/`
**Files:** `router.js` (no service.js — all logic inline)
**Base URL:** `/api/v2/:db/audit/...`
**Auth:** JWT + `admin` role required.

## Purpose

Unified read-only endpoint to query and export audit logs from all four audit sources: EAV object changes, schema changes, report structure changes, and AI agent tool calls. Merges and sorts results by timestamp. This is the endpoint to use when a regulator asks for "the whole audit trail" — it must stay a superset of the per-source endpoints.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/audit` | Query audit log (JSON, paginated) |
| GET | `/audit/export` | Bulk export (JSON or CSV file download) |

## Query Parameters (both endpoints)

| Param | Description |
|-------|-------------|
| `type` | Source filter: `objects` \| `schema` \| `reports` \| `ai` \| `all` (default: `all`) |
| `actor` | Filter by username |
| `dateFrom` | ISO datetime start |
| `dateTo` | ISO datetime end |
| `action` | Filter by action (partial match: `create`, `update`, `delete`) |
| `objectId` | Filter by EAV object ID (objects source only) |
| `typeId` | Filter by table/type ID (schema source only) |
| `reportId` | Filter by report ID (reports source only) |
| `toolName` | Filter by exact AI tool name (ai source only, `GET /audit` only) |
| `limit` | Max rows (GET /audit: default 100, max 1000; export: default 5000, max 50000) |
| `offset` | Pagination offset (GET /audit only) |

## Response Format

Each entry in the merged result has a normalized shape:

```json
{
  "source": "objects",
  "id": 1,
  "resourceId": 456,
  "resourceType": "object",
  "fieldName": "Status",
  "action": "update",
  "oldValue": "Draft",
  "newValue": "Active",
  "actor": "john@example.com",
  "timestamp": "2025-01-01T12:00:00Z"
}
```

- `source`: `objects` | `schema` | `reports` | `ai`
- `resourceType`: `object` | `table` | `report` | `ai_tool_call`

For `source: "ai"` the fields carry: `action` = tool name, `target` = agent id,
`resourceId` = session id, `details` = `{ riskTier, args, result, success, hitlRequired,
hitlApproved, lowConfidence, highRiskAutomation, durationMs }`. In the CSV export the same
row uses `oldValue` = tool arguments and `newValue` = the `details` object.

## Export

`GET /audit/export?format=csv` returns a semicolon-delimited CSV with BOM for Excel compatibility. `format=json` returns JSONL-style JSON.

## Graceful Degradation

Each source table is queried with `safeQuery()` — if the table doesn't exist yet (lazy-init not triggered), returns empty results instead of failing.

## DB Tables

Per-workspace, lazy-init (queried on `req.pool`):

- `_v2_audit_log` — EAV field-level changes (objects)
- `_v2_schema_audit` — table/column DDL changes
- `_v2_report_audit` — report structure changes

Global, on the system pool (queried via `getPool()`, scoped by the `workspace_db` column):

- `_v2_ai_audit_log` — every AI agent tool call (written by `ai/agent/middleware.js`,
  also exposed per-source at `GET /:db/ai/audit-log`). A human caller is a numeric
  `_v2_users.id` in `user_id`, so the username is `LEFT JOIN`ed in to keep the `actor`
  field consistent with the other three sources. Callers with no user row — A2A clients,
  delegating agents, users identified only by login — are stored verbatim in `actor_ref`.
  Both the `actor` field and the `actor=` filter resolve through
  `COALESCE(u.username, a.actor_ref)`.

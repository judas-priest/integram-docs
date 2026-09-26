# Module: computed-reqs

**Path:** `src/api/v2/modules/computed-reqs/`
**Files:** `router.js` (thin wrapper — all logic in `utils/computed-reqs.js`)
**Base URL:** `/api/v2/:db/schema/computed` (old `/api/v2/:db/computed-reqs/...` redirects 301)
**Auth:** JWT required for all endpoints.

## Purpose

REST interface for managing computed column definitions: LOOKUP, ROLLUP, and FORMULA. Computed columns are virtual — they have no stored EAV row value; their value is calculated at read time from other columns.

## Endpoints

Routes are in `schema/router.js` under `/api/v2/:db/schema/computed`.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/schema/computed?typeId=N` | List computed column definitions for a table |
| POST | `/schema/computed` | Create a computed column definition |
| PATCH | `/schema/computed/:id` | Update a computed column definition |
| DELETE | `/schema/computed/:id` | Delete a computed column definition |

## Computed Column Types

### LOOKUP
Pulls a value from a related record through a reference column.

```json
{
  "kind": "LOOKUP",
  "typeId": 123,
  "name": "Client Name",
  "config": {
    "sourceReqId": 45,
    "targetColId": 67
  }
}
```
- `sourceReqId`: the reference column in the current table
- `targetColId`: the column to pull from the referenced record

### ROLLUP
Aggregates values from child records.

```json
{
  "kind": "ROLLUP",
  "typeId": 123,
  "name": "Total Items",
  "config": {
    "linkReqId": 78,
    "targetColId": 89,
    "fn": "SUM"
  }
}
```
- `linkReqId`: the column in the child table that references this (parent) table
- `fn`: `SUM` | `COUNT` | `AVG` | `MIN` | `MAX` | `CONCAT`

#### CONCAT rollup

Concatenates text values from linked records. Supports two-hop resolution and optional variant/quantity display.

```json
{
  "kind": "ROLLUP",
  "typeId": 123,
  "name": "Assembly Components",
  "config": {
    "linkReqId": 78,
    "targetColId": 89,
    "fn": "CONCAT",
    "resolveReqId": 200
  }
}
```

- `resolveReqId` — two-hop resolution: child record → ref field (`targetColId`) → referenced record → fetch value from `resolveReqId` field

#### Parent-child ROLLUP (useUp)

Alternative config for aggregating via parent-child relationship instead of a link column:

```json
{
  "fn": "CONCAT",
  "useUp": true,
  "childTypeId": 300,
  "targetColId": 89,
  "resolveReqId": 200
}
```

- `useUp: true` — aggregate child records (where `up = parentId`) instead of following a link column
- `childTypeId` — type ID of the child table to aggregate from

### FORMULA
Calculates a value from an expression.

```json
{
  "kind": "FORMULA",
  "typeId": 123,
  "name": "Margin",
  "config": {
    "expr": "[Price] - [Cost]",
    "vars": { "Price": 42, "Cost": 43 }
  }
}
```

## Convenience Fields

The POST endpoint accepts individual fields (`sourceReqId`, `targetColId`, `linkReqId`, `rollupFn`, `formula`) and assembles the `config` object automatically if `config` is not provided directly.

## Core Logic

All computation logic lives in `utils/computed-reqs.js`:
- Dependency graph built at read time
- Topological sort to resolve chains (LOOKUP of a ROLLUP, etc.)
- Formula evaluation via `utils/formula-engine.js`

## AI Tools

| Tool | Tier | Description |
|------|------|-------------|
| `list_computed` | TIER_LOW | List computed columns for a table. Params: `typeId` (required) |
| `create_computed` | TIER_MEDIUM | Create a computed column (requires HITL). Params: `typeId`, `kind`, `alias`, `config` (all required) |
| `update_computed` | TIER_MEDIUM | Update a computed column (requires HITL). Params: `reqId` (required), `alias`, `config` |
| `delete_computed` | TIER_HIGH | Delete a computed column (requires HITL). Params: `reqId` (required), `reason` |

## DB Tables

- `_v2_computed_reqs` (per-workspace, lazy-init) — `id`, `type_id`, `name` (VARCHAR 255), `kind`, `config` (JSONB), `created_at`

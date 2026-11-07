# Module: lookups

**Path:** `src/api/v2/modules/lookups/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/:db/lookups/...`
**Auth:** JWT required for all endpoints.

## Purpose

Provides dropdown option lists for the frontend. Two types of lookups:
1. **Type lookup** (`GET /lookups/:typeId`) — returns objects from a type (EAV lookup table) for single-select/multi-select dropdowns
2. **Ref lookup** (`GET /lookups/:refId/refs`) — returns valid values for a reference column

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/lookups/:typeId` | Get lookup values for a type. Supports `q` (search), `restrict` (comma-separated IDs to restrict to), `limit` (max 1000, default 80) |
| GET | `/lookups/:refId/refs` | Get reference column options. `refId` is the column (req) ID. Supports `q`, `r` (restrict), `limit` |

## Query Parameters

- `q` — fuzzy text search (trigram GIN index)
- `restrict` — comma-separated IDs; return only these records (used to resolve stored ref IDs)
- `limit` — max results (default 80, max 1000)
- `r` — same as `restrict` for the refs endpoint

## How It Works

Lookups query the EAV `_value` column (the `val` field on root records where `t = up = typeId`). The service also respects row-level permissions — records the user cannot see are excluded.

This module is called heavily by the frontend whenever a dropdown is rendered or a user types in a reference field.

## DB Tables

- EAV workspace tables (`"db"."db"`) — queries root records by `t` (typeId)
- `"db"."_v2_reqs"` — for ref column resolution

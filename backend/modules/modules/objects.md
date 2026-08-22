# Module: objects

**Path:** `src/api/v2/modules/objects/`
**Files:** `router.js`, `service.js`, `bulk.js`, `history.js`, `csv-import.js`, `trash.js`, `schema.js`
**Base URL:** `/api/v2/:db/objects/...`
**Auth:** JWT required. Write operations require at least `editor` role or grant.

## Purpose

Core CRUD for EAV objects (records). The largest and most central backend module. Handles single-record operations, bulk operations, history/audit, CSV import, trash (soft delete), record sharing, and search.

## Endpoints

### Single object CRUD

| Method | Path | Description |
|--------|------|-------------|
| GET | `/objects` | List objects for a type. `?typeId=`, `?parentId=`, filters, sort, pagination |
| POST | `/objects` | Create object. Supports `X-Idempotency-Key` header. |
| GET | `/objects/:id` | Get object with all requisites + computed columns |
| PATCH | `/objects/:id` | Partial update (only specified fields) |
| DELETE | `/objects/:id` | Soft-delete to trash. Body param `cascade` (boolean, default false) also deletes referencing objects. |
| POST | `/objects/:id/move` | Move to different parent |
| POST | `/objects/:id/reorder` | Change sort order |
| POST | `/objects/:id/duplicate` | Duplicate (copy) object |
| POST | `/objects/:id/change-id` | Change object numeric ID (admin only) |

### Aggregation and analytics

| Method | Path | Description |
|--------|------|-------------|
| GET | `/objects/count` | Count objects matching filters |
| GET | `/objects/aggregate` | Aggregate column values (SUM, COUNT, etc.) |
| GET | `/objects/pivot` | Pivot table (rowField × colField) |
| GET | `/objects/grouped` | Group headers + counts (SSRM two-query pattern) |

### Bulk operations

Mounted separately at `/:db/objects/bulk/`:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/objects/bulk/create` | Create multiple objects |
| POST | `/objects/bulk/update` | Update multiple objects |
| POST | `/objects/bulk/delete` | Delete multiple objects |

### History and audit

Mounted at `/:db/objects/:objId/history` (separate sub-router `history.js`):

| Method | Path | Description |
|--------|------|-------------|
| GET | `/objects/:id/history` | Field-level audit trail for an object |
| POST | `/objects/:id/history/rollback` | Restore object to a previous audit snapshot |

### Import

| Method | Path | Description |
|--------|------|-------------|
| POST | `/objects/import` | CSV/XLSX import into a type (multipart or JSON body) |
| POST | `/objects/import/preview` | Preview file headers + sample rows for mapping UI |
| POST | `/objects/import/create-table` | Create new table from file and import all data (admin) |
| POST | `/objects/import/create-all-sheets` | Import all XLSX sheets as separate tables (admin) |

### PDF / DOCX rendering

| Method | Path | Description |
|--------|------|-------------|
| GET | `/objects/:id/pdf?template=DOC_ID` | Generate PDF from document template |
| POST | `/objects/:id/docx` | Render DOCX template via Carbone.io (multipart upload) |

### Record sharing

| Method | Path | Description |
|--------|------|-------------|
| GET | `/objects/:id/share` | Get current share token |
| POST | `/objects/:id/share` | Generate public share token |
| DELETE | `/objects/:id/share` | Revoke share token |

### Backlinks

| Method | Path | Description |
|--------|------|-------------|
| GET | `/objects/:id/backlinks` | Documents that reference this record |

### Trash

| Method | Path | Description |
|--------|------|-------------|
| GET | `/objects/trash` | List deleted objects (`?typeId=`, `?search=`) |
| GET | `/objects/trash/:id` | Get single trash item with requisites |
| POST | `/objects/trash/:id/restore` | Restore from trash |

## Query Parameters (List)

| Param | Description |
|-------|-------------|
| `typeId` | Required. EAV type to list |
| `parentId` | Filter by parent object ID |
| `page` / `pageSize` | Pagination (default pageSize: 50) |
| `sort` | Column alias + `asc`/`desc` |
| `q` | Full-text search query |
| `filter[field][op]` | Filter DSL via `parseFilterDsl` (e.g. `filter[Status][eq]=Active`, `filter[Price][lt]=1000`) |

## EAV Data Model

```
All records (both root objects and their requisites/field values) are stored
as rows in a single EAV table: "db"."db" (returned by eavTable()).

Root object row:
  id   — numeric primary key
  t    — typeId (which table/type this record belongs to)
  up   — parentId (workspace root=1 for top-level objects; objectId for child records)
  val  — _value / display name
  ord  — sort order

Field value row (requisite):
  id   — numeric PK
  t    — column definition ID (reqId)
  up   — objectId (the object this field belongs to)
  val  — stored value (text)
  ord  — sort order
```

## Auto-fields

On create: `created_at`, `created_by` (username) written to `_v2_autofields`.
On update: `updated_at`, `updated_by` updated.

## EAV Ref Write — Stale Row Cleanup

When writing a ref field using the **inverted pattern** (`t=refObjId, val=colDefId`), after upsert `saveRequisites` also deletes any stale **direct-pattern** row (`t=colDefId, val=refObjId`) for the same object+column. Without this, both patterns coexist → API returns an array instead of a scalar → Kanban/UI breaks.

Applied in three branches:
- Scalar ref (existing row update)
- Scalar ref (new row insert, when old direct-pattern row may exist)
- Multiselect ref (full replace: delete all inverted rows, then delete stale direct rows)

## Computed Columns

Resolved at read time by `utils/computed-reqs.js`. Topological sort handles dependency chains. Always included in the response — computed columns are not optional.

## Row-Level Permissions

Checked via `utils/row-permissions.js`. Rules: `OWNER_ONLY`, `FILTER` (SQL WHERE clause with user attributes), `DENY_ALL`.

## Event Bus Emissions

- `object.created` — on new record
- `object.updated` — on field change (with old/new values)
- `object.deleted` — on soft or hard delete
- `object.moved` — on parent change

## CSV Import (`csv-import.js`)

Maps CSV columns to EAV requisites by header name. Creates objects in batch. Reports created/skipped/error counts. Supports reference column lookup by value.

## DB Tables

- `"db"."db"` — single EAV table for all objects and their field values (eavTable())
- `_v2_autofields` (per-workspace, lazy-init) — created_at/by, updated_at/by
- `_v2_audit_log` (per-workspace, lazy-init) — field-level change history
- `_v2_trash` (per-workspace, lazy-init) — soft-deleted objects
- `_v2_record_share_tokens` (global) — public share tokens

## AI Tools

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `list_objects` | TIER_LOW | List objects by type with filters, sort, pagination (`hasMore`, `_truncated`, `_hint`), and optional summary aggregation |
| `get_object` | TIER_LOW | Get single object with all requisites |
| `create_object` | TIER_MEDIUM | Create object with field values |
| `update_object` | TIER_MEDIUM | Partial update of object fields |
| `delete_object` | TIER_HIGH | Delete object (soft-delete to trash) |
| `bulk_create` | TIER_MEDIUM | Create multiple objects in one call |
| `bulk_delete` | TIER_HIGH | Delete multiple objects by ID list |
| `aggregate_objects` | TIER_LOW | Aggregate column values (SUM, AVG, MIN, MAX, COUNT) |
| `group_objects` | TIER_LOW | Group objects by a column with counts |
| `pivot_objects` | TIER_LOW | Pivot table: rowField × colField with aggregation |
| `get_object_backlinks` | TIER_LOW | Find objects referencing a given record |
| `get_object_history` | TIER_LOW | Field-level audit trail for an object |
| `rollback_object` | TIER_HIGH | Restore object to a previous audit snapshot |
| `bulk_update` | TIER_HIGH | Bulk update multiple objects (HITL required) |
| `restore_from_trash` | TIER_MEDIUM | Restore a deleted object from trash |
| `duplicate_object` | TIER_MEDIUM | Duplicate (copy) an existing object |

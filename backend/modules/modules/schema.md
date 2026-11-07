# Module: schema

**Path:** `src/api/v2/modules/schema/`
**Files:** `router.js`, `service.js`, `schema.js`, `flat-views.js`, `schema-audit.js`, `schema-snapshots.js`, `type-meta.js`, `data-snapshots.js`
**Base URL:** `/api/v2/:db/schema/...`
**Auth:** JWT required. DDL operations require `admin` role or editor with DDL write grant (`grants[0] === 'WRITE'`).

## Purpose

DDL operations: manage EAV types (tables) and requisites (columns). Handles type/column CRUD, computed columns, column ordering, schema audit log, schema snapshots, type metadata (icons, colors), and flat view generation.

## Endpoints

### Types (tables)

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/schema` | any | List all types in workspace (`?include=`, `?q=`, `?sort=`) |
| POST | `/schema` | admin | Create type (`name`, `baseType` (number, default 3), `unique` (boolean), `icon`). Supports `X-Idempotency-Key`. |
| GET | `/schema/:typeId` | any | Get type with columns |
| PATCH | `/schema/:typeId` | admin | Update type `name`, `baseType`, `unique`, `icon`, `valueColumnName` |
| DELETE | `/schema/:typeId` | admin | Delete type (cascades objects) |

### Columns (requisites)

| Method | Path | Role | Description |
|--------|------|------|-------------|
| POST | `/schema/:typeId/columns` | admin | Add column to type. Supports `X-Idempotency-Key`. |
| PATCH | `/schema/columns/:reqId` | admin | Update column (name, type, options, required, unique) |
| DELETE | `/schema/columns/:reqId` | admin | Delete column (cascades values) |
| POST | `/schema/columns/:reqId/reorder` | admin | Reorder column |
| POST | `/schema/columns/:reqId/convert-to-ref` | admin | Convert text column to reference (creates lookup table) |
| GET | `/schema/columns/batch?typeIds=1,2` | any | Batch get columns for multiple types |

### Column validation rules

| Method | Path | Description |
|--------|------|-------------|
| GET | `/schema/columns/validation?ids=1,2` | Batch get validation rules for multiple columns |
| GET | `/schema/columns/:reqId/validation` | Get validation rules for a column |
| PUT | `/schema/columns/:reqId/validation` | Set validation rules for a column |

### Computed columns

| Method | Path | Description |
|--------|------|-------------|
| GET | `/schema/computed?typeId=` | List computed column defs |
| POST | `/schema/computed` | Create computed column |
| PATCH | `/schema/computed/:id` | Update computed column |
| DELETE | `/schema/computed/:id` | Delete computed column |

### Schema audit and snapshots

| Method | Path | Description |
|--------|------|-------------|
| GET | `/schema/:typeId/history` | Schema audit history for a type |
| GET | `/schema/:typeId/snapshots` | List schema snapshots for a type |
| GET | `/schema/:typeId/snapshots/:snapshotId` | Get full snapshot |

### Schema graph and utilities

| Method | Path | Description |
|--------|------|-------------|
| GET | `/schema/graph` | Schema graph (types as nodes, reference columns as edges) |
| GET | `/schema/:typeId/backlinks` | Columns in other types that reference this type (for ROLLUP UI) |
| POST | `/schema/flat-views/rebuild` | Rebuild all flat VIEWs for workspace (admin) |
| PATCH | `/:typeId/visibility` | editor | Toggle table visibility (`hidden: true/false`). Sets `hidden` flag in type meta. |

## Column Types

`text`, `memo`, `number`, `date`, `datetime`, `bool`, `file`, `pwd`, `uuid`, `url`, `duration`, `currency`, `percent`, `rating`, `status`, `choice`, `ai_button`, `collaborator`

Plus ref columns (reference to another type) configured via `refTypeId`. `collaborator` (type 1010) is a user picker over workspace members — see `objects.md`.

## Flat Views (`flat-views.js`)

On every schema change (`schema.type.*` or `schema.column.*` events), the flat view for the affected type is regenerated. A flat view is a PostgreSQL VIEW that joins the EAV root table with all requisites, pivoting each column into its own `MAX(CASE …)` expression. Ограничения (см. `utils/eav-ref.js:119-121`): ref-колонки во вью всегда NULL (хранение ссылки перевёрнуто — внешний BI должен брать их инвертированным паттерном, как `pivotValueSql`); DATETIME отдаётся как `CAST(unix-секунды AS TIMESTAMP)` — даты 1970-х, сортировка дат идёт через `dateSortSql`, не через вью. Движок отчётов платформы `_flat_*` НЕ читает (вью — для внешнего BI; пересборка на `schema.type.*`/`schema.column.*`).

## Schema Audit (`schema-audit.js`)

Every DDL operation (type/column create, update, delete, reorder) writes a row to `_v2_schema_audit` with actor, action, target, and details JSONB. Accessible via `GET /schema/:typeId/history`.

## Snapshots (`schema-snapshots.js`)

Captures the full schema (all types + columns) as JSON per type. Accessible via `GET /schema/:typeId/snapshots`. Restore replays the captured DDL. Useful for rolling back accidental schema changes.

## Schema Journal & Catch-Up (PM-358)

`_v2_schema_audit` is the schema **version journal**: workspace version = `MAX(id)`. `column.add` entries carry `size`, `unique` and `refName` (ref target NAME, not source-local id) — enough to replay the operation on another workspace via the schema API.

`POST /schema/journal/catch-up` (admin, body `{sourceDb, upTo?}`) replays the source workspace's journal onto this workspace up to `upTo` (default: source version). `createType`/`addColumn` are reused, ids are remapped by name. Safety guards (each logged in `skipped` with a reason):

- column.add with a numeric `type` that is a ref but has no `refName` → skipped (pre-enrichment legacy entry; its effect is already in a carried clone's schema);
- table/column that exists on the target → skipped;
- table absent on the source now (deleted later) → not re-created;
- column absent on the source now (deleted later) → not resurrected;
- journal entry without a table reference (type_id unresolvable) → skipped;
- deletes, renames, reorder, `column.update`, `size`/`unique` changes (column-validation) are **not replayed** in v1.

Consequence: catch-up replays only changes made **after** enrichment was enabled; a fresh clone already contains the pre-enrichment state via carry.

## Event Bus Emissions

- `schema.type.created/updated/deleted`
- `schema.column.created/updated/deleted/reordered`

These trigger flat-view regeneration and WebSocket broadcasts.

## DB Tables

- `"db"."db"` — type definitions are also stored as EAV records where `t = up` (root type records)
- `"db"."_v2_reqs"` — column (req) definitions where `t != up`
- `_v2_computed_reqs` (per-workspace, lazy-init) — computed column configs
- `_v2_column_validation` (per-workspace, lazy-init) — validation rules per column
- `_v2_column_ai_config` (per-workspace, lazy-init) — AI button prompt configs
- `_v2_schema_audit` (per-workspace, lazy-init) — DDL change log; `MAX(id)` = schema version; replayable via `POST /schema/journal/catch-up`
- `_v2_schema_snapshots` (per-workspace, lazy-init) — schema snapshots
- `_v2_type_meta` (per-workspace, lazy-init) — icon, color, description per type

## AI Tools

| Tool | Tier | Description |
|------|------|-------------|
| `get_schema_history` | LOW | Schema audit history for a type |
| `get_schema_snapshot` | LOW | Get schema snapshot |
| `get_schema_backlinks` | LOW | Columns in other types that reference this type |
| `get_validation_rules` | LOW | Get validation rules for columns |
| `set_validation_rules` | MEDIUM | Set validation rules for a column |
| `update_column` | MEDIUM | Update column (name, type, options) |
| `create_computed` | MEDIUM | Create computed column |
| `update_computed` | MEDIUM | Update computed column |
| `delete_computed` | HIGH | Delete computed column |
| `add_column` | HIGH | Add column to type |
| `delete_column` | HIGH | Delete column (cascades values) |
| `create_table` | HIGH | Create type (table) |
| `update_table` | HIGH | Update type name, icon, settings |
| `delete_table` | HIGH | Delete type (cascades objects) |
| `plan_schema` | HIGH | Batch schema creation with HITL confirmation |
| `convert_column_to_ref` | HIGH | Convert text column to reference |

`plan_schema` column aliases support dot-grouping: `"Contacts.Phone"` / `"Contacts.Email"` merge under a shared "CONTACTS" parent header (minimum 2 consecutive columns with the same prefix); aliases without a dot stay standalone. Use for tables with many related columns, skip for ≤4 columns.

# Module: views

**Path:** `src/api/v2/modules/views/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/:db/views/...`
**Auth:** JWT required for all endpoints. Create/update/delete: `editor`.

## Purpose

Saved view configurations for tables. A view captures filter state, visible columns, sort order, grouping, and display type (table, kanban, calendar, etc.). Users can have private views and shared views visible to the whole workspace.

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/views?typeId=N` | any | List views for a table (own + shared) |
| GET | `/views/:id` | any | Get view config |
| POST | `/views` | editor | Create view |
| PATCH | `/views/:id` | editor | Update view name/config/shared/isDefault flag |
| DELETE | `/views/:id` | editor | Delete view |
| GET | `/views/:id/share` | editor | Get current public share info |
| POST | `/views/:id/share` | editor | Create/regenerate public share token |
| DELETE | `/views/:id/share` | editor | Revoke public share token |

## View Config Schema

```json
{
  "typeId": 123,
  "name": "Open Orders",
  "config": {
    "viewType": "table",
    "filters": [{ "field": "Status", "op": "eq", "value": "Open" }],
    "sort": [{ "field": "CreatedAt", "dir": "desc" }],
    "columns": [42, 43, 44],
    "groupBy": null,
    "rowHeight": "compact"
  },
  "shared": false,
  "isDefault": false
}
```

`viewType` values: `table`, `kanban`, `calendar`, `gallery`, `gantt`, `timeline`, `pivot`, `graph`

## Public Sharing

`POST /views/:id/share` generates a UUID token and stores it in `_v2_view_share_tokens`. The public URL `GET /api/v2/:db/public/view/:token` renders the view for unauthenticated visitors. Optional fields: `expiresInDays`, `password`.

Shared views enforce the same column visibility and filters as the saved config.

## DB Tables

- `_v2_views` (per-workspace) — `id`, `type_id`, `owner`, `name`, `config` (JSONB), `is_shared`, `is_default`, `created_at`, `updated_at`
- `_v2_view_share_tokens` (global public schema) — `token`, `view_id`, `type_id`, `db`, `expires_at`, `password_hash`, `created_by`

# Module: desktop

**Path:** `src/api/v2/modules/desktop/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/desktop/...`  (no `:db` — global, user-scoped)
**Auth:** JWT required for all endpoints.

## Purpose

Persistent browser tabs for the user. Each tab references one item (table, report, or document) in a specific workspace. The frontend uses this to restore the user's open tabs across page reloads and devices.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/desktop` | List current user's tabs (ordered) |
| POST | `/desktop` | Open a new tab |
| DELETE | `/desktop/:tabId` | Close a tab |
| PATCH | `/desktop/:tabId` | Update tab label, icon, or config |
| POST | `/desktop/reorder` | Reorder tabs (pass full ordered array of `tabIds`) |

## Tab Schema

```json
{
  "dbName": "my-workspace",
  "itemType": "table",
  "itemId": 123,
  "label": "Clients",
  "icon": "pi pi-users",
  "config": {}
}
```

- `itemType`: `table` | `report` | `document`
- `config`: arbitrary JSONB, max 8 KB (e.g. current view filter, scroll position)
- `label`/`icon`: display overrides (null = use item's default name/icon)

## Notes

- Tabs are user-scoped globally (not per-workspace), so the same tab list is shared across all workspaces.
- Uses `getPool()` directly (global pool) since this endpoint has no `:db` param.

## DB Tables

- `_v2_desktop_tabs` (global public schema) — `id`, `user_id`, `db_name`, `item_type`, `item_id`, `label`, `icon`, `config`, `ord`, `created_at`

# Module: dashboards

**Path:** `src/api/v2/modules/dashboards/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/:db/dashboards/...`
**Auth:** JWT required for all endpoints.

## Purpose

Configurable dashboards containing widget layouts. Each dashboard is a named container; its `config` JSONB stores the widget grid (type, position, size, data source config for each widget).

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/dashboards` | List dashboards for this workspace |
| POST | `/dashboards` | Create dashboard (`title`, plus any extra fields in `config`) |
| GET | `/dashboards/:id` | Get dashboard (includes widget config) |
| PATCH | `/dashboards/:id` | Update title and/or config |
| DELETE | `/dashboards/:id` | Delete dashboard |

## Dashboard Config

The dashboard `config` field is a free-form JSONB object. The frontend defines the widget schema; the backend stores and returns it without interpretation. Typical structure:

```json
{
  "widgets": [
    {
      "id": "w1",
      "type": "KpiWidget",
      "x": 0, "y": 0, "w": 4, "h": 2,
      "config": { "reportId": 42, "colId": 7 }
    }
  ]
}
```

## Widget Types (frontend-defined)

`KpiWidget`, `ChartWidget`, `TableWidget`, `ReportWidget`, `KanbanWidget`, `GalleryWidget`, `PivotWidget`, `DocumentWidget`, `TextWidget`, `CardWidget`, `FormWidget`, `ActionWidget`

- **CardWidget** — карточка одной записи с cross-table parent resolution, поддержка crossFilter, linkedChildTypeId, ref-колонки через lookups
- **FormWidget** — form for record creation with automatic parent-table detection
- **ActionWidget** — 3 modes: `automation` (batch run), `ai_button` (run AI button on record), `clear_table` (delete all records from table)
- **KpiWidget** поддерживает `filterField`/`filterValue` в config для фильтрации данных
- **KanbanWidget** supports writable mode (add-row, delete-row) with group field auto-set from the selected column
- **KanbanWidget**, **TableWidget**, **CardWidget** support `parentId` in cross-filter events for parent-child table relationships
- **ReportWidget** and **TableWidget** support inline cell editing (double-click) and paste-rows (Ctrl+V from Excel/Sheets)

## DB Tables

- `_v2_dashboards` (global public schema) — `id`, `db`, `title`, `config` (JSONB), `owner`, `created_at`, `updated_at`

> Note: stored in the global schema (not per-workspace) because dashboards may be shared across workspaces in future.

## Events

Dashboard service does not emit bus events. Widget data is refreshed on demand by the frontend.

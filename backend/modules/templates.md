# Module: templates

**Path:** `src/api/v2/modules/templates/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/:db/templates/...`
**Auth:** JWT required. Create/delete: `editor`.

## Purpose

Record templates — pre-filled field values for a table. When a user creates a new record using a template, the fields are pre-populated with the template's values. Reduces repetitive data entry for common record patterns.

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/templates?typeId=N` | any | List templates for a table |
| POST | `/templates` | editor | Create template |
| DELETE | `/templates/:id` | editor | Delete template |

## Template Schema

```json
{
  "typeId": 123,
  "name": "Standard Contract",
  "fields": {
    "Status": "Draft",
    "Category": "Legal",
    "Owner": "current_user"
  }
}
```

- `fields`: map of column alias → default value
- `"current_user"` as a value substitutes the creating user at record creation time

## Idempotency

`X-Idempotency-Key` supported on POST, cached 30 seconds.

## DB Tables

- `_v2_templates` (per-workspace) — `id`, `type_id`, `name`, `fields` (JSONB), `created_by`, `created_at`

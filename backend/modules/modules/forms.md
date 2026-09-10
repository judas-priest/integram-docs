# Module: forms

**Path:** `src/api/v2/modules/forms/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/:db/forms/...`
**Auth:** `GET /:token` is public (no auth). Management endpoints require JWT + `admin`.

## Purpose

Public data collection forms. Each form is linked to a table (`typeId`) and generates a unique token. Anyone with the token URL can submit records to the linked table without logging in.

## Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/forms/:token` | Public | Get form schema (columns, config) by token |
| GET | `/forms` | JWT | List forms for this workspace |
| POST | `/forms` | JWT + admin | Create form |
| PUT | `/forms/:token` | JWT + admin | Update form config/settings |
| DELETE | `/forms/:token` | JWT + admin | Delete form |

## Form Schema

```json
{
  "typeId": 123,
  "config": {
    "title": "Contact Us",
    "fields": [42, 43, 44],
    "submitLabel": "Send",
    "successMessage": "Thank you!"
  },
  "parentId": 1,
  "expiresAt": "2026-12-31T23:59:59Z"
}
```

- `config.fields`: array of reqTypeIds to show in the form
- `parentId`: EAV parent node for submitted records (usually root = 1)
- `expiresAt`: optional expiry; submissions to expired forms are rejected

## Submission Flow

Form submission (creating an object) goes through the standard objects API authenticated with the form token — not via this module directly. The form token authenticates the submission as a "form submitter" role.

Triggers `on_form_submit` automations after successful submission.

## Idempotency

`X-Idempotency-Key` header supported on POST, cached for 30 seconds.

## DB Tables

- `_v2_forms` (global public schema) — `id`, `workspace_id`, `type_id`, `token` (UUID), `config` (JSONB), `parent_id`, `expires_at`, `created_by`, `created_at`

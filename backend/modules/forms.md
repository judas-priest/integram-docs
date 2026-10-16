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
- `config.consent`: optional 152-ФЗ consent block `{ "text": "...", "version": "1" }` (см. «Согласие на обработку ПД»)
- `parentId`: EAV parent node for submitted records (usually root = 1)
- `expiresAt`: optional expiry; submissions to expired forms are rejected

## Submission Flow

Form submission (creating an object) goes through the standard objects API authenticated with the form token — not via this module directly. The form token authenticates the submission as a "form submitter" role.

Triggers `on_form_submit` automations after successful submission.

## Согласие на обработку ПД (152-ФЗ, 02.10.2026)

Если в конфиге формы объявлен `consent: {text, version}`:

- `GET /forms/:token` (и портал-обёртка) возвращает блок `consent` — UI (`frontend/src/views/forms/FormPublic.vue`) рисует чекбокс с текстом и требует его перед отправкой.
- `submitForm` **отвергает** отправку без `body.consent === true` (400 «Требуется согласие…»). Проверка живёт внутри `submitForm` — публичная дверь (`api/v2/index.js`) и портал (`submitFormForPortal`, маршрут `portal/api/forms/:token/submit`) закрыты одной точкой; IP передаётся обоими вызовами.
- Факт согласия фиксируется в глобальной таблице `_v2_form_consents` (`token, db, object_id, ip, text_version, created_at`), заведение — `ensureConsentsTable`. Таблица внесена в реестр переноса (`workspace-carry`, `clone: skip` — token у клона перевыпускается) и каталоги уборки.

Пустой `config.consent` или его отсутствие — прежнее поведение: чекбокса нет, факт не пишется.

## Idempotency

`X-Idempotency-Key` header supported on POST, cached for 30 seconds.

## DB Tables

- `_v2_forms` (global public schema) — `id`, `workspace_id`, `type_id`, `token` (UUID), `config` (JSONB), `parent_id`, `expires_at`, `created_by`, `created_at`
- `_v2_form_consents` (global, 02.10.2026) — `token`, `db`, `object_id`, `ip`, `text_version`, `created_at`: факт согласия 152-ФЗ

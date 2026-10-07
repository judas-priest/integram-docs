# Module: objects

**Path:** `src/api/v2/modules/objects/`
**Files:** `router.js`, `service.js`, `bulk.js`, `history.js`, `csv-import.js`, `trash.js`, `versions.js`, `schema.js`
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
| `page` / `pageSize` | Pagination (default 50, потолок 1000); `limit` — алиас `pageSize` (#272), при обоих побеждает `pageSize` |
| `cursor` | Keyset-курсор (`nextCursor` из meta) вместо offset-страниц; только при дефолтной сортировке |
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

Checked via `utils/row-permissions.js`. Rules: `OWNER_ONLY`, `ROLE_ONLY` (only users whose role matches the rule's `role_id`), `FILTER` (SQL WHERE clause with user attributes), `DENY_ALL`.

## Unique groups (`uniqueGroup`)

Per-table unique constraint over a **group** of columns (issue #170). Columns opt in via the `uniqueGroup` rule in `_v2_column_validation`; columns sharing a group name form one constraint scoped as `typeId:group`.

Enforced on write inside the write transaction by `enforceUniqueGroups` (`utils/unique-fingerprint.js`, called from `objects/service.js:2862`): a fingerprint (sha256 of the sorted `colId=canonical` parts) is upserted into `_v2_unique_keys` under `PRIMARY KEY (key_scope, key_hash)` — the database resolves the conflict at COMMIT, so a parallel writer gets the error and no TOCTOU pre-check is possible to race. Only columns written by the current call participate; an incomplete tuple (any group column empty) is not indexed. Ref columns are read in the inverted storage pattern.

The `_v2_unique_keys` table is probed 42P01-safely (`to_regclass`) before `CREATE TABLE IF NOT EXISTS` and before delete, so a workspace without the table neither breaks writes nor leaves them silently unindexed. It is bootstrapped for new workspaces (lazy-init) and carried on workspace copy (`registry/workspace-carry.js`). A violation surfaces as a normal validation error: 422 `VALIDATION_ERROR` «Значение должно быть уникальным (группа «X»)».

The satellite probe (`ensureColumnRulesReady`: `_v2_column_rules`, `_v2_column_validation`, `_v2_unique_keys`) runs on every entry into a write transaction (`createObject`/`updateObject`/`deleteObject`/`createVersion`) BEFORE the transaction opens; satellite readers are probed with `to_regclass` and never abort the transaction. When the transaction does break, the secondary 25P02 error is annotated with `[первопричина: [ярлык] …]` carrying the original query (label from `execSql`/`withTransaction`, context via AsyncLocalStorage). (PM ai2o #671/#693/#748.)

## Конфликт по первичному ключу (issue #296)

Отставшая последовательность id EAV (`23505` + constraint `<db>_pkey`) — не ошибка
ввода. Отличается от уникальности поля по имени constraint (поле → `uq_*`).

- `objects/sequence-repair.js` — `isPkeyConflict`, `repairEavSequence`
  (`setval(seq, GREATEST(last_value, max(id)))`, через общий пул — сделка звавшего
  в этот момент прервана), `checkSequenceLag`, `withPkeyRetry` (ровно один ретрай).
- `saveRequisites` при `_pkey`-конфликте чинит счётчик и кидает `PK_SEQUENCE_LAG`
  (500); `createObject`/`updateObject` ретраят сделку автоматически.
- Диагностика: `GET /:db/objects/sequence-check` (admin) →
  `{ lastValue, maxId, behind }`; `behind > 0` — столько записей воркспейс потеряет,
  прежде чем начнёт писать снова.
- Причины отставания: заливка данных с явными `id` скриптом/коннектором.
  После таких заливок выравнивать счётчик (`repairEavSequence`, либо любая запись объекта — сработает авторетрай); ручка sequence-check только диагностирует, не чинит.

## Secret masking at read boundary

In the single-object read (`GET /objects/:id`, `getObject`), columns of type Secret (130) and columns matched by name (`isSecretColumn` in `utils/secret-mask.js`: `/секрет|пароль|password|token|ключ|secret/i`, plus PWD=6) are masked to `••••` (`SECRET_MASK`) in the response (objects/service.js:1147-1157, 1214-1224); the object's own `value` is masked too when its type is Secret. The column key stays in `requisites` — «a secret exists, its value is hidden» — unlike PM-570 `restrictedColumns`, which removes the key entirely. Name-based matching requires the colDefs query to include the column name (`val AS name` — fix 8392ec42c).

Masking applies to the read path only — writes are stored as given; list endpoints do not mask. Do NOT confuse with grant-level `maskFilter` (see admin.md).

## Колонка «Пользователь» (COLLABORATOR, тип 1010)

Тип колонки со списком участников воркспейса (аналог collaborator в Airtable / Person в Notion).

- Значение: `_v2_users.id` участника воркспейса (строкой). При записи принимаются число, email или username; значение обязано принадлежать участникам этого воркспейса, иначе 400.
- Чтение: REST возвращает сырое значение (id); имя и аватар подставляет фронтенд из списка участников. Выйдший из воркспейса участник остаётся в значении («бывший участник»).
- Фильтрация: в списках — обычный фильтр колонки (числовой id); в отчётах — числом или токеном `[USER_ID]` (текущий пользователь), `[USER]` (username).
- Создание: `POST /:db/schema/:typeId/columns` с `type: 1010`; ИИ-инструментом — `colTypeName: "collaborator"`.
- Ограничение: CSV-импорт канонизирует только числовые значения.

## Версии строк

Commit-style версионирование строки ВМЕСТЕ с поддеревом child-таблиц (PM-310, ADR-033 — `docs/adr/033-row-versions.md`). Версия — JSONB-снимок «области версии»: сама строка, её ячейки и рекурсивное поддерево (дети, внуки, их ячейки) — один рекурсивный CTE той же формы, что у корзины (`trash.js:56-65`). Автополя (`_v2_autofields`) не версионируются — при восстановлении генерируются заново.

### Endpoints (router.js:216-255)

| Метод | Путь | Описание |
|--------|------|-------------|
| GET | `/objects/:id/versions` | Список версий (В1, В2, …) с признаком активной: `versionNo`, `label`, `actor`, `isActive`, `rowsCount`, `createdAt` |
| POST | `/objects/:id/versions` | Новая версия; `value`/`requisites` применяются ПЕРЕД снимком — правка строки = новая версия Вn+1, а не перезапись (editor, CSRF) |
| POST | `/objects/:id/versions/:versionNo/activate` | Переключение на версию Вn (editor, CSRF). Живое состояние перед стиранием автосохраняется как новая версия (`autoSavedAs`); если содержимое уже равно — холостое переключение, только указатель (`restoredRows: 0`) |

### Хранение (`versions.js`)

- Таблица `_v2_row_versions` (per-workspace, lazy-init; versions.js:25-46): `root_obj_id`, `version_no`, `label`, `actor`, `is_active`, `snapshot` JSONB; `UNIQUE(root_obj_id, version_no)`.
- Два индекса: `idx_rv_root (root_obj_id, version_no DESC)` — список версий по строке; `idx_rv_active (root_obj_id) WHERE is_active` — ровно один активный указатель.
- Ретеншн: `MAX_VERSIONS = 100` (versions.js:21) на строку; лишние неактивные версии удаляются после каждой вставки, активная не чистится никогда (versions.js:126-133).
- Восстановление: стирание области, затем вставка с НОВЫМИ EAV-id, родители раньше детей; расхождение цепочки родителей — ошибка транзакции.
- `createVersion` сам вызывает `ensure` — на свежем воркспейсе таблица заводится до транзакции (versions.js:157-162). Без этого первый снимок падал `42P01` (`relation does not exist`): `listVersions`/`activateVersion` ensure зовут, `createVersion` — нет (замер на проде 21.09).
- Версии умирают вместе со строкой: слушатель `object.deleted` (событие после COMMIT) чистит снимки асинхронно, сбой пишется в лог.

### AI Tools

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `list_object_versions` | TIER_LOW | List versions of a row (В1, В2, …) with the active flag |
| `create_object_version` | TIER_MEDIUM | Create a new version; optional `value`/`requisites` applied before the snapshot |
| `activate_object_version` | TIER_HIGH | Activate a version (HITL required) — restores the row subtree from the snapshot, auto-saves the live state first |

## Event Bus Emissions

- `object.created` — on new record
- `object.updated` — on field change (with old/new values)
- `object.deleted` — on soft or hard delete
- `object.moved` — on parent change
- `object.moved.bulk` — on bulk move of all children from one parent to another
- `object.requisite.changed` — on a single field value write (old/new value)
- `object.edge.created` — on a ref field write (graph edge `REF_<typeId>`)
- `object.edge.deleted` — on a ref field clear/replace

## CSV Import (`csv-import.js`)

Maps CSV columns to EAV requisites by header name. Creates objects in batch. Reports created/skipped/error counts. Supports reference column lookup by value.

## DB Tables

- `"db"."db"` — single EAV table for all objects and their field values (eavTable())
- `_v2_autofields` (per-workspace, lazy-init) — created_at/by, updated_at/by
- `_v2_audit_log` (per-workspace, lazy-init) — field-level change history
- `_v2_trash` (per-workspace, lazy-init) — soft-deleted objects
- `_v2_record_share_tokens` (global) — public share tokens
- `_v2_row_versions` (per-workspace, lazy-init) — commit-style row versions (see «Версии строк»)
- `_v2_unique_keys` (per-workspace, lazy-init) — fingerprints for composite uniqueness (see «Unique groups»)

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
| `aggregate_objects` | TIER_LOW | Aggregate column values (`columns[]` = `colId:FUNC`, SUM/AVG/MIN/MAX/COUNT/COUNT_DISTINCT); `dateFrom`/`dateTo` supported, `filters` не поддерживается (400) |
| `group_objects` | TIER_LOW | Group objects by a column with counts |
| `pivot_objects` | TIER_LOW | Pivot table: rowField × colField with aggregation; ref columns read in both storage patterns, keys are target names; optional `valueField` + `agg` (COUNT/SUM/AVG/MIN/MAX) |
| `get_object_backlinks` | TIER_LOW | Find objects referencing a given record |
| `get_object_history` | TIER_LOW | Field-level audit trail for an object |
| `rollback_object` | TIER_HIGH | Restore object to a previous audit snapshot |
| `bulk_update` | TIER_HIGH | Bulk update multiple objects (HITL required) |
| `list_trash` | TIER_LOW | List deleted objects (trash) for a table |
| `get_trash_item` | TIER_LOW | Get a deleted object from trash by ID, with its saved requisites |
| `restore_from_trash` | TIER_MEDIUM | Restore a deleted object from trash |
| `duplicate_object` | TIER_MEDIUM | Duplicate (copy) an existing object |

# Module: synced-lookups

**Path:** `src/api/v2/modules/synced-lookups/`
**Files:** `router.js`, `service.js`, `worker.js`, `diff.js`
**Base URL:** `/api/v2/:db/synced-lookups/...`
**Auth:** JWT; writes require roles (create/patch/delete — admin, sync — editor) + CSRF. MCP: `synced_lookup_create`/`synced_lookup_sync` TIER_HIGH, `synced_lookup_list` TIER_LOW, `TOOL_MIN_ROLE` mirrors REST.

## Purpose

External lookup ("внешний справочник"): a table in workspace B configured as a **pull-synced read-only copy** of a table from another workspace A (same PG instance or remote DSN). Use case: a shared reference list (statuses, categories, nomenclature) maintained in one workspace and consumed in others.

Model follows the market pattern (SeaTable Common Dataset, Airtable Sync): source stays the single place to edit; each consumer holds a local materialized copy with **local EAV IDs** — refs/multiselect/filters in B work natively. Cross-workspace isolation (ADR-001 schema-per-workspace, ADR-021 no cross-server JOIN) is not weakened: nothing references A's IDs from B's data.

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/synced-lookups?typeId=N` | any member | List sync configs (optionally for one table). Items include `last_status`, `last_error`, `last_counts` |
| POST | `/synced-lookups` | admin + CSRF | Create config `{typeId, srcDb, srcTypeId, schedule}`. `srcDb` accepts slug or db_name |
| PATCH | `/synced-lookups/:id` | admin + CSRF | Change `schedule` |
| POST | `/synced-lookups/:id/sync` | editor + CSRF | Run sync immediately |
| DELETE | `/synced-lookups/:id` | admin + CSRF | Remove config (target table and its data are NOT touched) |

`SCHEDULES` whitelist: `manual`, `0 * * * *` (hourly), `0 4 * * *` (daily 04:00). Arbitrary cron is not accepted.

## MCP tools

- `synced_lookup_list` — configs with sync status
- `synced_lookup_create` — connect an external reference
- `synced_lookup_sync` — run sync now

## How It Works

1. **Access is checked twice** (invitation-time AND access-time, per `docs/research/2026-09-14-cross-workspace-tables.md`): at create the user must have viewer+ role in the source workspace (owner > direct membership > org membership, resolved via the main pool `getPool()` — global tables are invisible to a remote workspace pool); every sync run re-checks the **config creator's** access and fails with `Доступ к области-источнику утрачен` if revoked.
2. **Sync** (`syncLookup`): reads source rows via `getPoolForDb(srcDb)` (`t = srcTypeId AND up > 0`, same shape as `lookups`), diffs against the mapping, then in ONE transaction: batched INSERT of new values (multi-VALUES chunks of 1000, identity `id` — no literal `0`), per-row UPDATE for renames, full mapping rebuild, status write. Events are NOT emitted (connectors precedent) — reference values must not fire webhooks/automations.
3. **Rename-safe**: `_v2_synced_lookup_map` (sync_id, src_obj_id → local_obj_id) survives renames; values deleted from the source are NOT removed locally (they may sit under refs) — counted as `orphans` in `last_counts`.
4. **Scheduling**: BullMQ repeatables on the module's OWN queue `synced-lookups` (NOT via `upsertCronJob`, which targets the `automations` queue and would be consumed by the automations worker). Bootstrap re-raises crons for all workspaces on server start. Without Redis only manual sync works.

## DB Tables

- `"db"."_v2_synced_lookups"` — config per target table (`UNIQUE (type_id)`), last sync status/error/counts. Carry: `настройка`/copy.
- `"db"."_v2_synced_lookup_map"` — src→local ID mapping (`UNIQUE (sync_id, src_obj_id)`). Carry: `содержимое`/copy (clone preserves IDs via `OVERRIDING SYSTEM VALUE`, so a cloned workspace keeps a valid mapping).

## UI

Settings of a table → «Внешний справочник» block (`SyncedLookupEditor.vue`): connect dialog (source workspace → table → schedule), status tag, last error, «Синхронизировать» / «Отключить». The first sync runs immediately after connecting.

## Known MVP Limits

- Values removed from the source are not deleted locally (only counted).
- Only the record name (`_value`) is synced; child tables and extra columns are out of scope.
- No push subscriptions (SeaTable-style master registry) — pull from the consumer side only.
- `CREATE TABLE IF NOT EXISTS` does not migrate the map table if it was created before the `id` column existed (module is new; no live installs affected).
